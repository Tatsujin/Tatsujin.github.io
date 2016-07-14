---
layout: post
title: "HOWTO: Secure IRC client over SSL with nginx, weechat, bitlbee, and glowingbear"
description: "Setup a secured, fully automated IRC client web portal using nginx, certbot/letsencrypt, weechat, bitlbee, and Glowing Bear"
category: howto
tags: [howto, dns, cloudflare, ddns, debian, nginx, letsencrypt, certbot, nginx, websockets, weechat, relay, bitlbee, glowingbear]
---
{% include JB/setup %}

In today's post, I'll go over how to setup a fully secured HTML5 IRC client web
portal that can be accessed in any web browser, phone, or tablet.

## Why?

Recently, I started working with an organization that for once has strict
network requirements, namely restricting direct egress except for HTTP/S.
I already had a VM running weechat with a secured relay, it didn't take much
additional work to proxy it through a secured webserver with websockets enabled
and to setup an auto-renewing certificate through letsencrypt.

## What You'll need

* A free DNS provider of some kind (I recommend CloudFlare)
* A virtual machine or server running a common, modern Linux server distribution (examples show Debian Jessie)
* nginx, a popular alternative webserver to Apache webserver
* certbot, the EFF's client for Let's Encrypt, a free and open certificate authority
* weechat, a popular extensible IRC client and modern alternative to irssi
* bitlbee, a specially programmed IRCd designed to function as a gateway to various chat networks, such as Facebook Messenger
* Glowing Bear, an HTML5 web frontend for weechat's relay functionality

## Repositories and Packages

(Now would be a good time to take a snapshot if doing this on a VM)

Append the following to your /etc/apt/sources.list file:

```
#weechat - required for weechat-devel*
deb http://weechat.org/debian jessie main

#bitlbee - required for bitlbee, bitlbee-common
deb http://code.bitlbee.org/debian/devel/jessie/amd64/ ./

#bitlbee-additional - required for bitlbee-facebook
deb http://download.opensuse.org/repositories/home:/jgeboski/Debian_8.0 ./
```

Add keys to apt keyring, and install some packages

```
# apt-get install -y apt-transport-https
# apt-key adv --keyserver pool.sks-keyservers.net --recv-keys 11E9DE8848F2B65222AA75B8D1820DB22A11534E
# wget -O- https://jgeboski.github.io/obs.key | sudo apt-key add -
# wget -O- https://code.bitlbee.org/debian/release.key | sudo apt-key add -
# apt-get update
# apt-get install -y bitlbee-common bitlbee-facebook ca-certificates nginx-full screen weechat-devel weechat-devel-curses weechat-devel-plugins
# systemctl enable nginx
# systemctl start nginx
```

## DNS setup

Since we'll be using HTTPS a lot here, it is a good idea to setup DNS so your
SSL certificate is tied to a domain name that actually resolves. I use GKG.net
as a domain registrar, and CloudFlare's free tier for DNS, because it's free and
it's a more redundant, secure, and available solution for a minor project than
anything I would ever want to setup, configure, and maintain. Simply register
an account, get your domain verified, point your domain registar over, and setup
A/AAAA records for your VM/Server's hostname and domain name. Easy!

## Firewall check

Depending on your VM provider, you may have to either configure incoming/outgoing
connections at the hypervisor level, or locally on the VM. If so, ensure the
following ports are open inbound:

22 TCP (SSH)
80 TCP (HTTP)
443 TCP (HTTPS)
ICMP (all)

This is pretty important since bitlbee is technically an IRCd and opening those
to the outside world is a Bad Idea(tm). Weechat's relay also listens on a
non-standard port, but like bitlbee it will only need to listen locally.
Another important variable, depending on your VM/IaaS/ISP provider, IPv6 may be
available, so make sure their connections are also managed in the same way.
My provider doesn't have external connection control by default, but it *does*
support IPv6, so I just use iptables/ip6tables directly on the VM (firewalld is
also a great frontend to both). Here are my ip[6]tables config files for reference:

```
# cat /etc/iptables.up.rules
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -d 127.0.0.0/8 ! -i lo -j REJECT --reject-with icmp-port-unreachable
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7
-A INPUT -j REJECT --reject-with icmp-port-unreachable
-A FORWARD -j REJECT --reject-with icmp-port-unreachable
-A OUTPUT -j ACCEPT
COMMIT

# cat /etc/ip6tables.up.rules
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -d ::1/64 ! -i lo -j REJECT --reject-with icmp6-port-unreachable
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p icmpv6 -m icmpv6 --icmpv6-type 8 -j ACCEPT
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7
-A INPUT -j REJECT --reject-with icmp6-port-unreachable
-A FORWARD -j REJECT --reject-with icmp6-port-unreachable
-A OUTPUT -j ACCEPT
COMMIT
```

