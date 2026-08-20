# How to Back Up TrueNAS to Backblaze

## 1. Configure the rclone Remote

```console
> rclone config
```

```
No remotes found, make a new one?
n) New remote
s) Set configuration password
q) Quit config
n/s/q> n
```

**Enter a name for the new remote:**

```
Enter name for new remote.
name> Immich
```

**Choose the storage type:**

```
Option Storage.
Type of storage to configure.
Choose a number from below, or type in your own value.

 5 / Backblaze B2
   \ (b2)
```

**Enter your account ID / Application Key ID** (from the Backblaze B2 portal):

```
Option account.
Account ID or Application Key ID.
Enter a value.
account> xxxxxxxx
```

**Enter your application key** (from the Backblaze B2 portal):

```
Option key.
Application Key.
Enter a value.
key> xxxxx
```

**Confirm the remote was created:**

```
Current remotes:

Name                 Type
====                 ====
Immich               b2
```

## 2. Verify the Configuration

```console
> rclone config show
[Immich]
type = b2
account = xxxxxx
key = xxxxxx
```

## 3. Inspect the Bucket

List all directories / containers / buckets in the path:

```console
> rclone lsd Immich:
 -1 2000-01-01 00:00:00        -1 ImmichBackup-Bucket
```

List files in the bucket (empty for now):

```console
> rclone ls Immich:ImmichBackup-Bucket
```

> **Note:** You don't need to back up the Postgres database here — this is already handled in the Immich GUI settings. Database backups can be found in `/mnt/Red/immich/data/backups`.

## 4. Run the Backup

A cron job has been created in the TrueNAS GUI, run as the `admin` user:

```console
> rclone sync /mnt/Red/immich/data Immich:Wills-Immich-Bucket \
    --config /mnt/Red/immich/rclone.conf \
    --transfers 10 \
    --fast-list \
    --exclude "thumbs/**" \
    --exclude "encoded-video/**" \
    --log-file /mnt/Red/immich/rclone-backup.log \
    --log-level INFO
```

## 5. Restore Backups

```bash
docker-compose down -v   # CAUTION! Deletes all Immich data to start from scratch.
docker-compose pull      # Update to the latest version of Immich (if desired)
docker-compose create    # Create Docker containers for Immich apps without running them.

docker start immich_postgres   # Start Postgres server
sleep 10                       # Wait for Postgres server to start up

# Restore the database backup
gunzip < "/path/to/backup/dump.sql.gz" | docker exec -i immich_postgres psql -U postgres -d immich

docker-compose up -d     # Start remainder of Immich apps
```
