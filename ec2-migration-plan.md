# Runbook: migrate dev sequencers from i8g.12xlarge to i8g.4xlarge with no downtime

> Nothing in this runbook has been executed. All commands are meant to be run by a human, in order, one environment at a time.
> There's no Terraform for the dev servers yet. AWS resources are created and removed **manually with the `aws` CLI**, and hosts are configured with **Ansible**. Ansible snippets are templates: adapt role, variable and inventory names to your Ansible repo. Every AWS resource created here is tagged and logged (§3.0), so the future Terraform codebase can import it.
> Everything about sequencer behaviour (roles, leader election, endpoints) comes from the SDK source in this repo.

---

## 0. Background (read once)

### 0.1 Hardware change (confirm in §2.9)

| | i8g.12xlarge | i8g.4xlarge |
|---|---|---|
| vCPU | 48 | 16 |
| RAM | 384 GiB | 128 GiB |
| Instance-store NVMe | 3 × 3750 GB → RAID0 ≈ 11.25 TB | **1 × 3750 GB, no RAID** |
| Disk IOPS/throughput | about 3 drives striped | **about ⅓ of today** |

### 0.2 How a no-downtime cutover works (SDK behaviour)

- Config `[…postgres_config]`: `postgres_connection_string`, `node_id` (must be unique), `node_role` ∈ `Leader | Replica | ReplicaNoLeaderSync | DbElected`, `leader_election { leader_timeout_millis=1000, grace_period_millis=10000, heartbeat_interval_millis=100 }`.
- `Replica`: syncs from the leader through Postgres and **never** competes for leadership. Used while the new nodes catch up.
- `DbElected`: starts as a replica. If the leader stops heartbeating for >1 s (and has been leader for >10 s), it takes leadership, **exits**, and must be restarted by systemd. It then comes back as the batch producer.
- `Leader`: **always** starts as batch producer, even if another node already leads. It exits on its first heartbeat if it isn't the leader. ⇒ never (re)start a `Leader`-configured node while another node leads. That's why old nodes are `mask`ed below.
- Result: reads never stop. Writes (tx submit / soft confirmations) pause for **about 1 s plus the new leader's restart time** at cutover. The SDK has no hot handover.
- **Hard prerequisite:** dev must run with `postgres_config` set. If it uses the RocksDB backend (no `postgres_config`), you can't have a no-downtime cutover. Stop here and plan a maintenance window instead.

### 0.3 Strategy
1. Add new 4xlarge hosts next to the old ones (never modify the old ones in place).
2. Seed the new hosts' data by **restoring the latest snapshot taken by the backup node** (§7). The new nodes then only replay the few DA blocks since the snapshot. No old node is stopped during seeding.
3. Run the new hosts as `Replica` until they're at head, and soak them.
4. Set one new node to `DbElected`, stop and mask the old leader, and the new node takes over.
5. Soak, then destroy the old hosts.

---

## 1. Variables (set in every shell you use)

```bash
export ENV=<dev-env-name>
export AWS_REGION=<region>
export AWS_PROFILE=<profile>
export SVC=<rollup systemd unit, e.g. rollup.service>
export SVC_USER=<user the service runs as>
export DATA=<data mount point, e.g. /mnt/data>
export ROLLUP_DIR=<rollup data dir = [storage] path, e.g. /mnt/data/rollup>
export CFG=<rollup config toml path on hosts>
export API=<rollup REST port>
export PGURL='postgresql://<user>:<pass>@<host>:5432/<db>'
export SSH_USER=<ssh user>
export OLD_LEADER=<private ip>
export OLD_REPLICAS="<ip> <ip>"          # may be empty
export NEW1=<ip>                         # filled after §3.2; this one becomes the leader
export NEW2=<ip>                         # standby
export NEW_NODES="$NEW1 $NEW2"
export WRITE_ENDPOINT=<host:port clients use to submit txs>
export BACKUP_NODE=<ip of the backup node that takes snapshots>
export SNAP_URI=<s3://bucket/prefix where snapshots land, e.g. s3://<bucket>/<env>/>   # S3 variant (§7.A)
export TG_ARN=<target group arn, if an LB is used (§2.7 d)>
# OLD_LEADER_ID, OLD_IDS, NEW1_ID, NEW2_ID, NEW1, NEW2 are set in §3 and appended to the migration log:
# in any new shell run `source ~/i8g-migration-$ENV.env`
```

Find the values on an old host:

```bash
ssh $SSH_USER@$OLD_LEADER
systemctl list-units --type=service | grep -Ei 'rollup|sov|sequencer'
systemctl cat $SVC                                   # ExecStart → binary + config path, User=, Restart=
grep -nE '^\[|path|node_id|node_role|postgres_connection_string|bind_port|bind_host' $CFG
findmnt -no SOURCE,FSTYPE,OPTIONS $DATA
```

---

## 2. Pre-flight on the old hosts (read-only), go/no-go

Run on **each** old host and write the numbers down.

### 2.1 Disk layout and filesystem
```bash
lsblk -o NAME,MODEL,SIZE,TYPE,MOUNTPOINT
cat /proc/mdstat
sudo mdadm --detail /dev/md0
findmnt -no SOURCE,FSTYPE,OPTIONS $DATA     # reuse this FSTYPE + options on the new hosts
grep -E 'md0|'"$DATA" /etc/fstab
```

### 2.2 Disk usage and growth
```bash
df -h $DATA
sudo du -sh $ROLLUP_DIR/* | sort -h
sudo ls -la $ROLLUP_DIR                       # list everything; note any non-DB files (keys, configs)
```
Prometheus (node_exporter): growth per day and projected free space in 90 d:
```promql
-deriv(node_filesystem_avail_bytes{mountpoint="<DATA>", instance=~"<old hosts>"}[7d]) * 86400
predict_linear(node_filesystem_avail_bytes{mountpoint="<DATA>"}[30d], 90*86400)
```
**Go if:** used ≤ **3.0 TB** and projected usage in 90 d ≤ ~3.2 TB. Otherwise enable or tighten `[storage.pruner]`, or use i8g.8xlarge (2 × 3750 GB) instead.

### 2.3 NOMT hashtable size (can't be reduced after creation)
```bash
grep -nE 'hashtable_buckets|preallocate_ht|page_cache|leaf_cache|state_cache_size|commit_concurrency|num_cache_warmup_workers' $CFG
sudo find $ROLLUP_DIR -name 'ht' -o -name '*.ht' | xargs -r sudo ls -lh
```
The ht file is about `buckets × 4 KiB`. **Go if** all ht files plus the other data fit in 3.0 TB.

### 2.4 Memory
```bash
free -g
ps -o pid,rss,vsz,cmd -C <binary-name>          # RSS in KiB
```
PromQL: `max_over_time(process_resident_memory_bytes{instance=~"<old hosts>"}[7d])`
**Go if** peak RSS plus planned cache sizes ≤ ~90 GiB. Otherwise reduce caches in §5.

### 2.5 CPU and disk IO load
```bash
uptime; nproc
mpstat 5 6                                       # sysstat package
iostat -xm 5 6 /dev/md0 /dev/nvme*n1             # r/s, w/s, rMB/s, wMB/s, %util at peak
```
Write down peak `r/s + w/s` and MB/s on `md0`. You'll compare them to the fio result on the new host (§6.2). **Go if** the peak is < 50% of what a single new drive can do.

### 2.6 Cluster topology and Postgres
```bash
psql "$PGURL" -c "SELECT node_id, last_updated, leader_acquired_at FROM sequencer_leader;"
psql "$PGURL" -c "SELECT node_id, address, now()-last_updated AS age FROM nodes ORDER BY node_id;"
psql "$PGURL" -c "SHOW max_connections;" -c "SELECT count(*) FROM pg_stat_activity;"
```
Write down the leader's `node_id` and IP, every replica, and each node's `node_role` (in `$CFG`).
Seeding uses the backup node's snapshot (§2.11, §7), so having no old replica is fine. Old replicas only serve as a fallback data source (§7.D).

