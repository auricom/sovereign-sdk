# Runbook: migrate the relay-devnet sequencers (`SovRollupCdkStarterStack`) from c8gd.12xlarge to i8g.4xlarge with no downtime

> **Checked on 2026-11-17 and re-checked on 2026-11-19** against AWS account `313110633477` / `us-east-1` with the read-only profile `ReadOnlyAccess-313110633477`. Nothing has been changed in the account by this runbook.
> **Changes since 2026-11-17:** the stack was deployed three times on 2026-11-18 (14:33, 16:41, 19:41 UTC) by someone else, so every launch template and SSM document version, the AMI, the Ansible branch and the rollup commit are new (§0.1, §0.6, §2.1, §2.4). The Aurora drift is gone (§2.6). The CDK branch has moved on, and patch 0002 was rebuilt on top of it (§3.1). **None of these deploys was the migration**: the instances are still `c8gd.12xlarge` and still the ones launched in September (v39).
> Commands marked **[write]** need a profile with write access and are run by a human. Every other command is read-only.
> Node access is through **SSM Session Manager**: all nodes are registered and `Online`, running Ubuntu 24.04.

---

## 0. What the account really contains

### 0.1 Inventory

| Role | ASG (`desired/min/max`) | Instance | AZ / IP | Launch template | `DISK_PROFILE` today | Sequencer | Election role |
|---|---|---|---|---|---|---|---|
| Primary | `SovRollupCdkStarterStack-SovRollupComputePrimaryAsgASG448C4328-fVa4yPZ43E36` (1/1/2) | `i-0baeb42dfeb96cb28` | us-east-1a / 10.0.2.21 | `lt-02a6dc4c5d8cfdc99` (latest v48, instance from v39) | `aws_raid_0` | preferred | `DbElected` |
| Secondary | `SovRollupCdkStarterStack-SovRollupComputeSecondaryAsgASG8224C640-PKqRyQODK3bD` (1/0/3) | `i-0c9228ec8b0fa847a` | us-east-1b / 10.0.3.143 | `lt-05cf24139fde86881` (v48, instance v39) | `aws_raid_0` | preferred | `DbElected` |
| Backup | `SovRollupCdkStarterStack-SovRollupComputeBackupNodeAsgASGF12DEAEF-v241AoY1q69R` (1/1/2) | `i-0cc24aae7b0454089` | us-east-1a / 10.0.2.196 | `lt-0e54a3c6b1a4f422e` (v48, instance v39) | `aws_ebs_backup_node_raid_0` | standard (not part of the election) | – |
| Proxy (not migrated) | `…ProxyAsgASGB1045D50-nWkwreNMu6YC` | `i-035616ee95eaccd94` | us-east-1a / 10.0.0.9 | `lt-02bdfa1164fba9ea6` | – | – | – |

