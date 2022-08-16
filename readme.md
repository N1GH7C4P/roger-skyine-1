- [Installation](#installation)
  * [Adding non-root user](#adding-non-root-user)
  * [Network configuration](#network-configuration)
  * [SSH connection](#ssh-connection)
- [Network Security](#network-security)
  * [Firewall](#firewall)
  * [Fail2ban](#fail2ban)
    + [Unbanning myself](#unbanning-myself)
  * [Portsentry](#portsentry)
- [Monitoring & Updates](#monitoring---updates)
  * [Mail](#mail)
  * [Scripts](#scripts)
- [Resource Efficiency](#resource-efficiency)
  * [Disabled nonmandatory services](#disabled-nonmandatory-services)
- [SSL](#ssl)
- [DEPLOYMENT AUTOMATION](#deployment-automation)
  * [Git & GitHub access token](#git---github-access-token)
  * [SSH agent forwarding](#ssh-agent-forwarding)
  * [Deployment with GitHooks](#deployment-with-githooks)
    + [How it works](#how-it-works)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


# Installation

Installed Debian 11, disk size 8 GB and 4.2 GB home partition on VirtualBox VM.

apt-get update
apt-get upgrade

## Adding non-root user

adduser kpolojar
apt-get install sudo
usermod -aG sudo kpolojar

## Network configuration

Changed VM network setting NAT -> Bridged

Setup static ip address:

/etc/network/interfaces
#Primary network interface
auto enp0s3

/etc/network/interfaces.d/enp0s3
iface enp0s3 inet static
	address 10.13.199.214
	netmask 255.255.255.252 (/30)
	gateway 10.13.254.254

## SSH connection

/etc/ssh/sshd_config
Changed the ssh default port to 5555

Created key pair with ssh-keygen in VM

Changed SSH settings to allow for password connection.
/etc/ssh/sshd_config
uncomment PasswordAuthentication

Connected from iMac to virtual machine with SSH using password
ssh 10.13.199.214 -p 5555 (no need for username because same on both systems)

Copied key to VM with ssh-copy-id 

Changed settings back to refuse password connection
/etc/ssh/sshd_config
comment out PasswordAuthentication
uncommment PubkeyAuthentication

# Network Security

## Firewall

Installed ufw

Added rules to allow tcp at ports 5555 and 80 for SSH and HTTP respectively, 443 for https (udp & tcp)

## Fail2ban

Installed fail2ban

<-------->

fail2ban config stuff here

<-------->

Tested fail2ban with slowloris
slowloris 10.13.199.214
monitoring the log file shows thefailed DOS -attempts. 
sudo tail -f /var/log/fail2ban.log

### Unbanning myself

I got myself banned by trying to SSH the root.
ssh root@10.13.199.214

So I had to go and manually unban myself. 

Bans are listed in file:
/etc/fail2ban/jail.local
Clearing the file will end all current bans without affecting the filters.

## Portsentry

Installed portsentry

<-------->

portsentry config stuff here

<-------->

# Monitoring & Updates

## Mail

Installed mailutils and postfix

Delivering mail to the root has been disabled on any Debian distribution.
```
4.3. No deliveries to root!
No Exim 4 version released with any Debian OS can run deliveries as root. If you don't redirect mail for root via /etc/aliases to a nonprivileged account, the mail will be delivered to /var/mail/mail with permissions 0600 and owner mail:mail.
This redirection is done by the mail4root router which is last in the list and will thus catch mail for root that has not been taken care of earlier.
```
https://www.linuxquestions.org/questions/debian-26/root-not-getting-mail-4175423619/

This has been done because receiving mail as root is security vulnerability and bad practise.

https://www.howtogeek.com/124950/htg-explains-why-you-shouldnt-log-into-your-linux-system-as-root/

This is why we have the less privileged sudo account for system administrator in the first place. The subject mandates that mail is sent to root, But there is no mention of receiving or reading said mail on that account. Thus mails are sent to root and then redirected to sudo user kpolojar.

## Scripts

created script update.sh
Updates & upgrades all packages.

```
!/bin/bash
sudo apt-get update -y >> /var/log/update_script.log
sudo apt-get upgrade -y >> /var/log/update_script.log
```

created /etc/crontab.backup and gave it chmod 755
created script cron_monitor.sh
compares /etc/crontab and /etc/crontab.backup, if there is diff, creates new backup and notifies admins.

modified /etc/crontab

Update packages every sunday at 4 AM and at reboot.
0 4  * *  0 /scripts/update.sh
@reboot sudo /scripts/update.sh

Monitor crontab and send mail in case of change at midnight
0 0 * * *  sudo ~/home/kpolojar/cron_monitor.sh

```
#!/bin/bash
DIFF=$(diff /etc/crontab.backup /etc/crontab)
cat /etc/crontab > /etc/crontab.backup
if [ "$DIFF" != "" ]; then
        echo "crontab changed, sending mail to root" | mail -s "crontab updated" root
fi
```

# Resource Efficiency

## Disabled nonmandatory services

List enabled services
systemctl list-unit-files --type service | grep enabled

```
kpolojar@debian:~$ sudo systemctl disable console-setup.service
Removed /etc/systemd/system/multi-user.target.wants/console-setup.service.
kpolojar@debian:~$ sudo systemctl disable cryptdisks-early.service
Unit /lib/systemd/system/cryptdisks-early.service is masked, ignoring.
kpolojar@debian:~$ sudo systemctl disable cryptdisks.service
Unit /lib/systemd/system/cryptdisks.service is masked, ignoring.
kpolojar@debian:~$ sudo systemctl disable e2scrub_reap.service
Removed /etc/systemd/system/default.target.wants/e2scrub_reap.service.
kpolojar@debian:~$ sudo systemctl disable ifupdown-wait-online.service
kpolojar@debian:~$ sudo systemctl disable keyboard-setup.service
Removed /etc/systemd/system/sysinit.target.wants/keyboard-setup.service.
kpolojar@debian:~$ sudo systemctl disable nftables.service
```

# SSL

https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-debian-9

Creating certificate

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt

Country Name (2 letter code) [AU]:FI
State or Province Name (full name) [Some-State]:Uusimaa
Locality Name (eg, city) []:Helsinki
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Hive
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:Kimmo Polojarvi
Email Address []:kpolojar@debian.debbie

sudo nano /etc/apache2/conf-available/ssl-params.conf

```
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
SSLHonorCipherOrder On
# Disable preloading HSTS for now.  You can use the commented out header line that includes
# the "preload" directive if you understand the implications.
# Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
# Requires Apache >= 2.4
SSLCompression off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
# Requires Apache >= 2.4.11
SSLSessionTickets Off
```

sudo nano /etc/apache2/sites-available/default-ssl.conf

```
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerAdmin kpolojar@debian.debbie
                ServerName 10.13.199.214

                DocumentRoot /var/www/html

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                SSLEngine on

                SSLCertificateFile      /etc/ssl/certs/apache-selfsigned.crt
                SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
      
                <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                SSLOptions +StdEnvVars
                </FilesMatch>
                <Directory /usr/lib/cgi-bin>
                                SSLOptions +StdEnvVars
                </Directory>
        </VirtualHost>
</IfModule>
```
sudo nano /etc/apache2/sites-available/000-default.conf
```
<VirtualHost *:80>
        . . .

        Redirect permanent "/" "https://10.13.199.214/"

        . . .
</VirtualHost>
```

# DEPLOYMENT AUTOMATION

## Git & GitHub access token 

sudo apt-get install git

We need to setup GitHub access token to be able to fetch changes from my personal github repo.

Created GitHub access token
https://www.edgoad.com/2021/02/using-personal-access-tokens-with-git-and-github.html

To cache your GitHub credentials for HTTPS access follow this tutorial.

https://docs.github.com/en/get-started/getting-started-with-git/caching-your-github-credentials-in-git

## SSH agent forwarding

This is needed to use GitHub commands through SSH -connection without providing credentials each time.

https://docs.github.com/en/developers/overview/using-ssh-agent-forwarding

Make sure that you have the same SSH keys on your VM and local machine.
Otherwise the VM doesn't have permission to fetch from GitHub.

Then add GitHub to your known hossts file.
https://serverfault.com/questions/856194/securely-add-a-host-e-g-github-to-the-ssh-known-hosts-file

## Deployment with GitHooks

### How it works
You are developing in a working-copy on your local machine, lets say on the master branch. Most of the time, people would push code to a remote server like github.com or gitlab.com and pull or export it to a production server. Or you use a service like deepl.io to act upon a Web-Hook that's triggered that service.

But here, we add a "bare" git repository that we create on the production server and pusblish our branch (f.e. master) directly to that server. This repository acts upon the push event using a 'git-hook' to move the files into a deployment directory on your server. No need for a midle man.

This creates a scenario where there is no middle man, high security with encrypted communication (using ssh keys, only authorized people get access to the server) and high flexibility tue to the use of .sh scripts for the deployment.

https://gist.github.com/noelboss/3fe13927025b89757f8fb12e9066f2fa

https://towardsdatascience.com/how-to-create-a-git-hook-to-push-to-your-server-and-github-repo-fe51f59122dd

If you get permission problems do this.
https://stackoverflow.com/questions/14127255/remove-git-index-lock-permission-denied

Now when you do
```
git push server master
```
the changes are pushed straight to the VM server.