### 2.7 How clients reach the cluster
Find out how **writes** reach the leader today and write it down. Pick one:
- (a) proxy using `sov-proxy-utils` node discovery (follows `sequencer_leader` automatically): nothing to do at cutover
- (b) Elastic IP on the leader: reassociate at cutover (§11.4)
- (c) DNS record pointing at the leader: **lower TTL to 30 s at least 24 h before cutover** and update it at cutover (§11.4)
- (d) LB target group: register/deregister (§9, §11)

Also check who else talks to a specific host directly (prover, indexer, explorer, relayer, monitoring) and plan to repoint them.

### 2.8 Old leader systemd settings (needed for rollback)
```bash
systemctl show $SVC -p Restart,RestartUSec,TimeoutStopUSec,User
```

### 2.9 AWS checks
```bash
aws ec2 describe-instance-types --instance-types i8g.4xlarge i8g.12xlarge \
  --query 'InstanceTypes[].{type:InstanceType,vcpu:VCpuInfo.DefaultVCpus,mem:MemoryInfo.SizeInMiB,disks:InstanceStorageInfo.Disks,net:NetworkInfo.NetworkPerformance}' --output table
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters Name=instance-type,Values=i8g.4xlarge --query 'InstanceTypeOfferings[].Location'
aws ec2 describe-instances --filters "Name=tag:Name,Values=*<sequencer name pattern>*" \
  --query 'Reservations[].Instances[].{id:InstanceId,type:InstanceType,ip:PrivateIpAddress,az:Placement.AvailabilityZone,name:Tags[?Key==`Name`]|[0].Value,state:State.Name}' --output table
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A   # Running On-Demand Standard vCPUs
```
**Go if** i8g.4xlarge is offered in the same AZ(s) and the vCPU quota covers old + new instances running at the same time (+16 vCPU per new node).

### 2.10 Freeze
- Pause automatic deploys to dev (CI/CD, Ansible cron, Renovate auto-deploys) for the whole migration. An automated redeploy could restart an old node as `Leader` after cutover.
- Tell dev users which window the ~seconds-long write pause will happen in.

### 2.11 Backup node snapshots (read-only), go/no-go
The SDK has no built-in snapshot command, so snapshots are whatever the backup node's job produces. First find out exactly what that job does:

```bash
ssh $SSH_USER@$BACKUP_NODE
systemctl list-timers --all | grep -Ei 'snap|backup'
sudo crontab -l; ls /etc/cron.d/; grep -rlEi 'snap|backup' /etc/cron.d /etc/systemd/system 2>/dev/null
systemctl cat <snapshot unit>                      # find the script it runs
sudo cat <snapshot script>                         # read it end to end
grep -nE '^\[|path|_db_path|hashtable_buckets|preallocate_ht' <backup node config toml>
<rollup binary> --version                          # version that writes the snapshots
```

From the script, write down:

| Question | Why it matters |
|---|---|
| **Format and location**: tar(.zst/.gz/.lz4) archive in S3 (→ §7.A), EBS snapshot of a data volume (→ §7.B), or a directory/archive on the backup node's own disk (→ §7.C) | Decides which restore procedure to use |
| **Consistency**: is the rollup service **stopped** (or are NOMT and RocksDB checkpointed) while the snapshot is taken? | A copy of live NOMT/RocksDB files can't be opened. If the script copies files from a running node, don't use it: fix the script or use §7.D |
| **Contents**: which dirs are inside (`state-db`, `ledger`, `user_nomt_db`, … per `[storage]` paths) and the archive's top-level path | Must match the sequencer's `[storage]` layout. Decides `--strip-components` |
| **Height**: rollup slot / DA height at snapshot time (manifest file, object tags, or log line) | Used to check the state root after restore and to estimate catch-up time |
| **Checksum**: is a `.sha256` written next to the archive? | Integrity check after download |
| **Frequency and retention** | Pick a snapshot that's < 24 h old |

Then check a candidate snapshot (S3 example; adjust for EBS: `aws ec2 describe-snapshots --owner-ids self --filters Name=tag:Name,Values=<pattern> --query 'reverse(sort_by(Snapshots,&StartTime))[:5].[SnapshotId,StartTime,VolumeSize,State]' --output table`):

```bash
aws s3 ls $SNAP_URI --human-readable | sort | tail -5
export SNAP_OBJ=${SNAP_URI%/}/<latest-archive-name>          # e.g. .../rollup-20261117T0300Z.tar.zst
aws s3api head-object --bucket <bucket> --key <prefix/latest-archive-name> \
  --query '{size:ContentLength,modified:LastModified,meta:Metadata}'
aws s3 cp ${SNAP_OBJ}.sha256 - 2>/dev/null                    # if the job writes one
aws s3 cp ${SNAP_URI%/}/<manifest file> - 2>/dev/null         # if the job writes one (height, version, …)
# List the archive's top-level layout without downloading it all (stops after 50 entries)
aws s3 cp $SNAP_OBJ - | zstd -dc | tar -tf - | head -50
```

**Go if all are true:**
- The snapshot was taken from a **cleanly stopped** node (or a proper DB checkpoint).
- It was written by the **same rollup version** as the running cluster (§1 `systemctl cat`, `<binary> --version` on `$OLD_LEADER`). Never restore a snapshot from a newer version than the binary you'll run.
- The backup node's `*_hashtable_buckets` / `*_preallocate_ht` equal the sequencers' (§2.3). Buckets can't be changed after creation.
- Its **uncompressed** size is ≤ 3.0 TB. Check the manifest or `du` on the backup node.
- It's < 24 h old and its DA height is still served by the DA node (pruning window).
- It contains every dir the sequencer's `[storage]` config expects (compare with `sudo ls -la $ROLLUP_DIR` on an old sequencer from §2.2). If a sequencer has dirs the snapshot lacks (e.g. the backup node is a plain full node), don't guess: confirm with the SDK team that starting with those dirs missing is safe, or use §7.D.

---

## 3. AWS CLI: launch the new instances (never modify the old ones)

There's no Terraform yet, so every AWS change is a manual `aws` CLI command run by a human. Run each command that changes something with `--dry-run` first: it checks permissions and parameters and creates nothing. **Record every ID you create in the migration log (§3.0).** It's the only state, and it's what the future Terraform codebase will import.

Don't resize in place. `stop → modify-instance-attribute --instance-type → start` wipes the instance store (the RAID) and is downtime. **Never stop or start the old instances.**

### 3.0 Migration log
```bash
export LOG=~/i8g-migration-$ENV.env
echo "# i8g migration $ENV started $(date -u +%FT%TZ)" >> $LOG
# Append every ID you create, e.g. echo "NEW1_ID=i-…" >> $LOG
# In a new shell:  source $LOG
```

