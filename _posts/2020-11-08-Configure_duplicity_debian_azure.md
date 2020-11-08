---
layout: post
title: "Configure Duplicity on Debian with Azure Blob storage"
date: 2020-11-08 01:45:00 +0100
categories: azure duplicity backup tutorial
---
These instructions are based on [This work](https://www.digitalocean.com/community/tutorials/how-to-use-duplicity-with-gpg-to-back-up-data-to-digitalocean-spaces) by [Kathleen Juell](https://www.digitalocean.com/community/users/katjuell) from DigitalOcean

You can use the same instructions for AWS S3 storage by replacing *AZ* by *AW* in environment variable names and by replacing *azure://* by *s3://* in destination

## Install Duplicity

Instead of using the latest version I choose to use the one included in my Debian 9, which is in version 0.7.11. 
Why ? Because azure-storage module is deprecated since its version 0.37 and has been split in 5, including the one we're interested in here : azure-storeage-blob.
Unfortunately the support for this module is still not implemented in versions 0.8+

```bash
# Install dependances
sudo apt install python python-boto3 python-dev pyhton-fasteners librsync1 librsync-dev gnupg snapd

# install Azure Blob Storage SDK
sudo pip install azure-storage-blob

# Install Duplicity
sudo apt install duplicity
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

export AZ_ACCESS_KEY_ID="your-access-key"
export AZ_SECRET_ACCESS_KEY="your-secret-key"
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
--volu-size=1000 \
--log-file=/home/user/.duplicity/duplicity_$(date +"%Y%m%d_%I%M%p%S").log \
--exclude=/proc --exclude=/sys --exclude=/tmp \
/ azure://yourcontainer
```

The backup duration will highly depend of the storage and CPU speed. In my case, with an outdated Atom N2800 and 2GB of RAM it took 2 hours to backup 280k files eith an original size of 17GB and a final size of 11.6GB after compression.

## Automating the backup

Let's create a small script that we will run every day. This script will run a full backup if the last full is older than 7 days, otherwise it will be an incremental.

This job will also do a cleanup to keep only the last full and subsequent incrementals. This can be changed to keep more than 1 set of full backups (4 can be a good value as it represents a month of backups) at the cost of additional storage fees.
It's also possible to keep only the last n full and remove all the incrementals older than the current set with the *remove-all-inc-of-but-n-full* command in conjunction with the *remove-all-but-n-full* command.

```bash
nano ~/.duplicity/.backup.sh
```

### .backup.sh

```bash
#!/bin/bash

HOME="/home/user"

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
unset AZ_ACCESS_KEY_ID
unset AZ_SECRET_ACCESS_KEY
unset GPG_KEY
unset PASSPHRASE
```

Now we setup the cron job to run daily :

```bash
crontab -e
```

and add the following line :

```bash
@daily /home/user/.duplicity/.backup.sh
```

And we're all set !