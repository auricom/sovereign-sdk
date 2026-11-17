# Runbook: migrate the relay-devnet sequencers (`SovRollupCdkStarterStack`) from c8gd.12xlarge to i8g.4xlarge with no downtime

> **Checked on 2026-11-17** against AWS account `313110633477` / `us-east-1` with the read-only profile `ReadOnlyAccess-313110633477`. Nothing has been changed in the account.
> Commands marked **[write]** need a profile with write access and are run by a human. Every other command is read-only.
> Node access is through **SSM Session Manager**: all nodes are registered and `Online`, running Ubuntu 24.04.

---

## 0. What the account really contains

### 0.1 Corrections to the earlier version of this runbook

| Earlier assumption | What the account shows |
|---|---|
| Current type i8g.12xlarge | **c8gd.12xlarge**: 48 vCPU, **96 GiB** RAM, **3 × 950 GB** NVMe |
| Instances created by hand, no IaC | AWS **CDK / CloudFormation stack `SovRollupCdkStarterStack`**. Each node is the single instance of its own **Auto Scaling group** (min 1 / desired 1), launched from a launch template |
| RAID built by hand | Built at boot by **Ansible** (`Sovereign-Labs/rollup-starter-ansible`, branch `cdk-deployment-october-2026`, `roles/common/tasks/disks.yaml`) run via `ansible-pull` from user data. The layout comes from `DISK_PROFILE` in an SSM document. **Single-NVMe profiles already exist in that branch**, so no Ansible code changes are needed |
| Snapshot format unknown | Every hour the backup node stops rollup, rsyncs `/mnt/rollup` to an EBS volume on `/dev/sdf` and takes an **EBS snapshot** (retention ≈ 0 h / 6 h / 1 d / 1 w / 1 mo). Every node already has the restore script **`/home/ubuntu/restore-from-snapshot.sh <snap-id>`** |
| Write routing unknown | The proxy (t3.small, EIP `44.193.19.138`, `relay-devnet.sovereign-labs.xyz`) runs **node-discovery** against Postgres. Writes go to the leader. Reads go only to followers whose `/sequencer/ready` returns 200. **Nothing to switch by hand** |
| Postgres somewhere | **Aurora PostgreSQL 17.9**: writer in us-east-1a, reader in us-east-1b (db.r8g.large) |
| Node IDs set by hand | `rollup_node_id` = **EC2 instance ID**, set automatically by the render script |

### 0.2 Inventory

| Role | ASG (`desired/min/max`) | Instance | AZ / IP | Launch template | `DISK_PROFILE` today | Sequencer | Election role |
|---|---|---|---|---|---|---|---|
| Primary | `SovRollupCdkStarterStack-SovRollupComputePrimaryAsgASG448C4328-fVa4yPZ43E36` (1/1/2) | `i-0baeb42dfeb96cb28` | us-east-1a / 10.0.2.21 | `lt-02a6dc4c5d8cfdc99` (latest v44, instance from v39) | `aws_raid_0` | preferred | `DbElected` |
| Secondary | `SovRollupCdkStarterStack-SovRollupComputeSecondaryAsgASG8224C640-PKqRyQODK3bD` (1/0/3) | `i-0c9228ec8b0fa847a` | us-east-1b / 10.0.3.143 | `lt-05cf24139fde86881` (v44, instance v39) | `aws_raid_0` | preferred | `DbElected` |
| Backup | `SovRollupCdkStarterStack-SovRollupComputeBackupNodeAsgASGF12DEAEF-v241AoY1q69R` (1/1/2) | `i-0cc24aae7b0454089` | us-east-1a / 10.0.2.196 | `lt-0e54a3c6b1a4f422e` (v44, instance v39) | `aws_ebs_backup_node_raid_0` | standard (not part of the election) | – |
| Proxy (not migrated) | `…ProxyAsgASGB1045D50-nWkwreNMu6YC` | `i-035616ee95eaccd94` | us-east-1a / 10.0.0.9 | `lt-02bdfa1164fba9ea6` | – | – | – |