### 3.1 Capture the old instances' settings
```bash
export OLD_LEADER_ID=<instance id of the old leader>
export OLD_IDS="<old leader id> <old replica ids…>"
echo "OLD_IDS=\"$OLD_IDS\"" >> $LOG

aws ec2 describe-instances --instance-ids $OLD_LEADER_ID --query 'Reservations[0].Instances[0].{ami:ImageId,arch:Architecture,subnet:SubnetId,az:Placement.AvailabilityZone,sgs:SecurityGroups[].GroupId,profile:IamInstanceProfile.Arn,key:KeyName,root:RootDeviceName,imds:MetadataOptions.HttpTokens}' --output json
aws ec2 describe-instances --instance-ids $OLD_LEADER_ID --query 'Reservations[0].Instances[0].Tags' --output table
aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=$OLD_LEADER_ID \
  --query 'Volumes[].{dev:Attachments[0].Device,size:Size,type:VolumeType,iops:Iops,tp:Throughput,enc:Encrypted,kms:KmsKeyId,delOnTerm:Attachments[0].DeleteOnTermination}' --output table
aws ec2 describe-instance-attribute --instance-id $OLD_LEADER_ID --attribute userData --query 'UserData.Value' --output text | base64 -d   # usually empty when Ansible does everything
aws ec2 describe-addresses --filters Name=instance-id,Values=$OLD_LEADER_ID      # Elastic IP? (§2.7 b)
```
Copy the values into variables:
```bash
export AMI=<ImageId>
export SUBNET=<SubnetId of the old leader>        # same AZ as the old leader and Postgres
export SUBNET_B=<subnet for NEW2>                 # same as SUBNET, or the AZ the old replicas use
export SGS="<sg-… sg-…>"
export PROFILE_ARN=<IamInstanceProfile.Arn>
export KEY=<KeyName>
export ROOT_DEV=<RootDeviceName, e.g. /dev/sda1>
export ROOT_SIZE=<GiB of the root volume>
export EXTRA_TAGS='<tags from the old instance that monitoring / dynamic inventory / cost reports rely on, e.g. {Key=Project,Value=sovereign},>'
```
Check the AMI:
```bash
aws ec2 describe-images --image-ids $AMI --query 'Images[0].{name:Name,arch:Architecture,state:State,deprecated:DeprecationTime}'
```
It must be `arm64` (i8g is Graviton) and `available`. If it's been deregistered, use the current arm64 AMI of the **same distro release**. Ansible then brings the host to the same state.

### 3.2 Launch the two instances
```bash
# $1 = index (0|1), $2 = subnet, other args go straight to run-instances (e.g. --dry-run)
launch() {
  aws ec2 run-instances "${@:3}" \
    --image-id "$AMI" --instance-type i8g.4xlarge --count 1 \
    --subnet-id "$2" --security-group-ids $SGS \
    --iam-instance-profile Arn="$PROFILE_ARN" --key-name "$KEY" \
    --block-device-mappings "DeviceName=$ROOT_DEV,Ebs={VolumeSize=$ROOT_SIZE,VolumeType=gp3,Encrypted=true,DeleteOnTermination=true}" \
    --metadata-options HttpTokens=required,HttpEndpoint=enabled \
    --disable-api-termination --disable-api-stop \
    --tag-specifications \
      "ResourceType=instance,Tags=[${EXTRA_TAGS}{Key=Name,Value=$ENV-sequencer-v2-$1},{Key=Env,Value=$ENV},{Key=Role,Value=sequencer},{Key=Migration,Value=i8g-4xl},{Key=ManagedBy,Value=manual}]" \
      "ResourceType=volume,Tags=[{Key=Name,Value=$ENV-sequencer-v2-$1-root},{Key=Env,Value=$ENV}]" \
    --query 'Instances[0].InstanceId' --output text
}

launch 0 "$SUBNET"   --dry-run      # expect: "DryRunOperation … Request would have succeeded"
launch 1 "$SUBNET_B" --dry-run
```
- `--disable-api-stop` blocks accidental API stops (a stop wipes the instance store). Reboot still works.
- `--disable-api-termination` blocks accidental termination. Both are lifted in §13 or §14.
- If the old root volume uses a customer KMS key, add `KmsKeyId=<arn>` inside `Ebs={…}`.
- No block-device mapping is needed for the NVMe drive: instance store on i8g is attached automatically.

Launch for real (human):
```bash
NEW1_ID=$(launch 0 "$SUBNET");   echo "NEW1_ID=$NEW1_ID" >> $LOG
NEW2_ID=$(launch 1 "$SUBNET_B"); echo "NEW2_ID=$NEW2_ID" >> $LOG
aws ec2 wait instance-status-ok --instance-ids $NEW1_ID $NEW2_ID
aws ec2 describe-instances --instance-ids $NEW1_ID $NEW2_ID \
  --query 'Reservations[].Instances[].{id:InstanceId,name:Tags[?Key==`Name`]|[0].Value,type:InstanceType,ip:PrivateIpAddress,az:Placement.AvailabilityZone,state:State.Name}' --output table
```
```bash
export NEW1=<private ip of v2-0> NEW2=<private ip of v2-1>; export NEW_NODES="$NEW1 $NEW2"
echo "NEW1=$NEW1 NEW2=$NEW2" >> $LOG
```
`InsufficientInstanceCapacity` means AWS has no i8g.4xlarge free in that AZ right now. Retry later or use another subnet or AZ (Postgres latency matters for the leader: keep NEW1 close to Postgres).

### 3.3 Network access
The new hosts reuse the old SGs, so rules that reference the SG itself already apply. Rules that whitelist the **old hosts' IPs** don't. Check Postgres, the DA node, the LB/proxy and Prometheus:
```bash
aws ec2 describe-security-groups --group-ids <sg of Postgres / DA node / proxy> \
  --query 'SecurityGroups[].{sg:GroupId,in:IpPermissions[].{p:FromPort,cidr:IpRanges[].CidrIp,sg:UserIdGroupPairs[].GroupId}}' --output json
# Only if an old host's /32 is listed:
aws ec2 authorize-security-group-ingress --group-id <sg> --dry-run \
  --ip-permissions "IpProtocol=tcp,FromPort=<port>,ToPort=<port>,IpRanges=[{CidrIp=$NEW1/32,Description=$ENV-sequencer-v2-0},{CidrIp=$NEW2/32,Description=$ENV-sequencer-v2-1}]"
# then without --dry-run, and log it:
echo "SG_RULE <sg> <port> $NEW1/32 $NEW2/32" >> $LOG
```
If Postgres restricts clients in `pg_hba.conf` (Ansible-managed), add the new IPs there too.

### 3.4 Snapshot read access (§7.A only)
```bash
ROLE=$(aws iam get-instance-profile --instance-profile-name "${PROFILE_ARN##*/}" --query 'InstanceProfile.Roles[0].RoleName' --output text); echo $ROLE
aws iam list-role-policies --role-name $ROLE; aws iam list-attached-role-policies --role-name $ROLE   # maybe it can already read the bucket
cat > /tmp/snapshot-read.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "s3:ListBucket", "Resource": "arn:aws:s3:::<bucket>",
      "Condition": { "StringLike": { "s3:prefix": ["<prefix>/*"] } } },
    { "Effect": "Allow", "Action": "s3:GetObject", "Resource": "arn:aws:s3:::<bucket>/<prefix>/*" },
    { "Effect": "Allow", "Action": "kms:Decrypt", "Resource": "<kms key arn, remove this statement if the bucket uses SSE-S3>" }
  ]
}
EOF
aws iam put-role-policy --role-name $ROLE --policy-name $ENV-sequencer-snapshot-read --policy-document file:///tmp/snapshot-read.json
echo "IAM_INLINE_POLICY $ROLE $ENV-sequencer-snapshot-read" >> $LOG
ssh $SSH_USER@$NEW1 "aws s3 ls $SNAP_URI | tail -3"          # works within a few seconds
```
The role is shared with the old hosts, so they also get read access. That's harmless; remove it after the migration with `aws iam delete-role-policy --role-name $ROLE --policy-name $ENV-sequencer-snapshot-read`.

