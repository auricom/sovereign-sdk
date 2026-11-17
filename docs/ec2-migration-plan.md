# Runbook: migrate the relay-devnet sequencers (`SovRollupCdkStarterStack`) from c8gd.12xlarge to i8g.4xlarge with no downtime

> **Checked on 2026-11-17** against AWS account `313110633477` / `us-east-1` with the read-only profile `ReadOnlyAccess-313110633477`. Nothing has been changed in the account.
> Commands marked **[write]** need a profile with write access and are run by a human. Every other command is read-only.
> Node access is through **SSM Session Manager**: all nodes are registered and `Online`, running Ubuntu 24.04.

---

## 0. What the account really contains

### 0.1 Inventory

| Role | ASG (`desired/min/max`) | Instance | AZ / IP | Launch template | `DISK_PROFILE` today | Sequencer | Election role |
|---|---|---|---|---|---|---|---|
| Primary | `SovRollupCdkStarterStack-SovRollupComputePrimaryAsgASG448C4328-fVa4yPZ43E36` (1/1/2) | `i-0baeb42dfeb96cb28` | us-east-1a / 10.0.2.21 | `lt-02a6dc4c5d8cfdc99` (latest v44, instance from v39) | `aws_raid_0` | preferred | `DbElected` |
| Secondary | `SovRollupCdkStarterStack-SovRollupComputeSecondaryAsgASG8224C640-PKqRyQODK3bD` (1/0/3) | `i-0c9228ec8b0fa847a` | us-east-1b / 10.0.3.143 | `lt-05cf24139fde86881` (v44, instance v39) | `aws_raid_0` | preferred | `DbElected` |
| Backup | `SovRollupCdkStarterStack-SovRollupComputeBackupNodeAsgASGF12DEAEF-v241AoY1q69R` (1/1/2) | `i-0cc24aae7b0454089` | us-east-1a / 10.0.2.196 | `lt-0e54a3c6b1a4f422e` (v44, instance v39) | `aws_ebs_backup_node_raid_0` | standard (not part of the election) | – |
| Proxy (not migrated) | `…ProxyAsgASGB1045D50-nWkwreNMu6YC` | `i-035616ee95eaccd94` | us-east-1a / 10.0.0.9 | `lt-02bdfa1164fba9ea6` | – | – | – |

