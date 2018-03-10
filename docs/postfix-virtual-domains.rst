Virtual domains for postfix
===========================

Documentation used:

- http://www.postfix.org/VIRTUAL_README.html
- http://www.opendkim.org/opendkim-README


Domain alias without separation of mail per domain
--------------------------------------------------

- add the new domain to ``mydestination`` list in ``/etc/postfix/main.cf``
- .. code-block:: diff

    diff --git a/postfix/main.cf b/postfix/main.cf
    index 4963822..5f319ba 100644
    --- a/postfix/main.cf
    +++ b/postfix/main.cf
    @@ -36,7 +36,7 @@ myhostname = lists.codespeak.net
     alias_maps = hash:/etc/aliases
     alias_database = hash:/etc/aliases
     myorigin = /etc/mailname
    -mydestination = $myhostname, lists.codespeak.net, localhost.codespeak.net, , localhost
    +mydestination = $myhostname, codespeak.net, lists.codespeak.net, localhost.codespeak.net, , localhost
     relayhost =
     mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
     mailbox_size_limit = 0
- we use the ``default`` selector, because it was used on ``codespeak.net`` before
- For new **DKIM** key: ``opendkim-genkey --directory /etc/dkimkeys --selector default --domain codespeak.net``
- Or restore from backup and ``chmod 0600 /etc/dkimkeys/default.*``
- ``chown opendkim /etc/dkimkeys/default.*``
- Add the content of ``/etc/dkimkeys/default.txt`` to DNS
- add TXT record for spf in DNS: ``@ IN TXT "v=spf1 ip4:78.47.150.134/32 ~all"``
- ``systemctl reload opendkim``

Virtual domains in postfix
--------------------------

- add the new domain to ``virtual_mailbox_domains`` list in ``/etc/postfix/main.cf``
- list each new mailbox in ``/etc/postfix/virtual_mailboxes`` followed by an space and a string,
  we use OK as convention for the string. In the future if the need arises we could add a check on the value of
  the second string and do something about it. That's why is important to follow the convention
  run ``postmap /etc/postfix/virtual_mailboxes`` after editing the file
- list each email alias in ``/etc/postfix/virtual_aliases``, first field is the alias followed by the destinations
  run ``postmap /etc/postfix/virtual_aliases`` after editing the file
- to allow users to retrieve the emails using IMAP4 or POP3, they need to have a password
  edit ``/etc/dovecot/users``, add each mailbox followed by an : followed by the hashed password followed by : 6 times
  to generate the password, run ``doveadm pw -s SHA512-CRYPT`` the mailbox owner can do it on its own machine and provide the hash to us :)
  don't use more modern hashes, as the server may not support them. As Stretch Debian release SHA512-CRYPT is the best hash available