S3 path: without an S3 gateway endpoint, multi-TB downloads go through the NAT gateway and you pay data-processing charges.
```bash
VPC=$(aws ec2 describe-subnets --subnet-ids $SUBNET --query 'Subnets[0].VpcId' --output text)
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC Name=service-name,Values=com.amazonaws.$AWS_REGION.s3 \
  --query 'VpcEndpoints[].{id:VpcEndpointId,type:VpcEndpointType,routeTables:RouteTableIds}'
```
If none is attached to the new subnet's route table, either accept the NAT cost or have the VPC owner add one.

§7.B (EBS snapshot) only needs operator credentials with `ec2:CreateVolume`, `ec2:AttachVolume`, `ec2:DetachVolume`, `ec2:DeleteVolume` and `ec2:CreateTags`.

---

## 4. Ansible: inventory and disk provisioning

### 4.1 Inventory
Put the new hosts in their **own group**, so every run can be `--limit`ed to them and the old hosts are never touched. Make it a child of the existing sequencer group so it inherits the same vars (binary version, secrets, Postgres URL, monitoring):

```yaml
# inventories/<dev>/hosts.yml (static inventory)
all:
  children:
    sequencers:                     # existing group, old hosts unchanged
      hosts:
        <old hosts …>:
      children:
        sequencers_v2:
          hosts:
            <env>-sequencer-v2-0: { ansible_host: <NEW1 ip> }
            <env>-sequencer-v2-1: { ansible_host: <NEW2 ip> }
```
With the `amazon.aws.aws_ec2` dynamic inventory, add `keyed_groups: [{ key: tags.Migration, prefix: mig }]` and use the group `mig_i8g_4xl` instead of `sequencers_v2`.

Per-host and per-group vars (use the variable names your config template already uses: `grep -rn 'node_role\|node_id' roles/ templates/`):
```yaml
# inventories/<dev>/host_vars/<env>-sequencer-v2-0.yml
sequencer_node_id: <env>-seq-v2-0
sequencer_node_role: Replica          # → DbElected in §10

# inventories/<dev>/host_vars/<env>-sequencer-v2-1.yml
sequencer_node_id: <env>-seq-v2-1
sequencer_node_role: Replica          # → DbElected in §12

# inventories/<dev>/group_vars/sequencers_v2.yml  (child-group vars override the parent's)
data_mount: <same mount point as old, §2.1>
rollup_service_enabled: false         # don't start before the data is restored (§8 flips it)
rollup_service_state: stopped
# + tuned storage settings from §5.1
```
Check (safe):
```bash
ansible-inventory -i inventories/<dev> --graph
ansible-inventory -i inventories/<dev> --host <env>-sequencer-v2-0      # merged vars: node_id, role, tuned values
ansible -i inventories/<dev> sequencers_v2 -m ansible.builtin.ping
```

### 4.2 Disk role: 1 NVMe → no RAID, several → RAID0
Replace the fixed-3-device mdadm tasks in the disk role. Keep the **same mount point and filesystem type** as `findmnt` showed in §2.1, so no service or config paths change.
```yaml
- name: Discover instance-store NVMe devices
  ansible.builtin.set_fact:
    nvme_store: >-
      {{ ansible_devices | dict2items
         | selectattr('value.model', 'search', 'Instance Storage')
         | map(attribute='key') | map('regex_replace', '^', '/dev/') | sort | list }}

- name: Require at least one instance-store device
  ansible.builtin.assert:
    that: nvme_store | length >= 1

- name: Pick data device
  ansible.builtin.set_fact:
    data_dev: "{{ nvme_store[0] if nvme_store | length == 1 else '/dev/md0' }}"

- name: Create RAID0 (only on multi-disk instance types)
  ansible.builtin.command: >
    mdadm --create /dev/md0 --level=0 --raid-devices={{ nvme_store | length }} --run {{ nvme_store | join(' ') }}
  args: { creates: /dev/md0 }
  when: nvme_store | length > 1

- name: Persist mdadm config (only on multi-disk instance types)
  ansible.builtin.shell: mdadm --detail --scan > /etc/mdadm/mdadm.conf && update-initramfs -u
  when: nvme_store | length > 1

- name: Create filesystem (no-op if one exists)
  community.general.filesystem:
    fstype: xfs
    dev: "{{ data_dev }}"

- name: Read filesystem UUID
  ansible.builtin.command: blkid -o value -s UUID {{ data_dev }}
  register: data_uuid
  changed_when: false

- name: Mount data volume
  ansible.posix.mount:
    path: "{{ data_mount }}"
    src: "UUID={{ data_uuid.stdout }}"
    fstype: xfs
    opts: defaults,noatime,nofail
    state: mounted
```
Remove any hard-coded `/dev/md0` from other roles or templates (fstab, node_exporter mountpoint filters, alerts).

**Never run this updated role on the old hosts.** Their fstab entry would switch from `/dev/md0` to `UUID=…` and the data volume would be remounted. Every run stays `--limit sequencers_v2` until the old hosts are gone (§13).

### 4.3 No Ansible disk role today? One-off script
If the RAID on the old hosts was built by hand, run this once per new host instead of §4.2: `ssh $SSH_USER@$NEW1 'sudo bash -s' < setup-nvme.sh`
```bash
#!/usr/bin/env bash
# setup-nvme.sh: idempotent; 1 instance-store NVMe → plain fs, several → RAID0
set -euo pipefail
MOUNT=<same mount point as old, §2.1>
FSTYPE=xfs                        # match §2.1
MDADM_CONF=/etc/mdadm/mdadm.conf  # RHEL/AL2023: /etc/mdadm.conf

findmnt -rn "$MOUNT" >/dev/null && { echo "$MOUNT already mounted"; exit 0; }

# Instance-store NVMe only. The EBS root reports model "Amazon Elastic Block Store".
mapfile -t devs < <(lsblk -dpno NAME,MODEL | awk '/Amazon EC2 NVMe Instance Storage/{print $1}' | sort)
(( ${#devs[@]} >= 1 )) || { echo "no instance-store NVMe found" >&2; exit 1; }

if (( ${#devs[@]} == 1 )); then
  dev="${devs[0]}"
else
  dev=/dev/md0
  [ -e "$dev" ] || mdadm --create "$dev" --level=0 --raid-devices="${#devs[@]}" --run "${devs[@]}"
  mdadm --detail --scan > "$MDADM_CONF"
  update-initramfs -u || dracut -f
fi

# Format only if blank. Instance store survives reboot but is wiped on stop/start.
blkid -o value -s TYPE "$dev" >/dev/null || mkfs.$FSTYPE -f "$dev"
uuid=$(blkid -o value -s UUID "$dev")
mkdir -p "$MOUNT"
grep -q "$uuid" /etc/fstab || echo "UUID=$uuid $MOUNT $FSTYPE defaults,noatime,nofail 0 2" >> /etc/fstab
mount "$MOUNT"
```

---

## 5. Ansible: rollup config and systemd for the new hosts

### 5.1 Config differences vs the old hosts
The config template renders these from the §4.1 vars. Find the section that holds `node_role` in today's config with `grep -n -B5 node_role $CFG` on an old host. Same section, new values:
```toml
postgres_connection_string = "<same as old>"
node_id   = "<env>-seq-v2-0"           # NEW2: "<env>-seq-v2-1"; must be unique
node_role = "Replica"                  # Replica during sync; changed in §10 / §12
# [leader_election]: identical to old nodes (or omitted on all)
```
Storage and sequencer tuning for 16 vCPU / 128 GiB (in `group_vars/sequencers_v2.yml`). Start from the old values and scale:
```toml
# [storage]
state_cache_size          = <old ÷ 3>
user_commit_concurrency   = <min(old, 4)>
kernel_commit_concurrency = <min(old, 2)>
user_page_cache_size      = <old ÷ 3>
user_leaf_cache_size      = <old ÷ 3>
kernel_page_cache_size    = <old ÷ 3>
kernel_leaf_cache_size    = <old ÷ 3>
user_hashtable_buckets    = <UNCHANGED: must match the restored DB>
kernel_hashtable_buckets  = <UNCHANGED>
# [sequencer]
num_cache_warmup_workers  = <min(old, 8)>
```
Keep the same sequencer signing key and secrets, deployed by the same Ansible tasks as on the old hosts.

