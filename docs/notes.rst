Notes
=====

Vagrant Debian base boxes
-------------------------

https://atlas.hashicorp.com/debian/

Everything is based on Debian 9 "Stretch".

Preliminary notes
-----------------

rspamd and mailman3 aren't officially packaged on Debian at current versions

Hetzner
-------

- When ordering a VM, just use "Debian 9 minimal" or any other operating
  system.  After ordering **run the rescue system Linux 64-bit** and then

- With ``installimage`` use Debian 9 minimal

Installation
============

1. Updates
----------

- ``apt update``

2. etckeeper
------------

- ``apt install etckeeper``

- .. code-block:: diff

   diff --git a/.gitignore b/.gitignore
   index 9196cf5..74b2861 100644
   --- a/.gitignore
   +++ b/.gitignore
   @@ -46,6 +46,7 @@ check_mk/logwatch.state

    # editor temp files
    *~
   +*-
    .*.sw?
    .sw?
    \#*\#

- ``> /etc/.gitignore``

- You might want to edit ``/etc/etckeeper/etckeeper.conf`` to your liking.
  The most useful settings to look into are ``AVOID_DAILY_AUTOCOMMITS`` and
  ``AVOID_COMMIT_BEFORE_INSTALL``.

3. some basics
--------------

- ``apt install unattended-upgrades``
- use ``update-alternatives --config editor`` to choose the default editor (vim)
- **lists.codespeak.net** > ``/etc/hostname``
- check ``/etc/hosts``

- ``reboot`` to properly update the hostname

4. Postfix
----------

- ``apt install postfix``
- configuration type "Internet Site"
- mail name **lists.codespeak.net**
- add TXT record for spf in DNS: ``lists IN TXT "v=spf1 ip4:78.47.150.134/32 ~all"``

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

- ``apt install virtualenv``
- ``useradd --system --create-home --shell /bin/bash mailman``
- ``usermod -a -G mailman postfix``

- .. code-block:: diff

    diff --git a/postfix/main.cf b/postfix/main.cf
    index 5e1d87e..8b5350a 100644
    --- a/postfix/main.cf
    +++ b/postfix/main.cf
    @@ -43,3 +43,7 @@ mailbox_size_limit = 0
     recipient_delimiter = +
     inet_interfaces = all
     inet_protocols = all
    +owner_request_special = no
    +transport_maps = hash:/home/mailman/var/data/postfix_lmtp
    +local_recipient_maps = hash:/home/mailman/var/data/postfix_lmtp
    +relay_domains = hash:/home/mailman/var/data/postfix_domains

  >> ``/etc/postfix/main.cf``
- ``mkdir /etc/mailman3``
- .. code-block:: ini

    [mta]
    incoming: mailman.mta.postfix.LMTP
    outgoing: mailman.mta.deliver.deliver
    lmtp_host: 127.0.0.1
    lmtp_port: 8024
    smtp_host: localhost
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
    ExecStartPre=/bin/mkdir -p /run/mailman3
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
- ``mkdir /home/mailman/var/etc``
- ``ln -sf /etc/mailman3/mailman.cfg /home/mailman/var/etc/mailman.cfg``
- ``/home/mailman/mailman/bin/mailman aliases``


Back as user root:

- ``systemctl enable mailman3-core.service``
- ``systemctl start mailman3-core.service``
- ``systemctl reload postfix``

6. Postorius (Mailman Web UI) with nginx
----------------------------------------

Documentation used:

- https://uwsgi.readthedocs.io/en/latest/tutorials/Django_and_nginx.html
- https://gitlab.com/mailman/postorius/tree/master/example_project
- http://docs.mailman3.org/en/latest/prodsetup.html
- ``zless /usr/share/doc/uwsgi/README.Debian.gz`` (after uwsgi installation)

Setup steps:

