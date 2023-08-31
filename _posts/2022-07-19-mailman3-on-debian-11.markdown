---
layout: post
title:  "mailman3 on debian 11"
date:   2022-07-19 12:17:00 -0500
categories: mailman3 debian
---

I just struggled to get mailman3 set up on a Debian 11 (Bullseye) server,
so below are some hints and tips. I was converting an old server running 
mailman2, and thought it would be easy. Took me more than a few tries and
the documentation seems all over the place on this one.

I originally tried the debian packages, but couldn't get everything to work.
After reading some posts about problems with the debian packages, decided
to try the [virtualenv install][venv]. That's what I ended up getting to
work, but it may just have been a bad setting earlier, so the debian packages might 
work for you. There are lots of pieces to this one (mailman software, hyperkitty
archiver, django/postorius web interface, database server, email server, web server). 
Hopefully these notes and tips will help anyone struggling with all of these pieces.

## docs I followed

- [the main virtualenv install instructions][venv]
- [migrating lists and archives from mailman2][migrating]
- [list of all mailman settings][settings]
- [issues with hyperkitty not archiving][hkissues]
- [issues with mailman after debian upgrade][mmissues]
- [how to do stuff from the command line][adding]
- [using mailmanclient in scripts][examples]

## overview

Here's the overview:

* note: if you have debian mailman3 and python-django packages installed, 
  you probably want to purge/delete those
* follow the [venv install instructions][venv] (see details below if you get stuck)
* use the `mailman info` and `mailman-web createsuperuser` commands (as user
  mailman, in the virtualenv) to create a django admin user
