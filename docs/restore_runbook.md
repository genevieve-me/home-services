# Restic Restore Runbook

This document provides a step-by-step guide to restoring service data from a restic backup. The process involves restoring files to a temporary location, stopping services, copying data, importing volumes and databases, and then restarting services.

**IMPORTANT:** This is a destructive operation. Read through all the steps carefully before proceeding. It is recommended to perform a dry run or test the restore process in a non-critical environment first.

## Prerequisites

1.  **Restic Installed:** The `restic` command-line tool must be installed on the server.
2.  **SFTP Access:** You must have network access and credentials for the SFTP backup repository (`de3946@de3946.rsync.net`).
3.  **Restic Password:** The `RESTIC_PASSWORD` environment variable must be set to the correct password for your restic repository.
    ```bash
    export RESTIC_PASSWORD='your-restic-repository-password'
    ```

For complete safety, verify that there are no locks on the restic repository.
If there are stale locks (e.g., from a failed backup) they can be removed once the associated
process is confirmed to be ended with `restic unlock`.

## Step 1: Identify the Backup Snapshot

First, list the available snapshots in the repository to identify which one you want to restore from. Usually, this will be the most recent one.

```bash
restic -r sftp:de3946@de3946.rsync.net:restic snapshots
```

Take note of the `ID` of the snapshot you wish to restore. You can use the ID itself or the keyword `latest` for the most recent snapshot in the next step.

## Step 2: Restore Data to a Temporary Location

For safety, restore the entire backup to a temporary directory. This allows you to inspect the restored files before moving them to their final destination and avoids accidentally overwriting current data.

```bash
# This command restores the latest snapshot. Replace 'latest' with a specific snapshot ID if needed.
restic -r sftp:de3946@de3946.rsync.net:restic restore latest --target /tmp/restore
```

After this command completes, the backed-up data (including file directories and the `/data/dumps` directory containing database and volume backups) will be available in `/tmp/restore`.

### Alternative: Partial Restores

If you only need to restore a specific file or directory instead of the entire backup, you can perform a partial restore. This is much faster and less disruptive.

First, use `restic find` to locate the exact path of the file or directory you need within the backup snapshots. You can search for a pattern across all snapshots.

```bash
# Example: Find the beets database file in all snapshots
restic -r sftp:de3946@de3946.rsync.net:restic find beets.db

# Example: Find a specific Nextcloud configuration file
restic -r sftp:de3946@de3946.rsync.net:restic find config.php
```

The output will show you the full path(s) to the file within the backup, like `/data/beets.db`.

Once you have the full path, use `restic restore` with the `--include` flag to restore only that item.

```bash
# Example: Restore only the beets.db file from the latest snapshot
restic -r sftp:de3946@de3946.rsync.net:restic restore latest --target /tmp/restore --include "/data/beets.db"
```

The file will be restored to `/tmp/restore/data/beets.db`.
From there, you can inspect it and manually copy it to its final destination.

## Step 3: Stop Services

Stop all running services to prevent data corruption or conflicts while you copy files and restore databases.

```bash
# Add or remove services as needed
systemctl --user stop immich.pod miniflux.pod paperless.pod koito.pod navidrome.pod nextcloud-aio.service
```

## Step 4: Restore File & Directory Data

Use `rsync` to copy the restored file-based data from the temporary directory to the primary `/data` directory. `rsync` is efficient and will only copy what's necessary.

```bash
# This command will overwrite existing files with the restored versions.
rsync -avh /tmp/restore/data/ /data/
```

## Step 5: Restore Podman Volumes

Import the podman volumes from the `.tar` files located in the restored dumps directory.

**Note:** Ensure the corresponding pods/containers are stopped before running the import. The volume must not be in use.

```bash
# The import command may fail if a volume with the same name already exists.
# You may need to remove the existing volume first with 'podman volume rm <volume_name>'
podman volume import koito.volume /tmp/restore/data/dumps/koito-volume.tar
podman volume import nextcloud_aio_mastercontainer /tmp/restore/data/dumps/nextcloud_aio_mastercontainer-volume.tar
```

## Step 6: Restore Databases

Restore the PostgreSQL databases by piping the SQL dump files into `psql` within the respective database containers.

Note: The database container must be running for the `podman exec` command to work.
If you stopped the entire pod, you may need to start just the database container.

As per the last step, recreate database volumes if necessary.

```bash
# Restore Miniflux Database
podman exec -i miniflux-pg psql -U miniflux < /tmp/restore/data/dumps/miniflux-db-backup.sql

# Restore Paperless Database
podman exec -i paperless_ngx_db psql -U paperless < /tmp/restore/data/dumps/paperless-db-backup.sql

# Restore Koito Database
podman exec -i koito-psql psql -U postgres < /tmp/restore/data/dumps/koito-db-backup.sql
```

You can also use a temporary container (similar to how a major Postgres version upgrade might be done):

```
podman run --name miniflux-db-new --rm -d --volume miniflux-db:/var/lib/postgresql/data \
  -e POSTGRES_USER=miniflux --secret miniflux-db-password,target=POSTGRES_PASSWORD \
  -e POSTGRES_DB=miniflux \
  docker.io/library/postgres:17-alpine -- psql -U miniflux < backup.sql
```

## Step 7: Restart Services and Verify

Once all data has been restored, restart the services.

```bash
systemctl --user start immich.pod miniflux.pod paperless.pod koito.pod navidrome.pod nextcloud-aio.service
```

Check the status and logs of the services to ensure they started correctly.

```bash
# Example: Check the miniflux pod or database logs
journalctl --user -u miniflux.pod -f
podman logs --since=5m miniflux-db
```

Finally, access the web frontends for your services to verify that the data has been restored correctly.


## Step 8: Clean Up

After you have confirmed that the restore was successful, you can remove the temporary restore directory.

```bash
rm -rf /tmp/restore
```
