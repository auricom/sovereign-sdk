# Runbook: migrate the relay-mainnet sequencers (`SovRollupCdkStarterStack`, account `751545121805`) from c8gd.12xlarge to i8g.4xlarge with no downtime

> **Status (2026-11-20): written from the CDK repo and the devnet migration. Not yet checked against the mainnet account.**
> - No AWS profile for `751545121805` existed when this was written. Every account-specific value (ASGs, instances, AZs, snapshot volume, template and document versions) is therefore **discovered by the commands in §1**, not hard-coded. The sizing go/no-go (§2.2) uses mainnet's own CloudWatch data.
> - Fill in the `Result:` lines as you go, like the devnet runbook does.
> - **Rehearsal:** this is the same procedure as [`ec2-migration-plan.md`](ec2-migration-plan.md) (devnet, account `313110633477`). Run mainnet only after devnet is fully migrated (§2.8).
> - Commands marked **[write]** need a profile with write access and are run by a human. Every other command is read-only.
> - Node access is through **SSM Session Manager** (`aws ssm start-session`); there is no direct SSH.

---

## 0. What's different from devnet

| | Devnet | **Mainnet** |
|---|---|---|
| Account / region | `313110633477` / us-east-1 | **`751545121805`** / us-east-1 (`deployments/relay/mainnet/config.json`) |
| Users | internal + Relay testing | **Relay production traffic** |
| Public endpoint | `relay-devnet.sovereign-labs.xyz` | **`rpc.chain.relay.link`** (answers `/sequencer/ready` 200 and `/ledger/slots/latest` on 2026-11-20, slot ≈ 6.79 M) |
| CDK deployment | `npm run deploy relay devnet` | **`npm run deploy relay mainnet`**, same branch `theodore/inplace-config-updates` (PR #92), same stack name |
| Migration CDK change | patches 0001 (code) + 0002 (devnet param) | **code already merged** (PR #93 into the deploy branch, `bae2097`). Mainnet needs **one parameter**: [`mainnet/0001-Move-relay-mainnet-rollup-nodes-to-i8g.4xlarge.patch`](ec2-migration-cdk/mainnet/) |
| Rollup commit / Ansible branch (repo, 2026-11-20) | `8df3d0bc…` / `cdk-deployment-november-2026` | **`9d5cada5…`** / `cdk-deployment-november-2026`. That branch has both single-NVMe profiles (`aws_single_nvme_ebs_logs`, `aws_ebs_backup_node_single_nvme`), and its `restore-from-snapshot.sh` keeps volumes passed by ID |
| Aurora version parameter | `AuroraEngineVersion: 17.9` | **not set** (template default `17.5`). Check the live version in §2.7 |
| Other mainnet-only parameters | – | `BridgedGasTokenId`, `ContractDeploymentRestrictionsEnabled: true` + whitelist, `DatabaseDeletionProtection: True`, `Otel*` (metrics to `otel.relay.link`). **None is touched by the migration** |
| Soak between nodes | none | **an observation gate after run 1 and after run 2** (§4.5): real traffic is the only true load test |
| Failover communication | dev users | **Relay**, with a scheduled window (§2.9) |

Everything else is identical: the stack layout (`compute-infrastructure.ts`), the three ASGs (min 1 / desired 1, no rolling-update policy), the SSM render/values documents, `DbElected` leader election in Aurora, the proxy's node-discovery routing, and the hourly EBS snapshots of the backup node. The background is in the devnet runbook:
- §0.3: how the no-downtime switch works;
- §0.5: what goes through CDK and what doesn't (no drift);
- §0.6: how a new i8g.4xlarge instance gets installed (boot chain and disk layout).

**What the mainnet CDK change does** (resolved from the synthesized template at `bae2097` with `deployments/relay/mainnet/parameters.json`): adding `RollupNodeInstanceType=i8g.4xlarge` changes **exactly 6 resources**:

| Resource | Before | After |
|---|---|---|
| Primary / Secondary launch templates | `c8gd.12xlarge`, `/dev/sda1` | `i8g.4xlarge`, `/dev/sda1` + `/dev/sdg` (64 GiB gp3 logs) |
| BackupNode launch template | `c8gd.12xlarge`, `/dev/sda1`, `/dev/sdf` | `i8g.4xlarge`, `/dev/sda1`, `/dev/sdf`, `/dev/sdg` |
| Values documents primary / secondary | `DISK_PROFILE=aws_raid_0` | `aws_single_nvme_ebs_logs` |
| Values document backup | `aws_ebs_backup_node_raid_0` | `aws_ebs_backup_node_single_nvme` |

The 3 ASGs follow (`LaunchTemplate.Version` only). `npm test` passes 108/108 with the patch.

---

## 1. Access and variables

### 1.1 AWS profiles for the mainnet account
Add to `~/.aws/config` (same SSO portal as devnet). Use the role names the portal offers for `751545121805`. List them with `aws sso list-account-roles --account-id 751545121805 --access-token …`, or check the portal.
```ini
[profile ReadOnlyAccess-751545121805]
sso_start_url = https://celestiaorg.awsapps.com/start/
sso_region = eu-west-1
sso_account_id = 751545121805
sso_role_name = ReadOnlyAccess
region = us-east-1

[profile AdministratorAccess-751545121805]
sso_start_url = https://celestiaorg.awsapps.com/start/
sso_region = eu-west-1
sso_account_id = 751545121805
sso_role_name = AdministratorAccess
region = us-east-1
```
```bash
aws sso login --profile ReadOnlyAccess-751545121805
aws sts get-caller-identity --profile ReadOnlyAccess-751545121805 --query Account --output text      # 751545121805
```
- **Read-only** is enough for §2 and all checks.
- **Write steps** and `ssm start-session` need a role with `ssm:StartSession`, ASG/EC2 write, and `sts:AssumeRole` on the CDK deploy role `cdk-hnb659fds-deploy-role-751545121805-us-east-1` (for §3).
- You also need the Session Manager plugin, installed as for devnet.

### 1.2 Variables, discovered from the stack
Run once in a fresh shell. It writes the starting state to `$LOG`; in later shells, re-run only the first 6 lines and `source $LOG`.
```bash
export AWS_REGION=us-east-1
export RO=ReadOnlyAccess-751545121805
export RW=AdministratorAccess-751545121805
export STACK=SovRollupCdkStarterStack
export DOMAIN=rpc.chain.relay.link
export LOG=~/i8g-migration-relay-mainnet.env

out() { aws --profile $RO cloudformation describe-stacks --stack-name $STACK --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }
asg() { aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names "$1" --query "AutoScalingGroups[0].$2" --output text; }
az()  { aws --profile $RO ec2 describe-instances --instance-ids "$1" --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' --output text; }

P_ASG=$(out PrimaryAutoScalingGroupName); S_ASG=$(out SecondaryAutoScalingGroupName); B_ASG=$(out BackupNodeAutoScalingGroupName)
for n in P S B; do a=${n}_ASG
  [ "$(asg ${!a} 'length(Instances)')" = 1 ] || echo "STOP: ${!a} does not have exactly one instance"
  eval "${n}_LT=$(asg ${!a} LaunchTemplate.LaunchTemplateId) ${n}_OLD=$(asg ${!a} 'Instances[0].InstanceId')"
done
for n in P S B; do o=${n}_OLD; eval "AZ_${n}=$(az ${!o})"; done
P_DOC=$STACK-runtime-config-values-primary; S_DOC=$STACK-runtime-config-values-secondary; B_DOC=$STACK-runtime-config-values-backup
SNAP_VOL=$(aws --profile $RO ec2 describe-volumes --filters Name=attachment.instance-id,Values=$B_OLD Name=attachment.device,Values=/dev/sdf --query 'Volumes[0].VolumeId' --output text)
LT_BASE=$(aws --profile $RO ec2 describe-launch-templates --launch-template-ids $P_LT --query 'LaunchTemplates[0].LatestVersionNumber' --output text)
DOC_BASE=$(aws --profile $RO ssm describe-document --name $P_DOC --query Document.DefaultVersion --output text)
{ echo "# relay mainnet i8g migration, started $(date -u +%FT%TZ)"
  for v in P_ASG S_ASG B_ASG P_LT S_LT B_LT P_OLD S_OLD B_OLD AZ_P AZ_S AZ_B P_DOC S_DOC B_DOC SNAP_VOL LT_BASE DOC_BASE; do echo "$v=${!v}"; done
} | tee $LOG
set -a; source $LOG; set +a
```
- Every value must be non-empty and not `None`.
- `SNAP_VOL` is the backup node's `/dev/sdf`, the source of the restore volumes.
- Copy the output into §2.1.

---

## 2. Pre-flight (read-only), go/no-go

### 2.1 Inventory
```bash
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG \
  --query 'AutoScalingGroups[].{asg:AutoScalingGroupName,desired:DesiredCapacity,min:MinSize,max:MaxSize,ltv:LaunchTemplate.Version,suspended:SuspendedProcesses[].ProcessName,inst:Instances[].[InstanceId,LifecycleState,HealthStatus,LaunchTemplate.Version]}' --output json
aws --profile $RO ec2 describe-instances --filters Name=tag:Stack,Values=$STACK Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,Placement.AvailabilityZone,PrivateIpAddress,LaunchTime,Tags[?Key==`ASGType`]|[0].Value]' --output table
aws --profile $RO cloudformation describe-stacks --stack-name $STACK --query 'Stacks[0].[StackStatus,LastUpdatedTime]' --output text
aws --profile $RO ssm describe-instance-information --query 'InstanceInformationList[].[InstanceId,PingStatus,PlatformVersion]' --output table
```
Expect:
- the three rollup instances to be `c8gd.12xlarge`;
- max size ≥ 2 on every ASG (devnet: primary 2, secondary 3, backup 2), with nothing suspended;
- the stack `UPDATE_COMPLETE`;
- every node `Online` in SSM.

**Result:** _fill in: ASG names, instance IDs, AZ / IP, launch template IDs and versions (`LT_BASE`), values document version (`DOC_BASE`), stack last update._

| Role | ASG | Instance | AZ / IP | Launch template (latest / instance's) | `DISK_PROFILE` | Sequencer | Election |
|---|---|---|---|---|---|---|---|
| Primary | | | | | `aws_raid_0` | preferred | `DbElected` |
| Secondary | | | | | `aws_raid_0` | preferred | `DbElected` |
| Backup | | | | | `aws_ebs_backup_node_raid_0` | standard | – |

### 2.2 Sizing go/no-go (mainnet traffic is the deciding factor)
Devnet passed easily (1-min peak 14% of 48 vCPU). **Mainnet carries Relay's production traffic, so decide from its own numbers.** The target has a third of the vCPUs (16 instead of 48), more RAM (128 vs 96 GiB) and more local disk (3.75 TB vs 1.9 TB RAID0).

**CPU, last 14 days, per node:**
```bash
S=$(date -u -d '-14 days' +%FT%TZ); E=$(date -u +%FT%TZ)
for n in P S B; do o=${n}_OLD
  aws --profile $RO cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=${!o} \
    --statistics Average Maximum --period 3600 --start-time $S --end-time $E --output json \
  | jq -r --arg n $n '[.Datapoints[]] | "\($n): hours=\(length) avg=\((map(.Average)|add/length*100|floor)/100)% max_hourly_avg=\(map(.Average)|max)% max_peak=\(map(.Maximum)|max)% -> on 16 vCPU: max_hourly_avg x3=\(map(.Average)|max*3)% peak x3=\(map(.Maximum)|max*3)%"'
done
```
**GO** if, on the primary and secondary, `max_hourly_avg x3` ≤ **50%** and `peak x3` ≤ **80%**. If not, it's **NO-GO** for i8g.4xlarge. The next step up, i8g.8xlarge (32 vCPU, 2 NVMe), would need a new disk layout in the CDK and Ansible: neither `aws_raid_0` (needs 3 disks) nor the single-NVMe profiles fit 2 disks.

**Memory:** on each node, `free -g` and `ps -o rss= -C <rollup binary> | awk '{s+=$1} END {print s/1048576 " GiB"}'` (the binary name is in `systemctl cat rollup` → `rollup-manager.sh`). It has to fit in 128 GiB with room for the page cache; today it fits in 96 GiB.

**Network:** i8g.4xlarge is an "up to 25 Gbps" type; check its sustained baseline against the c8gd's peak:
```bash
aws --profile $RO ec2 describe-instance-types --instance-types i8g.4xlarge c8gd.12xlarge \
  --query 'InstanceTypes[].[InstanceType,NetworkInfo.NetworkPerformance,NetworkInfo.NetworkCards[0].BaselineBandwidthInGbps,EbsInfo.EbsOptimizedInfo.BaselineThroughputInMBps]' --output table
for n in P S; do o=${n}_OLD
  aws --profile $RO cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkOut --dimensions Name=InstanceId,Value=${!o} \
    --statistics Sum --period 300 --start-time $S --end-time $E --output json \
  | jq -r --arg n $n '"\($n): peak NetworkOut ≈ \(([.Datapoints[].Sum]|max)*8/300/1e9) Gbps (busiest 5-min average)"'
done
```
**GO** if each node's peak is below 50% of the i8g.4xlarge baseline.

**Disk:** the backup mirrors all rollup data onto a 1800 GiB EBS volume, so the data already fits in 1.8 TB, well under the new 3.75 TB NVMe. Record the size anyway, because restore and catch-up times scale with it:
```bash
aws --profile $RO ec2 describe-snapshots --owner-ids self --filters Name=volume-id,Values=$SNAP_VOL Name=tag:AutomatedBackup,Values=true Name=status,Values=completed \
  --query 'sort_by(Snapshots,&StartTime)[-1].[SnapshotId,StartTime,FullSnapshotSizeInBytes]' --output text
```

**Capacity and quota:**
```bash
aws --profile $RO ec2 describe-instance-type-offerings --location-type availability-zone --filters Name=instance-type,Values=i8g.4xlarge --query 'InstanceTypeOfferings[].Location' --output text   # must include $AZ_P $AZ_S $AZ_B
aws --profile $RO service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A --query Quota.Value --output text           # ≥ current vCPUs + 16
```

**Result:** _fill in the CPU, memory, network, data size and capacity numbers, then GO / NO-GO._

### 2.3 Which node is the leader
In a session on each preferred node (`aws --profile $RW ssm start-session --target $P_OLD`, then `$S_OLD`):
```bash
curl -s localhost:12346/sequencer/role; echo                        # "BatchProducer" = leader, "PgSyncReplica" = replica
curl -s -o /dev/null -w 'ready=%{http_code}\n' localhost:12346/sequencer/ready
curl -s localhost:12346/ledger/slots/latest | jq -c '{number,state_root}'
```
**Result:** `LEADER=` _P or S_. **The other one is migrated first.**

### 2.4 Logs volume size
On each old node: `df -h /mnt/logs; sudo du -sh /mnt/logs/* | sort -h | tail`. New nodes get a **64 GiB** `/dev/sdg` (`ROLLUP_LOGS_VOLUME_GIB`).
- If `/mnt/logs` uses more than ~30 GiB on any node, raising it means a code change in `lib/compute-infrastructure.ts`, which also affects devnet's next deploy (new instances only).
- Mainnet probably logs more than devnet, so check this one carefully.

**Result:** _fill in._

### 2.5 Same rollup version everywhere
```bash
aws --profile $RO ssm get-document --name $P_DOC --query Content --output text | jq -r '.mainSteps[0].inputs.runCommand[]' \
  | grep -E '^(ROLLUP_COMMIT_HASH|ANSIBLE_BRANCH)=' | sed -E "s/^([A-Z_]+)=.*'([^']+)'.*/\1 \2/" | while read k v; do echo "$k=$(printf '%s' "$v" | base64 -d)"; done
```
On each old node:
```bash
grep -E '^(rollup_commit_hash|disk_profile|rollup_node_role|rollup_sequencer_kind|rollup_node_id)' /home/ubuntu/runtime_vars.yaml
systemctl show rollup -p ActiveEnterTimestamp
```
- Every node must run the commit in the live SSM values document. That's `9d5cada50cfcd82ad727bf385ccf68f16d501d26` if `82b426f` is deployed (§2.7); new nodes build that commit.
- **If an old node runs a different commit, stop.** Its `ActiveEnterTimestamp` must be after the SSM document version that set that commit (`aws ssm list-document-versions --name $P_DOC`).

**Result:** _fill in._

### 2.6 Snapshots are healthy
```bash
aws --profile $RO ec2 describe-snapshots --owner-ids self --filters Name=volume-id,Values=$SNAP_VOL Name=tag:AutomatedBackup,Values=true \
  --query 'reverse(sort_by(Snapshots,&StartTime))[:5].[SnapshotId,StartTime,State,FullSnapshotSizeInBytes]' --output table
```
- The latest `completed` snapshot must be < 2 h old.
- On the backup node, `sudo tail -40 /mnt/logs/rollup-snapshot.log` should show `=== Snapshot complete ===` every hour, with no `ERROR`.
- Only ~5 snapshots exist: the retention keeps one near 0 h, 6 h, 1 d, 1 w and 1 month.

**Result:** _fill in._

### 2.7 The repo must match the live stack, with no drift
Mainnet may have **undeployed commits** on the deploy branch. On 2026-11-20 these mainnet-relevant commits sit on `theodore/inplace-config-updates`; check whether each is deployed:
- `82b426f`: mainnet rollup `9d5cada5…`, Ansible `cdk-deployment-november-2026`, Relay rate-limit exemption;
- `47b4a64`: proxy replacement race fix in the proxy user data, which **changes the proxy launch template**;
- `d0ef2ae`, `bae2097` (PR #93): new parameters with defaults, which resolve to no change for mainnet.

**a) Parameters: live vs repo.**
```bash
# in the CDK repo, on theodore/inplace-config-updates (no migration patch yet)
aws --profile $RO cloudformation describe-stacks --stack-name $STACK --query 'Stacks[0].Parameters' --output json \
  | jq -S 'map({(.ParameterKey): .ParameterValue}) | add' > /tmp/mainnet-live-params.json
jq -S '.parameters | map_values(tostring)' deployments/relay/mainnet/parameters.json > /tmp/mainnet-repo-params.json
jq -rn --slurpfile l /tmp/mainnet-live-params.json --slurpfile r /tmp/mainnet-repo-params.json \
  '$r[0] | to_entries[] | select($l[0][.key] != .value) | "\(.key): live=\($l[0][.key]) repo=\(.value)"'
```
No output means every repo parameter is live.

**b) Code: `npx cdk diff`** with an admin profile, from the deploy branch: `AWS_PROFILE=$RW npx cdk diff SovRollupCdkStarterStack`. This compares templates, not parameter values. It lists code changes not yet deployed to mainnet, for example:
- the `AuroraEngineVersion` / `RollupNodeInstanceType` parameters and their `Ref` / `Fn::If`;
- the proxy user data from `47b4a64`.

**c) CloudFormation drift:**
```bash
DID=$(aws --profile $RO cloudformation detect-stack-drift --stack-name $STACK --query StackDriftDetectionId --output text)
sleep 60; aws --profile $RO cloudformation describe-stack-drift-detection-status --stack-drift-detection-id $DID --query '[DetectionStatus,StackDriftStatus,DriftedStackResourceCount]' --output text
aws --profile $RO cloudformation describe-stack-resource-drifts --stack-name $STACK --stack-resource-drift-status-filters MODIFIED DELETED \
  --query 'StackResourceDrifts[].[LogicalResourceId,PropertyDifferences[].[PropertyPath,ExpectedValue,ActualValue]]' --output json
aws --profile $RO rds describe-db-clusters --query 'DBClusters[].[DBClusterIdentifier,EngineVersion]' --output text
```
- Devnet had exactly one drift: Aurora auto-upgraded from 17.5 to 17.9. Expect the same here.
- If the cluster reports a version other than 17.5, set `"AuroraEngineVersion": "<that version>"` in `deployments/relay/mainnet/parameters.json` and include it in Deploy 0 (§3.1).
- Anything else that's drifted must be understood and resolved first.

**Result:** _fill in a), b), c), and whether Deploy 0 is needed._

