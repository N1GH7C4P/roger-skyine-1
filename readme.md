# Installation

Installed Debian 11, disk size 8 GB and 4.2 GB home partition on VirtualBox VM.

apt-get update
apt-get upgrade

# Adding non-root user

adduser kpolojar
apt-get install sudo
usermod -aG sudo kpolojar

# Network configuration

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

# SSH connection

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

# Firewall

Installed ufw

Added rules to allow tcp at ports 5555 and 80 for SSH and HTTP respectively, 443 for https (udp & tcp)

# Fail2ban

Installed fail2ban

<-------->

fail2ban config stuff here

<-------->

Tested fail2ban with slowloris
slowloris 10.13.199.214
monitoring the log file shows thefailed DOS -attempts. 
(sudo tail -f /var/log/fail2ban.log)

# Portsentry

Installed portsentry

<-------->

portsentry config stuff here

<-------->

# Mail

Installed mailutils and postfix

```
4.3. No deliveries to root!
No Exim 4 version released with any Debian OS can run deliveries as root. If you don't redirect mail for root via /etc/aliases to a nonprivileged account, the mail will be delivered to /var/mail/mail with permissions 0600 and owner mail:mail.
This redirection is done by the mail4root router which is last in the list and will thus catch mail for root that has not been taken care of earlier.
https://www.linuxquestions.org/questions/debian-26/root-not-getting-mail-4175423619/
```

Mails sent to root are redirected to sudo user kpolojar.

# Scripts

created script update.sh
Updates & upgrades all packages.

created /etc/crontab.backup and gave it chmod 755
created script cron_monitor.sh
compares /etc/crontab and /etc/crontab.backup, if there is diff, creates new backup and notifies admins.

modified /etc/crontab

Update packages every sunday at 4 AM and at reboot.
0 4  * *  0 /scripts/update.sh
@reboot sudo /scripts/update.sh

Monitor crontab and send mail in case of change at midnight
0 0 * * *  sudo ~/home/kpolojar/cron_monitor.sh

#!/bin/bash
DIFF=$(diff /etc/crontab.backup /etc/crontab)
cat /etc/crontab > /etc/crontab.backup
if [ "$DIFF" != "" ]; then
        echo "crontab changed, sending mail to root" | mail -s "crontab updated>
fi

List enabled services
systemctl list-unit-files --type service | grep enabled

Disabled services:
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