## Let's Encrypt, certbot, and nginx

certbot is an ACME (Automated Certificate Management Environment) protocol
compliant client for Let's Encrypt, the open source Certificate Authority.
What does this mean? It means that you can automate requesting and renewing
certificates, which is useful since Let's Encrypt only has an issuance period of
90 days. Unfortunately, certbot currently only has limited alpha support for
automated configuration of nginx, but it can still be used to automatically
renew the cert file that nginx (and later on weechat relay) reads. Let's see how
we can properly generate a certificate request (CSR) and private key, submit the
CSR, and securely store the issued certificate and private key.

In order to use automated nginx support, we need to build certbot from source
and use it. Luckily, this is not terribly hard. We have to clone the project,
setup the environment, and then we can use the certbot command. Use the commands
below as a reference. If you already have nginx or another webserver installed
and pre-configured, you can actually install certbot from Debian Jessies's
backports repo with command 'sudo apt-get install certbot -t jessie-backports'
and save yourself some trouble, since the nginx plugin included with the source
code doesn't work well if nginx is already configured.

```
# apt-get install -y git
# FUN NOTE: one of the commands below compiles some python code, and my 1 GB VM
# did not like this. I was able to quickly enable swap space just for that command
# dd if=/dev/zero of=/mnt/1GiB.swap bs=1024 count=1048576
# mkswap /mnt/1GiB.swap
# swapon /mnt/1GiB.swap
# exit
$ cd ~
$ git clone https://github.com/certbot/certbot
$ cd certbot
$./letsencrypt-auto-source/letsencrypt-auto --os-packages-only
# NOTE: Above command will invoke sudo to install required packages to build certbot
$ ./tools/venv.sh
# NOTE: Above command installs pip and local python modules to your account and builds/installs certbot
$ sudo -i
# NOTE: At this point you can cleanup your temporary swap drive
# swapoff /mnt/1GiB.swap
# rm -f /mnt/1GiB.swap
# cd /home/$NONPRIVILERGEDUSER/certbot
# source ./venv/bin/activate
# NOTE: Above command allows you to run certbot command and will give you a prompt like below
(venv)
```

Once you have the (venv) prompt, you can use the certbot command. If you plan to
use it non-interactively, I would recommend that you use a screen/tmux session
before invoking the activate command.

Generate an automatically signed certificate request, save the certificate, and
automatically configure a blank nginx install to use it:

`# certbot --nginx -n --agree-tos --email=EMAIL -w /PATH/TO/WEBROOT -d domain.com -d www.domain.com -d alias.domain.com`

Generate an automatically signed certificate request, and save the certificate
(no automatic webserver configuration, requires domain is already setup):

`# certbot certonly --webroot -n --agree-tos --email=EMAIL -w /PATH/TO/WEBROOT -d domain.com -d www.domain.com -d alias.domain.com`

If you run into errors, remove the non-interactive mode and associated flags
(-n --agree-tos --email=EMAIL), and check /var/log/letsencrypt.

If everything was successful, your necessary cert files will be at
/etc/letsencrypt/live/domain.com/

Some interesting things I noticed about the auto-configuration option for nginx,
it appears to setup individual SSL vhosts with specific SSL options and reference
/etc/letsencrypt/options-ssl-nginx.conf. I'm posting both below for reference if
manually configuring nginx yourself, like me!

```
# SSL Vhost snippet
ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
include /etc/letsencrypt/options-ssl-nginx.conf;
ssl_trusted_certificate /etc/letsencrypt/live/domain.com/chain.pem;
ssl_stapling on;
ssl_stapling_verify on;

# cat /etc/letsencrypt/options-ssl-nginx.conf
ssl_session_cache shared:SSL:1m;
ssl_session_timeout 1440m;

ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;

# Using list of ciphers from "Bulletproof SSL and TLS"
ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES128-SHA ECDHE-ECDSA-AES256-SHA ECDHE-ECDSA-AES128-SHA256 ECDHE-ECDSA-AES256-SHA384 ECDHE-RSA-AES128-GCM-SHA256 ECDHE-RSA-AES256-GCM-SHA384 ECDHE-RSA-AES128-SHA ECDHE-RSA-AES128-SHA256 ECDHE-RSA-AES256-SHA384 DHE-RSA-AES128-GCM-SHA256 DHE-RSA-AES256-GCM-SHA384 DHE-RSA-AES128-SHA DHE-RSA-AES256-SHA DHE-RSA-AES128-SHA256 DHE-RSA-AES256-SHA256 EDH-RSA-DES-CBC3-SHA";
```

If it helps you, here is the server portion of my SSL VHost, which also includes
some stuff we need for weechat later.