### 2.8 Devnet rehearsal done
- The devnet runbook's definition of done is met.
- Its real durations are copied into §6: bootstrap, restore and catch-up times, and the failover's write gap measured by the probe.
- Any surprise found on devnet is fixed here first.

**Result:** _fill in._

### 2.9 Freeze and communication
- **Relay:** agree a window for run 2 (leader failover, a few seconds of write pause) and send them the plan. New nodes report to their OTEL endpoint (`otel.relay.link`) under new instance IDs (`relay-chain-prod-<type>_<instance-id>`); tell them so their dashboards follow.
- **Freeze until §5 is done:**
  - no mainnet deploy other than §3 (and §5's optional proxy rollout);
  - no deploy from `master`;
  - no `MigrateLiveNodesCommand`;
  - no `render_config.sh` / `run_ansible.sh` on old nodes after §3. They would read the single-NVMe `DISK_PROFILE` and fail on the c8gd hardware.

**Go/no-go:** _GO only if 2.2 is GO, 2.5 passes, 2.6 passes, 2.7 is understood, 2.8 is done, and Relay has agreed a window._

---

## 3. Deploy the CDK change **[write, admin profile]**

Same repo and branch as devnet: `Sovereign-Labs/sov-rollup-cdk-starter`, **`theodore/inplace-config-updates`** (PR #92). **Never deploy from `master`.** `npm run deploy relay mainnet` asks you to type `yes` (`production: true`).

```bash
git clone git@github.com:Sovereign-Labs/sov-rollup-cdk-starter.git && cd sov-rollup-cdk-starter
git switch theodore/inplace-config-updates && git log -3 --oneline     # a9b2642 or newer (same tree as bae2097)
npm ci && npx tsc --noEmit -p . && npm test                             # 108 passed
export AWS_PROFILE=$RW
npm run validate relay mainnet
```

### 3.1 Deploy 0: bring mainnet up to the branch, without the migration (only if §2.7 found differences)
This gets every non-migration change live first, so the migration change set stays tiny. It probably **rolls the mainnet proxy** (the `47b4a64` user data change, and a newer Ubuntu AMI). The proxy ASG replaces the instance one for one, behind the same EIP. **Do it in an agreed quiet window with Relay**, and watch the probe from §4.6.
```bash
# only if §2.7 c) needs it: add "AuroraEngineVersion": "<live version>" to deployments/relay/mainnet/parameters.json
npm run deploy relay mainnet -- --no-execute --change-set-name pre-migration-sync
aws cloudformation describe-change-set --stack-name $STACK --change-set-name pre-migration-sync \
  --query 'Changes[].ResourceChange.{action:Action,id:LogicalResourceId,type:ResourceType,replacement:Replacement,scope:join(`,`,Scope),details:Details[].Target.Name}' --output table
```
Every row must trace back to a cause found in §2.7:
- undeployed parameter: SSM values documents (`Content`), launch templates;
- proxy user data or AMI: proxy launch template and proxy ASG;
- Aurora: `SovRollupAuroraClusterC528DD46` `EngineVersion`;
- `CDKMetadata`.

**Stop** if any row says `Replacement: True` or `Remove`, or touches an ASG's `MinSize`/`MaxSize`/`DesiredCapacity`, or a resource you can't explain; drop it with `aws cloudformation delete-change-set --stack-name $STACK --change-set-name pre-migration-sync`. Otherwise:
```bash
aws cloudformation execute-change-set --stack-name $STACK --change-set-name pre-migration-sync
aws cloudformation wait stack-update-complete --stack-name $STACK
```
- Commit the `AuroraEngineVersion` change, if any, to the branch.
- Then re-run §2.7 a) and c): no parameter differences, `IN_SYNC 0`.
- If the SSM documents changed (a new rollup commit), the **running** nodes don't pick it up on their own. Their in-place update (`render_config.sh` + `run_ansible.sh`) is a separate operation, and §2.5 must pass again before §3.2.