- ``apt install nginx sqlite3 uwsgi uwsgi-plugin-python3``
- ``useradd --system --create-home --shell /bin/bash postorius``
- ``mkdir /var/www/postorius``
- ``chown postorius:www-data /var/www/postorius``
- .. code-block:: ini

    [uwsgi]
    plugins = python3
    chdir = /home/postorius/mailman_postorius
    module = mailman_postorius.wsgi:application
    venv = /home/postorius/postorius
    uid = postorius

  >> ``/etc/uwsgi/apps-available/postorius.ini``
- ``ln -s /etc/uwsgi/apps-available/postorius.ini /etc/uwsgi/apps-enabled``

As user ``postorius`` (``su - postorius``):

- ``virtualenv -p python3 /home/postorius/postorius``
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

     MIDDLEWARE = [
         'django.middleware.security.SecurityMiddleware',
         'django.contrib.sessions.middleware.SessionMiddleware',
         'django.middleware.common.CommonMiddleware',
         'django.middleware.csrf.CsrfViewMiddleware',
    +    'django.middleware.locale.LocaleMiddleware',
         'django.contrib.auth.middleware.AuthenticationMiddleware',
    +    #'django.contrib.auth.middleware.SessionAuthenticationMiddleware',
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
    +STATIC_ROOT = '/var/www/postorius'
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
    +POSTORIUS_TEMPLATE_BASE_URL = 'https://lists.codespeak.net'
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
    +from django.urls import reverse_lazy
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
    - ``update django_site set domain='lists.codespeak.net' where id=1;``
    - ``update django_site set name='lists.codespeak.net' where id=1;``

Back as user root:

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

  > ``/etc/nginx/sites-available/lists``
- ``ln -s /etc/nginx/sites-available/lists /etc/nginx/sites-enabled/``
- ``systemctl stop uwsgi``
- ``systemctl start uwsgi``
- ``systemctl reload nginx``
- You shouldn't use this until after the next step which adds https!

7. Let's Encrypt
----------------

Documentation used:

- https://github.com/lukas2511/dehydrated
- https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
- http://www.postfix.org/TLS_README.html

Setup steps:

- ``apt install dehydrated``
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
- If you restore from backup, you don't need ``staging.sh``, but it's recommended for testing and setting up new domains to prevent hitting any rate limits
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
- At this point you might want to restore ``/etc/dehydrated`` from backup
- Or for a new setup: **lists.codespeak.net** > ``/etc/dehydrated/domains.txt``
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
    +       ssl_certificate /etc/dehydrated/certs/lists.codespeak.net/fullchain.pem;
    +       ssl_certificate_key /etc/dehydrated/certs/lists.codespeak.net/privkey.pem;
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
- ``openssl dhparam -out /etc/nginx/dhparam.pem 4096`` — take a long walk for this, or restore from backup
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
- .. code-block:: diff

    diff --git a/postfix/main.cf b/postfix/main.cf
    index a2e2f1d..aac0715 100644
    --- a/postfix/main.cf
    +++ b/postfix/main.cf
    @@ -22,8 +22,8 @@ readme_directory = no
     compatibility_level = 2

     # TLS parameters
    -smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
    -smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
    +smtpd_tls_cert_file=/etc/dehydrated/certs/lists.codespeak.net/fullchain.pem
    +smtpd_tls_key_file=/etc/dehydrated/certs/lists.codespeak.net/privkey.pem
     smtpd_use_tls=yes
     smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
     smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
    diff --git a/postfix/master.cf b/postfix/master.cf
    index ff58b4d..f135d8c 100644
    --- a/postfix/master.cf
    +++ b/postfix/master.cf
    @@ -25,10 +25,10 @@ smtp      inet  n       -       y       -       -       smtpd
     #  -o smtpd_recipient_restrictions=
     #  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
     #  -o milter_macro_daemon_name=ORIGINATING
    -#smtps     inet  n       -       y       -       -       smtpd
    -#  -o syslog_name=postfix/smtps
    -#  -o smtpd_tls_wrappermode=yes
    -#  -o smtpd_sasl_auth_enable=yes
    +smtps     inet  n       -       y       -       -       smtpd
    +  -o syslog_name=postfix/smtps
    +  -o smtpd_tls_wrappermode=yes
    +  -o smtpd_sasl_auth_enable=yes
     #  -o smtpd_reject_unlisted_recipient=no
     #  -o smtpd_client_restrictions=$mua_client_restrictions
     #  -o smtpd_helo_restrictions=$mua_helo_restrictions
- ``systemctl reload postfix``

8. Mailman admins
-----------------

Now that we have a secure connection, we can continue with Mailman

- Create a new user account on https://lists.codespeak.net
- You should get a email to activate it
- Now to make that account a superuser, do: ``sqlite3 /home/postorius/mailman_postorius/postorius.db``
    - ``update auth_user set is_superuser=1 where email='mail@florian-schulze.net';``
- "Lists" -> "Create New Domain" -> ``lists.codespeak.net``
- "Lists" -> "Create New List" -> ``admins``, ``lists.codespeak.net``
- Go to the new list to "Mass operations" -> "Mass subscribe" and add admin email addresses (at least yourself)
- Maybe edit "Subject prefix" in "List Identity" to ``[Admins lists.codespeak.net]``

- Subscribe root@lists.codespeak.net to the list
- Disable mail delivery for root@lists.codespeak.net
- Let cron send output to the list:
- .. code-block:: diff

    diff --git a/crontab b/crontab
    index 95edd9b..d923b2a 100644
    --- a/crontab
    +++ b/crontab
    @@ -6,6 +6,7 @@

     SHELL=/bin/sh
     PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
    +MAILTO=admins@lists.codespeak.net

     # m h dom mon dow user command
     17 *   * * *   root    cd / && run-parts --report /etc/cron.hourly

9. HyperKitty mailing list archiver
------------------------------------

Documentation used:

- https://hyperkitty.readthedocs.io/en/latest/install.html
- https://gitlab.com/mailman/hyperkitty/blob/master/example_project/qcluster.service

A prerequisit is the Sass CSS precompiler:

- ``aptitude install sassc``

First we do the Django side.
We use the ``postorius`` installation for this and change the following.
For ``MAILMAN_ARCHIVER_KEY`` you have to replace ``!!REPLACE_WITH_YOUR_KEY!!`` with a randomly generated key.
One way to do that is with ``openssl rand -base64 32``.

- .. code-block:: diff

    diff -ru a/mailman_postorius/mailman_postorius/settings.py b/mailman_postorius/mailman_postorius/settings.py
    --- a/mailman_postorius/mailman_postorius/settings.py  2017-07-05 10:14:04.271688080 +0200
    +++ b/mailman_postorius/mailman_postorius/settings.py 2018-10-08 09:09:35.046133578 +0200
    @@ -28,9 +28,14 @@
     ALLOWED_HOSTS = ['127.0.0.1', 'localhost', 'lists.codespeak.net']


    +# Archiver
    +MAILMAN_ARCHIVER_KEY = !!REPLACE_WITH_YOUR_KEY!!
    +MAILMAN_ARCHIVER_FROM = ('127.0.0.1', '::1')
    +
     # Application definition

     INSTALLED_APPS = [
    +    'compressor',
         'django.contrib.admin',
         'django.contrib.auth',
         'django.contrib.contenttypes',
    @@ -38,22 +41,25 @@
         'django.contrib.sites',
         'django.contrib.messages',
         'django.contrib.staticfiles',
    +    'haystack',
    +    'hyperkitty',
         'postorius',
    +    'django_extensions',
         'django_mailman3',
         'django_gravatar',
    +    'django_q',
         'allauth',
         'allauth.account',
         'allauth.socialaccount',
     ]
    @@ -78,6 +84,7 @@
                     'django.contrib.auth.context_processors.auth',
                     'django.contrib.messages.context_processors.messages',
                     'django_mailman3.context_processors.common',
    +                'hyperkitty.context_processors.common',
                     'postorius.context_processors.postorius',
                 ],
             },
    @@ -136,6 +143,15 @@

     STATIC_URL = '/static/'

    +# List of finder classes that know how to find static files in
    +# various locations.
    +STATICFILES_FINDERS = (
    +    'django.contrib.staticfiles.finders.FileSystemFinder',
    +    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
    +    # 'django.contrib.staticfiles.finders.DefaultStorageFinder',
    +    'compressor.finders.CompressorFinder',
    +)
    +
     # Absolute path to the directory static files should be collected to.
     # Don't put anything in this directory yourself; store your static files
     # in apps' "static/" subdirectories and in STATICFILES_DIRS.
    @@ -178,3 +197,33 @@
     ACCOUNT_DEFAULT_HTTP_PROTOCOL = "https"
     ACCOUNT_UNIQUE_EMAIL  = True

    +# Async task settings
    +
    +Q_CLUSTER = {
    +    'timeout': 300,
    +    'save_limit': 100,
    +    'orm': 'default',
    +}
    +
    +COMPRESS_PRECOMPILERS = (
    +  ('text/x-scss', 'sassc -t compressed {infile} {outfile}'),
    +  ('text/x-sass', 'sassc -t compressed {infile} {outfile}'),
    +)
    +
    +COMPRESS_ENABLED = True
    +
    +# On a production setup, setting COMPRESS_OFFLINE to True will bring a
    +# significant performance improvement, as CSS files will not need to be
    +# recompiled on each requests. It means running an additional "compress"
    +# management command after each code upgrade.
    +# http://django-compressor.readthedocs.io/en/latest/usage/#offline-compression
    +COMPRESS_OFFLINE = True
    +
    +# Full text search config
    +HAYSTACK_CONNECTIONS = {
    +    'default': {
    +        'ENGINE': 'haystack.backends.whoosh_backend.WhooshEngine',
    +        'PATH': os.path.join(BASE_DIR, "fulltext_index"),
    +    },
    +}
    +
    +# HyperKitty
    +FILTER_VHOST = False
    +HYPERKITTY_DISABLE_SINGLETON_TASKS = False
    +HYPERKITTY_TASK_LOCK_TIMEOUT = 10 * 60
    +