```
#Weechat SSL relay

# Set connection header based on upgrade header
map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name domain.com www.domain.com alias.domain.com;
        root <WEBROOT_GOES_HERE>;
        index index.html index.htm

        ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
        ssl_trusted_certificate /etc/letsencrypt/live/domain.com/chain.pem;
        ssl_stapling on;
        ssl_stapling_verify on;
      }

      location /weechat {
              access_log /var/log/nginx/weechat_relay.access.log;
              error_log /var/log/nginx/weechat_relay.error.log notice;
              expires epoch;
              proxy_ignore_client_abort on;
              proxy_buffering off;
              proxy_request_buffering off;
              proxy_cache off;
              proxy_pass https://127.0.0.1:9001;
              proxy_http_version 1.1;
              proxy_set_header Upgrade $http_upgrade;
              proxy_set_header Connection $connection_upgrade;
              proxy_read_timeout 4h;
      }
}
```

Note that the SSL options in the options-ssl-nginx.conf file I put in the
main nginx.conf file, which is useful if you don't have automatic nginx
configuration support.

Regarding the weechat related stuff, the important part is that the proxy is
enabled on localhost:9001, and that the header manipulation set to upgrade,
which is required for WebSockets, which is what the weechat relay uses to talk
to the webserver when making a proxied connection.

## weechat and bitlbee

weechat is a text-based IRC client similar to irssi with some impressive plugins.
It also has an intuitive command line configuration tool within the application
that doesn't require direct edits to the config files.

bitlbee is a specially programmed IRC daemon (IRCd) used to function as a gateway
to chat networks such as Google Talk/Hangouts, and Facebook Messenger.

All of bitlbee's configuration can be done within your IRC client, but it is
advised to use systemctl to enable and start the service on boot. At this point,
you should also copy your letsencrypt SSL to weechat

```
$ mkdir -p ~/.weechat/ssl
$ touch ~/.weechat/ssl/relay.pem
$ chmod 0600 ~/.weechat/ssl/relay.pem
$ sudo -i
# systemctl enable bitlbee
# systemctl start bitlbee
# cat /etc/letsencrypt/live/domain.com/fullchain.pem /etc/letsencrypt/live/domain.com/privkey.pem > /home/USERNAME/.weechat/ssl/relay.pem
```

bitlbee will run locally. We can run weechat and configure it along with
bitlbee. It is recommend you run weechat in a screen or tmux session as your
nonprivileged user (e.g. $ screen -S weechat weechat).

The following directives to weechat should get everything configured to optimize
automation:

```
/fifo enable
/set irc.color.nick_prefixes "q:lightred;a:lightcyan;o:lightgreen;h:lightmagenta;v:yellow;*:lightblue"
/set irc.server_default.autoconnect on
/set irc.server_default.autorejoin on
/set irc.server_default.autorejoin_delay 5
/set irc.server_default.nicks "nick1,nick2,nickN"
/set irc.server_default.ssl_verify off
/set irc.server_default.username "SYSTEM USERNAME"
/server add bitlbee localhost
/server add OTHER_IRC hostname.irc.com/PORT -ssl
#NOTE: IPv4
/set relay.network.bind_address = "127.0.0.1"
# NOTE: IPv6
/set relay.network.bind_address = "::ffff:127.0.0.1"
/set relay.network.ipv6 on
/set relay.network.password "PASSWORD_FOR_RELAY"
/set relay.port.ipv4.ipv6.ssl.weechat = 9001
# NOTE: IPv4
/relay add ipv4.ssl.weechat 9001
# NOTE: IPv6
/relay add ipv4.ipv6.ssl.weechat 9001
/save
```

Now that weechat is configured, we can setup bitlbee. Note that for many chat
networks, they either require that you use a app-specific password (Facebook),
or they can generate a special link code you can access to grant that application
access (Google).

https://www.facebook.com/settings?tab=security&section=per_app_passwords&view

To begin using bitlbee, use Alt+Right_Arrow to switch over to the channel marked
'&bitlbee'. This is bitlbee's control channel that you can use to issue commands.
We will first create an identity, and then add accounts (chat networks)

```
register BITLBEE_ACCOUNT_PASSWORD
account add jabber EMAIL@gmail.com
account gtalk set oauth on
acc gtalk on
# NOTE: Follow instructions to authenticate via oauth
acc add facebook USERNAME APP_PASSWORD
acc facebook on
```

In order to automatically login to your bitlbee identity, you can add the following
to your bitlbee server setting in weechat

```
/set irc.server.bitlbee.command "/msg &bitlbee identify BITLBEE_ACCOUT_PASSWORD"
/save
```

At this point, we should have our DNS, Webserver, Webserver SSL, IRC Client,
and IRC Relay with SSL as well. At this point, the only missing piece of the
puzzle is the relay frontend.

