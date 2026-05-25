# Git Automation: Auto-Sync All Repositories with Cron

## Problem
I have multiple GitHub repositories on my server:
- Sys_Admin.projects
- Linux-Plus-Study
- Mycology-Lab
- Docker-Certified-Associate-Study
- Security-Plus-701-Study
- Network-Plus-Study

Manually running `git pull` and `git push` for each repo every time I wanted to sync was tedious and easy to forget.

## Solution
I created an automated script that runs daily at 2 AM to:
1. Pull latest changes from GitHub (downloading edits made from my phone or laptop)
2. Push local commits up to GitHub (uploading work done on my server)

## The Script: `~/update-all-repos.sh`

```bash
#!/bin/bash

LOGFILE="/home/anureyki/git-sync.log"

echo "========== SYNC STARTED: $(date) ==========" >> $LOGFILE

# SSH agent setup for cron
eval "$(ssh-agent -s)"
ssh-add /home/anureyki/.ssh/id_ed25519

# List of repositories to sync
for repo in /home/anureyki/Sys_Admin.projects \
            /home/anureyki/Linux-Plus-Study \
            /home/anureyki/Mycology-Lab \
            /home/anureyki/Docker-Certified-Associate-Study \
            /home/anureyki/Security-Plus-701-Study \
            /home/anureyki/Network-Plus-Study; do

    echo "Syncing $repo..." >> $LOGFILE
    cd $repo

    # Pull latest changes from GitHub
    git pull origin main >> $LOGFILE 2>&1

    # Push local commits to GitHub
    git push origin main >> $LOGFILE 2>&1

    echo "Done with $repo" >> $LOGFILE
    echo "" >> $LOGFILE
done

echo "========== SYNC FINISHED: $(date) ==========" >> $LOGFILE
