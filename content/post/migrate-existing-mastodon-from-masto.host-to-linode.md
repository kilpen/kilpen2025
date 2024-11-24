---
author: KilPen Tech
title: Migrate Mastodon
date: 2024-01-11
description: From paid hosting to self-hosted.  Moving Mastodon from Masto.host to Linode.
images: ["/images/solutions/linode_masto.png"]
---

![Image](/images/solutions/linode_masto.png)

## Existing Setup ##
For this tutorial, we will be moving a production instance of Mastodon from masto.host.  This instance is running on the lowest tier of service from masto.host.

Because this tutorial used a test data, there are minimal accounts, posts, avatars, and cache involved.  Real instances may be considerable larger.

Our DNS is via Cloudflare.

### Things to confirm after migration ###
Here are a list of settngs configured on the existing instance that we want to ensure carry over to the new instance.

**Instance settings**
1. Server name
2. Contact username
3. Contact email
4. Server description
5. Server thumbnail
6. Extended description
7. Privacy policy

**User content**
1. Bookmarks
2. Toots
3. Following
4. Followers
5. Direct Messages
6. Cached content

# Prepare for the migration #
Do this stuff a week or so before migration day.

## Build out the future VPS in Linode ##
Now it's time to provision the Virtual Private Server (VPS). 
1) Go to [create a new Linode](https://cloud.linode.com/linodes/create)
2) Select Debian 11
3) Select a region (us-southeast)
4) Select a Plan.  Shared CPU on Linode 4GB
5) Add a Linode Label
6) Create a root user password
7) Add SSH Keys
8) Select the Backups Add-on
9) Click "Create Linode" button.

Once the server has completed provisioning and booting, the status will turn to a green and state *Running*.


## Setup and secure the VPS ##
Let's do some configuration.
1. Set a hostname
```bash
hostnamectl set-hostname YOUR.DOMAIN
```
2. Set the timezone
```bash
timedatectl set-timezone 'TIME/ZONE'
```

Now that the VPS is up and running, we'll want to make some adjustments fairly quickly.
```bash
apt update && apt upgrade
```

Create a new limited user
```bash
adduser USER
```

Add to sudo
```bash
usermod -aG sudo USER
```

Enable SSH and Multi-factor auth
1. Generate a SSH Key via preferred method
2. Create an .ssh directory
```bash
mkdir /home/USER/.ssh
```
3. Create the auth keys file
```bash
touch /home/USER/.ssh/authorized_keys
```
4. Change ownership
```bash
chown -R USER:USER /home/USER/.ssh
```
5. Change permissions of auth file
```bash
chmod go-rw /home/USER/.ssh/authorized_keys
```
6. Open auth file in nano
```bash
nano /home/USER/.ssh/authorized_keys
```
7. Paste the public key and save

Install Google Authenticator
```bash
apt install -y libpam-google-authenticator
```

Edit sshd_config to be more 
```bash
nano /etc/ssh/sshd_config
```
1. Change *PermitRootLogin yes* to *PermitRootLogin no*
2. Change *PasswordAuthentication yes* to *PasswordAuthentication no*
3. Save

Restart SSHD
```bash
systemctl restart sshd
```

Exit and log back in as limited user.

Once logged in as the limited user, run Google Authenticator
```bash
google-authenticator
```
1.  Time-based tokens is yes
2.  Enroll using the code into authenticator app
3.  Save backup codes
4.  Last four quesitons are y,y,n,y

Edit sshd_config one more time for 2-factor login
```bash
sudo nano /etc/ssh/sshd_config
```
1. Add *ChallengeResponseAuthentication yes*
2. Add *AuthenticationMethods publickey,keyboard-interactive*
3. Save

Edit Pam.d config
```bash
sudo nano /etc/pam.d/sshd
```
1. Confirm *@include common-auth* exists
2. Add *auth   required   pam_google_authenticator.so*

Restart SSH
```bash
sudo systemctl restart ssh
```

Exit and log back in to confirm.

## Create object storage at Linode ##
1.  Go here: https://cloud.linode.com/object-storage/buckets
2.  Click ***Create Bucket***
3.  Give it a label and select a region
4.  Go here: https://cloud.linode.com/object-storage/access-keys
5.  Click ***Create Access Key***
6.  Give it a label and limit access to Read/Write only on the new bucket you just created.
7.  Save credentail and click ***I have my secret key***

## Create MailGun Credential ##
1. Create a MailGun account, or use an existing one.
2. Downgrade your plan to the Flex plan.   You'll need to Begin cancellation to do this.
3. Add your domain and verify using Cloudflare DNS TXT entry.
4. Adjust your domain's SPF record, DKIM record, and add MX if one doesn't already exist.
5. Once all verified, create SMTP credentials.
6. Save these credentials for later.    

