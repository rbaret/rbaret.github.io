layout: page
title: "Configure Duplicity on Debian with Azure Blob storage"
permalink: /configure-duplicity-debian-azure/

# Configure Duplicity on Debian with Azure Blob storage

## Install Duplicity

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
duplicity full \ # Force a full backup
--encrypt-sign-key=$GPG_KEY \ # Use the GPG key we set in the .env_variable.conf
--volu-size=1000 \ # Size of a volume in MB, default is 200MB. Duplicity will split the backup in files of this size
--log-file=/home/user/duplicity_$(date +"%Y%m%d_%I%M%p%S").log \ # target log file with the date of the command
--exclude=/proc --exclude=/sys --exclude=/tmp \ # folders to exclude
/ azure://kimsufi # Source path and destination
```

The backup duration will highly depend of the storage and CPU speed. In my case, with an outdated Atom N2800 and 2GB of RAM it took 2 hours to backup 280k files eith an original size of 17GB and a final size of 11.6GB after compression.