### 3.2 Switch to the migration branch
The change is pushed as branch **`i8g-4xlarge-mainnet`** (`1e073d7`, on top of `a9b2642`). Its only diff against the deploy branch adds `"RollupNodeInstanceType": "i8g.4xlarge"` to `deployments/relay/mainnet/parameters.json`. The same change is also kept as a patch in this fork: [`docs/ec2-migration-cdk/mainnet/`](ec2-migration-cdk/mainnet/).
```bash
git fetch origin && git switch i8g-4xlarge-mainnet
git diff origin/theodore/inplace-config-updates --stat                  # only deployments/relay/mainnet/parameters.json, 1 line added
npm test                                                                # 108 passed
```
If Deploy 0 committed an `AuroraEngineVersion` change to the deploy branch, rebase first: `git rebase origin/theodore/inplace-config-updates`.

Open the PR from `i8g-4xlarge-mainnet` against `theodore/inplace-config-updates` and get it reviewed. Deploy from it, and merge after the deploy succeeds.

The deploy branch history contains `f8fb636` and its revert `a9b2642`. `f8fb636` was this change, pushed to the deploy branch by mistake on 2026-11-20 and reverted minutes later; the branch's tree is identical to `bae2097`. Don't be surprised to see both commits in the log.

### 3.3 Preview and deploy
```bash
npm run deploy relay mainnet -- --no-execute --change-set-name i8g-4xlarge-migration
aws cloudformation describe-change-set --stack-name $STACK --change-set-name i8g-4xlarge-migration \
  --query 'Changes[].ResourceChange.{action:Action,id:LogicalResourceId,type:ResourceType,replacement:Replacement,scope:join(`,`,Scope),details:Details[].Target.Name}' --output table
```
Expected, all `Modify`, never `Replacement: True`. Devnet's change set (2026-11-19) had exactly this shape, plus the proxy rows that Deploy 0 has already taken care of here:

