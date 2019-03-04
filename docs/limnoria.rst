IRC Bot
=======

Bot configuration
-----------------

- Create a dedicated user

``useradd --create-home --shell /bin/bash ghbot``

- Install limnoria

``apt install limnoria``

- run the wizard as an unprivileged user

``su - ghbot``

``supybot-wizard``

- on a different location, clone the plugins repository

``git clone https://github.com/ProgVal/Supybot-plugins.git``

- symlink the GitHub plugin into limnoria plugin's directory

- most defaults are ok, things that need change

  *supybot.plugins.Github.announces.secret*
  space separated list of secrets to authenticate GitHub payloads,
  to avoid other people to send messages to our bot

  *supybot.plugins.Github.announces* autocrypt/autocrypt | freenode | #autocrypt || deltachat/deltachat-core | freenode | #deltachat
  list of GitHub projects and irc channels were to publish updates, it is
  recommended to modify the list using commands ('@Github announce add' or '@Github announce remove')

  *supybot.servers.http.port*
  port used for http

  *supybot.networks.freenode.ssl*
  set `True` to use ssl

  *supybot.protocols.ssl.verifyCertificates*
  set `True` to verify the ssl certificates


- limnoria/supybot will want to modify their config files at runtime,
  if that is not possible (permissions) the bot will run the same



Firewall
--------

- Github publishes their IP ranges at `https://api.github.com/meta` we'll
  use that to only allow access to GitHub IP Addresses to the bot.

- 2 ipsets and iptables rules calling them are required

- .. code-block:: shell

   ipset create github-webhooks4 hash:ip family inet hashsize 1024 maxelem 65536
   ipset create github-webhooks6 hash:ip family inet6 hashsize 1024 maxelem 65536
   iptables -A INPUT -m set --match-set github-webhooks4 src -p tcp -m tcp --dport 8093 -j ACCEPT
   ip6tables -A INPUT -m set --match-set github-webhooks6 src -p tcp -m tcp --dport 8093 -j ACCEPT

- the ipset has to be rutinelly updated, here is a cronjob for that

- .. code-block:: shell

   #!/bin/sh
   # this script updates github webhook ranges in an ipset
   set -e

   ipset create github-webhooks4-temp hash:ip family inet hashsize 1024 maxelem 65536
   ipset create github-webhooks6-temp hash:ip family inet6 hashsize 1024 maxelem 65536

   for ip in $(curl https://api.github.com/meta  | jq .hooks | sed 's/[^0-9\.\/]*//g' | grep -v :)
     do
          ipset add github-webhooks4-temp $ip
   done

   for ip in $(curl https://api.github.com/meta  | jq .hooks | sed 's/[^0-9\.\/]*//g' | grep :)
     do
          ipset add github-webhooks6-temp $ip
   done


   ipset swap github-webhooks4-temp github-webhooks4
   ipset destroy github-webhooks4-temp

   ipset swap github-webhooks6-temp github-webhooks6
   ipset destroy github-webhooks6-temp

*NOTE*: limnoria's default http port is 8080, here the config was changed to use the port 8093
*NOTE2*: limnoria http port is not using ssl, this is ok in this case as the repos are public