- All three rollup nodes use security group `sg-0e90e09ed3975a293`. Aurora accepts 5432 from that SG, and the proxy reaches 12346/3000 through it. **New instances get the same SG from the launch template, so no SG changes are needed.**
- Today's `aws_raid_0` layout: NVMe 1 + 2 in RAID0 at `/mnt/rollup` (ext4, 1.9 TB), NVMe 3 at `/mnt/logs`. The backup node also has EBS `/dev/sdf` (`vol-0b7f605d6d0900a4a`, 1800 GiB gp3, 30k IOPS) at `/mnt/snapshots`.
- Launch template **v44** (the ASGs' current version) differs from **v39** (what the running instances booted from) only in the AMI (`ami-09a93d801db0bafae`, Ubuntu 24.04 arm64) and the Ansible branch (`cdk-deployment-june-2026` → `cdk-deployment-october-2026`).

### 0.3 Sizing check: GO

| | c8gd.12xlarge (today) | i8g.4xlarge (target) | Verdict |
|---|---|---|---|
| vCPU | 48 | **16** | Over 14 days, CloudWatch shows: average ≤ 0.4%, highest hourly average 1.4%, highest 1-min peak **14%** (secondary). That's ≈ 6.7 vCPU, or ≈ 42% of 16. **OK**. Ansible compiles the rollup at boot, so first boot is ~3× slower |
| RAM | 96 GiB | **128 GiB** | More RAM than today |
| Rollup disk | RAID0 2 × 950 GB = 1.9 TB | **1 × 3750 GB**, no RAID | Data = **944 GB** (last snapshot's `FullSnapshotSizeInBytes`), including the ~800 GiB NOMT hashtable (`user_hashtable_buckets: 210000000`). **Fits** |
| Logs disk | 3rd NVMe (950 GB) | new **EBS gp3 on `/dev/sdg`** | Size it in §2.3 |
| Capacity | – | i8g.4xlarge offered in us-east-1a/b/c/d | **OK** |
| vCPU quota | 512 | +16 during each swap (peak 162) | **OK** |

### 0.4 How the no-downtime switch works (SDK behaviour)
- Primary and secondary run with `node_role = "DbElected"` and leader election in Aurora (`leader_timeout` 1 s, `grace_period` 10 s, `heartbeat` 100 ms, which are the defaults because the CDK parameters are empty).
- A `DbElected` node that isn't leader runs as a Postgres-synced replica. When the leader stops heart-beating for > 1 s, a replica takes leadership, **exits**, and is restarted by systemd as the batch producer. `rollup.service` has `Restart=always`, `RestartSec=3`.
- **Reads** never stop. **Writes** pause for about 1 s plus the restart time when the leader node is retired (§4.6).
- A node that is still syncing returns non-200 on `/sequencer/ready`, so node-discovery keeps it out of read routing. Clients never see a stale node.
- The backup node is a *standard* sequencer and isn't part of the election. Swapping it has no client impact.

### 0.5 Strategy (one node at a time)
For each node:
1. Activate a new launch template version (i8g.4xlarge plus an EBS logs volume) and the single-NVMe `DISK_PROFILE`.
2. Scale its ASG from 1 to 2. The new instance boots and Ansible sets up the single NVMe.
3. Restore the latest backup snapshot onto it. It catches up and becomes ready, and the proxy adds it automatically.
4. Detach the old instance from the ASG and stop rollup on it. For the leader, this is the failover.
5. Let it run for a while, then terminate the old instance.

**Order: non-leader preferred node → leader preferred node → backup node.** The backup goes last so its snapshots stay the restore source for the first two.

### 0.6 ⚠ CDK drift
The CDK stack owns the launch templates and SSM documents changed here by hand. Running instances are never touched by a deploy: the ASGs have no rolling-update policy. **But the next `cdk deploy` that modifies those resources reverts them** (for example, bumping `RollupCommitHash` rewrites the values documents). The next launched node would then come up as c8gd.12xlarge, or with `aws_raid_0` on an i8g.4xlarge, which fails at boot with *"Not enough instance store disks for RAID"*. **Port the changes to the CDK code (§7) before the next deploy.** I couldn't find the CDK source repo in the Sovereign-Labs GitHub org; it may be private.

---

## 1. Variables

```bash
export AWS_REGION=us-east-1
export RO=ReadOnlyAccess-313110633477           # read-only checks
export RW=<profile with write access to 313110633477>   # [write] steps
export DOMAIN=relay-devnet.sovereign-labs.xyz
export API=12346
export LT_BASE=44                               # current latest version of the 3 launch templates
export DOC_BASE=10                              # current default version of the 3 values documents
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
If any value differs (someone deployed), update `LT_BASE` / `DOC_BASE` and re-read §0.2.

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
Set `LOGS_GB` (EBS `/dev/sdg`) to at least 2 × the current `/mnt/logs` usage, with a minimum of 100. The runbook defaults to 200.

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

### 2.6 Freeze
Until §5 is done, don't do any of the following:
- `cdk deploy` on `SovRollupCdkStarterStack`
- the stack's `MigrateLiveNodesCommand`
- `render_config.sh` / `run_ansible.sh` on the old nodes. They would read the new single-NVMe `DISK_PROFILE`, and Ansible would fail on hardware that doesn't match it.

Tell dev users when the leader swap (a few seconds of write pause) will happen.

---

## 3. One-time preparation **[write]**

### 3.1 Stop the ASGs from replacing instances on their own
The instance role can call `autoscaling:SetInstanceHealth`, so something on the nodes may mark an instance unhealthy. The ASG would then terminate it, possibly halfway through a restore.
```bash
for g in $P_ASG $S_ASG $B_ASG; do
  aws --profile $RW autoscaling suspend-processes --auto-scaling-group-name $g --scaling-processes ReplaceUnhealthy AZRebalance
done
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG --query 'AutoScalingGroups[].SuspendedProcesses[].ProcessName'
```

### 3.2 New launch template versions (not used until §4.1)
`--source-version 44` copies everything: AMI, user data, SG, instance profile and key pair. Fields given in `--launch-template-data` replace the source's. **`BlockDeviceMappings` is replaced as a whole**, so the root volume (and `/dev/sdf` on the backup) must be repeated.
```bash
LOGS_GB=200
ROOT='{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":64,"VolumeType":"gp3"}}'
LOGS='{"DeviceName":"/dev/sdg","Ebs":{"VolumeSize":'$LOGS_GB',"VolumeType":"gp3","Encrypted":true,"DeleteOnTermination":true}}'
SNAP='{"DeviceName":"/dev/sdf","Ebs":{"VolumeSize":1800,"VolumeType":"gp3","Iops":30000,"Throughput":1000,"Encrypted":true,"DeleteOnTermination":true}}'
newver() {  # $1 = template id, $2 = block device mappings, extra args (e.g. --dry-run)
  aws --profile $RW ec2 create-launch-template-version "${@:3}" --launch-template-id "$1" --source-version $LT_BASE \
    --version-description "i8g.4xlarge single-NVMe migration" \
    --launch-template-data "{\"InstanceType\":\"i8g.4xlarge\",\"BlockDeviceMappings\":[$2]}" \
    --query LaunchTemplateVersion.VersionNumber --output text
}
newver $P_LT "$ROOT,$LOGS" --dry-run; newver $S_LT "$ROOT,$LOGS" --dry-run; newver $B_LT "$ROOT,$SNAP,$LOGS" --dry-run
P_LTV=$(newver $P_LT "$ROOT,$LOGS"); S_LTV=$(newver $S_LT "$ROOT,$LOGS"); B_LTV=$(newver $B_LT "$ROOT,$SNAP,$LOGS")
echo "P_LTV=$P_LTV S_LTV=$S_LTV B_LTV=$B_LTV" | tee -a $LOG
```
Check that only `InstanceType` and `BlockDeviceMappings` changed:
```bash
for x in "$P_LT $P_LTV" "$S_LT $S_LTV" "$B_LT $B_LTV"; do set -- $x
  diff <(aws --profile $RO ec2 describe-launch-template-versions --launch-template-id $1 --versions $LT_BASE --query 'LaunchTemplateVersions[0].LaunchTemplateData' | jq -S 'del(.InstanceType,.BlockDeviceMappings)') \
       <(aws --profile $RO ec2 describe-launch-template-versions --launch-template-id $1 --versions $2       --query 'LaunchTemplateVersions[0].LaunchTemplateData' | jq -S 'del(.InstanceType,.BlockDeviceMappings)') \
  && echo "$1 v$2: only type + disks changed"
done
```
Creating a version doesn't change anything yet: the ASGs still point at v44.

### 3.3 Prepare the new values documents (activated in §4.1)
The values documents store each value base64-encoded: `DISK_PROFILE=$(printf '%s' '<base64>' | base64 -d)`.

| Node | Old profile (base64) | New profile (base64) |
|---|---|---|
| primary, secondary | `aws_raid_0` (`YXdzX3JhaWRfMA==`) | `aws_single_nvme_ebs_logs` (`YXdzX3NpbmdsZV9udm1lX2Vic19sb2dz`): NVMe → `/mnt/rollup`, EBS `/dev/sdg` → `/mnt/logs` |
| backup | `aws_ebs_backup_node_raid_0` (`YXdzX2Vic19iYWNrdXBfbm9kZV9yYWlkXzA=`) | `aws_ebs_backup_node_single_nvme` (`YXdzX2Vic19iYWNrdXBfbm9kZV9zaW5nbGVfbnZtZQ==`): plus EBS `/dev/sdf` → `/mnt/snapshots` and the hourly snapshot cron |

```bash
mkdoc() {  # $1 = document, $2 = old profile, $3 = new profile
  aws --profile $RO ssm get-document --name $1 --document-version $DOC_BASE --document-format JSON --query Content --output text > /tmp/$1.v$DOC_BASE.json
  old=$(printf '%s' "$2" | base64 -w0); new=$(printf '%s' "$3" | base64 -w0)
  [ "$(grep -cF "'$old'" /tmp/$1.v$DOC_BASE.json)" = 1 ] || { echo "STOP: '$old' not found exactly once in $1"; return 1; }
  sed "s|'$old'|'$new'|" /tmp/$1.v$DOC_BASE.json > /tmp/$1.new.json
  diff <(jq . /tmp/$1.v$DOC_BASE.json) <(jq . /tmp/$1.new.json)      # exactly one line: the DISK_PROFILE line
}
mkdoc $P_DOC aws_raid_0 aws_single_nvme_ebs_logs
mkdoc $S_DOC aws_raid_0 aws_single_nvme_ebs_logs
mkdoc $B_DOC aws_ebs_backup_node_raid_0 aws_ebs_backup_node_single_nvme
```

---

## 4. Per-node procedure

Run §4.1–§4.7 three times, finishing each node before starting the next:

| Run | Node | Variables |
|---|---|---|
| 1 | the **non-leader** from §2.2 | `N=S ASG=$S_ASG LT=$S_LT LTV=$S_LTV DOC=$S_DOC OLD=$S_OLD` (or the `P_` set if the secondary leads) |
| 2 | the **leader** | the other `P_` / `S_` set |
| 3 | **backup** | `N=B ASG=$B_ASG LT=$B_LT LTV=$B_LTV DOC=$B_DOC OLD=$B_OLD` |

### 4.1 Activate the new template and disk profile **[write]**
```bash
DOCV=$(aws --profile $RW ssm update-document --name $DOC --content file:///tmp/$DOC.new.json --document-format JSON \
  --document-version '$LATEST' --query DocumentDescription.LatestVersion --output text); echo "${N}_DOCV=$DOCV" | tee -a $LOG
aws --profile $RW ssm update-document-default-version --name $DOC --document-version $DOCV
aws --profile $RW autoscaling update-auto-scaling-group --auto-scaling-group-name $ASG --launch-template LaunchTemplateId=$LT,Version=$LTV
# check
aws --profile $RO ssm get-document --name $DOC --document-format JSON --query Content --output text | grep -o "DISK_PROFILE=[^|]*"
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG --query 'AutoScalingGroups[0].LaunchTemplate'
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
If the bootstrap failed, the last Ansible task in the log explains why. The usual cause is *"Not enough instance store disks for RAID"*: the new `DISK_PROFILE` wasn't active, so redo §4.1. To discard the instance, remove its protection (`set-instance-protection --no-protected-from-scale-in`), then run `aws --profile $RW autoscaling terminate-instance-in-auto-scaling-group --instance-id $NEW --should-decrement-desired-capacity`.

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

The stack output `RestoreFromVolumeInstructions` points at `/tmp/restore-from-volume.sh`. That path is out of date: the script lives at `/home/ubuntu/restore-from-snapshot.sh`.

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
For every run, remove the old instance from the ASG but keep it running as a rollback option. Then stop rollup on it:
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
- Keep `$LOG`.

---

## 6. Rollback

| When | Action |
|---|---|
| Before §4.6 for a node | Remove the new instance: `set-instance-protection --no-protected-from-scale-in`, then `terminate-instance-in-auto-scaling-group --instance-id $NEW --should-decrement-desired-capacity`. Point the ASG back to v44: `update-auto-scaling-group --launch-template LaunchTemplateId=$LT,Version=$LT_BASE`. Restore the document: `update-document-default-version --name $DOC --document-version $DOC_BASE`. The old node was never touched. |
| After §4.6, before §4.7 | On `$OLD`: `sudo systemctl unmask rollup && sudo systemctl enable --now rollup`. It rejoins as a `DbElected` replica; wait for `ready=200`. Re-attach it: `aws autoscaling attach-instances --auto-scaling-group-name $ASG --instance-ids $OLD` (max size allows 2). Then handle the new instance as in the first row. For the backup, also restore the old cron line: `0 * * * * /home/ubuntu/snapshot-rollup-data.sh` in `crontab -u ubuntu`. |
| New leader crash-looping after run 2 | In sessions on both new preferred nodes: `sudo systemctl disable --now rollup && sudo systemctl mask rollup`. Then on the old leader: `sudo systemctl unmask rollup && sudo systemctl enable --now rollup`. It's `DbElected` and takes leadership about 1 s after the others stop heartbeating. Diagnose with `journalctl -u rollup`, `journalctl -k \| grep -i oom`, `df -h`. |

---

## 7. Port the changes to the CDK code (before the next `cdk deploy`)
- `PrimaryLaunchTemplate`, `SecondaryLaunchTemplate`, `BackupNodeLaunchTemplate`: `instanceType` → `i8g.4xlarge`. Add the `/dev/sdg` gp3 logs volume (`LOGS_GB`, encrypted, delete on termination). Keep `/dev/sdf` on the backup.
- Runtime config values: `DISK_PROFILE` = `aws_single_nvme_ebs_logs` for primary and secondary, `aws_ebs_backup_node_single_nvme` for backup.
- Fix the `RestoreFromVolumeInstructions` output (`/home/ubuntu/restore-from-snapshot.sh <snap-id>`).
- `cdk diff` should show only these changes: 3 launch templates, 3 ASG `LaunchTemplate.Version` values, 3 SSM documents. It must not show any instance replacement.

---

## 8. Time estimate

| Step | Duration |
|---|---|
| §2 pre-flight + §3 preparation | 1 h |
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
- The CDK code has been updated (§7), so the next `cdk deploy` doesn't revert the change.