| Rows | Detail |
|---|---|
| `SovRollupComputePrimaryLaunchTemplate…`, `…SecondaryLaunchTemplate…`, `…BackupNodeLaunchTemplate…` | `LaunchTemplateData`: `i8g.4xlarge`, `/dev/sdg` |
| `SovRollupComputePrimaryAsg…`, `…SecondaryAsg…`, `…BackupNodeAsg…` | `LaunchTemplate` only (version). No rolling-update policy, so **no instance is replaced**. Must not list `MinSize`/`MaxSize`/`DesiredCapacity` |
| `SovRollupComputeRuntimeConfigValuesPrimary…`, `…Secondary…`, `…Backup…` | `Content` (`DISK_PROFILE`) |
| `CDKMetadata` | harmless |
| Proxy launch template + ASG | **only** if Canonical published a newer Ubuntu AMI since Deploy 0. If so, it's a proxy roll, so stop and run it as its own quiet-window deploy first |

Anything else means **stop**: `aws cloudformation delete-change-set --stack-name $STACK --change-set-name i8g-4xlarge-migration`. Otherwise:
```bash
aws cloudformation execute-change-set --stack-name $STACK --change-set-name i8g-4xlarge-migration
aws cloudformation wait stack-update-complete --stack-name $STACK
```