- All three rollup nodes use security group `sg-0e90e09ed3975a293`. Aurora accepts 5432 from that SG, and the proxy reaches 12346/3000 through it. **New instances get the same SG from the launch template, so no SG changes are needed.**
- Today's `aws_raid_0` layout: NVMe 1 + 2 in RAID0 at `/mnt/rollup` (ext4, 1.9 TB), NVMe 3 at `/mnt/logs`. The backup node also has EBS `/dev/sdf` (`vol-0b7f605d6d0900a4a`, 1800 GiB gp3, 30k IOPS) at `/mnt/snapshots`.
- Launch template **v44** (the ASGs' current version) differs from **v39** (what the running instances booted from) only in the AMI (`ami-09a93d801db0bafae`, Ubuntu 24.04 arm64) and the Ansible branch (`cdk-deployment-june-2026` → `cdk-deployment-october-2026`).

### 0.2 Sizing check: GO

| | c8gd.12xlarge (today) | i8g.4xlarge (target) | Verdict |
|---|---|---|---|
| vCPU | 48 | **16** | Over 14 days, CloudWatch shows: average ≤ 0.4%, highest hourly average 1.4%, highest 1-min peak **14%** (secondary). That's ≈ 6.7 vCPU, or ≈ 42% of 16. **OK**. Ansible compiles the rollup at boot, so first boot is ~3× slower |
| RAM | 96 GiB | **128 GiB** | More RAM than today |
| Rollup disk | RAID0 2 × 950 GB = 1.9 TB | **1 × 3750 GB**, no RAID | Data = **944 GB** (last snapshot's `FullSnapshotSizeInBytes`), including the ~800 GiB NOMT hashtable (`user_hashtable_buckets: 210000000`). **Fits** |
| Logs disk | 3rd NVMe (950 GB) | new **EBS gp3 on `/dev/sdg`** | Size it in §2.3 |
| Capacity | – | i8g.4xlarge offered in us-east-1a/b/c/d | **OK** |
| vCPU quota | 512 | +16 during each swap (peak 162) | **OK** |

### 0.3 How the no-downtime switch works (SDK behaviour)
- Primary and secondary run with `node_role = "DbElected"` and leader election in Aurora (`leader_timeout` 1 s, `grace_period` 10 s, `heartbeat` 100 ms, which are the defaults because the CDK parameters are empty).
- A `DbElected` node that isn't leader runs as a Postgres-synced replica. When the leader stops heart-beating for > 1 s, a replica takes leadership, **exits**, and is restarted by systemd as the batch producer. `rollup.service` has `Restart=always`, `RestartSec=3`.
- **Reads** never stop. **Writes** pause for about 1 s plus the restart time when the leader node is retired (§4.6).
- A node that is still syncing returns non-200 on `/sequencer/ready`, so node-discovery keeps it out of read routing. Clients never see a stale node.
- The backup node is a *standard* sequencer and isn't part of the election. Swapping it has no client impact.

### 0.4 Strategy
1. **Change the CDK app and deploy once (§3).** The deploy creates new launch template versions (i8g.4xlarge plus an EBS logs volume) and new SSM values-document versions (single-NVMe `DISK_PROFILE`), and points the three ASGs at them. **Running rollup instances aren't touched:** their ASGs have no rolling-update policy, and nodes only read the SSM documents at boot.
2. **Then, one node at a time (§4):**
   1. Scale its ASG from 1 to 2. The new instance boots from the new template and Ansible sets up the single NVMe.
   2. Restore the latest backup snapshot onto it. It catches up and becomes ready, and the proxy adds it automatically.
   3. Detach the old instance with a desired-capacity decrement (back to 1) and stop rollup on it. For the leader, this is the failover.
   4. Let it run for a while, then terminate the old instance.

**Order: non-leader preferred node → leader preferred node → backup node.** The backup goes last so its snapshots stay the restore source for the first two.

### 0.5 No drift: what goes through CDK and what doesn't
| Change | How | CloudFormation drift? |
|---|---|---|
| Instance type, block devices, `DISK_PROFILE` | **CDK code + `cdk deploy`** (§3) | None: the stack is the source |
| ASG desired capacity 1 → 2 → 1 | `aws autoscaling` during §4. The ASGs carry `UpdatePolicy.AutoScalingScheduledAction.IgnoreUnmodifiedGroupSizeProperties: true`, so a deploy doesn't fight runtime scaling. Every swap ends back at the template value (1) | None once each swap is done. §5 checks the stack is `IN_SYNC` |
| Suspended processes, scale-in protection, detach, terminate | `aws autoscaling` / `aws ec2` | Not stack properties, so no drift. Suspended processes are resumed in §5 |
| Temporary restore volumes, new snapshots | Created and deleted by the stack's own scripts on the nodes | Not stack resources |

**No launch template or SSM document is ever edited by hand.**

- The CDK source isn't in any GitHub repo visible to this account (searched the `Sovereign-Labs` org and all `rollup-starter` branches), nor on this machine.
- CloudTrail shows every deploy is done by the IAM user **`aws-cdk-gentoo`**: the last ones were 2026-10-29, 2026-10-30 and 2026-11-09, all with CDK `2.220.0`. **§3 has to be run by that person, from the CDK repo.**
- The stack's own node-level config-drift check (`SovRollupCdkStarterStack-runtime-config-drift` alarm) **will fire** between the deploy and the last swap. The old nodes run a configuration that no longer matches the stack. It clears once the old nodes are gone.

### 0.6 How a new i8g.4xlarge instance gets installed (no Ansible change needed)
The single-NVMe support already exists in the Ansible branch the new instances boot from. It was added upstream by `rollup-starter-ansible` commit `e47315fb34` (2026-10-05, *"Rework EBS disk detection and add support for EBS log volume"*), and that branch's README documents it for single-NVMe instances. This migration only has to **select** it, through CDK (§3).

Boot chain, from launch template user data:
1. User data writes `/etc/rollup-node.env` (`ROLLUP_NODE_TYPE=primary|secondary|backup`) and runs `/home/ubuntu/render_config.sh`.
2. The render script fetches the SSM document `…-runtime-config-values-<type>` and writes `/home/ubuntu/runtime_vars.yaml`. That file includes `disk_profile`, `rollup_node_role`, `rollup_sequencer_kind`, the Aurora URL, and `rollup_node_id` (= instance ID).
3. User data runs `bootstrap.sh` and `ansible-pull` of `rollup-starter-ansible@cdk-deployment-october-2026` (`local.yml`, roles `common` + `rollup`).
4. Ansible `common` role:
   - `custom_vars.yaml` loads the profile.
   - `disk_discovery.yaml` lists instance-store disks by model (`Amazon EC2 NVMe Instance Storage`). It resolves the EBS volumes by **EC2 attachment name** (`/dev/sdg`, `/dev/sdf`) using `discover_ebs_disk.py`, which reads the NVMe controller metadata, so Linux device order doesn't matter.
   - `disks.yaml` formats and mounts the disks.
5. Ansible `rollup` role:
   - builds the rollup at commit `40c49a68…` and installs `rollup.service`;
   - installs `restore-from-snapshot.sh`;
   - on the backup only, installs the hourly snapshot cron;
   - starts rollup.

What `disks.yaml` does on each layout:

| | Today: c8gd.12xlarge, 3 NVMe | New: i8g.4xlarge, 1 NVMe |
|---|---|---|
| Profile (primary/secondary) | `aws_raid_0` | `aws_single_nvme_ebs_logs` |
| `/mnt/rollup` | `mdadm` RAID0 of NVMe 1 + 2 (`/dev/md/rollup`), ext4 | the single NVMe directly, ext4, **no mdadm** |
| `/mnt/logs` (+ journald) | NVMe 3 | EBS attached as **`/dev/sdg`** (new in the launch template) |
| Profile (backup) | `aws_ebs_backup_node_raid_0` | `aws_ebs_backup_node_single_nvme` |
| `/mnt/snapshots` (backup) | EBS `/dev/sdf` | EBS `/dev/sdf`, unchanged |
| Snapshot cron (backup) | yes (profile name contains `backup_node`) | yes, same rule |

Nothing else in the branch assumes a RAID device: no `md` device in monitoring, tuning or journald. The only other match is `mdadm` in the package list.

Failure modes, which all stop Ansible before any formatting:
- profile still `aws_raid_0` on 1 NVMe → *"Not enough instance store disks for RAID"*;
- `/dev/sdg` missing from the launch template → *"Expected exactly one EBS device attached as /dev/sdg; found 0"*.

---

## 1. Variables

```bash
export AWS_REGION=us-east-1
export RO=ReadOnlyAccess-313110633477           # read-only checks
export RW=<profile with write access to 313110633477>   # [write] steps
export DOMAIN=relay-devnet.sovereign-labs.xyz
export LT_BASE=44                               # latest version of the 3 launch templates before the deploy
export DOC_BASE=10                              # default version of the 3 values documents before the deploy
export STACK=SovRollupCdkStarterStack
export LOG=~/i8g-migration-relay-devnet.env     # every ID created is appended here

export P_ASG=SovRollupCdkStarterStack-SovRollupComputePrimaryAsgASG448C4328-fVa4yPZ43E36
export P_LT=lt-02a6dc4c5d8cfdc99
export P_OLD=i-0baeb42dfeb96cb28
export P_DOC=SovRollupCdkStarterStack-runtime-config-values-primary

export S_ASG=SovRollupCdkStarterStack-SovRollupComputeSecondaryAsgASG8224C640-PKqRyQODK3bD
export S_LT=lt-05cf24139fde86881
export S_OLD=i-0c9228ec8b0fa847a
export S_DOC=SovRollupCdkStarterStack-runtime-config-values-secondary

export B_ASG=SovRollupCdkStarterStack-SovRollupComputeBackupNodeAsgASGF12DEAEF-v241AoY1q69R
export B_LT=lt-0e54a3c6b1a4f422e
export B_OLD=i-0cc24aae7b0454089
export B_DOC=SovRollupCdkStarterStack-runtime-config-values-backup
export SNAP_VOL=vol-0b7f605d6d0900a4a           # old backup node's /dev/sdf: source of all restores
```
In a new shell: re-export the above, then `source $LOG`.

---

## 2. Pre-flight (read-only), go/no-go

### 2.1 Nothing changed since this runbook was checked
```bash
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG \
  --query 'AutoScalingGroups[].{asg:AutoScalingGroupName,desired:DesiredCapacity,max:MaxSize,ltv:LaunchTemplate.Version,inst:Instances[].[InstanceId,LifecycleState,HealthStatus]}' --output json
aws --profile $RO ec2 describe-launch-templates --launch-template-ids $P_LT $S_LT $B_LT \
  --query 'LaunchTemplates[].[LaunchTemplateId,LatestVersionNumber]' --output text            # all 44
for d in $P_DOC $S_DOC $B_DOC; do aws --profile $RO ssm describe-document --name $d --query 'Document.[Name,DefaultVersion]' --output text; done   # all 10
aws --profile $RO cloudformation describe-stacks --stack-name SovRollupCdkStarterStack --query 'Stacks[0].[StackStatus,LastUpdatedTime]' --output text
```
If any value differs (someone deployed), update `LT_BASE` / `DOC_BASE` and re-read §0.1.

### 2.2 Which node is the leader
Open a session on each preferred node (`aws --profile $RW ssm start-session --target $P_OLD`, then `$S_OLD`) and run:
```bash
curl -s localhost:12346/sequencer/role; echo
curl -s -o /dev/null -w 'ready=%{http_code}\n' localhost:12346/sequencer/ready
curl -s localhost:12346/ledger/slots/latest | jq -c '{number,state_root}'
```
Write down which one is the leader (`LEADER=P` or `LEADER=S`). **The other one is migrated first.**

### 2.3 Disks and logs volume size
On each old node:
```bash
cat /proc/mdstat; lsblk -o NAME,MODEL,SIZE,MOUNTPOINTS
df -h /mnt/rollup /mnt/logs /mnt/snapshots 2>/dev/null
sudo du -sh /mnt/logs/* 2>/dev/null | sort -h | tail
```
Set `LOGS_GB` (EBS `/dev/sdg`) to at least 2 × the current `/mnt/logs` usage. The Ansible README suggests 64 GiB as a starting point; today these logs sit on a 950 GB NVMe, so measure before choosing.

### 2.4 Same rollup version everywhere
On each old node:
```bash
grep -E '^(rollup_commit_hash|disk_profile|rollup_node_role|rollup_sequencer_kind|rollup_node_id)' /home/ubuntu/runtime_vars.yaml
```
Every node must show `rollup_commit_hash: "40c49a68fc67c0990a2d8bbe5a472f7527b44885"`, the value new nodes get from SSM. **If an old node runs a different commit, stop.** New nodes would run a different binary from the one that wrote the snapshot.

### 2.5 Snapshots are healthy
```bash
aws --profile $RO ec2 describe-snapshots --owner-ids self \
  --filters Name=volume-id,Values=$SNAP_VOL Name=tag:AutomatedBackup,Values=true \
  --query 'reverse(sort_by(Snapshots,&StartTime))[:4].[SnapshotId,StartTime,State,FullSnapshotSizeInBytes]' --output table
```
The latest `completed` snapshot must be < 2 h old. On the backup node: `sudo tail -40 /mnt/logs/rollup-snapshot.log` should show `=== Snapshot complete ===` every hour, with no `ERROR`.

### 2.6 Drift baseline: the stack must be in sync before we start
```bash
DID=$(aws --profile $RO cloudformation detect-stack-drift --stack-name $STACK --query StackDriftDetectionId --output text)
sleep 60; aws --profile $RO cloudformation describe-stack-drift-detection-status --stack-drift-detection-id $DID \
  --query '[DetectionStatus,StackDriftStatus,DriftedStackResourceCount]' --output text
aws --profile $RO cloudformation describe-stack-resource-drifts --stack-name $STACK --stack-resource-drift-status-filters MODIFIED DELETED \
  --query 'StackResourceDrifts[].[LogicalResourceId,ResourceType,PropertyDifferences[].PropertyPath|join(`,`,@)]' --output table
```
Expect `DETECTION_COMPLETE  IN_SYNC  0`. If anything is already drifted, sort it out with the CDK owner first: either fix it in code or revert it by hand. Otherwise the deploy in §3 will silently overwrite it.

**Result on 2026-11-17: `DRIFTED 1`.** `SovRollupAuroraClusterC528DD46` (`AWS::RDS::DBCluster`) has `/EngineVersion` expected `17.5`, actual `17.9`. Aurora applied an automatic minor version upgrade. Fix it in the CDK code **in a separate deploy before §3**, so the migration change set only contains migration changes:
- set the cluster engine to `rds.AuroraPostgresEngineVersion.of('17.9', '17')` (or whatever version `aws rds describe-db-clusters --query 'DBClusters[].EngineVersion'` reports on the day);
- preview it like §3.3. The change set must show only `Modify SovRollupAuroraClusterC528DD46` with `Replacement False`, because the version already matches;
- execute it, then re-run the drift detection above until it reports `IN_SYNC 0`.

To stop this from recurring, either pin `autoMinorVersionUpgrade: false` on the cluster instances, or accept that each AWS minor upgrade needs a follow-up one-line CDK bump. If the read-only profile can't call `detect-stack-drift`, run these three commands with `$RW`; drift detection doesn't modify anything.

### 2.7 Freeze
Until §5 is done, don't do any of the following:
- any `cdk deploy` other than the one in §3
- the stack's `MigrateLiveNodesCommand`
- `render_config.sh` / `run_ansible.sh` on the old nodes. After §3 they would read the new single-NVMe `DISK_PROFILE`, and Ansible would fail on hardware that doesn't match it.

Tell dev users when the leader swap (a few seconds of write pause) will happen.

---

## 3. Change the CDK app and deploy once **[write, run by the CDK owner]**

### 3.1 Start from what's deployed
In the CDK repo, on the machine and credentials normally used for deploys (CloudTrail user `aws-cdk-gentoo`):
```bash
git status && git log -1 --oneline        # clean tree, the commit that was deployed on 2026-11-09
npm ci
npx cdk --version                         # deployed with 2.220.0; same or newer
npx cdk diff SovRollupCdkStarterStack     # BEFORE editing: must print no differences
```
If `cdk diff` already shows differences, the code doesn't match the deployed stack. Reconcile that first, otherwise this deploy also ships changes nobody reviewed.

### 3.2 Code changes
Find the places to change:
```bash
grep -rnE "c8gd|C8GD|aws_raid_0|aws_ebs_backup_node_raid_0|/dev/sdf|DISK_PROFILE|diskProfile|restore-from-volume" --include='*.ts' bin lib
```
1. **Primary, Secondary and BackupNode launch templates:** set `instanceType: new ec2.InstanceType('i8g.4xlarge')`.
2. **Same three templates:** add a logs volume on `/dev/sdg`. That's the device name `aws_single_nvme_ebs_logs` and `aws_ebs_backup_node_single_nvme` look for. Keep the existing root volume, and keep `/dev/sdf` on the backup unchanged:
   ```ts
   { deviceName: '/dev/sdg',
     volume: ec2.BlockDeviceVolume.ebs(LOGS_GB /* from §2.3 */, {
       volumeType: ec2.EbsDeviceVolumeType.GP3, encrypted: true, deleteOnTermination: true }) },
   ```
3. **Runtime values documents:** set `DISK_PROFILE` to `aws_single_nvme_ebs_logs` for primary and secondary, and `aws_ebs_backup_node_single_nvme` for backup. Both profiles exist in `rollup-starter-ansible@cdk-deployment-october-2026` (`roles/common/defaults/main.yaml`); the backup profile requires `SEQUENCER_KIND=standard`, which the backup already uses.
4. Optional, same deploy: fix the `RestoreFromVolumeInstructions` output. The script is `/home/ubuntu/restore-from-snapshot.sh <snap-id>`, not `/tmp/restore-from-volume.sh`.

Commit the change on a branch and get it reviewed like any infra change.

### 3.3 Preview the exact change set (nothing is applied)
```bash
npx cdk diff SovRollupCdkStarterStack
npx cdk deploy SovRollupCdkStarterStack --no-execute --change-set-name i8g-4xlarge-migration
aws cloudformation describe-change-set --stack-name $STACK --change-set-name i8g-4xlarge-migration \
  --query 'Changes[].ResourceChange.{action:Action,id:LogicalResourceId,type:ResourceType,replacement:Replacement,scope:join(`,`,Scope)}' --output table
```
Expected rows, all `Modify` with `Replacement = False`:

| Resource | Why |
|---|---|
| `SovRollupComputePrimaryLaunchTemplate…`, `…SecondaryLaunchTemplate…`, `…BackupNodeLaunchTemplate…` | instance type, `/dev/sdg`, and a newer Ubuntu arm64 AMI (`ami-0f6435163a86ffc8c` instead of `ami-09a93d801db0bafae`). The AMI comes from the stack's SSM parameter and changes at every deploy |
| `SovRollupComputePrimaryAsg…`, `…SecondaryAsg…`, `…BackupNodeAsg…` | `LaunchTemplate.Version`. No rolling-update policy, so **no instance is replaced** |
| `SovRollupComputeRuntimeConfigValuesPrimary…`, `…Secondary…`, `…Backup…` | `DISK_PROFILE`. `UpdateMethod: NewVersion` creates version 11 and makes it the default |
| `SovRollupComputeProxyLaunchTemplate…` + `SovRollupComputeProxyAsg…` | **Not from our change.** The Ubuntu amd64 AMI parameter now resolves to `ami-08b275ac6cc5dbefb` (deployed: `ami-0d6dd9e8265d48906`). The proxy ASG has a rolling-update policy (`MinInstancesInService 1`, waits for the new proxy's signal, 30 min max), so the deploy **replaces the proxy instance one for one**. Every recent deploy did this (CloudTrail `SignalResource` after each one) |

**Stop and don't execute** if the change set shows anything else: any `Replacement: True`, any `Remove`, Aurora / EIP / VPC / security group / IAM changes, or changes to `MinSize`/`MaxSize`/`DesiredCapacity`. Drop it with `aws cloudformation delete-change-set --stack-name $STACK --change-set-name i8g-4xlarge-migration`.

### 3.4 Deploy
Run at a quiet time because of the proxy roll, and execute **the change set you reviewed**:
```bash
aws cloudformation execute-change-set --stack-name $STACK --change-set-name i8g-4xlarge-migration
aws cloudformation wait stack-update-complete --stack-name $STACK
aws cloudformation describe-stack-events --stack-name $STACK --max-items 40 \
  --query 'StackEvents[].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' --output table
```
During the proxy roll, run the write probe from §4.5 against `https://$DOMAIN/sequencer/ready`.

### 3.5 Check the result (read-only)
```bash
aws --profile $RO ec2 describe-launch-templates --launch-template-ids $P_LT $S_LT $B_LT --query 'LaunchTemplates[].[LaunchTemplateId,LatestVersionNumber]' --output text
for lt in $P_LT $S_LT $B_LT; do
  aws --profile $RO ec2 describe-launch-template-versions --launch-template-id $lt --versions '$Latest' \
    --query 'LaunchTemplateVersions[0].[LaunchTemplateId,VersionNumber,LaunchTemplateData.InstanceType,join(`,`,LaunchTemplateData.BlockDeviceMappings[].DeviceName)]' --output text
done                                  # i8g.4xlarge, /dev/sda1,/dev/sdg (backup: + /dev/sdf)
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG \
  --query 'AutoScalingGroups[].[AutoScalingGroupName,LaunchTemplate.Version,DesiredCapacity,Instances[0].InstanceId,Instances[0].LaunchTemplate.Version]' --output text
                                      # ASG version = latest; instances still the old ones (v39), desired 1
for d in $P_DOC $S_DOC $B_DOC; do printf '%s: ' $d
  aws --profile $RO ssm get-document --name $d --query Content --output text | jq -r '.mainSteps[0].inputs.runCommand[]' \
    | grep '^DISK_PROFILE=' | sed -E "s/.*'([^']+)'.*/\1/" | base64 -d; echo
done
```
Record the new versions: `echo "P_LTV=<n> S_LTV=<n> B_LTV=<n>" | tee -a $LOG`. Then repeat the drift check from §2.6: it must still be `IN_SYNC`.

From now on, any instance launched by these ASGs is an i8g.4xlarge with the single-NVMe layout, including an unplanned replacement.

---

## 4. Per-node procedure

### 4.0 Before the first node: stop the ASGs from replacing instances on their own **[write]**
The instance role can call `autoscaling:SetInstanceHealth`, so something on the nodes may mark an instance unhealthy. The ASG would then terminate it, possibly halfway through a restore. Suspended processes aren't a stack property, so this causes no drift; they're resumed in §5.
```bash
for g in $P_ASG $S_ASG $B_ASG; do
  aws --profile $RW autoscaling suspend-processes --auto-scaling-group-name $g --scaling-processes ReplaceUnhealthy AZRebalance
done
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG --query 'AutoScalingGroups[].SuspendedProcesses[].ProcessName'
```

Run §4.1–§4.7 three times, finishing each node before starting the next:

| Run | Node | Variables |
|---|---|---|
| 1 | the **non-leader** from §2.2 | `N=S ASG=$S_ASG LTV=$S_LTV OLD=$S_OLD` (or the `P_` set if the secondary leads) |
| 2 | the **leader** | the other `P_` / `S_` set |
| 3 | **backup** | `N=B ASG=$B_ASG LTV=$B_LTV OLD=$B_OLD` |

### 4.1 Confirm the ASG launches the new template
```bash
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG \
  --query 'AutoScalingGroups[0].[LaunchTemplate.Version,DesiredCapacity,length(Instances)]' --output text   # $LTV  1  1
```

### 4.2 Launch the new instance **[write]**
```bash
aws --profile $RW autoscaling set-desired-capacity --auto-scaling-group-name $ASG --desired-capacity 2
watch -n15 "aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG \
  --query 'AutoScalingGroups[0].Instances[].[InstanceId,LifecycleState,HealthStatus,LaunchTemplate.Version]' --output text"
NEW=<instance id whose LaunchTemplate.Version = $LTV>; echo "${N}_NEW=$NEW" | tee -a $LOG
aws --profile $RW autoscaling set-instance-protection --auto-scaling-group-name $ASG --instance-ids $NEW --protected-from-scale-in
aws --profile $RO ec2 describe-instances --instance-ids $NEW \
  --query 'Reservations[0].Instances[0].{type:InstanceType,az:Placement.AvailabilityZone,ip:PrivateIpAddress,disks:BlockDeviceMappings[].DeviceName}'
```
- Expect `i8g.4xlarge`, the same AZ as the old node, and disks `/dev/sda1`, `/dev/sdg` (plus `/dev/sdf` for the backup).
- If no instance appears, check `aws --profile $RO autoscaling describe-scaling-activities --auto-scaling-group-name $ASG --max-items 3`. `InsufficientInstanceCapacity` means AWS has no capacity right now: retry later. Primary and backup ASGs are pinned to us-east-1a, the secondary to us-east-1b.

### 4.3 Watch the bootstrap and check the disk layout
```bash
aws --profile $RW ssm start-session --target $NEW
sudo tail -f /var/log/user-data.log         # until "User data script completed"; ~30–90 min (Ansible compiles the rollup)
```
Then:
```bash
lsblk -o NAME,MODEL,SIZE,TYPE,MOUNTPOINTS
cat /proc/mdstat                                   # no md device
findmnt /mnt/rollup /mnt/logs /mnt/snapshots       # rollup = "Amazon EC2 NVMe Instance Storage" (ext4, ~3.4T),
                                                   # logs = EBS /dev/sdg, snapshots = EBS /dev/sdf (backup only)
nproc; free -g                                     # 16 / ~124
grep -E '^(disk_profile|rollup_node_role|rollup_sequencer_kind|rollup_commit_hash|rollup_node_id)' /home/ubuntu/runtime_vars.yaml
systemctl is-active rollup                         # active: syncing from the DA start height on an empty disk
curl -s -o /dev/null -w 'ready=%{http_code}\n' localhost:12346/sequencer/ready   # not 200, so not routed by the proxy
```
If the bootstrap failed, the last Ansible task in the log explains why. The usual cause is *"Not enough instance store disks for RAID"*: the instance read the old `DISK_PROFILE`, meaning the deploy in §3 didn't take effect. Re-check §3.5. To discard the instance, remove its protection (`set-instance-protection --no-protected-from-scale-in`), then run `aws --profile $RW autoscaling terminate-instance-in-auto-scaling-group --instance-id $NEW --should-decrement-desired-capacity`.

Primary and secondary nodes are `DbElected`. Until the restore is done, make sure the current leader stays up, because this node isn't caught up yet.

### 4.4 Restore the latest backup snapshot onto the new node
In the session on `$NEW`:
```bash
sudo -iu ubuntu
tmux new -s restore
SNAP=$(aws ec2 describe-snapshots --region us-east-1 --owner-ids self \
  --filters Name=volume-id,Values=vol-0b7f605d6d0900a4a Name=tag:AutomatedBackup,Values=true Name=status,Values=completed \
  --query 'sort_by(Snapshots,&StartTime)[-1].[SnapshotId,StartTime]' --output text); echo $SNAP
SNAP=${SNAP%%[[:space:]]*}

# primary / secondary:
/home/ubuntu/restore-from-snapshot.sh $SNAP
# backup: hold the snapshot lock so the hourly cron can't snapshot a half-restored disk
flock -n /home/ubuntu/.rollup-snapshot.lock /home/ubuntu/restore-from-snapshot.sh $SNAP
```
Type `yes`. The script (from Ansible) runs these steps:
1. Stops rollup.
2. Creates a gp3 volume from the snapshot (80k IOPS, 2000 MB/s, initialization rate 300 MiB/s) and attaches it as `/dev/sdr`.
3. Wipes `/mnt/rollup` and rsyncs `/rollup/` into it, then fixes ownership.
4. Detaches and deletes the temporary volume, then starts rollup.

Allow about 1–1.5 h for 944 GB. The log is in `/mnt/logs/rollup-restore.log`. The instance role already has the EBS permissions it needs.

If it stops halfway, its exit trap detaches and deletes the temporary volume. Check for leftovers with `aws --profile $RO ec2 describe-volumes --filters Name=tag:Purpose,Values=restore-temp --query 'Volumes[].[VolumeId,State]'`, then re-run.

Unless it was fixed in §3.2, the stack output `RestoreFromVolumeInstructions` still points at `/tmp/restore-from-volume.sh`. That path is out of date: the script lives at `/home/ubuntu/restore-from-snapshot.sh`.

### 4.5 Catch-up checks
On `$NEW`:
```bash
curl -s localhost:12346/rollup/sync-status | jq -c
curl -s localhost:12346/ledger/slots/latest | jq -c '{number,state_root}'     # starts near the snapshot height, not genesis, then rises
curl -s -o /dev/null -w 'ready=%{http_code}\n' localhost:12346/sequencer/ready # 200 once caught up
curl -s localhost:12346/sequencer/role; echo
sudo journalctl -u rollup --since '-30min' | grep -Ei 'error|panic|mismatch' | tail
```
- **Same state root as the cluster:** pick a slot `N` slightly below the head and compare `curl -s localhost:12346/ledger/slots/$N | jq -r .state_root` on `$NEW` and on the leader. The values must be identical.
- Once `ready=200`, node-discovery puts the node in read routing automatically.
- **Soak:** at least 24 h for run 1, at least 2 h for runs 2 and 3. Watch `uptime; free -g; df -h /mnt/rollup /mnt/logs` and CloudWatch CPU:
  ```bash
  aws --profile $RO cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=$NEW \
    --statistics Average Maximum --period 300 --start-time $(date -u -d '-6 hours' +%FT%TZ) --end-time $(date -u +%FT%TZ) \
    --query 'sort_by(Datapoints,&Timestamp)[-12:].[Timestamp,Average,Maximum]' --output table
  ```
- Run 3 (backup) only: after the restore, the hourly cron writes to the new `/dev/sdf`. Wait for the first `=== Snapshot complete ===` in `/mnt/logs/rollup-snapshot.log`, then check the snapshot exists:
  ```bash
  aws --profile $RO ec2 describe-snapshots --owner-ids self --filters Name=tag:InstanceId,Values=$NEW --query 'Snapshots[].[SnapshotId,StartTime,State]'
  ```

### 4.6 Retire the old instance (in run 2 this is the leader failover)
Run 2 only. Before you start, every preferred node that's running must return `ready=200` and be at the same height. Then open these watches:
```bash
# laptop: write-path probe through the proxy (/sequencer/ready is always routed to the leader)
while true; do printf '%s ' "$(date +%T)"; curl -s -o /dev/null -m 2 -w '%{http_code}\n' https://$DOMAIN/sequencer/ready; sleep 0.5; done
# sessions on both NEW preferred nodes
sudo journalctl -u rollup -f | grep -Ei 'leader|BatchProducer'
```
For every run, remove the old instance from the ASG but keep it running until the soak is over. Then stop rollup on it:
```bash
aws --profile $RW autoscaling detach-instances --auto-scaling-group-name $ASG --instance-ids $OLD --should-decrement-desired-capacity
echo "${N}_DETACHED=$OLD" | tee -a $LOG
aws --profile $RW ssm start-session --target $OLD
sudo systemctl disable --now rollup && sudo systemctl mask rollup; systemctl is-active rollup      # inactive
# run 3 (backup) only: stop the old node's snapshot cron
sudo crontab -u ubuntu -l | grep -v snapshot-rollup-data.sh | sudo crontab -u ubuntu -
```
- Run 2: within seconds, one of the new `DbElected` nodes logs *"Replica acquired leadership! Exiting to restart as leader."*. systemd restarts it, and it logs *"DbElected node acquired leadership, running as BatchProducer"*. The probe shows a short run of non-200 responses, then 200. Either new node may win.
- Then put the new instance back under normal ASG rules:
  ```bash
  aws --profile $RW autoscaling set-instance-protection --auto-scaling-group-name $ASG --instance-ids $NEW --no-protected-from-scale-in
  ```
- Post-check: `curl -s https://$DOMAIN/ledger/slots/latest | jq .number` keeps rising. Send a test transaction with your usual tool and confirm it's accepted.

### 4.7 Terminate the old instance (after the soak) **[write]**
```bash
aws --profile $RO ec2 describe-instances --instance-ids $OLD \
  --query 'Reservations[0].Instances[0].{type:InstanceType,asg:Tags[?Key==`aws:autoscaling:groupName`]|[0].Value,state:State.Name}'   # c8gd.12xlarge, the old one
aws --profile $RW ec2 terminate-instances --instance-ids $OLD --dry-run
aws --profile $RW ec2 terminate-instances --instance-ids $OLD
echo "${N}_TERMINATED=$OLD" | tee -a $LOG
```
Instance store and root volume go with it, along with `/dev/sdf` on the backup (`DeleteOnTermination=true`). Check for orphaned volumes using the stack's own command:
```bash
aws --profile $RO ec2 describe-volumes --filters "Name=tag:Stack,Values=SovRollupCdkStarterStack" "Name=status,Values=available" --query 'Volumes[].[VolumeId,Size,CreateTime]' --output table
```
**Before terminating the old backup node (run 3),** set `SNAP_VOL` to the new backup's `/dev/sdf` volume for any future restore: `aws --profile $RO ec2 describe-volumes --filters Name=attachment.instance-id,Values=$NEW Name=attachment.device,Values=/dev/sdf --query 'Volumes[0].VolumeId' --output text`.

---

## 5. Finish
```bash
for g in $P_ASG $S_ASG $B_ASG; do aws --profile $RW autoscaling resume-processes --auto-scaling-group-name $g --scaling-processes ReplaceUnhealthy AZRebalance; done
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG \
  --query 'AutoScalingGroups[].{asg:AutoScalingGroupName,desired:DesiredCapacity,ltv:LaunchTemplate.Version,inst:Instances[].[InstanceId,LifecycleState,ProtectedFromScaleIn]}'
aws --profile $RO ec2 describe-instances --filters Name=tag:Stack,Values=SovRollupCdkStarterStack Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,PrivateIpAddress,Tags[?Key==`ASGType`]|[0].Value]' --output table
```
- **Proxy static fallback:** `backend-select.lua` has the leader/follower IPs baked in at render time. They're used only before node-discovery's first update. On the proxy, run `sudo grep -rnE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+:12346' /etc/openresty /usr/local/openresty 2>/dev/null`. If it still shows `10.0.2.21` / `10.0.3.143`, refresh the proxy config the way the proxy is normally re-rendered.
- **Alarms:** check that `SovRollupCdkStarterStack-health-check-failure` and `SovRollupCdkStarterStack-runtime-config-drift` are back to OK.
- **Old snapshots:** once the old backup is terminated, its snapshots on `vol-0b7f605d6d0900a4a` are no longer pruned. The same applies to the September snapshots of `vol-03be4d7901208fbe7` already in the account. Delete them when you no longer need them: `aws ec2 delete-snapshot --snapshot-id …`.
- **The stack ends without drift:** repeat the §2.6 drift detection. It must report `IN_SYNC 0`; in particular every ASG must be back to `DesiredCapacity 1`.
- Keep `$LOG`.

---

## 6. Time estimate

| Step | Duration |
|---|---|
| §2 pre-flight, incl. the Aurora drift fix deploy (§2.6) | 1–2 h |
| §3 CDK change, review, change set, deploy (incl. proxy roll) | 1–2 h + review time |
| Per node: bootstrap (§4.3) | 30–90 min |
| Per node: snapshot restore (§4.4) | 1–1.5 h |
| Per node: catch-up + soak (§4.5) | 24 h (run 1), ≥ 2 h (runs 2, 3) |
| Leader failover (§4.6, run 2) | seconds of write pause |
| §4.7 terminations + §5 | 1 h |

## Definition of done
- The Primary, Secondary and BackupNode ASGs each run exactly one **i8g.4xlarge** instance from the new launch template version. Scale-in protection is removed and the suspended processes are resumed.
- On each node, `/mnt/rollup` is on the single instance-store NVMe (no md device) and `/mnt/logs` is on EBS `/dev/sdg`. The backup also has `/mnt/snapshots` on EBS `/dev/sdf`, and new hourly snapshots of its new volume complete.
- One new preferred node is leader. Both preferred nodes return `ready=200`, have the same height and the same state roots. `https://relay-devnet.sovereign-labs.xyz` serves reads and writes. The only client-visible impact was the leader failover pause.
- The old c8gd.12xlarge instances are terminated, with no orphaned volumes, and the alarms are OK.
- The CDK repo holds the change (instance type, `/dev/sdg`, `DISK_PROFILE`). `cdk diff` on its main branch prints no differences, and CloudFormation drift detection reports `IN_SYNC`.