### 5.2 systemd restart policy (required for DbElected takeover)
Add to the rollup role, unless the unit template already has these settings:
```yaml
- name: Rollup service restart policy (DbElected takeover needs an automatic restart)
  ansible.builtin.copy:
    dest: "/etc/systemd/system/{{ rollup_service }}.d/override.conf"     # rollup_service = e.g. rollup.service
    content: |
      [Service]
      Restart=always
      RestartSec=1
      TimeoutStopSec=300
    mode: "0644"
  notify: Reload systemd
```
```yaml
# handlers
- name: Reload systemd
  ansible.builtin.systemd:
    daemon_reload: true
```
The task that manages the service must follow `rollup_service_enabled` / `rollup_service_state` (§4.1), so the new hosts get the binary and config **without starting**.

### 5.3 Run the playbook on the new hosts only
Safe checks:
```bash
ansible-lint roles/<disk-role> roles/<rollup-role>
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit sequencers_v2 --syntax-check
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit sequencers_v2 --list-tasks
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit sequencers_v2 --check --diff
```
Apply (human):
```bash
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit sequencers_v2
```
Verify:
```bash
ansible -i inventories/<dev> sequencers_v2 -b -m ansible.builtin.shell \
  -a "systemctl is-active $SVC; systemctl show $SVC -p Restart; grep -nE 'node_id|node_role' $CFG; findmnt -no SOURCE,FSTYPE $DATA"
# expect: inactive, Restart=always, the v2 node_id, Replica, the NVMe device (not md0)
```

---

## 6. Verify each new host

```bash
ssh $SSH_USER@$NEW1
lsblk -o NAME,MODEL,SIZE,TYPE,MOUNTPOINT        # exactly 1 "Instance Storage" disk, mounted at $DATA
cat /proc/mdstat                                # no md devices
findmnt $DATA; df -h $DATA                      # ~3.4 TiB, expected fs type
grep $DATA /etc/fstab                           # UUID=…, nofail
nproc; free -g                                  # 16 / ~124
psql "$PGURL" -c 'select 1'                     # Postgres reachable from new host
curl -s -o /dev/null -w '%{http_code}\n' http://<DA RPC>/   # DA reachable
```

### 6.1 Reboot test (instance store must survive and remount)
```bash
sudo reboot
# after reconnect:
findmnt $DATA
```

### 6.2 Disk benchmark (compare with §2.5)
```bash
sudo apt-get install -y fio     # or dnf
sudo fio --name=rr --filename=$DATA/fio.test --size=20G --rw=randread  --bs=4k --iodepth=64 --numjobs=4 --ioengine=io_uring --direct=1 --runtime=60 --time_based --group_reporting
sudo fio --name=rw --filename=$DATA/fio.test --size=20G --rw=randwrite --bs=4k --iodepth=64 --numjobs=4 --ioengine=io_uring --direct=1 --runtime=60 --time_based --group_reporting
sudo rm -f $DATA/fio.test
```
**Go if** the old peak `r/s + w/s` from §2.5 is < 50% of the fio IOPS.

---

## 7. Seed data on the new hosts from the backup node's snapshot

Both new hosts restore **the same** snapshot, **in parallel** and independently of each other. The service stays stopped (§5.2) until §8. No old node is touched. Use the subsection that matches the format you found in §2.11:

- §7.A: archive in S3 (most common for instance-store hosts)
- §7.B: EBS snapshot
- §7.C: archive or directory kept on the backup node itself
- §7.D / §7.E: fallbacks if no usable snapshot exists

### 7.0 Prepare each new host (all variants)
```bash
ssh $SSH_USER@$NEW1                            # then the same on NEW2
systemctl is-active $SVC                       # must print 'inactive'
sudo apt-get install -y zstd pv jq tmux        # or dnf; use pigz / lz4 instead of zstd if the archive uses them
aws --version && aws sts get-caller-identity   # §7.A only: snapshot read access from §3.4 works
sudo mkdir -p $ROLLUP_DIR
sudo ls -A $ROLLUP_DIR                         # must print nothing (empty dir)
df -h $DATA                                    # free ≥ uncompressed snapshot size + 20%
```

Pin the snapshot chosen in §2.11 (same values on both hosts):
```bash
export SNAP_OBJ=<exact s3://… URI of the archive>          # §7.A
export SNAP_ID=<snap-… id>                                 # §7.B
export SNAP_SLOT=<rollup slot recorded for this snapshot>  # from the manifest/log; used in §8
export STRIP=<n>     # --strip-components: number of leading path parts in the archive above the data dirs
                     # e.g. entries "rollup/state-db/…" → 1 ; entries "state-db/…" → 0
```

### 7.A Snapshot is an archive in S3

Stream download → checksum → decompress → extract in **one pipeline**. Don't download the archive to disk first: the archive plus the extracted data won't both fit on the single 3.75 TB drive.

```bash
# 1) Faster S3 transfers (written to ~/.aws/config of the current user)
aws configure set default.s3.max_concurrent_requests 64
aws configure set default.s3.multipart_chunksize 256MB

# 2) Restore inside tmux so an SSH drop doesn't kill it (≈1–2 h per 3 TB)
tmux new -s restore
BUCKET=$(echo "$SNAP_OBJ" | cut -d/ -f3); KEY=$(echo "$SNAP_OBJ" | cut -d/ -f4-)
SIZE=$(aws s3api head-object --bucket "$BUCKET" --key "$KEY" --query ContentLength --output text)
aws s3 cp "$SNAP_OBJ" - \
  | pv -s "$SIZE" \
  | tee >(sha256sum > /tmp/snapshot.sha256) \
  | zstd -dc -T0 \
  | sudo tar -xf - -C "$ROLLUP_DIR" --strip-components="$STRIP" --numeric-owner
echo "exit codes: ${PIPESTATUS[*]}"          # every value must be 0, else wipe $ROLLUP_DIR and retry

# 3) Integrity (if the backup job publishes a checksum)
sleep 5; cut -d' ' -f1 /tmp/snapshot.sha256
aws s3 cp "${SNAP_OBJ}.sha256" - | cut -d' ' -f1         # must be identical
```
For other formats, replace `zstd -dc -T0`: `.tar.gz` → `pigz -dc`, `.tar.lz4` → `lz4 -dc`, plain `.tar` → remove that stage. zstd also checks its own frame checksums, so a non-zero exit code means the archive is corrupt.

Retry after a failure:
```bash
sudo find "$ROLLUP_DIR" -mindepth 1 -delete      # wipe the partial extract, then re-run step 2
```

### 7.B Snapshot is an EBS snapshot

Create a temporary volume from the snapshot in the new host's AZ, attach it, copy the data onto the NVMe, then delete the volume. Do this for NEW1 and NEW2 (one volume each, in parallel).

From an operator machine:
```bash
export NEW_ID=<instance id of NEW1 (then NEW2)>
AZ=$(aws ec2 describe-instances --instance-ids $NEW_ID --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' --output text)
VOL=$(aws ec2 create-volume --snapshot-id $SNAP_ID --availability-zone $AZ \
  --volume-type gp3 --iops 16000 --throughput 1000 \
  --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=$ENV-seq-v2-restore-tmp}]" \
  --query VolumeId --output text); echo $VOL
aws ec2 wait volume-available --volume-ids $VOL
aws ec2 attach-volume --volume-id $VOL --instance-id $NEW_ID --device /dev/sdf
aws ec2 wait volume-in-use --volume-ids $VOL
```