### 3.4 Check the result
```bash
for lt in $P_LT $S_LT $B_LT; do
  aws --profile $RO ec2 describe-launch-template-versions --launch-template-id $lt --versions '$Latest' \
    --query 'LaunchTemplateVersions[0].[LaunchTemplateId,VersionNumber,LaunchTemplateData.InstanceType,join(`,`,LaunchTemplateData.BlockDeviceMappings[].DeviceName)]' --output text
done                                  # i8g.4xlarge, /dev/sda1,/dev/sdg (backup: /dev/sda1,/dev/sdf,/dev/sdg)
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $P_ASG $S_ASG $B_ASG \
  --query 'AutoScalingGroups[].[AutoScalingGroupName,LaunchTemplate.Version,DesiredCapacity,Instances[0].InstanceId,Instances[0].InstanceType]' --output text
                                      # ASG version = latest; instances still the old c8gd ones; desired 1
for d in $P_DOC $S_DOC $B_DOC; do printf '%s: ' $d
  aws --profile $RO ssm get-document --name $d --query Content --output text | jq -r '.mainSteps[0].inputs.runCommand[]' \
    | grep '^DISK_PROFILE=' | sed -E "s/.*'([^']+)'.*/\1/" | base64 -d; echo
done
```
Record the versions with `echo "P_LTV=<n> S_LTV=<n> B_LTV=<n>" | tee -a $LOG`, then re-run §2.7 c): `IN_SYNC 0`.
- From now on, any instance these ASGs launch is an i8g.4xlarge with the single-NVMe layout, including an unplanned replacement.
- The `SovRollupCdkStarterStack-runtime-config-drift` alarm fires for the old nodes until they're replaced. That's expected.

