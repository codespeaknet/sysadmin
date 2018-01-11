Administration guide
====================

Adding a user
-------------

- ``useradd --create-home --shell /bin/bash [username]``

Adding a user to a group
------------------------

- ``usermod -a -G [groupname] [username]``

Adding public ssh key for a user
--------------------------------

- ``mkdir /home/[username]/.ssh``
- ``chmod 0700 /home/[username]/.ssh``
- ``vi /home/[username]/.ssh/authorized_keys`` to add the public key
- ``chmod 0600 /home/[username]/.ssh/authorized_keys``
- ``chown -R [username]:[usergroup] /home/[username]/.ssh`` (*usergroup* is usually the same as *username*)

Give sudo access
----------------

- ``usermod -a -G sudo [username]``

Fixup commit message after bad ``etckeeper commit``
---------------------------------------------------

- ``git commit --amend``