On the new host:
```bash
export VOL=<vol-… from above>
DEV=$(readlink -f /dev/disk/by-id/nvme-Amazon_Elastic_Block_Store_${VOL/-/})
lsblk -f $DEV                                   # check partitions / fs type / md member
sudo mkdir -p /mnt/snap
sudo mount -o nouuid $DEV /mnt/snap             # xfs; for ext4 drop -o nouuid. Mounted rw so the journal can replay; the volume is thrown away after.
                                                # partitioned → mount ${DEV}p1 ; md member(s) → sudo mdadm --assemble --run /dev/md9 <devs> then mount /dev/md9
sudo ls -la /mnt/snap                           # locate the rollup data dir
export SNAP_SRC=/mnt/snap/<path to rollup data dir>
tmux new -s restore
sudo rsync -aH --numeric-ids --info=progress2 $SNAP_SRC/ $ROLLUP_DIR/
sudo umount /mnt/snap
```
Volumes created from snapshots load lazily from S3, so the copy is slower than the gp3 figures (expect ~200–500 MB/s). Fast Snapshot Restore on `$SNAP_ID` in that AZ removes the lazy loading but is billed per hour. Optional.

Clean up from the operator machine:
```bash
aws ec2 detach-volume --volume-id $VOL && aws ec2 wait volume-available --volume-ids $VOL
aws ec2 delete-volume --volume-id $VOL
aws ec2 describe-volumes --filters "Name=tag:Name,Values=$ENV-seq-v2-restore-tmp" --query 'Volumes[].VolumeId'   # must be empty
```

### 7.C Snapshot kept on the backup node's own disk

Pull it straight from the backup node. That node keeps running: you read a **finished** snapshot, never its live data dir. First pause the snapshot job so it doesn't rotate or delete the file mid-copy:
```bash
ssh $SSH_USER@$BACKUP_NODE "sudo systemctl stop <snapshot>.timer"     # or comment out the cron entry
```
On each new host:
```bash
ssh -A $SSH_USER@$NEW1
tmux new -s restore
# archive file:
ssh $SSH_USER@$BACKUP_NODE "sudo cat <path/to/snapshot archive>" \
  | pv | tee >(sha256sum > /tmp/snapshot.sha256) | zstd -dc -T0 \
  | sudo tar -xf - -C "$ROLLUP_DIR" --strip-components="$STRIP" --numeric-owner
echo "exit codes: ${PIPESTATUS[*]}"
# …or a snapshot directory:
sudo --preserve-env=SSH_AUTH_SOCK rsync -aH --numeric-ids --info=progress2 \
  --rsync-path="sudo rsync" -e "ssh -o Compression=no" \
  $SSH_USER@$BACKUP_NODE:<snapshot dir>/ $ROLLUP_DIR/
```
When both hosts are done:
```bash
ssh $SSH_USER@$BACKUP_NODE "sudo systemctl start <snapshot>.timer"
```

### 7.D Fallback: copy from a stopped old replica
Use this only if no snapshot passes §2.11. Pick `SRC=<one of OLD_REPLICAS>`. Only this replica goes offline during the copy.

```bash
export SRC=<old replica ip>
# 1) Take SRC out of read routing (§9 commands), then stop it gracefully
ssh $SSH_USER@$SRC "sudo systemctl stop $SVC; systemctl is-active $SVC"      # must print 'inactive'
# 2) On NEW1: copy
ssh -A $SSH_USER@$NEW1
sudo --preserve-env=SSH_AUTH_SOCK rsync -aH --numeric-ids --delete --info=progress2 \
  --rsync-path="sudo rsync" -e "ssh -o Compression=no -o StrictHostKeyChecking=accept-new" \
  $SSH_USER@$SRC:$ROLLUP_DIR/ $ROLLUP_DIR/
# 3) Sizes must match exactly
ssh $SSH_USER@$SRC "sudo du -sb $ROLLUP_DIR"; sudo du -sb $ROLLUP_DIR
# 4) Restart SRC and put it back in routing
ssh $SSH_USER@$SRC "sudo systemctl start $SVC"
# 5) NEW1 → NEW2 (NEW1 not started yet, so its copy is consistent)
ssh -A $SSH_USER@$NEW2
sudo --preserve-env=SSH_AUTH_SOCK rsync -aH --numeric-ids --delete --info=progress2 \
  --rsync-path="sudo rsync" -e "ssh -o Compression=no" \
  $SSH_USER@$NEW1:$ROLLUP_DIR/ $ROLLUP_DIR/
```

### 7.E Fallback: sync from DA
Leave `$ROLLUP_DIR` empty. This only works if the DA node still serves the rollup's first DA height and the NOMT ht file (§2.3) fits. Can take hours to days.

### 7.1 After the restore (all variants), on each new host
```bash
sudo chown -R $SVC_USER: $ROLLUP_DIR
sudo ls -la $ROLLUP_DIR          # same top-level dirs as an old sequencer (§2.2), with no extra nesting level (wrong $STRIP)
sudo du -sh $ROLLUP_DIR/*        # sizes close to the old sequencer's / the manifest's
df -h $DATA                      # ≥ 20% free
grep -nE '^path|_db_path' $CFG   # [storage] paths point at the restored dirs
```
Only the **data** comes from the snapshot. The config (unique `node_id`, `node_role = "Replica"`) and the keys come from §5. If the snapshot contains config or key files, they're the backup node's, so don't use them.

---

## 8. Start the new nodes as `Replica` and wait for head

In `inventories/<dev>/group_vars/sequencers_v2.yml` set `rollup_service_enabled: true` and `rollup_service_state: started`, then:
```bash
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit sequencers_v2 --check --diff   # only the service state should change
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit sequencers_v2
ssh $SSH_USER@$NEW1 "journalctl -u $SVC -f"     # watch for errors; Ctrl-C when calm
```

### 8.0 Checks right after the first start from the snapshot
```bash
# The node must resume near the snapshot height, NOT from genesis
for h in $NEW_NODES; do printf '%-16s ' $h; curl -s http://$h:$API/ledger/slots/latest | jq .number; done   # ≈ $SNAP_SLOT, then rising
for h in $NEW_NODES; do curl -s http://$h:$API/rollup/sync-status | jq -c; done                          # Syncing → Synced

# The restored state must be the canonical one: root at the snapshot height matches the leader's
for h in $OLD_LEADER $NEW_NODES; do printf '%-16s ' $h; curl -s http://$h:$API/ledger/slots/$SNAP_SLOT | jq -r .state_root; done
```
**Stop the service and go back to §7** if:
- the height starts at 0 or 1: the node didn't find the restored data (wrong `$STRIP` or `[storage]` path)
- the service fails to open the DB: an inconsistent snapshot or a version mismatch (check `journalctl -u $SVC`)
- the state roots differ

To redo it: `sudo systemctl stop $SVC && sudo find $ROLLUP_DIR -mindepth 1 -delete`, then restore again.

Catch-up time is roughly the DA blocks since the snapshot ÷ the replay rate. That's minutes to a few hours for a snapshot < 24 h old.