---

## 4. Per-node procedure

Identical to devnet §4, with the variables from §1.2 and the **observation gates** below. The devnet runbook has more detail on each step.

### 4.0 Preparation, once **[write]**
**a) Pause ASG self-healing** (not a stack property, so no drift; resumed in §5):
```bash
for g in $P_ASG $S_ASG $B_ASG; do aws --profile $RW autoscaling suspend-processes --auto-scaling-group-name $g --scaling-processes ReplaceUnhealthy AZRebalance; done
```

**b) Disable the old backup's snapshot cron.** A day or two without snapshots is acceptable; it also freezes the restore source.
```bash
aws --profile $RW ssm start-session --target $B_OLD
sudo crontab -u ubuntu -e        # comment out:  0 * * * * /home/ubuntu/snapshot-rollup-data.sh
sudo crontab -u ubuntu -l; pgrep -af snapshot-rollup-data   # line commented, no run in progress
```

**c) Pre-warm one restore volume per AZ in use**, from the latest completed snapshot, all in parallel. Volumes are AZ-locked, and the restore script keeps a volume passed by ID, so one volume serves several nodes in its AZ, one at a time.
```bash
SNAP=$(aws --profile $RO ec2 describe-snapshots --owner-ids self \
  --filters Name=volume-id,Values=$SNAP_VOL Name=tag:AutomatedBackup,Values=true Name=status,Values=completed \
  --query 'sort_by(Snapshots,&StartTime)[-1].SnapshotId' --output text); echo "SNAP=$SNAP" | tee -a $LOG
for z in $(printf '%s\n' $AZ_P $AZ_S $AZ_B | sort -u); do
  v=$(aws --profile $RW ec2 create-volume --snapshot-id $SNAP --availability-zone $z \
        --volume-type gp3 --iops 16000 --throughput 1000 --volume-initialization-rate 300 \
        --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=i8g-migration-restore-$z},{Key=Purpose,Value=i8g-migration-restore},{Key=SourceSnapshot,Value=$SNAP}]" \
        --query VolumeId --output text)
  echo "VOL_${z//-/_}=$v" | tee -a $LOG
done
set -a; source $LOG; set +a
VOLS=$(grep -o '^VOL_[a-z0-9_]*=vol-[0-9a-f]*' $LOG | cut -d= -f2 | sort -u | xargs)
aws --profile $RO ec2 wait volume-available --volume-ids $VOLS
aws --profile $RO ec2 describe-volume-status --volume-ids $VOLS \
  --query 'VolumeStatuses[].[VolumeId,AvailabilityZone,InitializationStatusDetails.Progress,InitializationStatusDetails.EstimatedTimeToCompleteInSeconds]' --output table   # repeat until 100
```
The initialization time scales with `FullSnapshotSizeInBytes` (§2.2): about 1 h per TB at 300 MiB/s.

**d) Per-run variables.** In every shell: re-export the first 6 lines of §1.2, `set -a; source $LOG; set +a`, then define:
```bash
setrun() { local az v; case $1 in
  P) N=P ASG=$P_ASG LTV=$P_LTV OLD=$P_OLD az=$AZ_P NEW=${P_NEW:-} ;;
  S) N=S ASG=$S_ASG LTV=$S_LTV OLD=$S_OLD az=$AZ_S NEW=${S_NEW:-} ;;
  B) N=B ASG=$B_ASG LTV=$B_LTV OLD=$B_OLD az=$AZ_B NEW=${B_NEW:-} ;;
esac; v="VOL_${az//-/_}"; VOL=${!v}
export N ASG LTV OLD VOL NEW; echo "N=$N ASG=$ASG LTV=$LTV OLD=$OLD AZ=$az VOL=$VOL NEW=$NEW"; }
```

| Run | Node | Call | Gate after the run |
|---|---|---|---|
| 1 | the non-leader | `setrun P` if `LEADER=S`, `setrun S` if `LEADER=P` | **Gate 1** (§4.5) |
| 2 | the leader | the other one | **Gate 2** (§4.5). §4.6 is the failover, inside the window agreed with Relay |
| 3 | backup | `setrun B` | none (no client traffic) |

Check the `setrun` echo line before every write command. Two nodes in the same AZ share one volume, so it must be `available` again before the second restore.

### 4.1 Confirm the ASG launches the new template
```bash
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG --query 'AutoScalingGroups[0].[LaunchTemplate.Version,DesiredCapacity,length(Instances)]' --output text   # $LTV  1  1
```

### 4.2 Launch the new instance and protect it **[write]**
```bash
aws --profile $RW autoscaling set-desired-capacity --auto-scaling-group-name $ASG --desired-capacity 2
watch -n15 "aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG --query 'AutoScalingGroups[0].Instances[].[InstanceId,LifecycleState,HealthStatus,LaunchTemplate.Version]' --output text"
NEW=<instance id whose LaunchTemplate.Version = $LTV>; echo "${N}_NEW=$NEW" | tee -a $LOG
aws --profile $RW autoscaling set-instance-protection --auto-scaling-group-name $ASG --instance-ids $NEW --protected-from-scale-in
aws --profile $RW ec2 modify-instance-attribute --instance-id $NEW --disable-api-termination
aws --profile $RO ec2 describe-instances --instance-ids $NEW --query 'Reservations[0].Instances[0].{type:InstanceType,az:Placement.AvailabilityZone,ip:PrivateIpAddress,disks:BlockDeviceMappings[].DeviceName}'
```
- Expect `i8g.4xlarge`, the same AZ as the old node, and `/dev/sda1`, `/dev/sdg` (plus `/dev/sdf` on the backup).
- `InsufficientInstanceCapacity` shows in `aws autoscaling describe-scaling-activities --auto-scaling-group-name $ASG --max-items 3`: retry later.

