---
tags:
  - Other
---

# ZFS Send/Receive
`zfs send` from main server and `zfs receive` on backup server

## General Facts For Consideration
1. Both `zfs send` and `zfs receive` operations require plenty of permissions, which can be assigned to an unprivileged user via `zfs allow` (we should not run the script as root on either end as most guides suggest)
2. If the source dataset is auto-mounted, it will be attempted to be mounted on the receiving end, which can be a problem
3. If a backup dataset is mounted, there is a possibility (highly likely if `atime` is on) that it will be modified on the receiving end, semi-breaking future incremental updates
4. For scripted operations, one would need to have an ssh key without passphrase on either the main or the backup server, for the other one.
5. Unlike rsync, which compares source and destination data for diff calculation, `zfs send` only compares 2 snapshots on the source end to calculate diff, meaning the source and destination must contain a common snapshot, which should be used as the base of the diff on the sending end
6. All properties of the source dataset will be set on the backup dataset as is, which means any property that is `inherited` on the source will also be set as `inherited` on the backup, so the actual value may differ (such as the mountpoint)
7. If the machine with the ssh key gets compromised, the other machine is also compromised due to no ssh key passphrase
8. If the source machine gets compromised (ransomwared), the changes can propagate to backups
9. If `zfs allow` has the `destroy` permission, a compromised machine can potentially destroy all zfs data including the pools, datasets and the snapshots.
10. Initial `send` requires that the destination does not have the existing dataset (you can force but I'd rather keep it clean and simple)
11. Incremental `send` requires the destination state matches the base snapshot used in `send`

## Plan
- Due to `1.` and `4.` above, we will use a dedicated unprivileged linux user `zfsbackup` (not in sudoers) on both the sending and the receiving ends
  - On the sending end, the user will need the `zfs allow` perms `create,mount,rename,send,snapshot` (`create` and `mount` are there because they are required by others) (we do not assign the `destroy` perm due to `9.`)
  - On the receiving end, the user will need the `zfs allow` perms `atime,canmount,create,mount,readonly,receive,rename` (`create` and `mount` are there because they are required by others) (we do not assign the `destroy` perm due to `9.`)
  - Allow perms are per user per dataset and are inherited. Existing perms can be checked via `zfs allow <pool>/<dataset>`
- To prevent issues due to `2.` and `3.`, we will use the options `-o canmount=noauto -u -o readonly=on` because the first 2 will prevent mounting on the receiving end and the 3rd will prevent modifications (for good measure).
  - We can also add `-o atime=off` if we do want to mount them, but if it's read only, that should not be necessary.
  - These options are overridden permanently on the receiving end. If we ever have to recover from these backups, we need to kee in mind that our backup datasets have altered properties, which may need to be reverted after a restore. It is prudent to keep these alterations to a minimum so a potential restore doesn't get too complicated.
- Picking the receiving end as the initator and holder of the ssh key with no passphrase because that machine is a dedicated backup machine, has no public facing services and thus less likely to be compromised.
- On the receiving end, we create a new encrypted dataset dedicated to holding backups `pool/remotebackups` and set the options `readonly:on` and `canmount=noauto`. Keep in mind that while the `readonly` setting is inherited, the `canmount` setting is not.
- Because we have multiple pools on the source machine (one for the OS on NVMEs and one for the data/media on HDDs), we'll replicate the same structure on the receiving end to make things simpler:
  - `zarray/Books` on the source will be backed up to `pool/remotebackups/zarray/Books` on the receiving end
  - `zroot/ROOT/debian` on the source will be backed up to `pool/remotebackups/zroot/ROOT/debian` on the receiving end
- Since we have to specifically define the source snapshot on the sending end during incremental, and that snapshot state has to match the current state of the backup dataset, we will hardcode snapshot names and make assumptions based on that
  - We could label the snapshot with date stamps that would match on the sending and the receiving ends, but that would require parsing zfs list outputs and/or keeping a file or db with previous run info. That complicates things and is error prone (hardcoding the snapshot names makes it safer for a cron script to delete them as we reduce the risk of accidentally deleting the wrong snapshot or worse, a dataset)
  - It's much simpler to rely on the following format for snapshot names:
    - `remotebackup-snapshot` for the snapshot that will be sent and received
    - `remotebackup-snapshot-1` for the snapshot that will be used as the diff base
  - Prior to send, we rename the snapshots `remotebackup-snapshot` as `remotebackup-snapshot-1` on both ends, create a new snapshot `remotebackup-snapshot` on the sending end, and send that with the diff base `remotebackup-snapshot-1`
  - At the end of the send operation, both ends will have the matching snapshots `remotebackup-snapshot` and `remotebackup-snapshot-1`
  - Nightly, we have to delete the `remotebackup-snapshot-1` snapshots on both ends but that has to be done by a privileged user like root cron, otherwise the rename will fail due to existing snapshot with that name
  - At the start of each send operation, we will have only the `remotebackup-snapshot` on both ends
  - If the structure and naming of the snapshots gets messed up due to errors when running the script, we'll have the script notify us so we can take manual action

## Specific Steps

### Initial sync
1. Let's assume we're syncing `zarray/Books` on main machine to `pool/remotebackups/zarray/Books` on the backup machine
2. Add permissions to source dataset on main machine: `sudo zfs allow zfsbackup create,mount,rename,send,snapshot zarray/Books`
3. Create the parent pool/dataset on the backup machine: `sudo zfs create -o canmount=noauto -o readonly=on pool/remotebackups/zarray`
4. Add permissions to backup parent pool/dataset on backup machine: `sudo zfs allow zfsbackup atime,canmount,create,mount,readonly,receive,rename pool/remotebackups/zarray`
5. Log into `zfsbackup` on the backup machine: `su zfsbackup`
6. Create snapshot on the source dataset via ssh: `ssh zfsbackup@MAINSERVERIP zfs snapshot zarray/Books@remotebackup-snapshot`
7. Perform initial sync via ssh: `ssh zfsbackup@MAINSERVERIP zfs send -v zarray/Books@remotebackup-snapshot | zfs receive -o canmount=noauto -u -o readonly=true pool/remotebackups/zarray/Books`

### Subsequent Incremental Sync
1. Log into `zfsbackup` on the backup machine: `su zfsbackup`
2. Rotate snapshot on the source dataset via ssh: `ssh zfsbackup@MAINSERVERIP zfs rename zarray/Books@remotebackup-snapshot zarray/Books@remotebackup-snapshot-1`
3. Create new snapshot on the source dataset via ssh: `ssh zfsbackup@MAINSERVERIP zfs snapshot zarray/Books@remotebackup-snapshot`
4. Rotate snapshot on the backup dataset: `zfs rename pool/remotebackups/zarray/Books@remotebackup-snapshot pool/remotebackups/zarray/Books@remotebackup-snapshot-1`
5. Perform incremental sync via ssh: `ssh zfsbackup@MAINSERVERIP zfs send -vi zarray/Books@remotebackup-snapshot-1 zarray/Books@remotebackup-snapshot | zfs receive -o canmount=noauto -u -o readonly=true pool/remotebackups/zarray/Books`

### Clean Up After Sync
1. On the main machine edit the root crontab via `sudo crontab -e` and add an entry to delete the rotated snapshot: `17 0 * * * zfs destroy zarray/Books@remotebackup-snapshot-1`
2. On the backup machine edit the root crontab via `sudo crontab -e` and add an entry to delete the rotated snapshot: `17 0 * * * zfs destroy pool/remotebackups/zarray/Books@remotebackup-snapshot-1`

## My Ugly Script

```sh
#!/bin/bash

# Edit the below values

# Set Source Datasets
SOURCEDATASETS='zarray/Books zarray/Misc zroot/home zroot/ROOT/debian'
# Set Local Backup Base
BACKUPBASE='pool/remotebackups'
# Set remote server
REMOTESERVER='zfsbackup@<SERVERIP>'

# notify via self hosted server or if that fails, via the public instance
fn_notify () {
  curl -Lfs -H "Title: $1" -H "Authorization: Bearer <my token>" -d "$2"  https://ntfy.<mydomain>/zfshost || \
    curl -Ls -H "Title: $1" -d "$2" https://ntfy.sh/<my custom topic>
}

# Do not make edits below this line

echo "**** Starting sync on $(date)"
# Check to make sure remote server is up and connectable
if ssh -o ConnectTimeout=10 "${REMOTESERVER}" true; then
  echo "[Success] Remote server is up and connectable"
else
  echo "[Failed] Remote server does not seem to be reachable. Aborting zfs sync."
  fn_notify "Zima Zfs Sync Failed" "Remote server does not seem to be reachable. Aborting zfs sync."
  exit 1
fi

for SRC in ${SOURCEDATASETS}; do
  echo "**********************************"
  echo "**********************************"
  echo "**** Processing dataset ${SRC}"
  # check to make sure the latest snapshot exists on local
  if zfs list -t snapshot "${BACKUPBASE}/${SRC}" | grep -q "@remotebackup-snapshot "; then
    echo "[Success] ${BACKUPBASE}/${SRC}@remotebackup-snapshot exists on local as expected"
  else
    echo "[Failed] ${BACKUPBASE}/${SRC}@remotebackup-snapshot does not exist on local. Skipping backup of ${SRC}"
    fn_notify "Zima Zfs Sync Failure" "${BACKUPBASE}/${SRC}@remotebackup-snapshot does not exist on local. Skipping backup of ${SRC}"
    continue
  fi
  # check to make sure the renamed snapshot does not exist on local
  if ! zfs list -t snapshot "${BACKUPBASE}/${SRC}" | grep -q "@remotebackup-snapshot-1 "; then
    echo "[Success] ${BACKUPBASE}/${SRC}@remotebackup-snapshot-1 does not exist on local as expected"
  else
    echo "[Failed] ${BACKUPBASE}/${SRC}@remotebackup-snapshot-1 already exists on local. Skipping backup of ${SRC}"
    fn_notify "Zima Zfs Sync Failure" "${BACKUPBASE}/${SRC}@remotebackup-snapshot-1 already exists on local. Skipping backup of ${SRC}"
    continue
  fi
  # check to make sure the latest snapshot exists on remote
  if ssh "${REMOTESERVER}" "zfs list -t snapshot ${SRC} | grep -q \"@remotebackup-snapshot \""; then
    echo "[Success] ${SRC}@remotebackup-snapshot exists on remote as expected"
  else
    echo "[Failed] ${SRC}@remotebackup-snapshot does not exist on remote. Skipping backup of ${SRC}"
    fn_notify "Zima Zfs Sync Failure" "${SRC}@remotebackup-snapshot does not exist on remote. Skipping backup of ${SRC}"
    continue
  fi
  # check to make sure the renamed snapshot does not exist
  if ! ssh "${REMOTESERVER}" "zfs list -t snapshot ${SRC} | grep -q \"@remotebackup-snapshot-1 \""; then
    echo "[Success] ${SRC}@remotebackup-snapshot-1 does not exist on remote as expected"
  else
    echo "[Failed] ${SRC}@remotebackup-snapshot-1 already exists on remote. Skipping backup of ${SRC}"
    fn_notify "Zima Zfs Sync Failure" "${SRC}@remotebackup-snapshot-1 already exists on remote. Skipping backup of ${SRC}"
    continue
  fi
  # Rename snapshots on both local and remote
  echo "Renaming snapshot on the local"
  zfs rename "${BACKUPBASE}/${SRC}@remotebackup-snapshot" "${BACKUPBASE}/${SRC}@remotebackup-snapshot-1"
  if [ "$?" -ne 0 ]; then
    echo "[Failed] Renaming snapshot on the local failed. Skipping backup of $SRC"
    fn_notify "Zima Zfs Sync Failure" "Renaming snapshot on the local failed. Skipping backup of $SRC"
    continue
  fi
  echo "Renaming snapshot on the remote"
  ssh "${REMOTESERVER}" "zfs rename ${SRC}@remotebackup-snapshot ${SRC}@remotebackup-snapshot-1"
  if [ "$?" -ne 0 ]; then
    echo "[Failed] Renaming snapshot on the remote failed. Skipping backup of $SRC"
    fn_notify "Zima Zfs Sync Failure" "Renaming snapshot on the remote failed. Skipping backup of ${SRC}"
    continue
  fi
  # create snapshot on remote
  echo "Creating snapshot on the remote"
  ssh "${REMOTESERVER}" "zfs snapshot ${SRC}@remotebackup-snapshot"
  if [ "$?" -ne 0 ]; then
    echo "[Failed] Creating snapshot on the remote failed. Skipping backup of $SRC"
    fn_notify "Zima Zfs Sync Failure" "Creating snapshot on the remote failed. Skipping backup of ${SRC}"
    continue
  fi
  # Initiate zfs send
  echo "Initiating zfs send for ${SRC}"
  ssh "${REMOTESERVER}" "zfs send -vi ${SRC}@remotebackup-snapshot-1 ${SRC}@remotebackup-snapshot" | zfs receive -o readonly=on -o canmount=noauto -u "${BACKUPBASE}/${SRC}"
  if [ "$?" -ne 0 ]; then
    echo "[Failed] Sending snapshot failed. Skipping backup of $SRC"
    fn_notify "Zima Zfs Sync Failure" "Sending snapshot failed. Skipping backup of ${SRC}"
    continue
  fi
  SRC_COMPLETED="${SRC} ${SRC_COMPLETED}"
  echo "[Success] Zfs send for ${SRC} completed"
  echo "**********************************"
  echo "**********************************"
  echo "**********************************"
done
if [ -n "${SRC_COMPLETED}" ]; then
  fn_notify "Zima Zfs Sync Completed" "The following datasets were successfully synced: ${SRC_COMPLETED}"
fi
echo "Ended sync on $(date)"
```

This script is not pretty, has plenty of duplicate commands and statements. I prefer function over form. My primary goals with this script are error detection and verbosity. Because I rely on hardcoded snapshot names, and because zfs send receive needs the snapshots perfectly synced on both sides, I really want to make sure I preserve the snapshots. Otherwise, incremental sync will break and I'll need to do a full sync again. If anything goes wrong, this script will immediately halt, log the error and notify via ntfy so I can go in and fix it manually.

I have this script set to run weekly via the `zfsbackup` user's cron on the backup machine. I also have the root cron scripts running nightly to delete the rotated snapshots (named `@remotebackup-snapshot-1`) for each source and backup dataset on both machines.