### 8.1 Sync checks
```bash
# Height + root on every node (repeat until NEW* equal the leader and keep up)
for h in $OLD_LEADER $OLD_REPLICAS $NEW_NODES; do
  printf '%-16s ' $h; curl -s http://$h:$API/ledger/slots/latest | jq -c '{number, state_root}'
done

# Sync status and role of the new nodes
for h in $NEW_NODES; do curl -s http://$h:$API/rollup/sync-status | jq -c; curl -s http://$h:$API/sequencer/role | jq -c; done

# State roots must be identical at the same slot
N=$(( $(curl -s http://$OLD_LEADER:$API/ledger/slots/latest | jq .number) - 5 ))
for h in $OLD_LEADER $NEW_NODES; do printf '%-16s ' $h; curl -s http://$h:$API/ledger/slots/$N | jq -r .state_root; done

# Registered in the cluster with a reachable address (private ip:port, not 0.0.0.0)
psql "$PGURL" -c "SELECT node_id, address, now()-last_updated AS age FROM nodes ORDER BY node_id;"

# No root mismatch / panics
ssh $SSH_USER@$NEW1 "journalctl -u $SVC --since '-1h' | grep -Ei 'mismatch|panic|error' | tail -50"
```

### 8.2 Soak ≥ 24 h as replicas, under real load
```bash
ssh $SSH_USER@$NEW1 'free -g; ps -o rss,cmd -C <binary-name>; df -h '$DATA'; iostat -xm 5 3; mpstat 5 3'
```
PromQL to check after 24 h:
```promql
max_over_time(process_resident_memory_bytes{instance=~"<new hosts>"}[24h])
max_over_time(rate(node_disk_io_time_seconds_total{instance=~"<new hosts>",device=~"nvme.*"}[5m])[24h:])   # disk busy fraction
1 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle",instance=~"<new hosts>"}[5m]))
```
**Go to §9 if:** heights stay equal to the leader, state roots match, RSS < 75% of RAM, disk busy < 60%, CPU < 70%, no OOM (`journalctl -k | grep -i oom` is empty).

---

## 9. Add the new nodes to read routing

ALB/NLB example:
```bash
export TG_ARN=<target group arn>
aws elbv2 describe-target-groups --target-group-arns $TG_ARN --query 'TargetGroups[0].{port:Port,hc:HealthCheckPath}'
aws elbv2 register-targets --target-group-arn $TG_ARN --targets Id=$NEW1_ID,Port=$API Id=$NEW2_ID,Port=$API
echo "TG_TARGETS $TG_ARN $NEW1_ID $NEW2_ID" >> $LOG
aws elbv2 describe-target-health --target-group-arn $TG_ARN --output table   # wait for 'healthy'
```
(Proxy with node discovery: nothing to do, the nodes are in the `nodes` table.)
Watch the LB 5xx rate for 30 min.

---

## 10. Make NEW1 the only leader candidate

Only NEW1 becomes `DbElected` now, so the winner is known in advance (simplifies EIP/DNS moves). NEW2 stays `Replica` until §12.
In `inventories/<dev>/host_vars/<env>-sequencer-v2-0.yml` set `sequencer_node_role: DbElected`, then run on **that host only**:
```bash
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit <env>-sequencer-v2-0 --check --diff   # only node_role in $CFG changes
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit <env>-sequencer-v2-0
ssh $SSH_USER@$NEW1 "grep -n node_role $CFG; systemctl show $SVC -p ActiveEnterTimestamp"
```
If the role has no restart handler on config changes, restart it yourself: `ssh $SSH_USER@$NEW1 "sudo systemctl restart $SVC"`. Then:
```bash
ssh $SSH_USER@$NEW1 "journalctl -u $SVC --since '-2min' | grep -E 'DbElected node did not acquire leadership|PgSyncReplica'"
```
Expect *"DbElected node did not acquire leadership, running as PgSyncReplica"*. Then re-run the §8.1 height check until NEW1 is back at head.

---

## 11. Leader cutover

### 11.1 Retire the old replicas first
For each old replica:
```bash
aws elbv2 deregister-targets --target-group-arn $TG_ARN --targets Id=<old replica id>,Port=$API
aws elbv2 describe-target-health --target-group-arn $TG_ARN --output table   # wait until 'draining' finishes
ssh $SSH_USER@<old replica> "sudo systemctl disable --now $SVC && sudo systemctl mask $SVC"
psql "$PGURL" -c "SELECT node_id FROM nodes ORDER BY node_id;"     # old replica gone
```

### 11.2 Last checks (all must be true)
```bash
psql "$PGURL" -c "SELECT node_id, now()-leader_acquired_at AS leading_for FROM sequencer_leader;"   # old leader, leading_for > 10s
for h in $OLD_LEADER $NEW_NODES; do printf '%-16s ' $h; curl -s http://$h:$API/ledger/slots/latest | jq .number; done  # equal
ssh $SSH_USER@$NEW1 "systemctl show $SVC -p Restart; grep node_role $CFG"   # Restart=always, DbElected
```

### 11.3 Open 3 terminals to watch the cutover
```bash
# T1: leadership table
watch -n0.5 "psql \"$PGURL\" -Atc 'SELECT node_id, last_updated, leader_acquired_at FROM sequencer_leader'"
# T2: NEW1 logs
ssh $SSH_USER@$NEW1 "journalctl -u $SVC -f" | grep -Ei 'leader|BatchProducer|heartbeat|error'
# T3: write-path probe
while true; do printf '%s ' "$(date +%T.%N | cut -c1-12)"; curl -s -o /dev/null -m 1 -w '%{http_code}\n' http://$WRITE_ENDPOINT/sequencer/ready; sleep 0.2; done
```

### 11.4 Cutover
```bash
# Stop old leader gracefully (SIGTERM; never kill -9) and prevent any restart (Restart=, Ansible, CI)
ssh $SSH_USER@$OLD_LEADER "sudo systemctl disable --now $SVC && sudo systemctl mask $SVC && systemctl is-active $SVC"   # 'inactive'
```
What should happen within seconds:
1. T1: `node_id` flips to `<env>-seq-v2-0`.
2. T2: *"Replica acquired leadership! Exiting to restart as leader."* → systemd restart → *"DbElected node acquired leadership, running as BatchProducer"*.
3. T3: a short run of non-200 responses, then 200 again.

Move the write endpoint now **if** it's an EIP, DNS, or LB (§2.7 b/c/d):
```bash
# (b) Elastic IP
aws ec2 associate-address --allocation-id <eipalloc-…> --instance-id $NEW1_ID --allow-reassociation
# (c) Route53
aws route53 change-resource-record-sets --hosted-zone-id <ZONE> --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"<write.fqdn>","Type":"A","TTL":30,"ResourceRecords":[{"Value":"'$NEW1'"}]}}]}'
# (d) LB
aws elbv2 deregister-targets --target-group-arn $TG_ARN --targets Id=$OLD_LEADER_ID,Port=$API
```
From now until §13, **never run the playbook against the old hosts** (always `--limit sequencers_v2`). A role that enforces `state: started` would try to bring the old leader back.

### 11.5 Post-cutover verification
```bash
curl -s http://$NEW1:$API/sequencer/role | jq
psql "$PGURL" -c "SELECT node_id, last_updated FROM sequencer_leader;"
for i in 1 2; do for h in $NEW_NODES; do printf '%-16s ' $h; curl -s http://$h:$API/ledger/slots/latest | jq .number; done; sleep 15; done   # advancing, NEW2 follows
ssh $SSH_USER@$NEW1 "journalctl -u $SVC --since '-5min' | grep -Ei 'blob|submit|batch' | tail -20"   # batches published to DA
```
Submit a test tx with your usual tool or script, then:
```bash
curl -s http://$WRITE_ENDPOINT/sequencer/txs/<tx_hash>/status | jq
```

---

## 12. Make NEW2 a standby candidate
In `inventories/<dev>/host_vars/<env>-sequencer-v2-1.yml` set `sequencer_node_role: DbElected`, then:
```bash
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit <env>-sequencer-v2-1 --check --diff
ansible-playbook -i inventories/<dev> playbooks/<sequencer>.yml --limit <env>-sequencer-v2-1
ssh $SSH_USER@$NEW2 "journalctl -u $SVC --since '-2min' | grep -E 'did not acquire leadership'"   # restart by hand first if the role has no handler
```
Commit all inventory, host_vars, group_vars and role changes to the Ansible repo.

