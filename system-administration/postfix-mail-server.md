# Postfix Mail Server

_source: https://www.linuxbabe.com/mail-server/setup-basic-postfix-mail-sever-ubuntu_

This article is a quick note about how to install, setup and manage a mail server using Postfix.

## Prerequsites

Before I even get to Postfix, there are a set of tasks that should be finished before.

### Setup DNS

The DNS records should look like this:

```text
HOSTNAME                   TTL      TYPE    PRIO    VALUE
mail.digitao.io            57600    A               <IP-ADDRESS>       # Where to find the server
digitao.io                 57600    MX      10      mail.digitao.io    # If the request is related to email,
                                                                       # then forward to mail.digitao.io
autoconfig.digitao.io      57600    CNAME           mail.digitao.io    # Optional, only used for setting up MUA
                                                                       # (mail user agent)
autodiscover.digitao.io    57600    CNAME           mail.digitao.io    # Optional, only used for setting up MUA
```

Basically the `A` record defines a new sub-domain `mail.digitao.io` pointing to the actual mail server. And the `MX` server forwards the mail request from the main domain `digitao.io` to the sub-domain `mail.digitao.io`. Of course if there is IPv6 Address, I should also setup `AAAA` record.

### Setup the reverse DNS

After DNS has been done, I went to the hosting server provider, and set the reverse DNS from the IP-Adress back to the domain name (`mail.digitao.io`).

### Enable port 25 and 465

This has to be done for both inbound and also outbound, otherwise either sending or recieving mails won't work. Note that the outbound rules sometimes are disabled by hosting service providers, so I have to create a service ticket for them to enable the 2 ports for me.

### Change the hostname

Since the Postfix reads the hostname of my server to identify itself while communicating with other MTA (message transfer agent), it is important to change the hostname to match the hostname in the DNS record:

Use following command to see what the current hostname is:

```console
$ hostname -f
```

If it is not the one configured in the DNS record, I should do:

```console
# hostnamectl set-hostname mail.digitao.io
```

## Installation

This is a basic installation guide for installing the postfix, the basic barebone mail server framework.

### Basic Installation

Update the server first:

```console
# apt updage
# apt upgrade
# reboot
# apt install postfix
```

Then a dialog will be displayed asking me how I want to configure Postfix. In my case, I should set it to `Internet Site`, this is useful for most of the cases.

In the next step, I should enter my "System mail name". Here I should make it `digitao.io` instead of `mail.digitao.io`, because if I set it to the latter, I won't be able to send/receive Emails from `digitao.io` domain.

To verify the mail service, use following command:

```console
# postconf mail_version
mail_version = 3.8.6
```

And also checking who is now listening to port 25:

```console
# ss -lnpt | grep master
LISTEN 0      100          0.0.0.0:25        0.0.0.0:*    users:(("master",pid=2633,fd=13))                     
LISTEN 0      100             [::]:25           [::]:*    users:(("master",pid=2633,fd=14))
```

### Send/Receive test mails

To send Email, I can use the `mail` command in the `mailutils` package in Ubuntu:

```conosle
# apt install mailutils
```

And then write a short mail:

```console
$ mail -a FROM:<system-user-name>@digitao.io <gmail-user-name>@gmail.com
Cc: 
Subject: Test
This Email is sent from a mail server setup by myself just now.
```

To send the Email, simply press `Ctrl+D`, and the mail will be found in the GMail now.

To receive Email, simply type use mail command:

```console
$ mail
```

It is interactive, so type `?` and `Enter` for a short command instructions. And some useful commands are:

* `1`: Read the first mail
* `h`: List the head of the mails
* `d 1`: Delete first mail
* `q`: Quit mail program

## Setup TLS Encryption and IMAP

Now there are 2 problems:

1. All the Emails are transmitted in Internet in clear text;
2. I cannot use and mail client to connect to the server.

So this parts of the note provides a solution for these two problems.

### Enable more ports in Firewall

To setup TLS and IMAP, I have to enable more ports:
* `80` and `443` - to sign the TLS signature using certbot
* `587` - for encrypted SMTP
* `465` - for encrypted SMTP (deprecated)
* `143` - for non-encrypted IMAP
* `993` - for encrypted IMAP

