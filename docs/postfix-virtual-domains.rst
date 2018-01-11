Virtual domains for postfix
===========================

Documentation used:

- http://www.postfix.org/VIRTUAL_README.html

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