* go to the django admin site (https://your_domain/admin), log in as your
  django admin user
* edit the site (example.com) by clicking on Sites, then click on example.com,
  then change the DisplayName and DomainName to be what you want
* restart mailman3 and mailmanweb
* you should now be able to LogIn and create a test mailing list, and try subscribing
  and posting to it
* I had to specifically add 127.0.0.1 to the ALLOWED_HOSTS entry in my settings.py
  file (could also be your mailman-web.py file) and restart everything before
  my posts showed up in the archiver (hyperkitty)
* if you have old lists from a previous server, use the [migrating][migrating]
  doc to import the list settings and the archived posts
* when a doc lists a python/django command, such as `python manage.py hyperkitty_import....`,
  that can be run with this: `mailman-web hyperkitty_import...` (as the mailman
  user, in the virtualenv
* write some scripts that use [mailmanclient][examples] to add/rm users

More details are below (but not every step).

## the details

#### set up postgres

```
$ sudo apt install python3-dev python3-venv sassc lynx
$ sudo apt install postgresql
$ sudo -u postgres psql
postgres=# create database mailman;
postgres=# create database mailmanweb;
postgres=# create user mailman with encrypted password 'MYPGPASSWORD';
postgres=# grant all privileges on database mailman to mailman;
postgres=# grant all privileges on database mailmanweb to mailman;
postgres=# \q
```

#### add mailman user and software

```
$ sudo useradd -m -d /opt/mailman -s /usr/bin/bash mailman
$ sudo chown mailman:mailman /opt/mailman
$ sudo su mailman
mm$ cd /opt/mailman
mm$ python3 -m venv venv
mm$ echo 'source /opt/mailman/venv/bin/activate' >> /opt/mailman/.bashrc
mm$ source /opt/mailman/venv/bin/activate
(venv)$ pip install wheel mailman psycopg2-binary\<2.9
```

#### mailman configs

At this point you set up the `/etc/mailman3/mailman.cfg` file:

```
$ cat /etc/mailman3/mailman.cfg
[paths.here]
var_dir: /opt/mailman/mm/var

[mailman]
layout: here
site_owner: ME@MYDOMAIN.EDU
noreply_address: root
default_language: en
listname_chars: [-_.0-9a-z]

[database]
class: mailman.database.postgresql.PostgreSQLDatabase
url: postgres://mailman:MYPGPASSWORD@localhost/mailman

[archiver.prototype]
enable: yes

[archiver.hyperkitty]
class: mailman_hyperkitty.Archiver
enable: yes
configuration: /etc/mailman3/mailman-hyperkitty.cfg

[shell]
history_file: $var_dir/history.py

[mta]
verp_confirmations: yes
verp_personalized_deliveries: yes
verp_delivery_interval: 1

[webservice]
admin_user: restadmin
admin_pass: myrestadminpassword
```

#### postfix configs

I installed mailman3 on my email server, so I already had postfix running
for this site. Just added/fixed a few things in `/etc/postfix/main.cf`:

```
$ cat /etc/postfix/main.cf
...
relay_domains = MYDOMAIN.EDU, hash:/opt/mailman/mm/var/data/postfix_domains
transport_maps = hash:/opt/mailman/mm/var/data/postfix_lmtp
...
alias_maps = hash:/etc/aliases, ldap:/etc/postfix/ldap-aliases.cf, hash:/opt/mailman/mm/var/data/postfix_lmtp
...
```

#### systemd stuff and cron

I followed the 
[venv install instructions][venv] 
for systemd and cron.

After this, running mailman info (as mailman user with virtualenv active) should 
give you an output which looks something like below:

```
(venv)$ mailman info
GNU Mailman 3.3.2 (Tom Sawyer)
Python 3.8.5 (default, Jul 28 2020, 12:59:40)
[GCC 9.3.0]
config file: /etc/mailman3/mailman.cfg
db url: postgres://mailman:MYPGPASSWORD@localhost/mailman
devmode: DISABLED
REST root url: http://localhost:8001/3.1/
REST credentials: restadmin:myrestadminpassword
```

Note: make sure them `mailman` user can read the mailman.cfg file.

#### hyperkitty

```
(venv) $ pip install mailman-web mailman-hyperkitty
```

And the `settings.py` config file for hyperkitty:

```
$ cat /etc/mailman3/settings.py

from mailman_web.settings.base import *
from mailman_web.settings.mailman import *

ADMINS = (
    ('Mailman Suite Admin', 'root@mydomain.edu'),
    ('Me', 'me@mydomain.edu'),
)

# Postgresql database setup.
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'mailmanweb',
        'USER': 'mailman',
        'PASSWORD': 'MYPGPASSWORD',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}

STATIC_ROOT = '/opt/mailman/web/static'
# Make sure that this directory is created or Django will fail on start.
LOGGING['handlers']['file']['filename'] = '/opt/mailman/web/logs/mailmanweb.log'

# had to add the 127.0.0.1 before hyperkitty started archiving
ALLOWED_HOSTS = [
    "127.0.0.1",
    "localhost",  
    "myserver.mydomain.edu",
]

SITE_ID = 1

# Set this to a new secret value (not sure where this is used...).
SECRET_KEY = 'dskjfjdljflkj908943urjijf09fu0'

# Set this to match the api_key setting in
# /opt/mailman/mm/mailman-hyperkitty.cfg (quoted here, not there).
MAILMAN_ARCHIVER_KEY = 'ThisIsTheAPIKey'
MAILMAN_REST_API_USER = 'restadmin'
MAILMAN_REST_API_PASS = 'myrestadminpassword'

DEFAULT_FROM_EMAIL = 'root@mydomain.edu'
SERVER_EMAIL = 'root@mydomain.edu'

LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'America/New_York'
USE_I18N = True
USE_L10N = True
USE_TZ = True
EMAILNAME = 'mydomain.edu'
```

NOTE: I had to add the 127.0.0.1 to `ALLOWED_HOSTS` before hyperkitty started archiving!!

#### more DB stuff???

Note: also had to delete/purge python-django debian pkgs.

```
(venv)$  pip install --upgrade pip
(venv)$  pip install wheel mailman pymysql 
(venv)$  pip install mistune==2.0.0rc1
[due to a version mismatch bug???]
(venv)$  mailman-web migrate
(venv)$  pip install python-gettext
(venv)$  mailman-web compilemessages  (this one failed???) 
```

#### uwsgi config

```
(venv)$ pip install uwsgi
```

And set up the uwsgi.ini file as 
[venv install instructions][venv] say to.

Same for the systemd stuff for mailman-web.

```
$ sudo systemctl start mailmanweb
$ systemctl status mailmanweb
```

And the mailman-web cron setup...

#### nginx config

I also already had nginx running on this server to serve
out rspamd web pages, so I just added to the enabled site config
(replace myserver.mydomain with your FQHN):

```
upstream mailman3 {
    server unix:/run/mailman3-web/uwsgi.sock fail_timeout=0;
}

# send anything coming to port 80 to ssl port
server {
    listen 80;
    server_name myserver.mydomain;
    location / {
        return 301 https://$server_name$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name myserver.mydomain;
    root   /var/www/html;
    index index.html index.htm index.nginx-debian.html;

    ssl_certificate /etc/letsencrypt/live/myserver.mydomain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myserver.mydomain/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    server_tokens off;

# mailman stuff...only allow admin access from specific IPs
    location / {
        allow          130.20.0.0/16;
        allow          1.2.3.4/32;
        deny all; 
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }

    location /static/ {
        allow          130.20.0.0/16;
        allow          1.2.3.4/32;
        deny all; 
        alias /opt/mailman/web/static/;
    }

    access_log /var/log/nginx/mailman3/access.log combined;
    error_log /var/log/nginx/mailman3/error.log;
}
```


#### made an admin user

```
(venv) mailman@myserver:/etc/mailman3$ cd
(venv) mailman@myserver:~$ pwd
/opt/mailman
(venv) mailman@myserver:~$ mailman info
GNU Mailman 3.3.5 (Tom Sawyer)
Python 3.9.2 (default, Feb 28 2021, 17:03:44)
[GCC 10.2.1 20210110]
config file: /etc/mailman3/mailman.cfg
db url: postgres://mailman:longer_passwords_are_better@localhost/mailman
devmode: DISABLED
REST root url: http://localhost:8001/3.1/
REST credentials: restadmin:hPieJra7EzGaRhXoa84J6FqQcwzLlaip3zbcWYKEgVERtGsD
(venv) mailman@myserver:~$ mailman-web createsuperuser
Username (leave blank to use 'mailman'): superadmin
Email address: you@myserver.mydomain
Password:
Password (again):
Superuser created successfully.
(venv) mailman@myserver:~$
```

Then go to admin site 
`https://myserver.mydomain/admin/sites/site/`
and edit example.com site, change name to myserver.mydomain.


## importing an old list

- create list in web app 
- import the old list config.pck file
- migrate list archives
- rebuild index
```
(venv) mailman@myserver:/etc/mailman3$ mailman import21 list@mydomain oldserver/lists/all/config.pck
(venv) mailman@myserver:/etc/mailman3$ mailman-web hyperkitty_import -l list@mydomain oldserver/archives/private/list.mbox/list.mbox
(venv) mailman@myserver:/etc/mailman3$ mailman-web update_index_one_list list@mydomain
```


## script to add/rm users

```
$ sudo -u mailman /opt/mailman/bin/rmmail.py --help
Usage: rmmail.py [OPTIONS]

Options:
  --user TEXT   username to remove [required]
  --mlist TEXT  remove from this mailing list (e.g., list1)
  --help        Show this message and exit.
$ sudo -u mailman /opt/mailman/bin/rmmail.py --user testf204 --mlist list2
$ sudo -u mailman /opt/mailman/bin/rmmail.py --user testf204 --mlist list1
$ cat /opt/mailman/bin/rmmail.py
#!/opt/mailman/venv/bin/python3
"""
Summer 2022
Jeff Knerr

use mailmanclient to control mailman3 stuff
https://docs.mailman3.org/projects/mailmanclient/en/latest/src/mailmanclient/docs/using.html
"""                  
from utils import *   
import os                   
import click                   
import urllib
import sys

@click.command()
@click.option("--user", required=True, help="username to add/remove")
@click.option("--mlist", default="mmtest", help="remove from this mailing list (e.g., all)")
@click.option("--add", is_flag=True, default=False, help="add to list (e.g., default is remove)")
def main(user, mlist, add):
    mmAuthFile = os.environ["HOME"] + "/bin/AUTH"
    client = getCredentials(mmAuthFile)
    if add:
        addlist(user, mlist, client)
    else:
        rmlist(user, mlist, client)

def addlist(user, mlist, client):
    domain = 'mydomain'
    try:
        thelist = client.get_list('%s@%s' % (mlist, domain))
    except urllib.error.HTTPError:
        print("list %s not found???" % (mlist))
        sys.exit(1)
    try:
        thelist.subscribe('%s@%s' % (user, domain),
                pre_verified=True, pre_confirmed=True)
    except urllib.error.HTTPError:
        print("%s@%s already subscribed to %s???" % (user, domain, mlist))
        sys.exit(1)

def rmlist(user, mlist, client):
    domain = 'mydomain'
    thelist = client.get_list('%s@%s' % (mlist, domain))
    if "@" in user:
        try:
            thelist.unsubscribe('%s' % (user))
        except ValueError:
            print("%s@ not subscribed to %s" % (user, mlist))
            sys.exit(1)
    else:
        try:
            thelist.unsubscribe('%s@%s' % (user, domain))
        except ValueError:
            print("%s@%s not subscribed to %s" % (user, domain, mlist))
            sys.exit(1)

main()
```

## still to do

- check all config files, make sure everything is secure

[venv]: https://docs.mailman3.org/en/latest/install/virtualenv.html
[migrating]: https://docs.mailman3.org/en/latest/migration.html
[settings]: https://docs.mailman3.org/projects/mailman-web/en/latest/settings.html
[hkissues]: https://gitlab.com/mailman/hyperkitty/-/issues/412
[mmissues]: https://lists.mailman3.org/archives/list/mailman-users@mailman3.org/thread/YRPFYMQD6J5AYLC3EWWTESSNPODGDH2V/
[adding]: https://docs.mailman3.org/projects/mailman/en/latest/src/mailman/commands/docs/addmembers.html
[examples]: https://docs.mailman3.org/projects/mailmanclient/en/latest/src/mailmanclient/docs/using.html
