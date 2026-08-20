How to backup Truenas to backblaze
> rclone config

No remotes found, make a new one?
n) New remote
s) Set configuration password
q) Quit config
n/s/q> n

Enter name for new remote.
name> Immich

Option Storage.
Type of storage to configure.
Choose a number from below, or type in your own value.

 5 / Backblaze B2
   \ (b2)

Option account.
Account ID or Application Key ID.
Enter a value.
account> xxxxxxxx < Enter Key ID from Backblaze B2 portal

Option key.
Application Key.
Enter a value.
key> xxxxx < Enter application key from Backblaze B2 portal

Current remotes:

Name                 Type
====                 ====
Immich               b2

# Verify configuration settings
> rclone config show
[Immich]
type = b2
account = xxxxxx
key = xxxxxx

# List all directories/containers/buckets in the path
> rclone lsd Immich:
 -1 2000-01-01 00:00:00        -1 ImmichBackup-Bucket

# List files in the bucket, in this case it is empty for now.
> rclone ls Immich:ImmichBackup-Bucket

# Don't need to backup postgres database as this is already done in Immich GUI settings
# Database backups can be found in /mnt/Red/immich/data/backups

# cron job has been created in Truenas GUI, run as admin user
> rclone sync /mnt/Red/immich/data Immich:Wills-Immich-Bucket --config /mnt/Red/immich/rclone.conf --transfers 10 --fast-list --exclude "thumbs/**" --exclude "encoded-video/**" --log-file /mnt/Red/immich/rclone-backup.log --log-level INFO

# Restoring Backups
docker-compose down -v  # CAUTION! Deletes all Immich data to start from scratch.
docker-compose pull     # Update to the latest version of Immich (if desired)
docker-compose create   # Create Docker containers for Immich apps without running them.
docker start immich_postgres    # Start Postgres server
sleep 10    # Wait for Postgres server to start up
gunzip < "/path/to/backup/dump.sql.gz" | docker exec -i immich_postgres psql -U postgres -d immich    # Restore Backup
docker-compose up -d    # Start remainder of Immich apps