## Installing Mastodon and other packages ##
[Reference:](https://docs.joinmastodon.org/admin/install/)
Install some necessary stuff
```bash
sudo apt install -y curl wget gnupg apt-transport-https lsb-release ca-certificates
```

Install Node.js
```bash
sudo curl -sL https://deb.nodesource.com/setup_16.x | sudo bash -
```

Add PostgreSQL to source.list
```bash
sudo wget -O /usr/share/keyrings/postgresql.asc https://www.postgresql.org/media/keys/ACCC4CF8.asc
```

```bash
sudo echo "deb [signed-by=/usr/share/keyrings/postgresql.asc] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > sudo /etc/apt/sources.list.d/postgresql.list
```

Install System Packages
```bash
sudo apt update
```

```bash
  sudo apt install -y \
  imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git-core \
  g++ libprotobuf-dev protobuf-compiler pkg-config nodejs gcc autoconf \
  bison build-essential libssl-dev libyaml-dev libreadline6-dev \
  zlib1g-dev libncurses5-dev libffi-dev libgdbm-dev \
  nginx redis-server redis-tools postgresql postgresql-contrib \
  certbot python3-certbot-nginx libidn11-dev libicu-dev libjemalloc-dev
  ```
  
Install Yarn
 ```bash
 sudo npm install --global yarn
 ```

 ```bash
 yarn set version classic
 ```

 ### Create the PostgreSQL database ###
Switch to the Postgres user
```bash
sudo su - postgres
```

Launch psql
```bash
psql
```

Create the database, user, and change ownership
```bash
CREATE DATABASE mastodon_production;
```

```bash
CREATE USER mastodon;
```

```bash
ALTER USER mastodon createdb;
```

```bash
ALTER USER mastodon WITH ENCRYPTED PASSWORD 'SET_A_PASSWORD';
```

```bash
ALTER DATABASE mastodon_production OWNER TO mastodon;
```

Now exit back to the limited user account
```bash
exit
```
```bash
exit
```

### Create a Mastodon User
Add the new user
```bash
sudo adduser --disabled-login mastodon
```

Switch accounts
```bash
sudo su - mastodon
```

### Install Ruby ###
Install and Setup
```bash
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
```
```bash
cd ~/.rbenv && src/configure && make -C src
```
```bash
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
```
```bash
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
```
```bash
exec bash
```
```bash
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
```
```bash
RUBY_CONFIGURE_OPTS=--with-jemalloc rbenv install 3.0.6
```
```bash
rbenv global 3.0.6
```
```bash
gem install bundler --no-document
```

## Clone Mastodon Repo  ##
As the mastodon user, ensure you're in ~ and clone the repo
```bash
git clone https://github.com/mastodon/mastodon.git live && cd live
```

## Configure nginx ##
As the limited user, prepare the .conf files.
```bash
sudo cp /home/mastodon/live/dist/nginx.conf /etc/nginx/sites-available/mastodon
```
Edit the configuration file for the primary webserver
```bash
sudo nano /etc/nginx/sites-available/mastodon
```
Update these fields:
* server_name - replace *example.com* with your domain in both 80 and 443 sections
* Add snake oil lines for certbot
```javascript
ssl_certificate /etc/ssl/certs/ssl-cert-snakeoil.pem;
ssl_certificate_key /etc/ssl/private/ssl-cert-snakeoil.key;
```
Enable the mastodon site
```bash
sudo ln -s /etc/nginx/sites-available/mastodon /etc/nginx/sites-enabled/mastodon
```
Create another .conf to proxy cache and storage
```bash
sudo touch /etc/nginx/sites-available/files.DOMAIN.NAME
```
Edit the .conf
```bash
sudo nano /etc/nginx/sites-available/files.DOMAIN.NAME
```
Paste in the following config
*Source: https://docs.joinmastodon.org/admin/optional/object-storage-proxy/*
```javascript
server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name files.example.com;
  root /var/www/html;

  keepalive_timeout 30;

  location = / {
    index index.html;
  }

  location / {
    try_files $uri @s3;
  }

  set $s3_backend 'https://YOUR_BUCKET_NAME.YOUR_S3_HOSTNAME';

  location @s3 {
    limit_except GET {
      deny all;
    }

    resolver 8.8.8.8;
    proxy_set_header Host YOUR_BUCKET_NAME.YOUR_S3_HOSTNAME;
    proxy_set_header Connection '';
    proxy_set_header Authorization '';
    proxy_hide_header Set-Cookie;
    proxy_hide_header 'Access-Control-Allow-Origin';
    proxy_hide_header 'Access-Control-Allow-Methods';
    proxy_hide_header 'Access-Control-Allow-Headers';
    proxy_hide_header x-amz-id-2;
    proxy_hide_header x-amz-request-id;
    proxy_hide_header x-amz-meta-server-side-encryption;
    proxy_hide_header x-amz-server-side-encryption;
    proxy_hide_header x-amz-bucket-region;
    proxy_hide_header x-amzn-requestid;
    proxy_ignore_headers Set-Cookie;
    proxy_pass $s3_backend$uri;
    proxy_intercept_errors off;

    proxy_cache CACHE;
    proxy_cache_valid 200 48h;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_lock on;

    expires 1y;
    add_header Cache-Control public;
    add_header 'Access-Control-Allow-Origin' '*';
    add_header X-Cache-Status $upstream_cache_status;
    add_header X-Content-Type-Options nosniff;
    add_header Content-Security-Policy "default-src 'none'; form-action 'none'";
  }
}
```
Now make the following edits:
* Update the server_name for 443 server
* Edit the $s3_backend variable: https://kilpen-net-test.us-southeast-1.linodeobjects.com/kilpen-net-test
* Edit proxy_set_header to kilpen-net-test.us-southeast-1.linodeobjects.com
* Add snakeoil lines for Certbot
```javascript
ssl_certificate /etc/ssl/certs/ssl-cert-snakeoil.pem;
ssl_certificate_key /etc/ssl/pr

ivate/ssl-cert-snakeoil.key;
```
Lastly, activate this site in Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/files.DOMAIN.NAME /etc/nginx/sites-enabled
```

## Setting up Mastodon ##
As the Mastodon user from the ~/live directory

[TAKE A SNAPSHOT BACKUP BEFORE MOVING FORWARD!]

Select the version
```bash
git checkout v4.1.6
```
Run the following to configure and install Mastodon
```bash
bundle config deployment 'true'
```
```bash
bundle config without 'development test'
```
```bash
bundle install -j$(getconf _NPROCESSORS_ONLN)
```
```bash
yarn install --pure-lockfile
```

## DNS Prep ##
Cloudflare should have three A records entries:
* domain root
* www
* files

# Migration Day
## Retrieve data from MastoHost ##
Via the Masto.host interface, stop your production server and request a backup of the current data.  Once you receive the email with the link, download the contents to your local machine.  Unfortunately, MastoHost doesn't seem to allow download using *wget* from their servers directly to your VPS.

### Upload MastoHost database to the Linode VPS ###
Create a new SSH key just for this effort.  Add the public key to ~/.ssh/authorized_keys

```bash
scp -i ~/.ssh/id_rsa.pub FILENAME kilpen@[IP]:/home/kilpen/FILENAME
```

### Upload MastoHost files to the Linode Bucket ###

## Certbot ##
Run Certbot
```bash
sudo certbot --nginx
```
After successful certificate issue, restart nginx
```bash
sudo systemctl reload nginx
```

## Start Mastodon ##
Copy the services to systemd
```bash
sudo cp /home/mastodon/live/dist/mastodon-sidekiq.service /etc/systemd/system/ &&
sudo cp /home/mastodon/live/dist/mastodon-streaming.service /etc/systemd/system/ && sudo cp /home/mastodon/live/dist/mastodon-web.service /etc/systemd/system/
```
Reload and enable
```bash
sudo systemctl daemon-reload
```
```bash
sudo systemctl enable --now mastodon-web mastodon-sidekiq mastodon-streaming
```

## Sample .env.production file ##
```javascript
LOCAL_DOMAIN=kilpen.net
SINGLE_USER_MODE=false
SECRET_KEY_BASE=[Shh...]
VAPID_PRIVATE_KEY=[Shh...]
VAPID_PUBLIC_KEY=[Shh...]
DB_HOST=/var/run/postgresql
DB_PORT=5432
DB_NAME=mastodon_production
DB_USER=mastodon
DB_PASS=[Shh...]
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
S3_ENABLED=true
S3_PROTOCOL=https
S3_BUCKET=hometowndemo
S3_ENDPOINT=https://kilpen-net-test.us-southeast-1.linodeobjects.com
S3_REGION=us-southeast-1
S3_HOSTNAME=kilpen-net-test.us-southeast-1.linodeobjects.com
AWS_ACCESS_KEY_ID=[Shh...]
AWS_SECRET_ACCESS_KEY=[Shh...]
S3_ALIAS_HOST=files.kilpen.net
SMTP_SERVER=smtp.mailgun.org
SMTP_PORT=587
SMTP_LOGIN=postmaster@mg.kilpen.com
SMTP_PASSWORD=[Shh...]
SMTP_AUTH_METHOD=plain
SMTP_OPENSSL_VERIFY_MODE=none
SMTP_ENABLE_STARTTLS=auto
SMTP_FROM_ADDRESS='Mastodon <postmaster@mg.kilpen.com>'
```