### Install Nginx and certbot

Simply use following command to install Nginx:

```conosle
# apt install nginx
```

### Install certbot

The suggested way of install certbot is using `snap`. For the servers from my hosting service provider, the `snap` is already preinstalled, so I don't need to install any extra pacakge at all. I can simply run:

```console
# snap install --classic certbot
```

### Prepare Nginx for the SSL certificate challenge

Provide an Nginx site config like this:

```text
server {
        listen 80;
        listen [::]:80;

        server_name mail.digitao.io;

        root /usr/share/nginx/html/;

        location ~/.well-known/acme-challenge {
                allow all;
        }
}
```

And put it into `/etc/nginx/sites-enabled`, name the file as `mail.digitao.io`. For more information, please read [Nginx Administration](./nginx-administration.md).

### Request a new certificate using certbot

Use the following command:

```console
# certbot certonly --nginx -d mail.digitao.io
```

In this case, certbot won't change the settings in the nginx, it just tries to get a new certificate.

After the interactive script, the certificate files will be saved:
* Certificate file: `/etc/letsencrypt/live/mail.digitao.io/fullchain.pem`
* Key file: `/etc/letsencrypt/live/mail.digitao.io/privkey.pem`

### Setup SMTP

**Deeper understanding required ...**

Edit the `/etc/postfix/master.cf` file, and add following two blocks into the file:

```text
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_tls_wrappermode=no
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o smtpd_recipient_restrictions=permit_mynetworks,permit_sasl_authenticated,reject
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o smtpd_recipient_restrictions=permit_mynetworks,permit_sasl_authenticated,reject
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
```

Edit the `/etc/postfix/main.cf` file, modify the `# TLS parameters` section:

```text
# TLS parameters
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.digitao.io/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.digitao.io/privkey.pem
smtpd_tls_security_level = may
smtpd_tls_loglevel = 1
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache

smtp_tls_security_level = may
smtp_tls_loglevel = 1
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
```

And then restart Postfix service:

```console
# systemctl restart postfix
```

After that, if I check the port which is listened by `master`, I can see there are not only port `25`, but also `465` and `587` opened:

```console
# ss -lnpt | grep master
LISTEN 0      100          0.0.0.0:25        0.0.0.0:*    users:(("master",pid=12881,fd=13))
LISTEN 0      100          0.0.0.0:465       0.0.0.0:*    users:(("master",pid=12881,fd=22))
LISTEN 0      100          0.0.0.0:587       0.0.0.0:*    users:(("master",pid=12881,fd=18))
LISTEN 0      100             [::]:25           [::]:*    users:(("master",pid=12881,fd=14))
LISTEN 0      100             [::]:465          [::]:*    users:(("master",pid=12881,fd=23))
LISTEN 0      100             [::]:587          [::]:*    users:(("master",pid=12881,fd=19))
```

### Install and setup Dovecot to enable IMAP

**Deeper understanding required ...**

To install Dovecot, simply run:

```console
# apt install dovecot-core dovecot-imapd dovecot-lmtpd
```

After that, edit Dovecot configruation file `/etc/dovecot/dovecot.conf` to enable IMAP. Add one line:

```text
# Enable installed protocols
protocols = imap lmtp    # Add
!include_try /usr/share/dovecot/protocols.d/*.protocol
```

Edit `/etc/dovecot/conf.d/10-mail.conf` to config it using the `Maildir` instead of `mbox` directory structure:

```text
mail_location = maildir:~/Maildir    # Modify
mail_privileged_group = mail    # Should be already there, just check it
```

After that we want to add the user `dovecot` to the `mail` group:

```console
# adduser dovecot mail
```

Open the `/etc/dovecot/conf.d/10-master.conf` file and edit the `service lmtp` and `service auth` section:

```text
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }

  # Create inet listener only if you can't use the above UNIX socket
  #inet_listener lmtp {
    # Avoid making LMTP visible for the entire internet
    #address =
    #port = 
  #}
}

service auth {
  # auth_socket_path points to this userdb socket by default. It's typically
  # used by dovecot-lda, doveadm, possibly imap process, etc. Users that have
  # full permissions to this socket are able to get a list of all usernames and
  # get the results of everyone's userdb lookups.
  #
  # The default 0666 mode allows anyone to connect to the socket, but the
  # userdb lookups will succeed only if the userdb returns an "uid" field that
  # matches the caller process's UID. Also if caller's uid or gid matches the
  # socket's uid or gid the lookup succeeds. Anything else causes a failure.
  #
  # To give the caller full permissions to lookup all users, set the mode to
  # something else than 0666 and Dovecot lets the kernel enforce the
  # permissions (e.g. 0777 allows everyone full permissions).
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }

  # Postfix smtp-auth
  #unix_listener /var/spool/postfix/private/auth {
  #  mode = 0666
  #}

  # Auth process is run as this user.
  #user = $default_internal_user
}
```

Edit `/etc/postfix/main.cf` file, and add following 2 lines at the bottom of the file:

```text
mailbox_transport = lmtp:unix:private/dovecot-lmtp
smtputf8_enable = no
```

Edit `/etc/dovecot/conf.d/10-auth.conf` file, edit multiple places:

```text
disable_plaintext_auth = yes    # Uncomment
auth_username_format = %n    # Uncomment and modify
auth_mechanisms = plain login    # Modify
```

Edit `/etc/dovecot/conf.d/10-ssl.conf`, edit multiple places:

```text
ssl = required    # Modify
ssl_cert = </etc/letsencrypt/live/mail.digitao.io/fullchain.pem    # Modify
ssl_key = </etc/letsencrypt/live/mail.digitao.io/privkey.pem    # Modify
ssl_prefer_server_ciphers = yes    # Uncomment and modify
ssl_min_protocol = TLSv1.2    # Uncomment and modify
```

Edit `/etc/ssl/openssl.cnf` file, and comment one line to disable FIPS:

```text
#providers = provider_sect    # Comment
```

Edit `/etc/dovecot/conf.d/15-mailboxes.conf` file, and edit it:

```text
  mailbox Drafts {
    auto = create
    special_use = \Drafts
  }
  mailbox Junk {
    auto = create
    special_use = \Junk
  }
  mailbox Trash {
    auto = create
    special_use = \Trash
  }
  mailbox Sent { 
    auto = create
    special_use = \Sent
  }
```

This tells dovecot to create default folders automatically. Besides, it will also avoid to create 2 different folders for the `\Sent` usage (previously there are "Sent" and "Sent Messages").

And finally restart everything that I have configured:

```console
# systemctl restart postfix dovecot
```

And check whether dovecot is up and running:

```console
# ss -lnpt | grep dovecot
LISTEN 0      100          0.0.0.0:143       0.0.0.0:*    users:(("dovecot",pid=18756,fd=36))
LISTEN 0      100          0.0.0.0:993       0.0.0.0:*    users:(("dovecot",pid=18756,fd=38))
LISTEN 0      100             [::]:143          [::]:*    users:(("dovecot",pid=18756,fd=37))
LISTEN 0      100             [::]:993          [::]:*    users:(("dovecot",pid=18756,fd=39))
```

### Create user for mail

Postfix will use the system users as Email users. So create a user like this:

```console
# useradd -d /home/<unix_username> -s /bin/false <unix_username>
# mkdir /home/<unix_username>
# chown <unix_username>:<unix_usergrp> /home/<unix_username>
# passwd <unix_username>
```

This will create the home directory for the user (the mails are being stored in the `~/Maildir` folder).

## Use MUA to connect to the server

Using following data to setup the Mail Client:

```text
Name: <my_name>
EMail Address: <unix_username>@digitao.io

# IMAP Mail Server
Host: mail.digitao.io
Port: 143
User Name: <unix_username>
Security: STARTTLS
Authentication Method: Normal password

# SMTP Server
Host: mail.digitao.io
Port: 587
User Name: <unix_username>
Security: STARTTLS
Authentication Method: Normal password
```

## Setup SPF

To send Email to an GMail address, I have to setup either SPF or DKIM. Since SPF is easier than DKIM, so I will only setup SPF for the moment.

First we need a DNS record:

```text
HOSTNAME      TTL    TYPE    PRIO    VALUE
digitao.io           A               v=spf1 mx ~all
```

And it is done :-)