- .. code-block:: diff


    diff -ru a/mailman_postorius/mailman_postorius/urls.py b/mailman_postorius/mailman_postorius/urls.py
    --- a/mailman_postorius/mailman_postorius/urls.py  2017-07-05 09:49:19.208622793 +0200
    +++ b/mailman_postorius/mailman_postorius/urls.py 2018-10-08 08:46:34.126326388 +0200
    @@ -15,15 +15,18 @@
     """
     urlpatterns = [
         url(r'^$', RedirectView.as_view(
             url=reverse_lazy('list_index'),
             permanent=False)),
    +    url(r'^$', RedirectView.as_view(
    +        url=reverse_lazy('hk_root'),
    +        permanent=False)),
         url(r'^postorius/', include('postorius.urls')),
    -    #url(r'^hyperkitty/', include('hyperkitty.urls')),
    +    url(r'^hyperkitty/', include('hyperkitty.urls')),
         url(r'', include('django_mailman3.urls')),
         url(r'^accounts/', include('allauth.urls')),
         # Django admin

- ``/home/postorius/postorius/bin/pip install HyperKitty Whoosh``
- ``/home/postorius/mailman_postorius/manage.py migrate``
- ``/home/postorius/mailman_postorius/manage.py collectstatic``
- ``/home/postorius/mailman_postorius/manage.py compress``

As the ``mailman`` user we have to install the archiver plugin.

- ``/home/mailman/mailman/bin/pip install mailman-hyperkitty``

As ``root`` we need to place some config files and reload/restart the services.

Make hyperkitty URL available on localhost.

- .. code-block:: nginx

    server {
            listen 127.0.0.1;
            listen [::1];
            server_name localhost;

            location /hyperkitty {
                    include uwsgi_params;
                    uwsgi_pass unix:/run/uwsgi/app/postorius/socket;
            }
    }

  > ``/etc/nginx/sites-available/localhost``

- ``ln -s /etc/nginx/sites-available/localhost /etc/nginx/sites-enabled/``

Add archiver config to ``mailman.cfg``

- .. code-block:: ini

    [archiver.hyperkitty]
    class: mailman_hyperkitty.Archiver
    enable: yes
    configuration: /etc/mailman3/hyperkitty.cfg

  >> ``/etc/mailman3/mailman.cfg``

Add hyperkitty config. Replace the secret key with the same as above.

- .. code-block:: ini

    [general]

    # This is your HyperKitty installation, preferably on the localhost. This
    # address will be used by Mailman to forward incoming emails to HyperKitty
    # for archiving. It does not need to be publicly available, in fact it's
    # better if it is not.
    base_url: http://localhost/hyperkitty/

    # Shared API key, must be the identical to the value in HyperKitty's
    # settings.
    api_key: !!REPLACE_WITH_YOUR_KEY!!

  > ``/etc/mailman3/hyperkitty.cfg``

Add systemd service file for Django's qcluster task runner

- .. code-block:: ini

    [Unit]
    Description=HyperKitty async tasks runner
    After=network.target remote-fs.target

    [Service]
    ExecStart=/home/postorius/postorius/bin/django-admin qcluster --pythonpath /home/postorius/mailman_postorius --settings mailman_postorius.settings
    User=postorius
    Restart=always

    [Install]
    WantedBy=multi-user.target

  > ``/etc/systemd/system/qcluster.service``

- ``systemctl stop mailman3-core``

Add cron jobs for Django

- .. code-block::

    MAILTO=admins@lists.codespeak.net

    @hourly  postorius  /home/postorius/postorius/bin/django-admin runjobs hourly  --pythonpath /home/postorius/mailman_postorius --settings mailman_postorius.settings
    @daily   postorius  /home/postorius/postorius/bin/django-admin runjobs daily   --pythonpath /home/postorius/mailman_postorius --settings mailman_postorius.settings
    @weekly  postorius  /home/postorius/postorius/bin/django-admin runjobs weekly  --pythonpath /home/postorius/mailman_postorius --settings mailman_postorius.settings
    @monthly postorius  /home/postorius/postorius/bin/django-admin runjobs monthly --pythonpath /home/postorius/mailman_postorius --settings mailman_postorius.settings
    @yearly  postorius  /home/postorius/postorius/bin/django-admin runjobs yearly  --pythonpath /home/postorius/mailman_postorius --settings mailman_postorius.settings
    * * * * *  postorius  /home/postorius/postorius/bin/django-admin runjobs minutely --pythonpath /home/postorius/mailman_postorius --settings mailman_postorius.settings
    2,17,32,47 * * * * postorius  /home/postorius/postorius/bin/django-admin runjobs quarter_hourly --pythonpath /home/postorius/mailman_postorius --settings mailman_postorius.settings

  > ``/etc/cron.d/hyperkitty``

Unfortunately systemctl doesn't really stop mailman.
Use ``systemctl status mailman3-core`` to see which PID the main ``/home/mailman/mailman/bin/python3 /home/mailman/mailman/bin/master -C /etc/mailman3/mailman.cfg`` process has.
Use ``kill`` with the PID you got.
Run ``systemctl status mailman3-core`` a few times until all sub-processes are shut down.

- ``systemctl start mailman3-core``
- ``systemctl start qcluster``
- ``systemctl restart uwsgi``


10. rspamd
----------

Documentation used:

- https://rspamd.com/

Setup steps:

- ``curl https://rspamd.com/apt-stable/gpg.key | apt-key add -``
- .. code-block::

    deb http://rspamd.com/apt-stable/ stretch main
    deb-src http://rspamd.com/apt-stable/ stretch main

  > ``/etc/apt/sources.list.d/rspamd.list``
- ``apt update``
- ``apt install rspamd``
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
    +smtpd_milters = inet:localhost:11332

- ``systemctl reload postfix``

- configure rspamd to sign email headers (dkim)

- .. code-block:: nginx

   allow_envfrom_empty = true;
   allow_hdrfrom_mismatch = false;
   allow_hdrfrom_multiple = false;
   allow_username_mismatch = false;
   auth_only = true;
   path = "/etc/dkimkeys/$selector.private.$domain";
   selector = "mail";
   sign_local = true;
   sign_networks = "/etc/dkimkeys/TrustedHosts";
   symbol = "DKIM_SIGNED";
   try_fallback = true;
   use_domain = "header";
   use_esld = false;
   use_redis = false;
   key_prefix = "DKIM_KEYS";
   sign_headers = 'from:subject:date:to'
   domain {
    # Domain name is used as key
    lists.codespeak.net {
      # Private key path
      path = "/etc/dkimkeys/lists.private";
      # Selector
      selector = "lists";
    }
   }
  > ``/etc/rspamd/local.d/dkim_signing.conf``

  Generate the keys using `opendkim-genkey` from the opendkim-tools package

  ``apt install opendkim-tools``

  ``opendkim-genkey --directory /etc/dkimkeys --selector lists --domain lists.codespeak.net``

  Keys in `/etc/dkimkeys/` have to allow the `_rspamd` user to read them



11. borgbackup
--------------

Documentation used:

- https://borgbackup.readthedocs.io/en/stable/index.html

Setup steps:

- ``apt install borgbackup``
- At this point you might want to restore your ssh key from backup,
- Or create a new one: ``ssh-keygen -t rsa -b 4096``
- Use public ssh key on destination host according to https://borgbackup.readthedocs.io/en/stable/deployment.html#restrictions
- Get your passphrase ready for an existing backup,
- Or create a new passphrase with ``python -c "import os, binascii; print binascii.hexlify(os.urandom(16))"``
- ¡Keep the passphrase in a safe place somewhere, so you can access the backup later on!
- For a new backup: ``borg init backup@backup:full`` with passphrase that is used as ``BORG_PASSPHRASE`` in next step,
- Or use you existing passphrase as ``BORG_PASSPHRASE``
- .. code-block:: bash

    #!/bin/sh
    set -e
    set -u
    export BORG_PASSPHRASE=
    export BORG_LOGGING_CONF=/etc/borg-logging.ini
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

  > ``/etc/cron.hourly/zz-backup``
- It's named ``zz-backup`` to run last
- ``chmod 0700 /etc/cron.hourly/zz-backup``
- .. code-block:: ini

    [loggers]
    keys = root

    [handlers]
    keys = logfile, stderr

    [formatters]
    keys = plain, timestamped

    [logger_root]
    level = INFO
    handlers = logfile, stderr

    [handler_logfile]
    class = FileHandler
    level = INFO
    formatter = timestamped
    args = ('/var/log/borg-backup.log',)

    [handler_stderr]
    class = StreamHandler
    args = (sys.stderr,)
    level = WARN
    formatter = plain

    [formatter_plain]
    format = %(message)s
    datefmt =
    class = logging.Formatter

    [formatter_timestamped]
    format = %(asctime)s,%(msecs)03d %(levelname)-5.5s %(message)s
    datefmt = %y-%m-%d %H:%M:%S
    class = logging.Formatter

  > /etc/borg-logging.ini

- .. code-block::

    /var/log/borg-backup {
      rotate 30
      daily
      compress
      delaycompress
      missingok
      notifempty
    }

  > /etc/logrotate.d/borg-backup

12. Commit hook for etckeeper
-----------------------------

- ``apt install mailutils``
- .. code-block:: bash

    #!/bin/sh
    git log -1 --stat HEAD | mail -s "etckeeper commit" admins@lists.codespeak.net

  > /etc/.git/hooks/post-commit
- ``chmod u+x /etc/.git/hooks/post-commit``

13. dovecot
-----------

- ``apt install dovecot-imapd``
- require SSL:
- .. code-block:: diff

    diff --git a/dehydrated/hook.sh b/dehydrated/hook.sh
    index b6df571..3b0b9b7 100755
    --- a/dehydrated/hook.sh
    +++ b/dehydrated/hook.sh
    @@ -4,6 +4,7 @@ set -u
     case "$1" in
         "deploy_cert")
             systemctl reload nginx
    +        systemctl reload dovecot
             ;;
         *)
             return
    diff --git a/dovecot/conf.d/10-ssl.conf b/dovecot/conf.d/10-ssl.conf
    index ab2dc01..9276be6 100644
    --- a/dovecot/conf.d/10-ssl.conf
    +++ b/dovecot/conf.d/10-ssl.conf
    @@ -3,7 +3,7 @@
     ##

     # SSL/TLS support: yes, no, required. <doc/wiki/SSL.txt>
    -ssl = no
    +ssl = required

     # PEM encoded X.509 SSL/TLS certificate and private key. They're opened before
     # dropping root privileges, so keep the key file unreadable by anyone but
    @@ -11,6 +11,8 @@ ssl = no
     # certificate, just make sure to update the domains in dovecot-openssl.cnf
     #ssl_cert = </etc/dovecot/dovecot.pem
     #ssl_key = </etc/dovecot/private/dovecot.pem
    +ssl_cert = </etc/dehydrated/certs/lists.codespeak.net/fullchain.pem
    +ssl_key = </etc/dehydrated/certs/lists.codespeak.net/privkey.pem

     # If key file is password protected, give the password here. Alternatively
     # give it when starting dovecot with -p parameter. Since this file is often
- Enable local delivery:
- .. code-block:: diff

    diff --git a/dovecot/conf.d/10-mail.conf b/dovecot/conf.d/10-mail.conf
    index cc0d35e..e3c4de0 100644
    --- a/dovecot/conf.d/10-mail.conf
    +++ b/dovecot/conf.d/10-mail.conf
    @@ -111,7 +111,7 @@ namespace inbox {
     # Group to enable temporarily for privileged operations. Currently this is
     # used only with INBOX when either its initial creation or dotlocking fails.
     # Typically this is set to "mail" to give access to /var/mail.
    -#mail_privileged_group =
    +mail_privileged_group = mail

     # Grant access to these supplementary groups for mail processes. Typically
     # these are used to set up access to shared mailboxes. Note that it may be
    diff --git a/postfix/main.cf b/postfix/main.cf
    index aac0715..8b00b7d 100644
    --- a/postfix/main.cf
    +++ b/postfix/main.cf
    @@ -45,7 +45,7 @@ inet_interfaces = all
     inet_protocols = all
     owner_request_special = no
     transport_maps = hash:/home/mailman/var/data/postfix_lmtp
    -local_recipient_maps = hash:/home/mailman/var/data/postfix_lmtp
    +local_recipient_maps = proxy:unix:passwd.byname hash:/home/mailman/var/data/postfix_lmtp
     relay_domains = hash:/home/mailman/var/data/postfix_domains
     smtpd_milters = unix:/run/opendkim/opendkim.sock, inet:localhost:11332
     non_smtpd_milters = unix:/run/opendkim/opendkim.sock
- Enable SASL auth
- .. code-block:: diff

    diff --git a/dovecot/conf.d/10-auth.conf b/dovecot/conf.d/10-auth.conf
    index 1c59eb4..187b262 100644
    --- a/dovecot/conf.d/10-auth.conf
    +++ b/dovecot/conf.d/10-auth.conf
    @@ -97,7 +97,7 @@
     #   plain login digest-md5 cram-md5 ntlm rpa apop anonymous gssapi otp skey
     #   gss-spnego
     # NOTE: See also disable_plaintext_auth setting.
    -auth_mechanisms = plain
    +auth_mechanisms = plain login

     ##
     ## Password and user databases
    diff --git a/dovecot/conf.d/10-master.conf b/dovecot/conf.d/10-master.conf
    index e3d6260..441f95a 100644
    --- a/dovecot/conf.d/10-master.conf
    +++ b/dovecot/conf.d/10-master.conf
    @@ -93,9 +93,9 @@ service auth {
       }

       # Postfix smtp-auth
    -  #unix_listener /var/spool/postfix/private/auth {
    -  #  mode = 0666
    -  #}
    +  unix_listener /var/spool/postfix/private/auth {
    +    mode = 0666
    +  }

       # Auth process is run as this user.
       #user = $default_internal_user
    diff --git a/postfix/main.cf b/postfix/main.cf
    index 8b00b7d..4963822 100644
    --- a/postfix/main.cf
    +++ b/postfix/main.cf
    @@ -49,3 +49,8 @@ local_recipient_maps = proxy:unix:passwd.byname hash:/home/mailman/var/data/post
     relay_domains = hash:/home/mailman/var/data/postfix_domains
     smtpd_milters = unix:/run/opendkim/opendkim.sock, inet:localhost:11332
     non_smtpd_milters = unix:/run/opendkim/opendkim.sock
    +smtpd_sasl_type = dovecot
    +smtpd_sasl_auth_enable = yes
    +smtpd_recipient_restrictions = permit_sasl_authenticated permit_mynetworks reject_unauth_destination
    +smtpd_sasl_path = private/auth
- ``systemctl reload dovecot``
- ``systemctl reload postfix``

14. Add sudo
------------

- ``apt install sudo``
- Use ``visudo`` to edit ``/etc/sudoers``
- Allow GIT* environmental vars to have proper names on git commits

 .. code-block:: diff

    diff --git a/sudoers b/sudoers
    index d4cc632..77b5539 100644
    --- a/sudoers
    +++ b/sudoers
    @@ -9,6 +9,7 @@
     Defaults       env_reset
     Defaults       mail_badpass
     Defaults       secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    +Defaults       env_keep ="GIT_AUTHOR_EMAIL GIT_AUTHOR_NAME GIT_COMMITTER_EMAIL GIT_COMMITTER_NAME"

     # Host alias specification

    @@ -20,7 +21,7 @@ Defaults      secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/b
     root   ALL=(ALL:ALL) ALL

     # Allow members of group sudo to execute any command
    -%sudo  ALL=(ALL:ALL) ALL
    +%sudo  ALL=(ALL:ALL) NOPASSWD: ALL

     # See sudoers(5) for more information on "#include" directives:

15. Add sshguard
----------------

- ``apt install sshguard``

16. Add fail2ban
----------------

- ``apt install fail2ban``

- .. code-block:: diff

     diff --git a/fail2ban/jail.conf b/fail2ban/jail.conf
     index d80e3d0..3b2b08c 100644
     --- a/fail2ban/jail.conf
     +++ b/fail2ban/jail.conf
     @@ -614,7 +614,7 @@ backend  = %(syslog_backend)s

      [postfix-sasl]

     -port     = smtp,465,submission,imap3,imaps,pop3,pop3s
     +port     = smtp,465,submission,imap2,imaps,pop3,pop3s
      # You might consider monitoring /var/log/mail.warn instead if you are
      # running postfix since it would provide the same log lines at the
      # "warn" level but overall at the smaller filesize.
     diff --git a/fail2ban/jail.d/defaults-debian.conf b/fail2ban/jail.d/defaults-debian.conf
     index 9eb356c..78630c7 100644
     --- a/fail2ban/jail.d/defaults-debian.conf
     +++ b/fail2ban/jail.d/defaults-debian.conf
     @@ -1,2 +1,8 @@
      [sshd]
     +enabled = false
     +
     +[dovecot]
     +enabled = true
     +
     +[postfix-sasl]
      enabled = true
     diff --git a/fail2ban/paths-common.conf b/fail2ban/paths-common.conf
     index 9072136..d4226b6 100644
     --- a/fail2ban/paths-common.conf
     +++ b/fail2ban/paths-common.conf
     @@ -66,7 +66,7 @@ vsftpd_log = /var/log/vsftpd.log
      postfix_log = %(syslog_mail_warn)s
      postfix_backend = %(default_backend)s

     -dovecot_log = %(syslog_mail_warn)s
     +dovecot_log = %(syslog_mail)s
      dovecot_backend = %(default_backend)s

      # Seems to be set at compile time only to LOG_LOCAL0 (src/const.h) at Notice level

- sshguard is used because supports IPv6, fail2ban does not support IPv6 but supports more services


