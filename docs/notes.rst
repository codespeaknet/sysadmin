Notes
=====

Vagrant Debian base boxes
-------------------------

https://atlas.hashicorp.com/debian/

Everything is based on Debian 9 "Stretch".
The installed and packaged software is much more current and the final release will be soon.

Stuff
-----

rspamd and mailman3 aren't officially packaged on Debian at current versions


Installation
============

1. Updates
----------

- ``aptitude update``
- ``aptitude upgrade``
- ``aptitude install unattended-upgrades``

2. etckeeper
------------

- ``aptitude install etckeeper``
- .gitignore

3. host names
-------------

- **lists.codespeak.net** > ``/etc/hostname``
- **lists.codespeak.net** > ``/etc/mailname``

4. Postfix
----------

- ``aptitude install postfix``
- configuration type "Internet Site"
- mail name **lists.codespeak.net**

5. Mailman 3
------------

Documentation used:

- https://mailman.readthedocs.io/en/latest/src/mailman/docs/mta.html#postfix
- https://github.com/P-EB/mailman3-core/blob/master/debian/mailman3-core.service
- https://wiki.list.org/DOC/Mailman%203%20installation%20experience?action=AttachFile&do=view&target=lm3o_mailman.cfg.txt

Getting the paths to work was tricky.
The key is to set ``layout: debian-manual`` in the ``[mailman]`` section in ``mailman.cfg`` and add a corresponding ``[paths.debian-manual]`` section.

Make sure the ``transport_file_type`` matches the type used for ``transport_maps``, ``local_recipient_maps`` and ``relay_domains``.

Setup steps:

- ``aptitude install virtualenv``
- ``useradd --system --create-home --shell /bin/bash mailman``
- ``usermod -G mailman postfix``
- .. code-block:: ini

    owner_request_special = no
    transport_maps = hash:/home/mailman/var/data/postfix_lmtp
    local_recipient_maps = hash:/home/mailman/var/data/postfix_lmtp
    relay_domains = hash:/home/mailman/var/data/postfix_domains

  >> ``/etc/postfix/main.cf``
- ``mkdir /etc/mailman3``
- .. code-block:: ini

    [mta]
    incoming: mailman.mta.postfix.LMTP
    outgoing: mailman.mta.deliver.deliver
    lmtp_host: mail.example.com
    lmtp_port: 8024
    smtp_host: mail.example.com
    smtp_port: 25
    configuration: /etc/mailman3/postfix-mailman.cfg

    [mailman]
    layout: debian-manual

    [paths.debian-manual]
    var_dir: /home/mailman/var
    bin_dir: /home/mailman/mailman/bin
    etc_dir: /etc/mailman3
    pid_file: /run/mailman3/master.pid

  > ``/etc/mailman3/mailman.cfg``
- .. code-block:: ini

    [postfix]
    transport_file_type: hash
    postmap_command: /usr/sbin/postmap

  > ``/etc/mailman3/postfix-mailman.cfg``
- .. code-block:: ini

    # systemd service template for mailman3 core program

    [Unit]
    Description=Mailman3 Core program
    ConditionPathExists=/etc/mailman3/mailman.cfg

    [Service]
    # Type is simple
    ExecStart=/home/mailman/mailman/bin/mailman -C /etc/mailman3/mailman.cfg start
    ExecReload=/bin/kill -HUP $MAINPID
    # The main PID receives SIGTERM and by default, SIGKILL 90s later
    KillMode=process
    PermissionsStartOnly=true
    ExecStartPre=/bin/mkdir /run/mailman3
    ExecStartPre=/bin/chown -R mailman:mailman /run/mailman3
    PIDFile=/run/mailman3/master.pid
    SyslogIdentifier=mailman3-core
    Restart=on-failure
    RestartPreventExitStatus=SIGINT SIGTERM SIGKILL
    User=mailman
    Group=mailman

    [Install]
    WantedBy=multi-user.target

  > ``/etc/systemd/system/mailman3-core.service``

As user ``mailman`` (``su - mailman``):

- ``virtualenv -p python3 /home/mailman/mailman``
- ``/home/mailman/mailman/bin/pip install mailman``
- ``/home/mailman/mailman/bin/mailman -C /etc/mailman3/mailman.cfg info``
- ``ln -sf /etc/mailman3/mailman.cfg /home/mailman/var/etc/mailman.cfg``

