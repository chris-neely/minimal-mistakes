---
title: "Configuring dynamic DNS updates for CloudFlare"
tags: 
  - DNS
---

One of my favorite dynamic DNS services is [DNS-O-Matic](http://www.dnsomatic.com/). DNS-O-Matic is a free service that will announce your dynamic IP changes to multiple dynamic DNS providers with a single update. DNS-O-Matic supports a large number of dynamic dns clients, [this page](https://dnsomatic.com/wiki/software) shows a few tested clients.

For my specific use case I configured [ddclient](https://sourceforge.net/p/ddclient/wiki/Home/) running on my Ubuntu AWS instance to update DNS-O-Matic which in return updates my websites A record at my DNS host.

In order to do this you will need to have your websites DNS service hosted at a site that supports IP updates from DNS-O-Matic. I happen to use [CloudFlare](http://cloudflare.com/) which supports DNS-O-Matic dynamic IP updates. You can find a list of DNS-O-Matic supported services in their [documentation](https://dnsomatic.com/wiki/supportedservices).


**Here is a quick rundown of how I configured these services...**

1. Register for an account at DNS-O-Matic
2. Configure your DNS Hosting service at the DNS-O-Matic website. Here is an example for [CloudFlare](https://support.cloudflare.com/hc/en-us/articles/206142407-Using-DNS-O-Matic-dynamic-DNS-updates-with-CloudFlare-):

   ```
   Username: CLOUDFLARE ACCOUNT EMAIL
   API Token: CLOUDFLARE API KEY
   Hostname: DNS HOSTNAME
   Domain: YOUR DOMAIN NAME
   ```

3. Install ddclient on your server:

   ```
   sudo apt-get install ddclient
   ```

4. Edit /etc/ddclient.conf to update your DNS-O-Matic account:

   ```
   use=web, web=myip.dnsomatic.com
   server=updates.dnsomatic.com
   protocol=dyndns2
   login=dnsomatic_username
   password='dnsomatic_password'
   all.dnsomatic.com
   ```

5. Start the ddclient service and configure to run on boot:

   ```
   sudo update-rc.d ddclient defaults
   sudo update-rc.d ddclient enable
   sudo /usr/sbin/service/ddclient start
   ```

6. Confirm the WAN IP of your instance has been pushed to DNS-O-Matic and CloudFlare