## Glowing Bear

Glowing Bear is an IRC client web frontend configured specifically for weechat.

It can be found on the android app store.

https://play.google.com/store/apps/details?id=com.glowing_bear&hl=en

There is a standard HTML5 web client available at https://www.glowing-bear.org
that works with modern browsers. Chrome is recommended though since it works
with Chrome's Push Notifications API for both desktop and mobile. If connecting
via a browser, most OSes offer a way to save the bookmarked page as a taskbar or
home screen item.

To connect, simply enter the hostname of your server or domain, the password
for the IRC relay you defined in the weechat configuration, and check the boxes
in this order: top/bottom/middle

At this point, you should be ready to go. If you run into issues, check the
troubleshooting section at the bottom

## Automation

Setting all this up is all well and good, but we still have a couple problems
if our server ever reboots or if your SSL cert expires

* On a reboot, weechat won't automatically run for your nonprivileged user
* If the letsencrypt certificate expires in 30 days, it has to be renewed manually via command line and sent over to weechat, and weechat has to be reloaded

We can tackle these one at a time

In newer OSes, the startup script file, /etc/rc.local has been removed due
to systemd doing away with runlevels. We can stil easily create a startup script
in systemd for the nonprivileged user. Create a service target in
/usr/lib/systemd/system/weechat_USERNAME.service

```
[Unit]
Description=Run screen session with weechat started for user USERNAME on boot

[Service]
Type=oneshot
ExecStart=/bin/bash -c "screen -dmS weechat weechat"
User=USERNAME

[Install]
WantedBy=multi-user.target
```

What does this do? It creates a service of type oneshot that executes a screen
command. Oneshot means that it will wait for the command in ExecStart to complete,
but the way the command is run it should finish immediately. the -dm option in
screen means to run the command and immediately detach, useful for startup scripts!
If you're using tmux, you can use the equivalent command:

`tmux new-session -d -s weechat 'weechat'`

Once the file is created, reload systemctl itself and enable the service

```
# systemctl daemon-reload
# systemctl enable weechat_USERNAME
```

Next, we need a script that can be called periodically to check if the certificate
can be renewed, and automatically reloads nginx and weechat if needed. This can
easily be done using a root cron job:

```
# crontab -l | grep certbot
27 4,16 * * * certbot renew --quiet --renew-hook "systemctl reload nginx; cat /etc/letsencrypt/live/domain.com/fullchain.pem /etc/letsencrypt/live/domain.com/privkey.pem > /home/USERNAME/.weechat/ssl/relay.pem; printf '%b' '*/upgrade\n*/relay sslcertkey\n' > /home/USERNAME/.weechat/weechat_fifo_*"
```

What's going on here? at an arbitrary minute past the hour twice a day
(4:23 AM, 4:23 PM), we have certbot renew with no output. The important thing is
that the --renew-hook flag is used, meaning it will run shell commands if the
certificate actually renews. We run a command to reload nginx, a command to
rewrite the newly renewed certificate and key to weechat's SSL relay file, and
we issue commands directly to weechat to reload (and process any pending upgrades)
and reload its relay certificate. The weechat_fifo_PID file is a FIFO pipe that
can accept commands from the system into weechat directly.

Let's Encrypt states that it is 'safe' to check for renewal status twice a day
at an arbitrary hour and minute.

NOTE: If using the compiled version, you may have to author a script that
changes directory to the certbot directory and invokes the venv command before
running certbot itself.

## Troubleshooting

If you are running into issues connecting to Glowing Bear after setup, you can
usually check in the following places:

nginx
/var/log/nginx/error.log
/var/log/nginx/weechat_relay.error_log

weechat
/home/USERNAME/.weechat/weechat.log

certbot
/var/log/letsencrypt/letsencrypt.log

## References

CloudFlare - DNS Documentation and Help
https://support.cloudflare.com/hc/en-us/categories/200276237

Let's Encrypt
https://letsencrypt.org/

certbot - Install instructions
https://certbot.eff.org/

certbot - Building from source
https://certbot.eff.org/docs/contributing.html#hacking

weechat - User Guide and Documentation
https://weechat.org/files/doc/stable/weechat_user.en.html

bitlbee - User Guide
https://www.bitlbee.org/user-guide.html

bitlbee - Wiki (custom connection reference)
https://wiki.bitlbee.org/

bitlbee - Facebook MQTT repository (Debian/Ubuntu only)
https://jgeboski.github.io/

Glowing Bear - github page
https://github.com/glowing-bear/glowing-bear

Man Pages (for further reading)
iptables
ip6tables
firewall-cmd
bitlbee
screen
tmux
certbot
nginx