- All three rollup nodes use security group `sg-0e90e09ed3975a293`. Aurora accepts 5432 from that SG, and the proxy reaches 12346/3000 through it. **New instances get the same SG from the launch template, so no SG changes are needed.**
- Today's `aws_raid_0` layout: NVMe 1 + 2 in RAID0 at `/mnt/rollup` (ext4, 1.9 TB), NVMe 3 at `/mnt/logs`. The backup node also has EBS `/dev/sdf` (`vol-0b7f605d6d0900a4a`, 1800 GiB gp3, 30k IOPS) at `/mnt/snapshots`.
- Launch template **v48** (the ASGs' current version, since 2026-11-18 19:42 UTC) differs from **v39** (what the running instances booted from) in the AMI (`ami-0a78311d2d2afa4a3`, Ubuntu 24.04 arm64 `20261118`) and, through the SSM values documents, in the Ansible branch (`cdk-deployment-june-2026` → `cdk-deployment-november-2026`) and the rollup commit (`8df3d0bc…`).
- The 2026-11-18 deploys created launch template versions 45 to 48 (all `c8gd.12xlarge`; v46 briefly reverted the AMI to `ami-09a93d801db0bafae`, the 2026-10-23 image) and raised the three values documents from version 10 to 14 (`ANSIBLE_BRANCH` `cdk-deployment-october-2026` → `cdk-deployment-november-2026`, `ROLLUP_COMMIT_HASH` `40c49a68…` → `8df3d0bc…`). The proxy ASG was also updated (19:42 to 19:59 UTC).

### 0.2 Sizing check: GO

| | c8gd.12xlarge (today) | i8g.4xlarge (target) | Verdict |
|---|---|---|---|
| vCPU | 48 | **16** | Over 14 days, CloudWatch shows: average ≤ 0.4%, highest hourly average 1.4%, highest 1-min peak **14%** (secondary). That's ≈ 6.7 vCPU, or ≈ 42% of 16. **OK**. Ansible compiles the rollup at boot, so first boot is ~3× slower |
| RAM | 96 GiB | **128 GiB** | More RAM than today |
| Rollup disk | RAID0 2 × 950 GB = 1.9 TB | **1 × 3750 GB**, no RAID | Data = **944 GB** (last snapshot's `FullSnapshotSizeInBytes`), including the ~800 GiB NOMT hashtable (`user_hashtable_buckets: 210000000`). **Fits** |
| Logs disk | 3rd NVMe (950 GB) | new **64 GiB EBS gp3 on `/dev/sdg`** | Check usage in §2.3 |
| Capacity | – | i8g.4xlarge offered in us-east-1a/b/c/d | **OK** |
| vCPU quota | 512 | +16 during each swap (peak 162) | **OK** |

### 0.3 How the no-downtime switch works (SDK behaviour)
- Primary and secondary run with `node_role = "DbElected"` and leader election in Aurora (`leader_timeout` 1 s, `grace_period` 10 s, `heartbeat` 100 ms, which are the defaults because the CDK parameters are empty).
- A `DbElected` node that isn't leader runs as a Postgres-synced replica. When the leader stops heart-beating for > 1 s, a replica takes leadership, **exits**, and is restarted by systemd as the batch producer. `rollup.service` has `Restart=always`, `RestartSec=3`.
- **Reads** never stop. **Writes** pause for about 1 s plus the restart time when the leader node is retired (§4.6).
- A node that is still syncing returns non-200 on `/sequencer/ready`, so node-discovery keeps it out of read routing. Clients never see a stale node.
- The backup node is a *standard* sequencer and isn't part of the election. Swapping it has no client impact.

### 0.4 Strategy
1. **Deploy the CDK change once (§3).**
   - It creates new launch template versions (i8g.4xlarge plus an EBS logs volume) and new SSM values-document versions (single-NVMe `DISK_PROFILE`), and points the three ASGs at them.
   - **Running rollup instances aren't touched:** their ASGs have no rolling-update policy, and nodes only read the SSM documents at boot.
   - The Aurora drift fix is already deployed (2026-11-18).
2. **Prepare once (§4.0):**
   - disable the old backup's snapshot cron;
   - pre-warm one restore volume per AZ from the latest snapshot, both in parallel.
3. **Then, one node at a time, without soak periods (§4):**
   1. Scale its ASG from 1 to 2. The new instance gets scale-in and termination protection, boots from the new template, and Ansible sets up the single NVMe.
   2. Restore it from the warm volume. It catches up and becomes ready, and the proxy adds it automatically.
   3. Detach the old instance with a desired-capacity decrement (back to 1) and stop rollup on it. For the leader, this is the failover.
   4. Terminate the old instance right away, confirm it's gone, then lift the new instance's protections.
4. **Finish (§5):** re-enable backups, clean up, optionally roll the proxy, and ask Relay for a load test.

**Order: non-leader preferred node → leader preferred node → backup node.**

### 0.5 No drift: what goes through CDK and what doesn't
| Change | How | CloudFormation drift? |
|---|---|---|
| Instance type, block devices, `DISK_PROFILE`, Aurora version | **CDK parameters in `parameters.json` + a reviewed change set via `npm run deploy`** (§3) | None: the stack is the source |
| ASG desired capacity 1 → 2 → 1 | `aws autoscaling` during §4. The ASGs carry `UpdatePolicy.AutoScalingScheduledAction.IgnoreUnmodifiedGroupSizeProperties: true`, so a deploy doesn't fight runtime scaling. Every swap ends back at the template value (1) | None once each swap is done. §5 checks the stack is `IN_SYNC` |
| Suspended processes, scale-in protection, detach, terminate | `aws autoscaling` / `aws ec2` | Not stack properties, so no drift. Suspended processes are resumed in §5 |
| Pre-warmed restore volumes, termination protection, new snapshots | `aws ec2` (§4.0, §4.2, §5) and the stack's own scripts on the nodes | Not stack resources or properties |

**No launch template or SSM document is ever edited by hand.**

- The CDK app is [`Sovereign-Labs/sov-rollup-cdk-starter`](https://github.com/Sovereign-Labs/sov-rollup-cdk-starter). The live stack was deployed from branch `theodore/inplace-config-updates` (open PR #92), not `master`; see §3.
- CloudTrail shows every deploy is made by the IAM user **`aws-cdk-gentoo`** (last ones 2026-10-29, 2026-10-30, 2026-11-09, 2026-11-18 ×3, CDK `2.220.0`; the 2026-11-18 ones weren't re-checked in CloudTrail, only seen as `User Initiated` stack updates). §3 needs those deploy credentials.
- The change is **parameter-driven** (`RollupNodeInstanceType`, `AuroraEngineVersion` in `deployments/relay/devnet/parameters.json`). The defaults keep every other deployment (`relay/mainnet`, test accounts) resolving to exactly what it runs today. `AuroraEngineVersion` is already on the deploy branch; `RollupNodeInstanceType` comes with the migration patches.
- The stack's own node-level config-drift check (`SovRollupCdkStarterStack-runtime-config-drift` alarm) **will fire** between the deploy and the last swap. The old nodes run a configuration that no longer matches the stack. It clears once the old nodes are gone.

### 0.6 How a new i8g.4xlarge instance gets installed (no Ansible change needed)
The single-NVMe support already exists in the Ansible branch the new instances boot from. It was added upstream by `rollup-starter-ansible` commit `e47315fb34` (2026-10-05, *"Rework EBS disk detection and add support for EBS log volume"*), and that branch's README documents it for single-NVMe instances. This migration only has to **select** it, through CDK (§3).

Boot chain, from launch template user data:
1. User data writes `/etc/rollup-node.env` (`ROLLUP_NODE_TYPE=primary|secondary|backup`) and runs `/home/ubuntu/render_config.sh`.
2. The render script fetches the SSM document `…-runtime-config-values-<type>` and writes `/home/ubuntu/runtime_vars.yaml`. That file includes `disk_profile`, `rollup_node_role`, `rollup_sequencer_kind`, the Aurora URL, and `rollup_node_id` (= instance ID).
3. User data runs `bootstrap.sh` and `ansible-pull` of `rollup-starter-ansible@cdk-deployment-november-2026` (`local.yml`, roles `common` + `rollup`).
4. Ansible `common` role:
   - `custom_vars.yaml` loads the profile.
   - `disk_discovery.yaml` lists instance-store disks by model (`Amazon EC2 NVMe Instance Storage`). It resolves the EBS volumes by **EC2 attachment name** (`/dev/sdg`, `/dev/sdf`) using `discover_ebs_disk.py`, which reads the NVMe controller metadata, so Linux device order doesn't matter.
   - `disks.yaml` formats and mounts the disks.
5. Ansible `rollup` role:
   - builds the rollup at commit `8df3d0bc…` and installs `rollup.service`;
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
export LT_BASE=48                               # latest version of the 3 launch templates before the deploy
export DOC_BASE=14                              # default version of the 3 values documents before the deploy
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
export SNAP_VOL=vol-0b7f605d6d0900a4a           # old backup node's /dev/sdf: its snapshots seed the restore volumes
```
In a new shell: re-export the above, then `source $LOG`. §4.0 adds `SNAP`, `VOL_1A` and `VOL_1B` to it; §3.4 adds the `*_LTV` versions, and §4 the `*_NEW` IDs.

---

## 2. Pre-flight (read-only), go/no-go

### 2.1 Nothing changed since this runbook was checked
```bash
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG \
  --query 'AutoScalingGroups[].{asg:AutoScalingGroupName,desired:DesiredCapacity,max:MaxSize,ltv:LaunchTemplate.Version,inst:Instances[].[InstanceId,LifecycleState,HealthStatus]}' --output json
aws --profile $RO ec2 describe-launch-templates --launch-template-ids $P_LT $S_LT $B_LT \
  --query 'LaunchTemplates[].[LaunchTemplateId,LatestVersionNumber]' --output text            # all 48
for d in $P_DOC $S_DOC $B_DOC; do aws --profile $RO ssm describe-document --name $d --query 'Document.[Name,DefaultVersion]' --output text; done   # all 14
aws --profile $RO cloudformation describe-stacks --stack-name SovRollupCdkStarterStack --query 'Stacks[0].[StackStatus,LastUpdatedTime]' --output text   # UPDATE_COMPLETE 2026-11-18T19:41:03+00:00
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

**Result on 2026-11-19: `LEADER=S`.** Secondary (`i-0c9228ec8b0fa847a`) reports `"BatchProducer"`; primary (`i-0baeb42dfeb96cb28`) reports `"PgSyncReplica"`, `ready=200`, slot 5741618 → 5741619. The primary is migrated first.

### 2.3 Disks and logs volume size
On each old node:
```bash
cat /proc/mdstat; lsblk -o NAME,MODEL,SIZE,MOUNTPOINTS
df -h /mnt/rollup /mnt/logs /mnt/snapshots 2>/dev/null
sudo du -sh /mnt/logs/* 2>/dev/null | sort -h | tail
```
New nodes get a **64 GiB** `/dev/sdg` logs volume (`ROLLUP_LOGS_VOLUME_GIB` in the CDK patch, the size the Ansible README suggests). If `/mnt/logs` uses more than ~30 GiB on any old node, raise that constant in patch 0001 before deploying.

### 2.4 Same rollup version everywhere
On each old node:
```bash
grep -E '^(rollup_commit_hash|disk_profile|rollup_node_role|rollup_sequencer_kind|rollup_node_id)' /home/ubuntu/runtime_vars.yaml
```
Every node must show `rollup_commit_hash: "8df3d0bc67268b1d249feff682edbe34a8c7d230"`, the value new nodes get from SSM. **If an old node runs a different commit, stop.** New nodes would run a different binary from the one that wrote the snapshot.

**Result on 2026-11-19: pass.** All three old nodes show `rollup_commit_hash: "8df3d0bc67268b1d249feff682edbe34a8c7d230"` (and still `disk_profile` `aws_raid_0` / `aws_ebs_backup_node_raid_0`). Their instances still date from September (v39), so the nodes were updated in place after the 2026-11-18 SSM change, not replaced. The `runtime_vars.yaml` value is what new nodes get, but check the *running* binary too: `systemctl show rollup -p ActiveEnterTimestamp` should be after that update.

**Running binary, checked on 2026-11-19:** `ActiveEnterTimestamp` is 2026-11-18 18:29:34 UTC on the primary and 18:29:37 UTC on the secondary, both after SSM v13 (16:42 UTC, `8df3d0bc…`). On the backup it is 2026-11-19 10:13:50 UTC, which is just the hourly snapshot restart, so it doesn't date the build; its `runtime_vars.yaml` already shows `8df3d0bc…`.

### 2.5 Snapshots are healthy
```bash
aws --profile $RO ec2 describe-snapshots --owner-ids self \
  --filters Name=volume-id,Values=$SNAP_VOL Name=tag:AutomatedBackup,Values=true \
  --query 'reverse(sort_by(Snapshots,&StartTime))[:4].[SnapshotId,StartTime,State,FullSnapshotSizeInBytes]' --output table
```
The latest `completed` snapshot must be < 2 h old. On the backup node: `sudo tail -40 /mnt/logs/rollup-snapshot.log` should show `=== Snapshot complete ===` every hour, with no `ERROR`.

**Result on 2026-11-19 10:19 UTC:** latest completed snapshot 09:13 (1 h 06 min old), 10:13 pending. Only 5 snapshots exist (2026-11-19 10:13 pending, 09:13, 08:13, 2026-11-17 23:13, 2026-11-07 03:13). That is expected, not a gap: `snapshot-rollup-data.sh` keeps one snapshot near ages 0 h, 6 h, 1 d, 1 w and 1 month and deletes the rest.

### 2.6 Drift baseline: the stack must be in sync before we start
```bash
DID=$(aws --profile $RO cloudformation detect-stack-drift --stack-name $STACK --query StackDriftDetectionId --output text)
sleep 60; aws --profile $RO cloudformation describe-stack-drift-detection-status --stack-drift-detection-id $DID \
  --query '[DetectionStatus,StackDriftStatus,DriftedStackResourceCount]' --output text
aws --profile $RO cloudformation describe-stack-resource-drifts --stack-name $STACK --stack-resource-drift-status-filters MODIFIED DELETED \
  --query 'StackResourceDrifts[].[LogicalResourceId,ResourceType,PropertyDifferences[].PropertyPath|join(`,`,@)]' --output table
```
Expect `DETECTION_COMPLETE  IN_SYNC  0`. If anything is already drifted, sort it out with the CDK owner first: either fix it in code or revert it by hand. Otherwise the deploy in §3 will silently overwrite it.

**Result on 2026-11-17: `DRIFTED 1`.** (**Re-run on 2026-11-19: `DETECTION_COMPLETE  IN_SYNC  0`.** The 2026-11-18 deploy applied the fix below, and the cluster still reports 17.9. Keep the rest of this paragraph for reference.) `SovRollupAuroraClusterC528DD46` (`AWS::RDS::DBCluster`) has `/EngineVersion` expected `17.5`, actual `17.9`, after an automatic Aurora minor upgrade.
- The fix (`AuroraEngineVersion=17.9`, §3) was deployed on 2026-11-18.
- Check the cluster still reports 17.9 before deploying (17.9 on 2026-11-19): `aws --profile $RO rds describe-db-clusters --query 'DBClusters[].EngineVersion'`. If it moved again, bump `AuroraEngineVersion` in `parameters.json` first.
- This drift is the only one acceptable here. Anything else must be sorted out first. If the read-only profile can't call `detect-stack-drift`, run these three commands with `$RW`; drift detection doesn't modify anything.

**Go/no-go on 2026-11-19: GO for §3.** 2.1 baselines updated (v48 / v14), 2.2 `LEADER=S`, 2.3 64 GiB logs volume is enough, 2.4 and 2.5 pass, 2.6 `IN_SYNC 0`. Before deploying, re-check that the stack's `LastUpdatedTime` is still `2026-11-18T19:41:03+00:00`, and re-run 2.5 if the pending 10:13 snapshot hasn't completed by about 11:13 UTC.

### 2.7 Freeze
Until §5 is done, don't do any of the following:
- any deploy of the stack other than §3 and the optional proxy rollout in §5 (and never from `master` while PR #92 is open)
- the stack's `MigrateLiveNodesCommand`
- `render_config.sh` / `run_ansible.sh` on the old nodes. After §3 they would read the new single-NVMe `DISK_PROFILE`, and Ansible would fail on hardware that doesn't match it.

Tell dev users when the leader swap (a few seconds of write pause) will happen.

---

## 3. Deploy the CDK change **[write, run by whoever deploys the stack]**

**Repo:** [`Sovereign-Labs/sov-rollup-cdk-starter`](https://github.com/Sovereign-Labs/sov-rollup-cdk-starter). Devnet is deployed with `npm run deploy relay devnet`, configured by `deployments/relay/devnet/{config.json,parameters.json}` (account `313110633477`, stack `SovRollupCdkStarterStack`, `production: true`, so the script asks you to type `yes`).

**Deploy from `theodore/inplace-config-updates` (open PR #92), never from `master`.**
- That branch is what's deployed. The live stack (last update 2026-11-18 19:41 UTC) already has the **Aurora drift fix** (`AuroraEngineVersion=17.9` is a live stack parameter), which came in with:
  - `d0ef2ae`: new `AuroraEngineVersion` parameter, default `17.5`;
  - `5e6cd19`: devnet sets it to `17.9`;
  - `03d9448`: devnet bump with the latest rollup (the `8df3d0bc…` commit).
- The branch head on 2026-11-19 is **`82b426f`** (2026-11-18 21:37 UTC), with two commits that are **not deployed yet**: `47b4a64` (devnet `AnsibleBranch` → `cdk-deployment-november-2026`, which the live SSM values already have; plus a fix to the proxy's replacement race in `lib/userdata/proxy/cfn-helpers.sh`) and `82b426f` (mainnet only). The next devnet deploy therefore also changes the proxy launch template and ASG (see §3.2).
- `master` lacks the SSM render-document scheme the nodes rely on.

### 3.1 Apply the migration patches
The patches are in this fork, in [`docs/ec2-migration-cdk/`](ec2-migration-cdk/), built on top of `82b426f` (patch 0002 was rebuilt on 2026-11-19: its context changed when `47b4a64` moved the devnet `AnsibleBranch` to `cdk-deployment-november-2026`; patch 0001 is unchanged):

| Patch | Change |
|---|---|
| `0001-Add-RollupNodeInstanceType-for-single-NVMe-rollup-no.patch` | New parameter **`RollupNodeInstanceType`** (`c8gd.12xlarge` default, or `i8g.4xlarge`). It sets the instance type of the primary, secondary and backup launch templates and selects the matching `DISK_PROFILE` (`aws_raid_0` / `aws_ebs_backup_node_raid_0`, or `aws_single_nvme_ebs_logs` / `aws_ebs_backup_node_single_nvme`). With `i8g.4xlarge` it also adds a 64 GiB gp3 encrypted `/dev/sdg` logs volume (`ROLLUP_LOGS_VOLUME_GIB`). It also fixes the `RestoreFromVolumeInstructions` output, and adds `test/rollup-node-hardware.test.ts`. Instance type and disk layout are one parameter, so they can't get out of step |
| `0002-Move-relay-devnet-rollup-nodes-to-i8g.4xlarge.patch` | `deployments/relay/devnet/parameters.json`: `"RollupNodeInstanceType": "i8g.4xlarge"` |

How I checked it:
- `npm test` passes **108/108**.
- Resolved per deployment:
  - devnet changes only the 3 rollup launch templates (`i8g.4xlarge`, plus `/dev/sdg`) and the 3 values documents (`DISK_PROFILE`);
  - mainnet and the test deployments change nothing.

```bash
git clone git@github.com:Sovereign-Labs/sov-rollup-cdk-starter.git && cd sov-rollup-cdk-starter
git switch theodore/inplace-config-updates && git log -3 --oneline     # 82b426f or newer; if newer, re-check that both patches still apply
git switch -c i8g-4xlarge-devnet
git am /path/to/sovereign-sdk/docs/ec2-migration-cdk/000*.patch
npm ci && npx tsc --noEmit -p . && npm test                             # 108 passed
```
Open the PR **against `theodore/inplace-config-updates`** and get it reviewed. Deploy from this branch, and merge it only after the deploy succeeds.

### 3.2 Preview the change set (nothing is applied)
With the deploy credentials (`aws-cdk-gentoo` in CloudTrail). `npm run deploy` forwards extra flags to `cdk deploy`:
```bash
npm run validate relay devnet
npm run deploy relay devnet -- --no-execute --change-set-name i8g-4xlarge-migration
aws cloudformation describe-change-set --stack-name $STACK --change-set-name i8g-4xlarge-migration \
  --query 'Changes[].ResourceChange.{action:Action,id:LogicalResourceId,type:ResourceType,replacement:Replacement,scope:join(`,`,Scope)}' --output table
```
Expected rows, all `Modify` with `Replacement = False`:

| Resource | Why |
|---|---|
| `SovRollupComputePrimaryLaunchTemplate…`, `…SecondaryLaunchTemplate…`, `…BackupNodeLaunchTemplate…` | `i8g.4xlarge`, `/dev/sdg`, and the Ubuntu arm64 AMI the stack's SSM parameter resolves to on the day (`ami-0a78311d2d2afa4a3` today; a newer one only if Canonical published one since) |
| `SovRollupComputePrimaryAsg…`, `…SecondaryAsg…`, `…BackupNodeAsg…` | `LaunchTemplate.Version` only. No rolling-update policy, so **no instance is replaced** |
| `SovRollupComputeRuntimeConfigValuesPrimary…`, `…Secondary…`, `…Backup…` | `DISK_PROFILE`. `UpdateMethod: NewVersion` creates a new version and makes it the default |
| `SovRollupComputeProxyLaunchTemplate…` + `SovRollupComputeProxyAsg…` | Expected: `47b4a64` changed the proxy user data (`cfn-helpers.sh`) after the last deploy, and a newer Ubuntu amd64 AMI may also apply. The proxy ASG's rolling-update policy replaces the proxy one for one, as every recent deploy did. Run at a quiet time and watch `https://$DOMAIN/sequencer/ready` |

**Stop and don't execute** if the change set shows anything else: any `Replacement: True`, any `Remove`, Aurora *instances* / EIP / VPC / security group / IAM changes, or `MinSize`/`MaxSize`/`DesiredCapacity`. Drop it with `aws cloudformation delete-change-set --stack-name $STACK --change-set-name i8g-4xlarge-migration`.

### 3.3 Deploy
```bash
aws cloudformation execute-change-set --stack-name $STACK --change-set-name i8g-4xlarge-migration
aws cloudformation wait stack-update-complete --stack-name $STACK
aws cloudformation describe-stack-events --stack-name $STACK --max-items 40 \
  --query 'StackEvents[].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' --output table
```

### 3.4 Check the result (read-only)
```bash
aws --profile $RO cloudformation describe-stacks --stack-name $STACK \
  --query 'Stacks[0].Parameters[?ParameterKey==`RollupNodeInstanceType` || ParameterKey==`AuroraEngineVersion`]' --output table
for lt in $P_LT $S_LT $B_LT; do
  aws --profile $RO ec2 describe-launch-template-versions --launch-template-id $lt --versions '$Latest' \
    --query 'LaunchTemplateVersions[0].[LaunchTemplateId,VersionNumber,LaunchTemplateData.InstanceType,join(`,`,LaunchTemplateData.BlockDeviceMappings[].DeviceName)]' --output text
done                                  # i8g.4xlarge, /dev/sda1,/dev/sdg (backup: /dev/sda1,/dev/sdf,/dev/sdg)
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG \
  --query 'AutoScalingGroups[].[AutoScalingGroupName,LaunchTemplate.Version,DesiredCapacity,Instances[0].InstanceId,Instances[0].LaunchTemplate.Version]' --output text
                                      # ASG version = latest; instances still the old ones (v39), desired 1
for d in $P_DOC $S_DOC $B_DOC; do printf '%s: ' $d
  aws --profile $RO ssm get-document --name $d --query Content --output text | jq -r '.mainSteps[0].inputs.runCommand[]' \
    | grep '^DISK_PROFILE=' | sed -E "s/.*'([^']+)'.*/\1/" | base64 -d; echo
done
```
Record the new versions: `echo "P_LTV=<n> S_LTV=<n> B_LTV=<n>" | tee -a $LOG`. Repeat the §2.6 drift detection: it must report `IN_SYNC 0`.

From now on, any instance launched by these ASGs is an i8g.4xlarge with the single-NVMe layout, including an unplanned replacement.

---

## 4. Per-node procedure

### 4.0 Preparation, once, before the first node **[write]**
These steps can run while §3 is in review.

**a) Stop the ASGs from replacing instances on their own.**
- The instance role can call `autoscaling:SetInstanceHealth`.
- The CDK repo has a health-check script (`lib/health-check-script.ts`, not wired in on the deployed branch) that would flag a node that's mid-bootstrap or mid-restore.
- Suspended processes aren't a stack property, so this causes no drift; they're resumed in §5.
```bash
for g in $P_ASG $S_ASG $B_ASG; do
  aws --profile $RW autoscaling suspend-processes --auto-scaling-group-name $g --scaling-processes ReplaceUnhealthy AZRebalance
done
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG --query 'AutoScalingGroups[].SuspendedProcesses[].ProcessName'
```

**b) Disable the snapshot cron on the old backup node.** Missing snapshots for a few hours, or even a couple of days, is fine. It also freezes the snapshot the restore volumes are built from.
```bash
aws --profile $RW ssm start-session --target $B_OLD
sudo crontab -u ubuntu -e        # comment out:  0 * * * * /home/ubuntu/snapshot-rollup-data.sh
sudo crontab -u ubuntu -l        # the line now starts with '#'
pgrep -af snapshot-rollup-data   # nothing running; if a run is in progress, wait for it to finish
```

**c) Pre-create and pre-warm one restore volume per AZ, from the latest completed snapshot.**
- Most of a restore's time is EBS initializing a new volume from the snapshot. Doing it now, for both AZs in parallel, takes it out of each node's turn.
- Volumes are AZ-locked:
  - `us-east-1a` → one volume, used by the **primary** and then the **backup**;
  - `us-east-1b` → one volume for the **secondary**.
- `restore-from-snapshot.sh` **keeps** a volume passed by ID: it detaches it after the restore and only deletes volumes it created from a snapshot itself. So one volume can serve several restores, one at a time.
- A snapshot that's a day or two old only adds catch-up time.
```bash
SNAP=$(aws --profile $RO ec2 describe-snapshots --owner-ids self \
  --filters Name=volume-id,Values=$SNAP_VOL Name=tag:AutomatedBackup,Values=true Name=status,Values=completed \
  --query 'sort_by(Snapshots,&StartTime)[-1].SnapshotId' --output text)
aws --profile $RO ec2 describe-snapshots --snapshot-ids $SNAP --query 'Snapshots[0].[SnapshotId,StartTime,State,FullSnapshotSizeInBytes]' --output text
newvol() {  # $1 = AZ; rate 300 MiB/s fully initializes ~944 GB in ~1 h
  aws --profile $RW ec2 create-volume --snapshot-id $SNAP --availability-zone $1 \
    --volume-type gp3 --iops 16000 --throughput 1000 --volume-initialization-rate 300 \
    --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=i8g-migration-restore-$1},{Key=Purpose,Value=i8g-migration-restore},{Key=SourceSnapshot,Value=$SNAP}]" \
    --query VolumeId --output text
}
VOL_1A=$(newvol us-east-1a); VOL_1B=$(newvol us-east-1b)
echo "SNAP=$SNAP VOL_1A=$VOL_1A VOL_1B=$VOL_1B" | tee -a $LOG
aws --profile $RO ec2 wait volume-available --volume-ids $VOL_1A $VOL_1B
# Initialization progress, repeat until both report 100
aws --profile $RO ec2 describe-volume-status --volume-ids $VOL_1A $VOL_1B \
  --query 'VolumeStatuses[].[VolumeId,AvailabilityZone,InitializationStatusDetails.Progress,InitializationStatusDetails.EstimatedTimeToCompleteInSeconds]' --output table
```
Delete both volumes in §5. They aren't stack resources, so they cause no drift.

Run §4.1–§4.7 three times, finishing each node before starting the next:

| Run | Node | Variables | Restore volume |
|---|---|---|---|
| 1 | the **non-leader** from §2.2 | `N=S ASG=$S_ASG LTV=$S_LTV OLD=$S_OLD` (or the `P_` set if the secondary leads) | `$VOL_1B` for the secondary, `$VOL_1A` for the primary |
| 2 | the **leader** | the other `P_` / `S_` set | same rule |
| 3 | **backup** | `N=B ASG=$B_ASG LTV=$B_LTV OLD=$B_OLD` | `$VOL_1A` |

### 4.1 Confirm the ASG launches the new template
```bash
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG \
  --query 'AutoScalingGroups[0].[LaunchTemplate.Version,DesiredCapacity,length(Instances)]' --output text   # $LTV  1  1
```

### 4.2 Launch the new instance and protect it **[write]**
```bash
aws --profile $RW autoscaling set-desired-capacity --auto-scaling-group-name $ASG --desired-capacity 2
watch -n15 "aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG \
  --query 'AutoScalingGroups[0].Instances[].[InstanceId,LifecycleState,HealthStatus,LaunchTemplate.Version]' --output text"
NEW=<instance id whose LaunchTemplate.Version = $LTV>; echo "${N}_NEW=$NEW" | tee -a $LOG
# Until the old instance is gone, protect the new one from both ASG scale-in and a mistaken terminate-instances
aws --profile $RW autoscaling set-instance-protection --auto-scaling-group-name $ASG --instance-ids $NEW --protected-from-scale-in
aws --profile $RW ec2 modify-instance-attribute --instance-id $NEW --disable-api-termination
aws --profile $RO ec2 describe-instances --instance-ids $NEW \
  --query 'Reservations[0].Instances[0].{type:InstanceType,az:Placement.AvailabilityZone,ip:PrivateIpAddress,disks:BlockDeviceMappings[].DeviceName}'
```
- Expect `i8g.4xlarge`, the same AZ as the old node, and disks `/dev/sda1`, `/dev/sdg` (plus `/dev/sdf` for the backup).
- EC2 termination protection doesn't stop the ASG from terminating an instance. That's what the scale-in protection is for. Together they cover both paths.
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
**Run 3 (backup) only:** Ansible installed the hourly snapshot cron. Disable it before restoring, the same way as in §4.0 b (`sudo crontab -u ubuntu -e`). It's re-enabled in §5.

If the bootstrap failed, the last Ansible task in the log explains why. The usual cause is *"Not enough instance store disks for RAID"*: the instance read the old `DISK_PROFILE`, so re-check §3.4. To discard the instance:
```bash
aws --profile $RW ec2 modify-instance-attribute --instance-id $NEW --no-disable-api-termination
aws --profile $RW autoscaling set-instance-protection --auto-scaling-group-name $ASG --instance-ids $NEW --no-protected-from-scale-in
aws --profile $RW autoscaling terminate-instance-in-auto-scaling-group --instance-id $NEW --should-decrement-desired-capacity
```

Primary and secondary nodes are `DbElected`. Until the restore is done, make sure the current leader stays up, because this node isn't caught up yet.

### 4.4 Restore from the pre-warmed volume
In the session on `$NEW`. Use `$VOL_1A` for the primary and backup, `$VOL_1B` for the secondary; the IDs are in `$LOG`. The volume must be `available`: one restore at a time per volume.
```bash
sudo -iu ubuntu
tmux new -s restore
aws ec2 describe-volumes --region us-east-1 --volume-ids <VOL> --query 'Volumes[0].[State,AvailabilityZone]' --output text   # available  <same AZ as this node>
/home/ubuntu/restore-from-snapshot.sh <VOL>
```
Type `yes`. The script (from Ansible):
1. checks the volume is `available` and in this node's AZ;
2. stops rollup, attaches the volume as `/dev/sdr` and mounts it read-only;
3. wipes `/mnt/rollup` and rsyncs `/rollup/` into it, then fixes ownership;
4. **detaches the volume and keeps it** for the next restore;
5. starts rollup.

With a warm volume this takes about 15–30 min for 944 GB. The log is in `/mnt/logs/rollup-restore.log`.

If it stops halfway, its exit trap unmounts and detaches the volume without deleting it. Check the volume is `available` again, then re-run.

### 4.5 Catch-up checks
On `$NEW`:
```bash
curl -s localhost:12346/rollup/sync-status | jq -c
curl -s localhost:12346/ledger/slots/latest | jq -c '{number,state_root}'     # starts near the snapshot height, not genesis, then rises
curl -s -o /dev/null -w 'ready=%{http_code}\n' localhost:12346/sequencer/ready # 200 once caught up
curl -s localhost:12346/sequencer/role; echo
sudo journalctl -u rollup --since '-30min' | grep -Ei 'error|panic|mismatch' | tail
uptime; free -g; df -h /mnt/rollup /mnt/logs
```
- **Same state root as the cluster:** pick a slot `N` slightly below the head and compare `curl -s localhost:12346/ledger/slots/$N | jq -r .state_root` on `$NEW` and on the leader. The values must be identical.
- Once `ready=200`, node-discovery puts the node in read routing automatically. There's no soak period: move on to §4.6. The load test comes at the end (§5).

### 4.6 Retire the old instance (in run 2 this is the leader failover)
**1. Check identities before touching anything.** OLD must be the c8gd.12xlarge and NEW the i8g.4xlarge with termination protection on, both in `$ASG`:
```bash
aws --profile $RO ec2 describe-instances --instance-ids $OLD $NEW \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name,Tags[?Key==`aws:autoscaling:groupName`]|[0].Value]' --output table
aws --profile $RO ec2 describe-instance-attribute --instance-id $NEW --attribute disableApiTermination --query 'DisableApiTermination.Value'   # true
```

**2. Run 2 only:** every running preferred node must return `ready=200` and be at the same height. Then open these watches:
```bash
# laptop: write-path probe through the proxy (/sequencer/ready is always routed to the leader)
while true; do printf '%s ' "$(date +%T)"; curl -s -o /dev/null -m 2 -w '%{http_code}\n' https://$DOMAIN/sequencer/ready; sleep 0.5; done
# sessions on both new preferred nodes
sudo journalctl -u rollup -f | grep -Ei 'leader|BatchProducer'
```

**3. Detach the old instance from the ASG** (desired goes back to 1) **and stop rollup on it:**
```bash
aws --profile $RW autoscaling detach-instances --auto-scaling-group-name $ASG --instance-ids $OLD --should-decrement-desired-capacity
echo "${N}_DETACHED=$OLD" | tee -a $LOG
aws --profile $RW ssm start-session --target $OLD
sudo systemctl disable --now rollup && sudo systemctl mask rollup; systemctl is-active rollup      # inactive
```
- Run 2: within seconds, one of the new `DbElected` nodes logs *"Replica acquired leadership! Exiting to restart as leader."*. systemd restarts it, and it logs *"DbElected node acquired leadership, running as BatchProducer"*. The probe shows a short run of non-200 responses, then 200. Either new node may win.
- Post-check: `curl -s https://$DOMAIN/ledger/slots/latest | jq .number` keeps rising. Send a test transaction with your usual tool and confirm it's accepted.

### 4.7 Terminate the old instance and confirm it's gone **[write]**
Do it straight away, before starting the next node. A detached instance still carries the `ASGType` tag the proxy uses to find nodes, so it mustn't linger.
```bash
# Guard: only terminate if OLD really is the old c8gd.12xlarge and no longer in the ASG
read TYPE STATE < <(aws --profile $RO ec2 describe-instances --instance-ids $OLD \
  --query 'Reservations[0].Instances[0].[InstanceType,State.Name]' --output text)
INASG=$(aws --profile $RO autoscaling describe-auto-scaling-instances --instance-ids $OLD --query 'length(AutoScalingInstances)' --output text)
echo "$OLD: $TYPE $STATE in-asg=$INASG"
[ "$TYPE" = c8gd.12xlarge ] && [ "$INASG" = 0 ] && [ "$OLD" != "$NEW" ] \
  && aws --profile $RW ec2 terminate-instances --instance-ids $OLD \
  && aws --profile $RO ec2 wait instance-terminated --instance-ids $OLD \
  && echo "${N}_TERMINATED=$OLD" | tee -a $LOG
```
Confirm the old instance is retired and only the new one remains:
```bash
aws --profile $RO ec2 describe-instances --instance-ids $OLD --query 'Reservations[0].Instances[0].State.Name' --output text      # terminated
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG \
  --query 'AutoScalingGroups[0].[DesiredCapacity,Instances[].[InstanceId,InstanceType,LifecycleState]]' --output json   # 1, only $NEW (i8g.4xlarge, InService)
aws --profile $RO ec2 describe-instances --filters Name=tag:Stack,Values=$STACK Name=instance-state-name,Values=running,stopped \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,Tags[?Key==`ASGType`]|[0].Value]' --output table              # no c8gd.12xlarge left for this role
```
Only now lift the protections on the new instance:
```bash
aws --profile $RW ec2 modify-instance-attribute --instance-id $NEW --no-disable-api-termination
aws --profile $RW autoscaling set-instance-protection --auto-scaling-group-name $ASG --instance-ids $NEW --no-protected-from-scale-in
```
The old root volume goes with the instance, along with `/dev/sdf` on the backup (`DeleteOnTermination=true`).

---

## 5. Finish
1. **Re-enable the snapshot cron on the new backup node.** In a session on `$B_NEW`, run `sudo crontab -u ubuntu -e` and uncomment the line. Wait for the next full hour and the first `=== Snapshot complete ===` in `/mnt/logs/rollup-snapshot.log`, then confirm:
   ```bash
   aws --profile $RO ec2 describe-snapshots --owner-ids self --filters Name=tag:InstanceId,Values=$B_NEW --query 'Snapshots[].[SnapshotId,StartTime,State]' --output table
   ```
   New snapshots come from the new backup's `/dev/sdf` volume. Future restores must look them up by that volume's ID:
   ```bash
   aws --profile $RO ec2 describe-volumes --filters Name=attachment.instance-id,Values=$B_NEW Name=attachment.device,Values=/dev/sdf --query 'Volumes[0].VolumeId' --output text
   ```
2. **Delete the restore volumes** (§4.0 c):
   ```bash
   aws --profile $RO ec2 describe-volumes --volume-ids $VOL_1A $VOL_1B --query 'Volumes[].[VolumeId,State]' --output text   # both available
   aws --profile $RW ec2 delete-volume --volume-id $VOL_1A && aws --profile $RW ec2 delete-volume --volume-id $VOL_1B
   ```
3. **Resume the ASG processes and check the end state:**
   ```bash
   for g in $P_ASG $S_ASG $B_ASG; do aws --profile $RW autoscaling resume-processes --auto-scaling-group-name $g --scaling-processes ReplaceUnhealthy AZRebalance; done
   aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG \
     --query 'AutoScalingGroups[].{asg:AutoScalingGroupName,desired:DesiredCapacity,ltv:LaunchTemplate.Version,suspended:SuspendedProcesses[].ProcessName,inst:Instances[].[InstanceId,LifecycleState,ProtectedFromScaleIn]}'
   aws --profile $RO ec2 describe-instances --filters Name=tag:Stack,Values=$STACK Name=instance-state-name,Values=running \
     --query 'Reservations[].Instances[].[InstanceId,InstanceType,PrivateIpAddress,Tags[?Key==`ASGType`]|[0].Value]' --output table
   ```
4. **Optional: roll the proxy** so its static fallback IPs point at the new nodes.
   - The proxy finds the primary and secondary IPs by EC2 tag when it boots. They're only used before node-discovery's first update.
   - Do this only after all old instances are terminated, because detached ones still carry the tags.
   - In `deployments/relay/devnet/parameters.json`, increment the dummy parameter **`ProxyRolloutToken`** (`"5"` → `"6"`). That changes the proxy user data and makes its ASG roll over.
   - Preview it like §3.2 (`--change-set-name proxy-rollout`). Expect only `Modify` on `SovRollupComputeProxyLaunchTemplate…` and `SovRollupComputeProxyAsg…`.
   - Execute it, watch `https://$DOMAIN/sequencer/ready`, and commit the parameter change.
5. **Alarms:** check that `SovRollupCdkStarterStack-health-check-failure` and `SovRollupCdkStarterStack-runtime-config-drift` are back to OK.
6. **The stack ends without drift:** repeat the §2.6 drift detection. It must report `IN_SYNC 0`; in particular every ASG must be back to `DesiredCapacity 1`.
7. **Old snapshots:** the snapshots of `vol-0b7f605d6d0900a4a` (old backup) are no longer pruned, nor are the September ones of `vol-03be4d7901208fbe7`. Delete them when you no longer need them: `aws ec2 delete-snapshot --snapshot-id …`.
8. **Ask Relay for a load test** on the fully migrated devnet.

---

## 6. Time estimate
Everything fits in one working day.

| Step | Duration |
|---|---|
| §2 pre-flight | 30 min |
| §3 apply patches, review, preview + deploy (plus proxy roll if it happens) | 30–60 min + review time |
| §4.0 prep: volumes pre-warm in parallel, during §3 | ~1 h, mostly waiting |
| Per node: bootstrap (§4.3) | 30–90 min |
| Per node: restore from warm volume (§4.4) + catch-up (§4.5) | 30–45 min |
| Per node: retire + terminate (§4.6–§4.7) | 10 min (run 2: seconds of write pause) |
| §5 | 30 min, plus up to 1 h waiting for the first new snapshot |

## Definition of done
- The Primary, Secondary and BackupNode ASGs each run exactly one **i8g.4xlarge** instance from the new launch template version. Scale-in and termination protection are removed and the suspended processes are resumed.
- On each node, `/mnt/rollup` is on the single instance-store NVMe (no md device) and `/mnt/logs` is on EBS `/dev/sdg`. The backup also has `/mnt/snapshots` on EBS `/dev/sdf`, its snapshot cron is enabled, and its new hourly snapshots complete.
- One new preferred node is leader. Both preferred nodes return `ready=200`, have the same height and the same state roots. `https://relay-devnet.sovereign-labs.xyz` serves reads and writes. The only client-visible impact was the leader failover pause.
- All three old c8gd.12xlarge instances are terminated, and the restore volumes are deleted. No orphaned volumes, alarms OK.
- The migration PR is merged into `theodore/inplace-config-updates`, and `deployments/relay/devnet/parameters.json` sets `RollupNodeInstanceType: i8g.4xlarge` and `AuroraEngineVersion: 17.9`. CloudFormation drift detection reports `IN_SYNC 0`.
- Relay has been asked for a load test.