### 4.3 Watch the bootstrap and check the disk layout
```bash
aws --profile $RW ssm start-session --target $NEW
sudo tail -f /var/log/user-data.log         # until "User data script completed" (Ansible compiles the rollup: see §6 for devnet's real time)
lsblk -o NAME,MODEL,SIZE,TYPE,MOUNTPOINTS; cat /proc/mdstat         # one Instance Storage NVMe, no md device
findmnt /mnt/rollup /mnt/logs /mnt/snapshots                        # NVMe ext4 ~3.4T / EBS /dev/sdg / (backup) EBS /dev/sdf
nproc; free -g                                                      # 16 / ~124
grep -E '^(disk_profile|rollup_node_role|rollup_sequencer_kind|rollup_commit_hash|rollup_node_id)' /home/ubuntu/runtime_vars.yaml
curl -s -o /dev/null -w 'ready=%{http_code}\n' localhost:12346/sequencer/ready      # not 200 yet, so not routed by the proxy
```
- **Backup (run 3) only:** disable the new node's snapshot cron (`sudo crontab -u ubuntu -e`) before restoring. It's re-enabled in §5.
- If the bootstrap failed, discard the instance the same way as on devnet (§4.3 there):
  1. `modify-instance-attribute --no-disable-api-termination`;
  2. `set-instance-protection --no-protected-from-scale-in`;
  3. `terminate-instance-in-auto-scaling-group --should-decrement-desired-capacity`.

### 4.4 Restore from the pre-warmed volume
```bash
sudo -iu ubuntu
tmux new -s restore
aws ec2 describe-volumes --region us-east-1 --volume-ids <VOL> --query 'Volumes[0].[State,AvailabilityZone]' --output text   # available  <this node's AZ>
/home/ubuntu/restore-from-snapshot.sh <VOL>                                                                                     # type yes
```
The script checks the volume and stops rollup. It then attaches, mounts the volume read-only and rsyncs it into `/mnt/rollup`, **detaches the volume and keeps it**, and starts rollup. The log is in `/mnt/logs/rollup-restore.log`.

### 4.5 Catch-up checks and observation gates
On `$NEW`:
```bash
curl -s localhost:12346/rollup/sync-status | jq -c
curl -s localhost:12346/ledger/slots/latest | jq -c '{number,state_root}'     # starts near the snapshot height, then rises
curl -s -o /dev/null -w 'ready=%{http_code}\n' localhost:12346/sequencer/ready # 200 once caught up
curl -s localhost:12346/sequencer/role; echo
sudo journalctl -u rollup --since '-30min' | grep -Ei 'error|panic|mismatch' | tail
```
**Same state root:** at a slot `N` slightly below the head, `curl -s localhost:12346/ledger/slots/$N | jq -r .state_root` must be identical on `$NEW` and on the leader.

```bash
# CPU of the new node, 5-min points over the last 2 h
aws --profile $RO cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=$NEW \
  --statistics Average Maximum --period 300 --start-time $(date -u -d '-2 hours' +%FT%TZ) --end-time $(date -u +%FT%TZ) \
  --query 'sort_by(Datapoints,&Timestamp)[].[Timestamp,Average,Maximum]' --output table
```
- **Gate 1 (after run 1):** the new node serves mainnet **reads** for at least **2 h**, including the busiest hour seen in §2.2. Pass if:
  - it stays `ready=200` and keeps up with the leader's height;
  - CPU average < 50%, max < 80%;
  - `free -g` is comfortable;
  - Relay sees no read errors.
- **Gate 2 (after run 2):** the i8g node is now the **leader** (batch production plus writes). Watch it for at least **1 h**: same thresholds, slots keep advancing, test transactions are accepted.
- If a gate fails, stop and review before going further. The c8gd ASG layout can come back through a CDK deploy with `RollupNodeInstanceType` removed.

### 4.6 Retire the old instance (in run 2 this is the leader failover)
1. **Check identities:**
   ```bash
   aws --profile $RO ec2 describe-instances --instance-ids $OLD $NEW --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name]' --output table   # OLD c8gd.12xlarge, NEW i8g.4xlarge
   aws --profile $RO ec2 describe-instance-attribute --instance-id $NEW --attribute disableApiTermination --query 'DisableApiTermination.Value'   # true
   ```
2. **Run 2 only, inside the agreed window.** Both preferred nodes that will remain must be `ready=200` at the same height. Then open these watches:
   ```bash
   while true; do printf '%s ' "$(date +%T)"; curl -s -o /dev/null -m 2 -w '%{http_code}\n' https://$DOMAIN/sequencer/ready; sleep 0.5; done   # laptop
   sudo journalctl -u rollup -f | grep -Ei 'leader|BatchProducer'                                                                              # each new preferred node
   ```
3. **Detach the old instance and stop rollup on it:**
   ```bash
   aws --profile $RW autoscaling detach-instances --auto-scaling-group-name $ASG --instance-ids $OLD --should-decrement-desired-capacity
   echo "${N}_DETACHED=$OLD" | tee -a $LOG
   aws --profile $RW ssm start-session --target $OLD
   sudo systemctl disable --now rollup && sudo systemctl mask rollup; systemctl is-active rollup      # inactive
   ```
   - Run 2: a new `DbElected` node logs *"Replica acquired leadership! Exiting to restart as leader."*, then *"DbElected node acquired leadership, running as BatchProducer"*. The probe shows a short run of non-200 responses, then 200.
   - Record the length of that gap, and send a test transaction.

