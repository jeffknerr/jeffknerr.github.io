---
layout: post
title:  "LDAP for small CS Department"
date:   2021-04-28 06:54:00 -0400
categories: LDAP linux debian ubuntu
---
Notes on setting up an LDAP server for a small CS department.

Our site is not large (10+ Professors, 400-600 active users), but we
run all of our own servers (email, NFS, web, LDAP, etc). Most of our servers
run debian (currently debian 10/buster), and our lab computers run ubuntu
(currently 20.04).

Our LDAP server is now a virtual machine running on a kvm host.
I set it up with 16GB of memory, but it barely uses any of that (8GB
would have been fine). I also gave it 10GB of disk, but it currently
uses only about 4.2GB.

In this post I will show how we:

- set up the LDAP server with users and groups on debian 10
- set up the LDAP clients using ubuntu 20.04
- add a custom LDAP attribute
- back up our LDAP server

Here are a bunch of links I have used while trying to figure this out:

- [server-world LDAP page] (has tons of good info, some I didn't use like replication)
- [debian ldap wiki]
- [zytrax book] (link currently gives SSL error, needs TLS 1.0 or 1.1)
- [tyler memberof page]
- [Cyrill Gremaud new schema page]

You will probably also want to clone my 
[ldapscripts repo](https://github.com/jeffknerr/ldapscripts), which 
contains the scripts and ldif files used below:

{% include codeHeader.html %}
```bash
git clone https://github.com/jeffknerr/ldapscripts.git
```

# LDAP server

## install pkgs on ldap server

```bash
$ sudo apt-get install slapd shelldap ldap-utils ldapscripts 
$ sudo dpkg-reconfigure slapd
```

Make sure to **use MDB**, and set up a good _admin_ password,
which you shouldn't forget (we store our password in an ansible vault file:
`ansible-vault create ldap_pw.yml`). 

You will also be asked for
your ldap domain info. For this page I will use `dc=cs,dc=college,dc=edu`
as an example (as in user@cs.college.edu).

Also, if you need/want to restart the whole process (I did many times), 
you can do this to wipe out the current _slapd_ files:

```bash
$ sudo service slapd stop
$ sudo apt-get remove --purge slapd
$ sudo \rm -rf /var/lib/slapd
```
then repeat above install and reconfigure commands.

## add people and groups org units

Add the organizational units using ldif files (which I created
already, using an editor like vim):

```bash
$ cat base.ldif
dn: ou=people,dc=cs,dc=college,dc=edu
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=cs,dc=college,dc=edu
objectClass: organizationalUnit
ou: groups

$ sudo ldapadd -x -D cn=admin,dc=cs,dc=college,dc=edu -W -f base.ldif
Enter LDAP Password:
adding new entry "ou=people,dc=cs,dc=college,dc=edu"

adding new entry "ou=groups,dc=cs,dc=college,dc=edu"

```

Running the `ldapadd` command will ask for the admin password you 
configured above.

Again, see my 
[ldapscripts repo](https://github.com/jeffknerr/ldapscripts) 
for copies of scripts and ldif files.

## add logging

```bash
$ cat logging.ldif
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats

$ sudo ldapmodify -H ldapi:/// -Y EXTERNAL -f logging.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"
```

## SSL cert

I use [certbot/letsencrypt](https://certbot.eff.org/) to create an SSL cert.

{% include codeHeader.html %}
```bash
sudo apt-get install certbot ; sudo certbot certonly --standalone
```

When certbot asks for your *domain name*, what it really wants is
your *hostname* (i.e., the hostname of your ldap server -- in my 
example case, `ldap.cs.college.edu`).

Also, if you have a firewall, you may need to allow ports 80/443 to 
your LDAP server for this step (and any renewals).

### make letsencrypt files all readable by openldap

```bash
$ cd /etc/letsencrypt
$ sudo chgrp -R openldap archive
$ sudo chmod -R g+rX archive
$ sudo chgrp -R openldap live
$ sudo chmod -R g+rX live
```

Also add these to the certbot post renew hook!

```bash
$ sudo cat /etc/letsencrypt/renewal-hooks/post/fixPermissions 
#!/bin/bash

# make SSL keys readable by openldap
cd /etc/letsencrypt
chgrp -R openldap live
chgrp -R openldap archive
chmod -R g+rX live
chmod -R g+rX archive

$ sudo chmod 755 /etc/letsencrypt/renewal-hooks/post/fixPermissions 
```

### now add SSL info to ldap
   
Assuming my ldap server's hostname is `ldap.cs.college.edu`:

```bash
$ cat ssl.ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/letsencrypt/live/ldap.cs.college.edu/cert.pem
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/letsencrypt/live/ldap.cs.college.edu/privkey.pem
-
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/letsencrypt/live/ldap.cs.college.edu/chain.pem
    
$ sudo ldapmodify -H ldapi:/// -Y EXTERNAL -f ssl.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

```


### other SSL config file changes

Again, assuming our ldap server is `ldap.cs.college.edu`:

```bash
$ cat /etc/ldap/ldap.conf
BASE        dc=cs,dc=college,dc=edu
URI         ldaps://ldap.cs.college.edu
TLS_CACERT  /etc/letsencrypt/live/ldap.cs.college.edu/chain.pem
```

Change to use ldaps:

```bash
$ sudo vim /etc/default/slapd      (change SLAPD_SERVICES="ldaps:/// ldapi:///")
$ sudo service slapd restart
```

And test a few things:

```bash
$ sudo ldapwhoami -H ldaps://ldap.cs.college.edu -x 
anonymous
```
    
```bash
$ sudo ldapwhoami -x -D cn=admin,dc=cs,dc=college,dc=edu -W -H ldaps://ldap.cs.college.edu
Enter LDAP Password: xxxxxxxxxx
dn:cn=admin,dc=cs,dc=college,dc=edu
```

Test if we can *see* the letsencrypt info from another computer (hit `Ctrl-d` when done):

```bash
labcomputer$ openssl s_client -connect ldap.cs.college.edu:636 -showcerts              
```

See if server is listening on ldaps port (636):

```bash
labcomputer$ nmap ldap.cs.college.edu
...
636/tcp open  ldapssl
...
```

## change db max size

```bash
$ cat maxsizedb.ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcDbMaxSize
olcDbMaxSize: 10737418240

$ sudo ldapmodify -H ldapi:/// -Y EXTERNAL -f maxsizedb.ldif
```

## add/change indexing

```bash
$ cat indexing.ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcDbIndex
olcDbIndex: sn pres,sub,eq
-
add: olcDbIndex
olcDbIndex: displayName pres,sub,eq
-
add: olcDbIndex
olcDbIndex: default sub
-
add: olcDbIndex
olcDbIndex: uniqueMember eq
-
add: olcDbIndex
olcDbIndex: givenName eq
-
add: olcDbIndex
olcDbIndex: dc eq
-
delete: olcDbIndex
olcDbIndex: cn,uid eq
-
add: olcDbIndex
olcDbIndex: cn pres,sub,eq
-
add: olcDbIndex
olcDbIndex: uid pres,sub,eq

$ sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f ./indexing.ldif
```

## change access

Allow admin to write, user to write their own info (for changing passwords,
gecos info, or shell), and read by others.

```bash
$ cat access.ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {1}to attrs=loginShell,gecos
  by dn="cn=admin,dc=cs,dc=college,dc=edu" write
  by self write
  by * read
    
$ sudo ldapmodify -H ldapi:/// -Y EXTERNAL -f access.ldif
```

## change password format

Use [sha512 with 12-character salt](https://en.wikipedia.org/wiki/Crypt_(C)):

```bash
$ cat passwd.ldif
dn: cn=config
add: olcPasswordHash
olcPasswordHash: {CRYPT}
-
add: olcPasswordCryptSaltFormat
olcPasswordCryptSaltFormat: $6$%.12s

$ sudo ldapmodify -H ldapi:/// -Y EXTERNAL -f passwd.ldif
```

## fix limits

For some reason the default limits are set very low (something like 500 users).
We have 1600+ total user accounts, so we need to increase these limits. 

```bash
$ cat limits.ldif
dn: cn=config
changetype: modify
replace: olcSizeLimit
olcSizeLimit: 8000
-
replace: olcTimeLimit
olcTimeLimit: 3500

$ sudo ldapmodify -H ldapi:/// -Y EXTERNAL -f limits.ldif
```

**NOTE: this also seems to revert back to the old default limits everytime
slapd is restarted or the server is rebooted.** I have a hack that forces
these new limits via cron every few hours. Would love to know why these new limits
don't stick...

## add users and groups

*Finally*, we can try to add a users and  groups! 

For our server we made a giant 
`usersgroups.ldif` file to import
all of out users and groups into the new LDAP setup:

```bash
$ sudo ldapadd -c -Wx -D "cn=admin,dc=cs,dc=college,dc=edu" -H ldaps://ldap.cs.college.edu -f usersgroups.ldif
```

Here are two sample entries from this `usersgroups.ldif` file, one
for a user (`user1`) and one for a group (`sudo`):

```
dn: uid=user1,ou=people,dc=cs,dc=college,dc=edu
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: user1
sn: Lastname
givenName: Firstname
cn: Firstname Lastname
displayName: Firstname Lastname
uidNumber: 5372
gidNumber: 100
gecos: Firstname Lastname
loginShell: /bin/bash
homeDirectory: /home/user1
mail: user1@cs.college.edu
shadowExpire: -1
shadowFlag: 0
shadowWarning: 7
shadowMin: 0
shadowMax: 99999
shadowLastChange: 10770
userPassword:: e0NSWVBUfSQ2JGpteExGM2FLUQ5STSRIa2sLcWovOGdmTTcwNkopWGdrWUdGOEp
 ySDVKb0pNYmRLMTJScEF2Y1BoR0dQekUuOXdPWmRxcmxPS0RjbU8xN3dINFFjdkZnNm9KNGRsMDhV
  Tm1ELw==

dn: cn=sudo,ou=groups,dc=cs,dc=college,dc=edu
objectClass: posixGroup
cn: sudo
gidNumber: 27
memberUid: user1
memberUid: user2
```

These entries and this file can either be produced by dumping your
current ldap server data (if you have one):

```bash
$ sudo ldapsearch -Wx -D "cn=admin,dc=cs,dc=college,dc=edu" -b "dc=cs,dc=college,dc=edu" -H ldaps://currentldapserver.cs.college.edu -LLL > usersgroups.ldif
```

*Or* by writing a script to convert unix `/etc/passwd,shadow,group` files
into the above format (with blank lines between each entry).

Once you have users and groups added, use `ldapsearch` to test:

```bash
# run these on the new ldap server

# see all entries
$ ldapsearch -x -H ldapi:///

# see all people
$ sudo ldapsearch -x -LLL -D 'CN=admin,DC=cs,DC=college,DC=edu' -W -H ldaps://ldap.cs.college.edu -b 'OU=people,DC=cs,DC=college,DC=edu'

# see one specific user
$ sudo ldapsearch -x -LLL -D 'CN=admin,DC=cs,DC=college,DC=edu' -W -H ldaps://ldap.cs.college.edu -b 'OU=people,DC=cs,DC=college,DC=edu' -s sub '(uid=user1)'

# see all groups
$ sudo ldapsearch -x -LLL -D 'CN=admin,DC=cs,DC=college,DC=edu' -W -H ldaps://ldap.cs.college.edu -b 'OU=groups,DC=cs,DC=college,DC=edu'
```

## configure ldapscripts

If you want to make changes from the command line, on the ldap server,
use the ldapscripts package/scripts. These will let you run (or script) things
like adding a group (`ldapaddgroup`) or adding a user to a group (`ldapaddusertogroup`).
From the new ldap server this would look something like this (adding `newgrp` with gid 2001,
then adding `user1` to the new group):

```bash
sudo ldapaddgroup newgrp 2001
sudo ldapaddusertogroup user1 newgrp
```

The config files are in `/etc/ldapscripts`. Here are the changes I made:

```bash
$ cd /etc/ldapscripts
$ sudo vim ldapscripts.conf
SERVER="ldaps://ldap.cs.college.edu"
SUFFIX="dc=cs,dc=college,dc=edu"
GSUFFIX="ou=groups"
USUFFIX="ou=people"
BINDDN="cn=admin,dc=cs,dc=college,dc=edu"
BINDPWDFILE="/etc/ldapscripts/ldapscripts.passwd"
```

Also add the LDAP admin password (without a new line) to 
the `/etc/ldapscripts/ldapscripts.passwd` file (and make sure others can't read that file).
In vim you can use `:set binary` and `:set noeol` to make a file that doesn't add a newline 
character at the end of the file.

## memberOf stuff

This might be optional. Some things like web apps need to know if 
a user is in a specific group. To be able to do that, you have 
to add the "memberOf" overlay.

### need to add overlay

```bash
$ cat memberof.ldif
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: memberof.la
$ sudo ldapmodify -H ldapi:/// -Y EXTERNAL -f memberof.ldif
```

### now config the overlay

```bash
$ cat refint.ldif
dn: olcOverlay=memberof,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcMemberOf
olcOverlay: memberof
olcMemberOfRefint: TRUE
olcMemberOfDangling: ignore
olcMemberOfGroupOC: groupOfNames
$ sudo ldapadd -H ldapi:/// -Y EXTERNAL -f refint.ldif
```

### now add the pwapp groupofnames group 

As an example, here I am adding a `pwapp` (short for password app) group. I will
add any admin users to this group, so my password app will be able to
check the group when allowing admin access. Not sure why, but this can't
be a posixGroup.

```bash
$ cat groupofnames.ldif
dn: cn=pwapp,ou=groups,dc=cs,dc=college,dc=edu
objectclass: groupofnames
cn: pwapp
description: pw app admins
member: 
$ sudo ldapadd -x -D cn=admin,dc=cs,dc=college,dc=edu -H ldapi:/// -W -f groupofnames.ldif
```

### now add a user to the pwapp group

```bash
$ cat pwappGROUP.ldif
dn: uid=user1,ou=people,dc=cs,dc=college,dc=edu
changetype: modify
add: memberOf
memberOf: cn=pwapp,ou=groups,dc=cs,dc=college,dc=edu
-

dn: cn=pwapp,ou=groups,dc=cs,dc=college,dc=edu
changetype: modify
add: member
member: uid=user1,ou=people,dc=cs,dc=college,dc=edu
-

$ sudo ldapmodify -c -x -D cn=admin,dc=cs,dc=college,dc=edu -H ldapi:/// -W -f pwappGROUP.ldif
```

### see the results

```bash
$ sudo ldapsearch -x -LLL -D 'CN=admin,DC=cs,DC=college,DC=edu' -W -H ldaps://ldap.cs.college.edu  -s sub '(cn=pwapp)'
Enter LDAP Password:
dn: cn=pwapp,ou=groups,dc=cs,dc=college,dc=edu
objectClass: groupOfNames
cn: pwapp
description: pw app admins
member: uid=user1,ou=people,dc=cs,dc=college,dc=edu
```

### simpler search

Or search a specific user, to see if they are a member of:

```bash
$ sudo ldapsearch -x -LLL  -H ldaps://ldap.cs.college.edu -b "uid=user1,ou=people,dc=cs,dc=college,dc=edu" "(memberOf=cn=pwapp,ou=groups,dc=cs,dc=college,dc=edu)" uid
    
dn: uid=user1,ou=people,dc=cs,dc=college,dc=edu
uid: user1
```

(will have no output if they are *not* a member of the group)

# adding CSUser attribute

For one of our web apps we needed new (custom) CSUser LDAP attributes.
We wanted to know if our users were logging in to our password
service for the first time and/or record the date and time of their
latest login. We also wanted to record if users had already agreed
to our "terms of service". So we added two new attributes: `csAgreed`
(a boolean) and `csLastSeen` (a datetime).  For the `csLastSeen` datetime,
we set all new users' "last seen" time to 1970 (so we could easily check
if they had ever logged in).

Here's one way to add these custom attributes. 

Note: there is a unique organization identifier here (10672). I 
[looked that up somewhere](https://www.iana.org/assignments/enterprise-numbers/enterprise-numbers)
for my college.

## add new schema and attributes

Made a new schema file called `passwordservice.schema`
in a tmp directory:

```bash
$ cd /tmp
$ cat passwordservice.schema
#
# 1.3.6.1.4.1   base OID
# 10672         organization idenfifier
# 1             if an objectclass
# 2             if an attribute
# yyyy.mm.dd    date of creation
# n             extra identifier

attributetype ( 1.3.6.1.4.1.10672.2.2020.06.20.1
        NAME 'csAgreed'
        DESC 'boolean for user agreement'
        SYNTAX 1.3.6.1.4.1.1466.115.121.1.7 )

attributetype ( 1.3.6.1.4.1.10672.2.2020.06.20.2
        NAME 'csLastSeen'
        DESC 'datetime of last login to password service'
        SYNTAX 1.3.6.1.4.1.1466.115.121.1.24 )

objectclass ( 1.3.6.1.4.1.10672.1.2020.06.20.1
        NAME 'CSUser'
        DESC 'container for cs password service info'
        AUXILIARY
        MAY ( csAgreed $ csLastSeen ) )
```

## convert to ldif

Use `slaptest` to turn the schema file into an ldif file:

```bash
$ cd /tmp
$ cat passwordservice.conf
include /tmp/passwordservice.schema
$ mkdir foo
$ slaptest -f passwordservice.conf -F /tmp/foo
$ cd foo
$ cd cn=config
$ cd cn=schema
$ cat cn\=\{0\}passwordservice.ldif
$ cp cn\=\{0\}passwordservice.ldif ~/passwordservice.ldif
$ cd
$ vim passwordservice.ldif
(keep only the stuff shown below, modify the dn and cn)
$ cat passwordservice.ldif
dn: cn=cs,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: cs
olcAttributeTypes: {0}( 1.3.6.1.4.1.10672.2.2020.06.20.1 NAME 'csAgreed'
  DESC 'boolean for user agreement' SYNTAX 1.3.6.1.4.1.1466.115.121.1.7 )
olcAttributeTypes: {1}( 1.3.6.1.4.1.10672.2.2020.06.20.2 NAME 'csLastSee
 n' DESC 'datetime of last login to password service' SYNTAX 1.3.6.1.4.1.146
 6.115.121.1.24 )
olcObjectClasses: {0}( 1.3.6.1.4.1.10672.1.2020.06.20.1 NAME 'CSUser' DE
 SC 'container for cs password service info' AUXILIARY MAY ( csAgre
 ed $ csLastSeen ) )
```

Again, see the [Cyrill Gremaud new schema page] for more details.

## use the ldif file

Now add the new schema/attributes using the ldif file and verify it's there:

```bash
$ sudo  ldapadd  -H ldapi:/// -Y EXTERNAL -f passwordservice.ldif
adding new entry "cn=cs,cn=schema,cn=config"
$ sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config cs*
...
...
dn: cn={4}cs,cn=schema,cn=config
...
...
```

Not sure if this is needed, but copied the ldif and schema files to `/etc/ldap/schema`
and restarted slapd:

```bash
$ cd /etc/ldap/schema/
$ cp ~/passwordservice.ldif .
$ cp /tmp/passwordservice.schema .
$ /etc/init.d/slapd status
$ /etc/init.d/slapd restart
$ /etc/init.d/slapd status
```

## add new stuff for each user

Now add objectClass and attributes to a user:

```bash
$ cat addpsinfo.ldif
dn: uid=user1,ou=people,dc=cs,dc=college,dc=edu
changetype: modify
add: objectClass
objectClass: CSUser
-
add: csAgreed
csAgreed: TRUE
-
add: csLastSeen
csLastSeen: 20140102030405.678Z
-

$ sudo ldapadd -Wx -D "cn=admin,dc=cs,dc=college,dc=edu" -H ldaps://ldap.cs.college.edu -f addpsinfo.ldif
# and see the new data
$ sudo ldapsearch -x -LLL -D 'CN=admin,DC=cs,DC=college,DC=edu' -W -H ldaps://ldap.cs.college.edu -b 'OU=people,DC=cs,DC=college,DC=edu' -s sub '(uid=user1)'
...
objectClass: CSUser
...
csAgreed: TRUE
csLastSeen: 20140102030405.678Z
```

Note: for the above user, we set them up to have already agreed to our "terms of service",
and to have a "last seen" date of Jan 2, 2014. For new accounts we set these attributes 
to `FALSE` and `19700101000000-0500`.

# Backing up the LDAP server

I use the following script (monthly via cron) to get a dump of all ldap 
data in an ldif file format. This is a klunky backup, but combined with 
the above info, allows me to recreate a new ldap server in less than a 
day, if needed.

```bash
#! /bin/sh

# dump the ldap data

if [ `id -u` != 0 ]
then
  echo "You must have root privilege to run this program."
  exit 1
fi

# assumes ldap admin password stored in this file
LDAPPASSFILE=/root/ldappasswd

# will put the ldif backup file here
BACKDIR=/root/ldapbackups

# ldap site info
LDAPCN="cn=admin,dc=cs,dc=college,dc=edu"
LDAPBASE="dc=cs,dc=college,dc=edu" 
LDAPURL="ldaps://ldap.cs.college.edu"

# check for/make dir
if [ ! -d $BACKDIR ] ; then
  mkdir $BACKDIR
  chmod 700 $BACKDIR
fi

# create unique file name
d=`date +'%m%d%Y'`
fn=ldap_dump-${d}.ldif

# make the backup
cd $BACKDIR
ldapsearch -Wx -D $LDAPCN -y $LDAPPASSFILE -b $LDAPBASE -H $LDAPURL -LLL > $fn
```

# ubuntu LDAP client

Here are the minimal steps needed to set up an ubuntu (20.04) ldap client:

- install some packages:

{% include codeHeader.html %}
```bash
sudo apt-get install libnss-ldapd libpam-ldapd ldap-utils python3-ldap3
```

The apt-get will install other packages, like `nslcd`, and
ask you for information about your ldap server and setup (make sure
to use **ldaps** for the uri):

* server uri: `ldaps://<IP address of your ldap server>`
* search base: `dc=cs,dc=college,dc=edu`
* check server's SSL cert: `never`
* services to configure: passwd, group, shadow (will set `/etc/nsswitch.conf`)

- check/edit `/etc/nslcd.conf` to make sure it is correct:

```bash
$ sudo cat /etc/nslcd.conf | grep -v ^#

uid nslcd
gid nslcd

uri ldaps://<IP address of your ldap server>

base dc=cs,dc=college,dc=edu

tls_reqcert never
tls_cacertfile /etc/ssl/certs/ca-certificates.crt
```

- see if the ldap accounts show up

```bash
$ getent passwd
```

- reboot the client computer:

Even though it looks like ldap is working, I always need a reboot here
before I can log in as a normal user. Not sure why...

{% include codeHeader.html %}
```bash
sudo sync ; sudo sync ; sudo reboot
```

[server-world LDAP page]: https://www.server-world.info/en/note?os=Debian_10&p=openldap&f=1
[debian ldap wiki]: https://wiki.debian.org/LDAP/OpenLDAPSetup
[zytrax book]: https://www.zytrax.com/books/ldap/
[our using ldap page]: https://www.cs.swarthmore.edu/syshelp/ldap.html
[THIS ldap page]: https://www.cs.swarthmore.edu/syshelp/ldap_install.html
[tyler memberof page]: https://tylersguides.com/guides/openldap-memberof-overlay/
[Cyrill Gremaud new schema page]: https://www.cyrill-gremaud.ch/how-to-add-new-schema-to-openldap-2-4/
