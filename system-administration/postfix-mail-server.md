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
# hostname -f
```

If it is not the one configured in the DNS record, I should do:

```console
# hostnamectl set-hostname mail.digitao.io
```

## Installation

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

## Send/Receive emails

To send Email, I can use the `mail` command in the `mailutils` package in Ubuntu:

```conosle
# apt install mailutils
```

And then write a short mail:

```console
user@mail:~$ mail -a FROM:<system-user-name>@digitao.io <gmail-user-name>@gmail.com
Cc: 
Subject: Test
This Email is sent from a mail server setup by myself just now.
```

To send the Email, simply press `Ctrl+D`, and the mail will be found in the GMail now.

To receive Email, simply type use mail command:

```console
# mail
```

It is interactive, so type `?` and `Enter` for a short command instructions. And some useful commands are:

* `1`: Read the first mail
* `h`: List the head of the mails
* `d 1`: Delete first mail
* `q`: Quit mail program

