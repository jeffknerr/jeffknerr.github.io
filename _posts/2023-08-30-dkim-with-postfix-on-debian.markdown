---
layout: post
title:  "dkim with postfix on debian"
date:   2023-08-30 14:20:00 -0500
categories: dkim postfix debian
---

I added [DKIM][open] to our postfix setup, as I was seeing more and more
rejections on emails that we forward to gmail. Not sure if they are 
getting rejected because *we* didn't have DKIM, or because the original
sending site didn't have DKIM. 

My email server is currently running Debian 11, using postfix
and rspamd. I already had SPF set up on this server.


## docs I followed

- [opendkim instructions][install]
- [validator site][validator]
- [digital ocean guide][ocean]
- [LB guide][lb]

## overview

Here's the overview:

* install debian packages (`opendkim opendkim-tools`)
* edit config file: `/etc/opendkim.conf`
* edit postfix `main.cf` file
* edit `/etc/default/opendkim` file
* add postfix user to opendkim group
* set up `/etc/opendkim/` files
* generate public/private key with `opendkim-genkey`
* add public part to DNS (maybe wait an hour for DNS to update?)
* test the key
* restart postfix and opendkim
* test with validators sites and emails...

## the details

### add packages

```
sudo apt-get install opendkim opendkim-tools
```

## opendkim configs

```
sudo vim /etc/opendkim.conf
```

These are the ones I changed (or left uncommented):

```
Syslog			yes
SyslogSuccess		yes
LogWhy			yes
Canonicalization	relaxed/simple
Mode			sv
OversignHeaders		From
UserID			opendkim
UMask			007
Socket			inet:8891@localhost
PidFile			/run/opendkim/opendkim.pid
TrustAnchorFile		/usr/share/dns/root.key
AutoRestart             Yes
AutoRestartRate         10/1h
ExternalIgnoreList      refile:/etc/opendkim/TrustedHosts
InternalHosts           refile:/etc/opendkim/TrustedHosts
KeyTable                refile:/etc/opendkim/KeyTable
SigningTable            refile:/etc/opendkim/SigningTable
SignatureAlgorithm      rsa-sha256
```

See the [digital ocean site][ocean] for descriptions of some of those.

Also change the *default* opendkim file:

```
cat /etc/default/opendkim | grep -v ^#
RUNDIR=/run/opendkim
SOCKET=inet:8891@localhost
USER=opendkim
GROUP=opendkim
PIDFILE=$RUNDIR/$NAME.pid
EXTRAAFTER=
```

## postfix configs

```
sudo vim /etc/postfix/main.cf
```

I already had rspamd running, so I added the opendkim
socket (localhost:8891) before the rspamd socket:

```
smtpd_milters = inet:localhost:8891, inet:localhost:11332
non_smtpd_milters = inet:localhost:8891, inet:localhost:11332
milter_protocol = 6
milter_mail_macros =  i {mail_addr} {client_addr} {client_name} {auth_authen}
milter_default_action = accept
```

Not sure if needed (since I am using the localhost socket, not a unix
file socket), but added the postfix user to the opendkim group:

```
grep opendkim /etc/group
opendkim:x:216:postfix,pfnobody
```

## create key, edit more opendkim files

Here I am creating a public/private key, using the SELECTOR `202308`.
You can pick whatever you want for the SELECTOR. And replace "mydomain"
with your domain name (e.g., yourdept.yourcollege.edu).

```
sudo mkdir -p /etc/opendkim/keys
cd /etc/opendkim/keys
sudo mkdir mydomain
cd mydomain
sudo opendkim-genkey -s 202308 -d mydomain -b 2048 -v
sudo chown opendkim: 202308.private
```

Edit more opendkim files (the ones listed above in the .conf file):

```
sudo vim /etc/opendkim/TrustedHosts
127.0.0.1
localhost
192.168.0.1/24
*.mydomain
```

```
sudo vim /etc/opendkim/KeyTable
202308._domainkey.mydomain  mydomain:202308:/etc/opendkim/keys/mydomain/202308.private
```

```
sudo vim /etc/opendkim/SigningTable
*@mydomain 202308._domainkey.mydomain
```

## add public key to your DNS

Lots of different ways to do this. I control my own DNS server, 
so I just edited one of my zone files and restarted bind9.

Here's an example of what I added (just output from `202308.txt` file):

```
202308._domainkey  600  IN      TXT     ( "v=DKIM1; h=sha256; k=rsa; "
          "p=BIjANBgkqhkiG9w0BAQc1Og2vKN9Kxdk/vsVD8c2xy8XSs4pEtdNT9pAiFI+09LaJyVF+g3pZkBpk8xDDq8h+mBo1OgGQGkrK76rJoo2TYaiv6XlbBeNMES8bqHKR0BmP5rcRANrRzaQAKZ/rJh2o"
          "JppRMIYhXe/DhlKNObT4AiixpMzhOP+WeBeg6HXSm6YHZaoSCBQjmHUfJJSIxFzRsDHWuQHag9I2yH+JXmyYT3sJkHAZye+pAahDY41cYQO7NfdZ2MAmS0nBu2QIDAQAB" )   ; ----- DKIM key 202308 for mydomain
```

I added `600` for the TTL, so if I needed to change it, it would
only take 10 minutes to propagate.

## test the key

```
sudo opendkim-testkey -d mydomain -s 202308 -vvv
opendkim-testkey: using default configfile /etc/opendkim.conf
opendkim-testkey: checking key '202308._domainkey.mydomain'
opendkim-testkey: key not secure
opendkim-testkey: key OK
```

## restart postfix and opendkim

Before you restart everything, you might want to check your DNS entries
from outside hosts, to see if everyone else can see your new DKIM TXT record.
Run a dig command if you have access to an outside computer:

```
dig 202308._domain.mydomain txt
```

If that worked, you should see your public key displayed.

Once that works, go ahead and restart postfix and opendkim:

```
sudo systemctl restart opendkim.service
sudo systemctl restart postfix
```

## test test test

Run `tail -f /var/log/mail.log`, and also check `mail.warn` and `mail.err`.

Also send a test email from your system to a gmail account. When you
read the email in gmail, select "Show original", and you should see
`DKIM:   'PASS' with domain mydomain`.

Maybe try one of the [validation][validator] sites (send email, look 
at the results).


## still to do

- watch the logs for a few days, make sure all ok
- figure out why lab computer "clients", when sending email with mutt, don't get dkim added
- change 600 TTL to something larger

[open]:http://www.opendkim.org
[install]:http://www.opendkim.org/INSTALL
[ocean]:https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-dkim-with-postfix-on-debian-wheezy
[validator]:https://dkimvalidator.com/
[lb]:https://www.linuxbabe.com/mail-server/setting-up-dkim-and-spf
