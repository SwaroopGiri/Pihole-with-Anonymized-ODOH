# How To Pihole-with-DNSCrypt-Anonymized-ODOH on Raspberry Pi 4
 
***
**Important Information**

This tutorial assumes that you have a Raspberry Pi setup with Rasberry Pi OS and Pi-hole cofigured on it.
This tutorial is an alternative to unbound and both can't be used simultaneously. If you want to use anonymoity and encryption capabilities of DNSCrypt, follow this tutorial. If you like recursive self contained DNS resolution, you should stick to unbound.

***
**Additional Information**

Oblivious DNS (ODNS) is a new design of the DNS ecosystem that allows current DNS servers to remain unchanged and increases privacy for data in motion and at rest. In the ODNS system, both the client is modified with a local resolver, and there is a new authoritative name server for .odns.

**dnscrypt-proxy** is a flexible DNS proxy. It runs on your computer or router, and can locally block unwanted content, reveal where your devices are silently sending data to, make applications feel faster by caching DNS responses, and improve security and confidentiality by communicating to upstream DNS servers over secure channels.

**Anonymized DNS** is a lightweight alternative to Tor and SOCKS proxies, dedicated to DNS traffic. They hide the client IP address to DNS resolvers, providing anonymity in addition to confidentiality and integrity.

DNS Anonymization is only compatible with servers supporting the DNSCrypt protocol.

***  

### DNSCrypt-proxy Installation

Enter the following commands to do a base DNSCrypt installation. Check the latest release <a href="https://github.com/DNSCrypt/dnscrypt-proxy/releases">here</a> and modify the wget and tar commands as needed to use the latest binary. Raspberry Pi 4 supports Linux-ARM repo, so make sure to copy the correct link among various release links.

```
cd /opt

sudo wget https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.1.0/dnscrypt-proxy-linux_arm-2.1.0.tar.gz

sudo tar -xf dnscrypt-proxy-linux_arm-2.1.0.tar.gz
sudo mv linux-arm dnscrypt-proxy && cd dnscrypt-proxy
sudo cp example-dnscrypt-proxy.toml dnscrypt-proxy.toml
sudo nano dnscrypt-proxy.toml
```
Under Global settings add one or more servers. You can find list of available servers in dnscrypt [repo](https://github.com/dnscrypt/dnscrypt-resolvers). For example, you can search for DNS servers that block ads, support doh (DNS over HTTPS), view locations, etc. V3 is the most recently updated list, hence, we will be using V3 ODOH [server list](https://github.com/DNSCrypt/dnscrypt-resolvers/blob/master/v3/odoh-servers.md) and pairing servers with [relay list](https://github.com/DNSCrypt/dnscrypt-resolvers/blob/master/v3/odoh-relays.md) to anonymize the queries. You can also check out the public DNSCrypt server list and pick one or more that fits your requirements. 

**Note**

ODOH servers can only be paired with ODOH Relays. Attempting to pair DoH servers with ODoH Relays or vice versa won't work.

```
server_names = ['odoh-cloudflare' , 'odohrelay-koki-ams']
```
Under List of Local addresses change the port number to something you like, above 1024. We'll be using 5350 in this example. Port 53 (standard for DNS) will be reserved for Pihole and hence, we must use a different custom port number for DNSCrypt.

```
listen_addresses = ['127.0.0.1:5350', '[::1]:5350']
```

Change the following:
```
require_dnssec = true   
require_nofilter = false
cache = false   (we will use the Pi-Hole cache)
```
Scroll down to the bottom of the TOML file. For **server_name** add the same server name you used above. For the 'via' servers, review the relay list <a href="https://github.com/DNSCrypt/dnscrypt-resolvers/blob/master/v3/odoh-relays.md">here</a>, and pick a couple that suite your needs. It is usually recommended to use relays nearest to your location for quick resolutiom but you can use servers in a different country or have other unique requirements. 

```
routes = [
     { server_name='odoh-cloudflare', via=['odohrelay-crypto-sx', 'odohrelay-surf'] },
 ]
 routes = [
     { server_name='odohrelay-koki-ams', via=['odohrelay-koki-ams', 'odohrelay-koki-bcn'] },
 ]
```

Save the configuration file and exit nano.
```
CTRL + X then Y and Enter
```

Now we need to start the service and test it to make sure it's working before we configure Pi-Hole to use it.
```
sudo ./dnscrypt-proxy -service install
sudo ./dnscrypt-proxy
sudo ./dnscrypt-proxy -service start
sudo systemctl status dnscrypt-proxy
```

The **sudo ./dnscrypt-proxy** command will provide detailed startup information and return any errors it encounters.  **sudo systemctl status dnscrypt-proxy** does the same for DNScrypt when it's started as a service. Both should have the same output, as shown below. If the Anonymized setting is properly configured those relay servers will be shown in the DNSCrypt output.

To run a quick test that DNSCrypt can perform name resolution type:

```
./dnscrypt-proxy -resolve www.google.com
```

### Configuring Pi-Hole to use DNSCrypt as resolver

Login to Pi-hole web interface, Goto settings --> DNS --> Select Custom 1 (IPv4) and Type the DNSCrypt-proxy listening IP 127.0.0.1#5350 below it.

Uncheck everything else in Upstream DNS Servers section.

Make sure "Never forward reverse lookups for private IP ranges" and "Never forward non-FQDNs" is checked in Advanced DNS section. Uncheck use DNSSEC option since DNSCrypt does that for us.

Only check "Listen on all interfaces, permit all origins" in 'Interface listening behavior' section if you're planning to setup a VPN server. For all other connections in local network, only selecting "Listen on all interfaces" should suffice.

Click save.

Lastly, Reboot Pi `sudo reboot`


Our Pi-Hole will now send all DNS requests to ODoH servers over an encrypted tunnel.

We can test this to check our work. Start with https://www.dnsleaktest.com/ --> it will tell us right away. You may see more than one DNS server listed and as long as your ISP isn't shown, you were successful!

Happy Adblocking :)
