---
layout: post
title: "Configure Duplicity on Debian with Azure Blob storage"
date: 2022-08-05 22:45:00 +0100
categories: azure duplicity backup tutorial

---
These instructions are based on [This work](https://www.digitalocean.com/community/tutorials/how-to-use-duplicity-with-gpg-to-back-up-data-to-digitalocean-spaces) by [Kathleen Juell](https://www.digitalocean.com/community/users/katjuell) from DigitalOcean

You can use the same instructions for AWS S3 storage by replacing *AZ* by *AW* in environment variable names and by replacing *azure://* by *s3://* in destination

## Install Duplicity

```bash
# Install dependances
sudo apt install python python-boto3 python-dev pyhton-fasteners librsync1 librsync-dev gnupg snapd

# install Azure Blob Storage SDK
sudo pip install azure-storage-blob

# Install Duplicity
pip install duplicity
```

## Configure GPG for encryption

```bash
# Check GPG is installed
gpg --version

# Give the  system some entropy for the generation time
sudo rngd -r /dev/urandom

# Generate new key
gpg --full-generate-key

# If GPG < v2.1.17 use this one instead
# gpg --default-new-key-algo rsa4096 --gen-key

# Choose (1) RSA and RSA (default)
# Type 2048 as key size - bigger keys will divide encryption performance by a big amount (8 in my case between 2048 and 4096)
# Choose 0 to avoid key expiration (We all knkow you will forget to mantain your key...)
# y to confirm choices
# Enter required info : Key name, email, description
# Validate with y
# Choose a secret passphrase to secure the key and note it

# Get the public fingerprint of the key
gpg --list-keys
```

Example of result :

```bash
pub   rsa2048 2020-11-07 [SC]
      ABCDEF341243BBBAFDE8C96EBF67513BDFEF9364ADFEFD234B12335 # This is the public fingerprint of your key
uid          [  ultime ] Your key description <your@emailaddress.com>
sub   rsa2048 2020-11-07 [E]
```

## Configure Duplicity

```bash
# Create hidden folder to store the configuration
mkdir ~/.duplicity

# Create file to store variables :
nano ~/.duplicity/.env_variables.conf

# Edit the file content :

export AZURE_CONNECTION_STRING="your-connection-string" # You can fetch this info from the storage account properties or via az CLI : az storage account show-connection-string --resource-group your-RSG-name --name your-account-name 
export GPG_KEY="your-GPG-public-key-id" # Only the last 8 characters of the pub fingerprint
export PASSPHRASE="your-GPG-key-passphrase"

# Secure the file :
chmod 0600 ~/.duplicity/.env_variables.conf

# Make the variables available for bash
source ~/.duplicity/.env_variables.conf

```

## Backup the data using Duplicity

Here is an example of a full backup of a whole server to an azure storage account with a dedicated blob container :

```bash
duplicity full \
--encrypt-sign-key=$GPG_KEY \
--vol-size=1000 \
--log-file=/home/user/.duplicity/duplicity_$(date +"%Y%m%d_%I%M%p%S").log \
--exclude=/proc --exclude=/sys --exclude=/tmp \
/ azure://yourcontainer
```

The backup duration will highly depend of the storage and CPU speed. In my case, with an outdated Atom N2800 and 2GB of RAM it took 2 hours to backup 280k files with an original size of 17GB and a final size of 11.6GB after compression.

## Automating the backup

Let's create a small script that we will run every day. This script will run a full backup if the last full is older than 7 days, otherwise it will be an incremental.

This job will also do a cleanup to keep only the last full and subsequent incremental. This can be changed to keep more than 1 set of full backups (4 can be a good value as it represents a month of backups) at the cost of additional storage fees.
It's also possible to keep only the last *n* full backups and remove all the incremental older than the current set with the *remove-all-inc-of-but-n-full* command in conjunction with the *remove-all-but-n-full* command.

```bash
nano ~/.duplicity/.backup.sh
```

### .backup.sh

```bash
#!/bin/bash

HOME="/home/user"
export PATH=$PATH:/usr/local/bin # Add local bin path so duplicity can be found when installed through pip
source "$HOME/.duplicity/.env_variables.conf"

duplicity \
    --verbosity info \
    --encrypt-sign-key="$GPG_KEY" \
    --full-if-older-than 7D \
    --log-file "$HOME/duplicity_$(date +"%Y%m%d_%I%M%p%S").log" \
    / \
    azure://yourcontainer

duplicity remove-all-but-n-full 1 --force
# Uncomment the line below if you want to keep more thant 1 full and want to clean intermediate inc backups except the last chain
# duplicity remove-all-inc-of-but-n-full 1
unset AZURE_CONNECTION_STRING
unset GPG_KEY
unset PASSPHRASE
```

Now we setup the cron job to run daily :

```bash
# Make the script file executable
chmod 0700 ~/.duplicity/.backup.sh

crontab -e
```

and add the following line :

```bash
@daily /home/user/.duplicity/.backup.sh
```

And we're all set !
