Hi

To create auto-backup (on on your backup node), one would need to execute a script using cron job

cron job could be created like this `sudo crontab -e`
```
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
0  12 *  *  * near      <WORK_DIR>/backup.sh >> <WORK_DIR>/backups/backup.log 2>&1
```
in example above script gonna be executed every day at 12:00.

auto-backup script
```
#!/bin/bash

DATE=$(date +%Y-%m-%d-%H-%M)
DATADIR=/home/near
BACKUPDIR=${DATADIR}/backups/
BACKUP_NAME=near_${DATE}
FULL_BACKUPDIR_PATH=${BACKUPDIR}/${BACKUP_NAME}
HEARTBEAT_LINK=https://hc-ping.com/xxxxxx-yyyyyyy-zzzzzzzz

mkdir ${FULL_BACKUPDIR_PATH}

sudo systemctl stop near.service

wait

echo "NEAR node was stopped" | ts

if [ -d "$FULL_BACKUPDIR_PATH" ]; then
    echo "Backup started" | ts

    cp -rf $DATADIR/.near/data/ ${FULL_BACKUPDIR_PATH}

    # archiving
    zip -rjD ${FULL_BACKUPDIR_PATH}.zip ${FULL_BACKUPDIR_PATH}

    # healthcheck
    curl -fsS -m 10 --retry 5 -o /dev/null ${HEARTBEAT_LINK}

    rm -rf ${FULL_BACKUPDIR_PATH}
    echo "Backup completed" | ts

    # delete old abckups
    echo "Deleting old backups:" | ts

    ls ${BACKUPDIR} | grep -xv "${FULL_BACKUPDIR_PATH}.zip" | xargs rm
    echo "Done" | ts
else
    echo $BACKUPDIR is not created. Check your permissions.
    exit 0
fi

sudo systemctl start near.service
echo "NEAR node was started" | ts
```

create a `backup.sh` file in <WORK_DIR> directory and paste script above. `chmod +x bash.sh` to convert file to the bash script
this script 
  - takes the node's data folder
  - archives it's content (stores in backups directory)
  - send healthcheck (I'm using healthchecks.io)
  - deletes all previous archived backups

to restore data from archived backup (in automated manner) you would need to run script below
```
#!/bin/bash

DATE=$(date +%Y-%m-%d-%H-%M)
DATADIR=/home/near
BACKUPDIR=${DATADIR}/backups
BACKUP_NAME=near_${DATE}
FULL_BACKUPDIR_PATH=${BACKUPDIR}/${BACKUP_NAME}

sudo systemctl stop near.service

wait

echo "NEAR node was stopped"
echo "Backup restore started"

latest_backup=`ls -t ${BACKUPDIR}/*.zip | sort | head -1`
echo "Backup to restore: ${latest_backup}"

# unzip latest archive
unzip -o ${latest_backup} -d ${DATADIR}/.near/data

echo "Backup restore completed"
sudo systemctl start near.service

echo "NEAR node was started"
```
script unzips all files from latest backup into <WORK_DIR>/.near/data