Back as user root:

- ``systemctl reload postfix``

6. OpenDKIM
-----------

Documentation used:

- http://www.opendkim.org/docs.html

Setup steps:

- ``aptitude install opendkim opendkim-tools``
- ``mkdir /var/spool/postfix/run``
- ``mkdir /var/spool/postfix/run/opendkim``
- ``chown opendkim:opendkim /var/spool/postfix/run/opendkim``
- ``chmod o-rx /var/spool/postfix/run/opendkim/``
- ``opendkim-genkey --directory /etc/dkimkeys --selector lists --domain lists.codespeak.net``
- ``chown opendkim /etc/dkimkeys/lists.*``
- Add the content of ``/etc/dkimkeys/lists.txt`` to DNS
- Edit ``/etc/defaults/opendkim`` and ``/lib/systemd/system/opendkim.service`` to use ``/var/spool/postfix/run/opendkim/`` instead of ``/var/run/opendkim``
- .. code-block::

    SenderHeaders Sender,From
    Domain lists.codespeak.net
    KeyFile /etc/dkimkeys/lists.private
    Selector lists

  >> ``/etc/opendkim.conf``
- .. code-block::

    smtpd_milters = unix:/run/opendkim/opendkim.sock
    non_smtpd_milters = unix:/run/opendkim/opendkim.sock

  >> ``/etc/postfix/main.cf``
- ``systemctl reload opendkim``
- ``systemctl reload postfix``

7. Postorius (Mailman Web UI)
-----------------------------

Documentation used:

- https://uwsgi.readthedocs.io/en/latest/tutorials/Django_and_nginx.html
- https://gitlab.com/mailman/postorius/tree/master/example_project
- http://docs.mailman3.org/en/latest/prodsetup.html
- ``zless /usr/share/doc/uwsgi/README.Debian.gz`` (after uwsgi installation)

Setup steps:

- ``aptitude install uwsgi-plugin-python``
- ``useradd --system --create-home --shell /bin/bash postorius``
- ``mkdir /var/www/postorius``
- ``chown postorius:www-data /var/www/postorius``
- .. code-block:: ini

    [uwsgi]
    chdir = /home/postorius/mailman_postorius
    module = mailman_postorius.wsgi:application
    venv = /home/postorius/postorius
    uid = postorius

  >> ``/etc/uwsgi/apps-available/postorius.ini``
- ``ln -s /etc/uwsgi/apps-available/postorius.ini /etc/uwsgi/apps-enabled``

As user ``mailman`` (``su - mailman``):

