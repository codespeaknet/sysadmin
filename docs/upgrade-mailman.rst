Upgrading Mailman
=================

Checking installed versions
---------------------------

For mailman:

.. code-block::

    # su - mailman
    $ ./mailman/bin/pip list | grep mailman

For postorius:

.. code-block::

    # su - postorius
    $ ./postorius/bin/pip list | grep -E "postorius|HyperKitty|mailman"


Changelogs
----------

Now take a look at the changelogs to see if there is anything to look out for:

https://mailman.readthedocs.io/en/latest/src/mailman/docs/NEWS.html
https://postorius.readthedocs.io/en/latest/news.html
https://hyperkitty.readthedocs.io/en/latest/news.html

Upgrading
---------

https://docs.mailman3.org/en/latest/upgrade-3.2.html#migration-steps
https://hyperkitty.readthedocs.io/en/latest/install.html#upgrading

First we disable the web interface by adding a ``return 503`` as first line to the ``location / {`` part in the nginx config:

.. code-block::

    # vi /etc/nginx/sites-available/lists
    # systemctl reload nginx

Check https://lists.codespeak.net/ to see if it worked.

Now we stop the services:

.. code-block::

    # systemctl stop qcluster

Unfortunately systemctl doesn't really stop mailman.
Use ``systemctl status mailman3-core`` to see which PID the main ``/home/mailman/mailman/bin/python3 /home/mailman/mailman/bin/master -C /etc/mailman3/mailman.cfg`` process has.
Use ``kill`` with the PID you got.
Run ``systemctl status mailman3-core`` a few times until all sub-processes are shut down.

Disable cron jobs:

.. code-block::

    # chmod 0 /etc/cron.d/hyperkitty

Upgrade mailman:

.. code-block::

    # su - mailman
    $ ./mailman/bin/pip install -U --upgrade-strategy eager mailman mailman-hyperkitty
    $ ./mailman/bin/pip freeze > mailman/requirements.txt

Upgrade postorius:

.. code-block::

    # su - postorius
    $ ./postorius/bin/pip install -U --upgrade-strategy eager postorius HyperKitty Whoosh
    $ ./postorius/bin/pip freeze > postorius/requirements.txt

Now we can start mailman3-core again, which according to the docs upgrades its own data automatically:

.. code-block::

    # systemctl start mailman3-core

Then we migrate postorius:

.. code-block::

    # su - postorius
    $ ./mailman_postorius/manage.py migrate
    $ ./mailman_postorius/manage.py collectstatic
    $ ./mailman_postorius/manage.py compress

Enable cron jobs again:

.. code-block::

    # chmod 644 /etc/cron.d/hyperkitty

Start qcluster service:

.. code-block::

    # systemctl start qcluster

Remove the 503 from the nginx config:

.. code-block::

    # vi /etc/nginx/sites-available/lists
    # systemctl reload nginx

Finally we can commit the changed requirements.txt files:

.. code-block::

    # cd /etc
    # etckeeper commit "Upgraded mailman and postorius."