---

## 13. Soak (24–48 h), then decommission

During the soak, **leave the old instances running with the service masked**. Stopping them wipes the instance store and makes a quick rollback impossible.

After the soak (all AWS commands are run by a human):

1. **Confirm exactly which instances you're about to delete.** They must be the 12xlarge ones, and none of the v2 IDs may be in the list:
   ```bash
   source $LOG
   aws ec2 describe-instances --instance-ids $OLD_IDS \
     --query 'Reservations[].Instances[].{id:InstanceId,name:Tags[?Key==`Name`]|[0].Value,type:InstanceType,ip:PrivateIpAddress,state:State.Name}' --output table
   for id in $OLD_IDS; do [ "$id" = "$NEW1_ID" ] || [ "$id" = "$NEW2_ID" ] && echo "STOP: $id is a v2 instance"; done
   ```
2. **List what's attached to them**, so nothing is orphaned or left billing:
   ```bash
   aws ec2 describe-addresses --filters Name=instance-id,Values=${OLD_IDS// /,} --query 'Addresses[].{ip:PublicIp,alloc:AllocationId,inst:InstanceId}'
   aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=${OLD_IDS// /,} \
     --query 'Volumes[].{id:VolumeId,inst:Attachments[0].InstanceId,delOnTerm:Attachments[0].DeleteOnTermination}' --output table
   aws elbv2 describe-target-health --target-group-arn $TG_ARN --query 'TargetHealthDescriptions[].Target.Id'   # old ids must be gone (§11)
   ```
3. **Remove termination protection, then terminate** (try `--dry-run` first):
   ```bash
   for id in $OLD_IDS; do aws ec2 modify-instance-attribute --instance-id $id --no-disable-api-termination; done
   aws ec2 terminate-instances --instance-ids $OLD_IDS --dry-run
   aws ec2 terminate-instances --instance-ids $OLD_IDS
   aws ec2 wait instance-terminated --instance-ids $OLD_IDS
   echo "TERMINATED $OLD_IDS $(date -u +%FT%TZ)" >> $LOG
   ```
4. **Clean up the leftovers found in step 2:**
   ```bash
   aws ec2 release-address --allocation-id <eipalloc-… of the old hosts, only if NOT moved to NEW1 in §11.4>
   aws ec2 delete-volume --volume-id <vol-… that had DeleteOnTermination=false>
   aws ec2 revoke-security-group-ingress --group-id <sg> --ip-permissions "IpProtocol=tcp,FromPort=<port>,ToPort=<port>,IpRanges=[{CidrIp=<old ip>/32}]"   # old-IP rules only
   aws iam delete-role-policy --role-name $ROLE --policy-name $ENV-sequencer-snapshot-read                                                         # §3.4
   ```
5. Clean up the rest:
   - Ansible: remove the old hosts from the inventory, merge `group_vars/sequencers_v2.yml` into the parent group, drop the `sequencers_v2` group, and remove the old fixed-3-disk RAID code. Re-run `ansible-playbook … --check --diff` on the whole group: it must show no changes.
   - Keep `$LOG` (instance IDs, SG rules, IAM policy, TG targets). Later, the Terraform codebase imports these resources from it instead of recreating them.
   - Prometheus scrape targets and file_sd, Grafana dashboards, alert rules that name hosts or instance IDs. Check with `promtool check config <prometheus.yml>` and `promtool check rules <rules.yml>`.
   - **Disk alerts:** the same % threshold now gives ⅓ of the absolute headroom. Re-tune thresholds and `for:`, or alert on `predict_linear(...[6h], 3*86400) < 0`.
   - SSH `known_hosts`, docs, and runbooks that mention old IPs.
6. Re-enable automatic deploys to dev, pointed at the new hosts.

---

## 14. Rollback

| When | Action |
|---|---|
| Before §11.4 | Stop and mask the service on the new nodes and deregister them from the LB (`aws elbv2 deregister-targets … Id=$NEW1_ID … Id=$NEW2_ID`). Then remove them: `aws ec2 modify-instance-attribute --instance-id <id> --no-disable-api-termination` for each, followed by `aws ec2 terminate-instances --instance-ids $NEW1_ID $NEW2_ID`. Revert the §3.3/§3.4 additions listed in `$LOG` and remove `sequencers_v2` from the Ansible inventory. Old cluster untouched. |
| After §11.4, new leader healthy but you want to go back | On OLD_LEADER: set `node_role = "DbElected"` (**not** `Leader`), `sudo systemctl unmask $SVC && sudo systemctl enable --now $SVC`, wait until it's at head (§8.1). Make sure it has `Restart=always` (§2.8). Then `sudo systemctl disable --now $SVC && sudo systemctl mask $SVC` on NEW2 and then NEW1. The old node takes over within ~1 s. Move the EIP/DNS back. |
| After §11.4, new leader crash-looping (OOM, disk, …) | 1) `sudo systemctl disable --now $SVC && sudo systemctl mask $SVC` on NEW1 **and** NEW2 (stops it from re-acquiring leadership on every restart). 2) On OLD_LEADER: set `DbElected`, unmask, `enable --now`. It acquires leadership after ~1 s, exits, and restarts as leader. 3) Move the EIP/DNS back. 4) Diagnose with `journalctl -u $SVC`, `journalctl -k \| grep -i oom`, `df -h`. |

---

## 15. Troubleshooting

- **Log says "Leadership lost! Another node has taken over."**: a second leader-capable node started. Find it with `SELECT * FROM nodes`, then mask it.
- **New node never reaches head:** check DA reachability, Postgres reachability, and IO saturation (`iostat -xm 5`). If disk-bound, the 4xlarge is too small (go to 8xlarge).
- **NEW1 doesn't take over after the old leader stops:** it isn't `DbElected`, its `Restart=` isn't `always`, or the old leader is still heartbeating (check it's really inactive and that nothing else uses its `node_id`).
- **Address in `nodes` is wrong or unreachable:** fix the bind host or port in the config. The proxy routes using that address.

---

## 16. Order of work and time estimate (per environment)

| Step | Duration |
|---|---|
| §2 pre-flight (incl. §2.11 snapshot checks) | 1.5 h |
| §3–§6 AWS CLI launch, Ansible, verify | 2–4 h |
| §7 snapshot restore (both hosts in parallel) | ~1–2 h per 3 TB (S3 stream); slower for EBS lazy-load |
| §8.0 catch-up from snapshot height | minutes to a few hours |
| §8 sync + soak | ≥ 24 h |
| §9–§12 routing + cutover | 1 h (write pause: seconds) |
| §13 soak + decommission | 24–48 h, then 1 h |

Repeat §1–§13 for each dev environment, one at a time.

## Definition of done
- Every dev env runs only i8g.4xlarge sequencers. The single NVMe is mounted at the original mount point by UUID with `nofail`, and there's no md device.
- `sequencer_leader` holds a v2 node. All v2 nodes are `DbElected`, registered in `nodes`, at head, with matching state roots. Tx submission, soft confirmations, and DA batch publishing work.
- The only client-visible impact was the seconds-long write pause at cutover.
- The Ansible repo contains the new hosts in the inventory, the device-count-aware disk provisioning, the restart policy, and the tuned config. The old hosts are gone from the inventory.
- The old instances are terminated, with no orphaned EIPs, volumes, SG rules or IAM policies. `$LOG` lists every AWS resource created by hand, ready for a later Terraform import. Monitoring and alerts are updated.
- After 48 h: RSS < 75% of RAM, disk < 80% used, no OOM or unexpected restarts.