- ``virtualenv -p python2 /home/postorius/postorius``
- ``/home/postorius/postorius/bin/pip install postorius``
- ``mkdir /home/postorius/mailman_postorius``
- ``/home/postorius/postorius/bin/django-admin startproject mailman_postorius /home/postorius/mailman_postorius``
- .. code-block:: diff

    diff -ru a/mailman_postorius/settings.py b/mailman_postorius/settings.py
    --- a/mailman_postorius/settings.py
    +++ b/mailman_postorius/settings.py
    @@ -25,7 +25,7 @@
     # SECURITY WARNING: don't run with debug turned on in production!
    -DEBUG = True
    +DEBUG = False

    -ALLOWED_HOSTS = []
    +ALLOWED_HOSTS = ['127.0.0.1', 'localhost', 'lists.codespeak.net']


     # Application definition
    @@ -35,18 +35,27 @@
         'django.contrib.auth',
         'django.contrib.contenttypes',
         'django.contrib.sessions',
    +    'django.contrib.sites',
         'django.contrib.messages',
         'django.contrib.staticfiles',
    +    'postorius',
    +    'django_mailman3',
    +    'django_gravatar',
    +    'allauth',
    +    'allauth.account',
    +    'allauth.socialaccount',
     ]

    -MIDDLEWARE = [
    +MIDDLEWARE_CLASSES = [
         'django.middleware.security.SecurityMiddleware',
         'django.contrib.sessions.middleware.SessionMiddleware',
         'django.middleware.common.CommonMiddleware',
         'django.middleware.csrf.CsrfViewMiddleware',
    +    'django.middleware.locale.LocaleMiddleware',
         'django.contrib.auth.middleware.AuthenticationMiddleware',
    +    'django.contrib.auth.middleware.SessionAuthenticationMiddleware',
         'django.contrib.messages.middleware.MessageMiddleware',
         'django.middleware.clickjacking.XFrameOptionsMiddleware',
    +    'postorius.middleware.PostoriusMiddleware',
     ]

     ROOT_URLCONF = 'mailman_postorius.urls'
    @@ -59,9 +68,16 @@
             'OPTIONS': {
                 'context_processors': [
                     'django.template.context_processors.debug',
    +                'django.template.context_processors.i18n',
    +                'django.template.context_processors.media',
    +                'django.template.context_processors.static',
    +                'django.template.context_processors.tz',
    +                'django.template.context_processors.csrf',
                     'django.template.context_processors.request',
                     'django.contrib.auth.context_processors.auth',
                     'django.contrib.messages.context_processors.messages',
    +                'django_mailman3.context_processors.common',
    +                'postorius.context_processors.postorius',
                 ],
             },
         },
    @@ -76,7 +92,7 @@
     DATABASES = {
         'default': {
             'ENGINE': 'django.db.backends.sqlite3',
    -        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    +        'NAME': os.path.join(BASE_DIR, 'postorius.db'),
         }
     }

    @@ -118,3 +134,44 @@
     # https://docs.djangoproject.com/en/1.11/howto/static-files/

     STATIC_URL = '/static/'
    +
    +# Absolute path to the directory static files should be collected to.
    +# Don't put anything in this directory yourself; store your static files
    +# in apps' "static/" subdirectories and in STATICFILES_DIRS.
    +# Example: "/var/www/example.com/static/"
    +STATIC_ROOT = os.path.join(BASE_DIR, 'static')
    +
    +SITE_ID = 1
    +SITE_URL = 'lists.codespeak.net'
    +SITE_NAME = 'lists.codespeak.net'
    +
    +LOGIN_URL = 'account_login'
    +LOGIN_REDIRECT_URL = 'list_index'
    +LOGOUT_URL = 'account_logout'
    +
    +# Mailman API credentials
    +MAILMAN_REST_API_URL = 'http://localhost:8001'
    +MAILMAN_REST_API_USER = 'restadmin'
    +MAILMAN_REST_API_PASS = 'restpass'
    +
    +# From Address for emails sent to users
    +DEFAULT_FROM_EMAIL = 'postorius@lists.codespeak.net'
    +# From Address for emails sent to admins
    +SERVER_EMAIL = 'root@lists.codespeak.net'
    +# Compatibility with Bootstrap 3
    +from django.contrib.messages import constants as messages
    +MESSAGE_TAGS = {
    +    messages.ERROR: 'danger'
    +}
    +
    +
    +AUTHENTICATION_BACKENDS = (
    +    'django.contrib.auth.backends.ModelBackend',
    +    'allauth.account.auth_backends.AuthenticationBackend',
    +)
    +
    +# Django Allauth
    +ACCOUNT_AUTHENTICATION_METHOD = "username_email"
    +ACCOUNT_EMAIL_REQUIRED = True
    +ACCOUNT_EMAIL_VERIFICATION = "mandatory"
    +ACCOUNT_DEFAULT_HTTP_PROTOCOL = "https"
    +ACCOUNT_UNIQUE_EMAIL  = True
    +
    +
    diff -ru a/mailman_postorius/urls.py b/mailman_postorius/urls.py
    --- a/mailman_postorius/urls.py
    +++ b/mailman_postorius/urls.py
    @@ -13,9 +13,19 @@
         1. Import the include() function: from django.conf.urls import url, include
         2. Add a URL to urlpatterns:  url(r'^blog/', include('blog.urls'))
     """
    -from django.conf.urls import url
    +from django.conf.urls import include, url
     from django.contrib import admin
    +from django.core.urlresolvers import reverse_lazy
    +from django.views.generic import RedirectView

     urlpatterns = [
    +    url(r'^$', RedirectView.as_view(
    +        url=reverse_lazy('list_index'),
    +        permanent=False)),
    +    url(r'^postorius/', include('postorius.urls')),
    +    #url(r'^hyperkitty/', include('hyperkitty.urls')),
    +    url(r'', include('django_mailman3.urls')),
    +    url(r'^accounts/', include('allauth.urls')),
    +    # Django admin
         url(r'^admin/', admin.site.urls),
     ]
    diff -ru a/manage.py b/manage.py
    --- a/manage.py
    +++ b/manage.py
    @@ -1,4 +1,4 @@
    -#!/usr/bin/env python
    +#!/home/postorius/postorius/bin/python
     import os
     import sys

- Make sure your domain is included in ``ALLOWED_HOSTS`` of ``settings.py``
- ``/home/postorius/mailman_postorius/manage.py collectstatic``
- ``/home/postorius/mailman_postorius/manage.py migrate``
- ``sqlite3 /home/postorius/mailman_postorius/postorius.db``
    - ``update auth_user set is_superuser=1 where email='mail@florian-schulze.net';``
    - ``update django_site set domain='lists.codespeak.net' where id=1;``
    - ``update django_site set name='lists.codespeak.net' where id=1;``

Back as user root:

- ``systemctl reload uwsgi``

8. Nginx
--------

- ``aptitude install nginx``
- .. code-block:: nginx

    server {
        listen 80;

        server_name lists.codespeak.net;

        location /static/ {
            alias /var/www/postorius/;
        }

        location / {
            include uwsgi_params;
            uwsgi_pass unix:/run/uwsgi/app/postorius/socket;
        }
    }

  > ``/etc/nginx/sites-available/lists
- ``ln -s /etc/nginx/sites-available/lists /etc/nginx/sites-enabled/``
- ``systemctl reload nginx``

9. Let's Encrypt
----------------

Documentation used:

- https://github.com/lukas2511/dehydrated
- https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/

Setup steps:

- ``aptitude install dehydrated``
- .. code-block:: bash

    CONTACT_EMAIL="admins@lists.codespeak.net"

  > ``/etc/dehydrated/conf.d/contact_email.sh``
- .. code-block:: bash

    BASEDIR=/etc/dehydrated

  > ``/etc/dehydrated/conf.d/basedir.sh``
- .. code-block:: bash

    HOOK="/etc/dehydrated/hook.sh"

  > ``/etc/dehydrated/conf.d/hook.sh``
- .. code-block:: bash

    #!/bin/sh
    set -e
    set -u
    case "$1" in
        "deploy_cert")
            systemctl reload nginx
            ;;
        *)
            return
    esac

  > ``/etc/dehydrated/hook.sh``
- ``chmod u+x /etc/dehydrated/hook.sh``
- .. code-block:: bash

    CA="https://acme-staging.api.letsencrypt.org/directory"
    CA_TERMS="https://acme-staging.api.letsencrypt.org/terms"

  > ``/etc/dehydrated/conf.d/staging.sh``
- .. code-block:: diff

    diff --git a/nginx/sites-available/lists b/nginx/sites-available/lists
    index 3b1ebee..0297b9f 100644
    --- a/nginx/sites-available/lists
    +++ b/nginx/sites-available/lists
    @@ -3,6 +3,10 @@ server {

        server_name lists.codespeak.net;

    +   location /.well-known/acme-challenge {
    +       alias /var/lib/dehydrated/acme-challenges;
    +   }
    +
        location /static/ {
            alias /var/www/postorius/;
        }
- **lists.codespeak.net** > ``/etc/dehydrated/domains.txt``
- ``systemctl reload nginx``
- ``dehydrated -c``
- .. code-block:: diff

    diff --git a/nginx/sites-available/lists b/nginx/sites-available/lists
    index 3b1ebee..0297b9f 100644
    --- a/nginx/sites-available/lists
    +++ b/nginx/sites-available/lists
    @@ -7,6 +7,18 @@ server {
                    alias /var/lib/dehydrated/acme-challenges;
            }

    +       location / {
    +               return 302 https://$http_host$request_uri;
    +       }
    +}
    +
    +server {
    +       listen 443 ssl;
    +       server_name lists.codespeak.net;
    +
    +       ssl_certificate /var/lib/dehydrated/certs/lists.codespeak.net/fullchain.pem;
    +       ssl_certificate_key /var/lib/dehydrated/certs/lists.codespeak.net/privkey.pem;
    +
            location /static/ {
                    alias /var/www/postorius/;
            }
- ``systemctl reload nginx``
- Check certificate with browser, should be from staging server
- ``git rm dehydrated/conf.d/staging.sh``
- Now we run dehydrated again with the real server and use ``-x`` to force certificate renewal
- ``dehydrated -c -x``
- The hook should have been called this time, so we don't need to reload nginx manually
- Check certificate with browser, should be valid now
- ``openssl dhparam -out /etc/nginx/dhparam.pem 4096`` — take a long walk for this
- .. code-block:: diff

    diff --git a/nginx/nginx.conf b/nginx/nginx.conf
    index 6e57ea9..55ae279 100644
    --- a/nginx/nginx.conf
    +++ b/nginx/nginx.conf
    @@ -31,8 +31,11 @@ http {
            # SSL Settings
            ##

    +       # see https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
            ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
            ssl_prefer_server_ciphers on;
    +       ssl_ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS;
    +       ssl_dhparam dhparam.pem;

            ##
            # Logging Settings
- ``systemctl reload nginx``
- Use https://www.ssllabs.com/ssltest/analyze.html?d=lists.codespeak.net&hideResults=on&latest to check your domain
- If wanted, you can do more, see https://observatory.mozilla.org/analyze.html?host=lists.codespeak.net
- .. code-block:: bash

    #!/bin/sh
    set -e
    set -u
    /usr/bin/dehydrated -c

  > ``/etc/cron.weekly/dehydrated``
- ``chmod u+x /etc/cron.weekly/dehydrated``

10. rspamd
----------

Documentation used:

- https://rspamd.com/

Setup steps:

- ``https://rspamd.com/apt-stable/gpg.key | apt-key add -``
- .. code-block::

    deb http://rspamd.com/apt-stable/ stretch main
    deb-src http://rspamd.com/apt-stable/ stretch main

  > ``/etc/apt/sources.list.d/rspamd.list``
- ``aptitude update``
- ``aptitude install rspamd``
- .. code-block::

    bind_socket = "localhost:11333";
    enabled = false;

  > ``/etc/rspamd/local.d/worker-normal.inc``
- .. code-block:: nginx

    bind_socket = "localhost:11332";
    milter = yes; # Enable milter mode
    timeout = 120s; # Needed for Milter usually
    upstream "local" {
      default = yes; # Self-scan upstreams are always default
      self_scan = yes; # Enable self-scan
    }

  > ``/etc/rspamd/local.d/worker-proxy.inc``
- ``systemctl reload rspamd``
- .. code-block:: diff

    diff --git a/postfix/main.cf b/postfix/main.cf
    index aaf6f7e..a400d58 100644
    --- a/postfix/main.cf
    +++ b/postfix/main.cf
    @@ -47,6 +47,6 @@ owner_request_special = no
     transport_maps = hash:/home/mailman/var/data/postfix_lmtp
     local_recipient_maps = hash:/home/mailman/var/data/postfix_lmtp
     relay_domains = hash:/home/mailman/var/data/postfix_domains
    -smtpd_milters = unix:/run/opendkim/opendkim.sock
    +smtpd_milters = unix:/run/opendkim/opendkim.sock, inet:localhost:11332
     non_smtpd_milters = unix:/run/opendkim/opendkim.sock

- ``systemctl reload postfix``

11. borgbackup
--------------

Documentation used:

- https://borgbackup.readthedocs.io/en/stable/index.html

Setup steps:

- ``aptitude install borgbackup``
- ``ssh-keygen -t rsa -b 4096``
- Use ssh key on destination host according to https://borgbackup.readthedocs.io/en/stable/deployment.html#restrictions
- Create passphrase with ``python -c "import os, binascii; print binascii.hexlify(os.urandom(16))"``
- ``borg init backup@backup:full`` with passphrase that is used as ``BORG_PASSPHRASE`` in next step
- ¡Keep the passphrase in a safe place somewhere, so you can access the backup later on!
- .. code-block:: bash

    #!/bin/sh
    set -e
    set -u
    export BORG_PASSPHRASE=
    REPOSITORY=backup@backup:full

    if [ -z $BORG_PASSPHRASE ]; then
        echo Missing passphrase!
        exit 1
    fi

    borg create -v --stats \
        $REPOSITORY::'{hostname}-{now:%Y%m%d-%H%M}' \
        /etc \
        /home \
        /root \
        /var

    borg prune -v --list $REPOSITORY --prefix '{hostname}-' \
        --keep-hourly=24 --keep-daily=7 --keep-weekly=4 --keep-monthly=6

  > ``/etc/cron.hourly/backup``
- ``chmod 0700 /etc/cron.hourly/backup``

Todo
====

- Try without ``SITE_ID``