### 4.7 Terminate the old instance and confirm it's gone **[write]**
Do it straight away: a detached instance still carries the `ASGType` tag the proxy uses at boot.
```bash
read TYPE STATE < <(aws --profile $RO ec2 describe-instances --instance-ids $OLD --query 'Reservations[0].Instances[0].[InstanceType,State.Name]' --output text)
INASG=$(aws --profile $RO autoscaling describe-auto-scaling-instances --instance-ids $OLD --query 'length(AutoScalingInstances)' --output text)
echo "$OLD: $TYPE $STATE in-asg=$INASG"
[ "$TYPE" = c8gd.12xlarge ] && [ "$INASG" = 0 ] && [ "$OLD" != "$NEW" ] \
  && aws --profile $RW ec2 terminate-instances --instance-ids $OLD \
  && aws --profile $RO ec2 wait instance-terminated --instance-ids $OLD \
  && echo "${N}_TERMINATED=$OLD" | tee -a $LOG
aws --profile $RO autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG \
  --query 'AutoScalingGroups[0].[DesiredCapacity,Instances[].[InstanceId,InstanceType,LifecycleState]]' --output json   # 1, only $NEW
aws --profile $RW ec2 modify-instance-attribute --instance-id $NEW --no-disable-api-termination
aws --profile $RW autoscaling set-instance-protection --auto-scaling-group-name $ASG --instance-ids $NEW --no-protected-from-scale-in
```

---

## 5. Finish
1. **Backup cron:** on `$B_NEW`, uncomment the line with `sudo crontab -u ubuntu -e`. Wait for the first `=== Snapshot complete ===` in `/mnt/logs/rollup-snapshot.log`, then check:
   ```bash
   aws --profile $RO ec2 describe-snapshots --owner-ids self --filters Name=tag:InstanceId,Values=$B_NEW --query 'Snapshots[].[SnapshotId,StartTime,State]' --output table
   ```
   Future restores use the new backup's `/dev/sdf` volume ID.
2. **Delete the restore volumes:**
   ```bash
   VOLS=$(grep -o '^VOL_[a-z0-9_]*=vol-[0-9a-f]*' $LOG | cut -d= -f2 | sort -u | xargs); echo $VOLS
   aws --profile $RO ec2 describe-volumes --volume-ids $VOLS --query 'Volumes[].[VolumeId,State]' --output text   # all available
   for v in $VOLS; do aws --profile $RW ec2 delete-volume --volume-id $v; done
   ```
3. **Resume ASG processes and check the end state:**
   ```bash
   for g in $P_ASG $S_ASG $B_ASG; do aws --profile $RW autoscaling resume-processes --auto-scaling-group-name $g --scaling-processes ReplaceUnhealthy AZRebalance; done
   aws --profile $RO ec2 describe-instances --filters Name=tag:Stack,Values=$STACK Name=instance-state-name,Values=running \
     --query 'Reservations[].Instances[].[InstanceId,InstanceType,PrivateIpAddress,Tags[?Key==`ASGType`]|[0].Value]' --output table   # 3 × i8g.4xlarge + proxy
   ```
4. **Optional proxy roll**, for its static fallback IPs:
   - only after all old instances are terminated, and in a quiet window agreed with Relay;
   - increment `ProxyRolloutToken` in `deployments/relay/mainnet/parameters.json`;
   - preview it (`--change-set-name proxy-rollout`): only the proxy launch template and ASG may change;
   - execute it, watch the probe, and commit the change.
5. **Alarms:** `SovRollupCdkStarterStack-health-check-failure` and `SovRollupCdkStarterStack-runtime-config-drift` must be back to OK.
6. **No drift:** §2.7 c) reports `IN_SYNC 0`, and every ASG is back to `DesiredCapacity 1`.
7. **Old snapshots** of `$SNAP_VOL` are no longer pruned. Delete them once you no longer need them.
8. **Tell Relay** the migration is complete: give them the time of the failover, the new instance IDs (OTEL), and ask them to report anything unusual over the next days.

---

## 6. Time estimate
Plan two working days: Deploy 0 and the migration deploy can be one day, and the node runs the next. Replace the devnet durations with the values measured in §2.8.

| Step | Duration |
|---|---|
| §1–§2 access + pre-flight | 1–2 h |
| §3.1 Deploy 0 (if needed, incl. proxy roll) | 30–60 min, in a quiet window |
| §3.2–§3.4 migration patch, review, deploy | 30 min + review time |
| §4.0 prep: volumes pre-warm in parallel | ~1 h per TB of snapshot data |
| Per node: bootstrap (§4.3) | devnet: _fill in_ |
| Per node: restore from warm volume + catch-up | devnet: _fill in_ (scales with data size) |
| Gate 1 / Gate 2 | ≥ 2 h / ≥ 1 h |
| Per node: retire + terminate | 10 min (run 2: seconds of write pause, devnet measured _fill in_) |
| §5 | 30 min + up to 1 h for the first new snapshot |

## Definition of done
- The Primary, Secondary and BackupNode ASGs in `751545121805` each run exactly one **i8g.4xlarge** from the new launch template version. Scale-in and termination protection are removed and the suspended processes are resumed.
- On each node, `/mnt/rollup` is on the single instance-store NVMe (no md device) and `/mnt/logs` is on EBS `/dev/sdg`. The backup also has `/mnt/snapshots` on `/dev/sdf`, its cron is re-enabled, and its new snapshots complete.
- One new preferred node is leader. Both preferred nodes return `ready=200`, have the same height and the same state roots. `https://rpc.chain.relay.link` serves reads and writes. The only client-visible impact was the agreed failover pause, and both gates passed.
- All three old c8gd.12xlarge instances are terminated, and the restore volumes are deleted. Alarms are OK.
- `deployments/relay/mainnet/parameters.json` sets `RollupNodeInstanceType: i8g.4xlarge` (and `AuroraEngineVersion` if mainnet drifted). That PR is merged into the deploy branch, and CloudFormation drift detection reports `IN_SYNC 0`.
- Relay has been told the migration is complete.
