# GeoBypass
Reliable way how to avoid Geo restrictions is to use VPN. Thou this is has several challenges which I'd like to cover in this article and document what I found about them. I'll be glad for any PRs which will clarify things or correct me.

## Challenges
- [IP of VPN Server is blocked](#IP-of-VPN-Server-is-blocked).
- [Device does not allow to configure VPN](#Device-does-not-allow-to-configure-VPN).
- [All traffic is going through VPN](#All-traffic-is-going-through-VPN).

## How to?
- [Configure VPN through Ubiquity Edge Router](#Configure-VPN-through-Ubiquity-Edge-Router)

## Domains for Popular services
This section contains domains which needs to be on the allowlist in order to bypass geo check for these services.

### Disney+
- bamgrid.com
- registerdisney.go.com
- disney-plus.net
- dssott.com
- disneyplus.com

### HBO Now
- hbogo.com
- hbo.com
- hbonow.com

### Showtime
- showtime.com

## IP of VPN Server is blocked
A lot of services are using banlist of IP ranges to be protected against VPN servers. This solution is pretty effective.

My experience is, that all (I tried Azure, Amazon and Linode) IP from cloud providers are always banned. Only chance to get through it is to either use some unknown hosting or buy commercial VPN service which has a lot of servers (a lot of them will still be blocked).

Solution with commercial VPN service is usually cheaper, but chance that IP you are using there will be blocked is higher (after what you will need to switch to new VPN server, which is usually available as VPN service providers are always adding new servers into pool ...).

One thing which I noticed, I was never able to bypass Geo check L2TP VPN server. Don't know why yet, but it simply didn't work (I checked that I have correct IP visible from public internet). I was more successful with OpenVPN.

I believe that there could be better ways with IPv6, but unfortunately that one is often not supported by the services with Geo restrictions (for example Disney+).

## Device does not allow to configure VPN
Some devices (Apple TV) does not allow to configure VPN. This is problematic when you want to bypass Geo restrictions, but solvable.

The way how to do it is on the router or in general on some device between your Apple TV and your internet provider.

It's possible to configure your router in a way, that it will redirect traffic over VPN.

## All traffic is going through VPN
Another challenge is, that typically you don't want all traffic to go through VPN for several reasons (higher latency for services where it's not needed, limited bandwidth/data, visibility of different content because of different region, ...).

It is possible to configure your router in a way, that it will forward to VPN just traffic, which needs to bypass Geo restrictions.

## Configure VPN through Ubiquity Edge Router
You can easily configure OpenVPN on Ubiquity routers and route through the VPN only traffic for specific domains. Currently (Dec 2019) it's not possible to do configuration through web interface and changes through CLI are not always reflected in UI.

__Be aware__, that for mentioned way, devices on your network must have configured DNS servers in a way that your router is your DNS server. Usually it's default, but in case you reconfigured it differently, it will not work.

### How to
- [Configure OpenVPN client](#Configure-OpenVPN-client)
- [Configure NAT](#Configure-NAT)
- [Configure rules for choosing right gateway](#Configure-rules-for-choosing-right-gateway)
- [Configure domains for redirect](#Configure-domains-for-redirect)

## Configure OpenVPN client

### Preparation of configuration files
At first you need to download file with OpenVPN configuration. This file has extension __ovpn__.

In case your VPN provider provides also credentials, it's necessary to modify __ovpn__ file. You can open it in text editor and add lines:

```
auth-user-pass to auth-user-pass /config/auth/vpnauth.txt
route-nopull
```

This will force OpenVPN client too look for credentials in file on specified path. File __vpnauth.txt__ will have two lines where first line will contain username and second line password.

When you have configuration files, they need to be uploaded to Ubiquity router into new folder auth on path __/config/auth__. Folder should have permissions set to 644 (owner read+write, everybody else just read).

On Windows you can do it through WinScp or Putty and on Mac through SSH or Midnight Commander. For more details see article on [lazyadmin.nl](https://lazyadmin.nl/home-network/edgerouter-as-vpn-client/).

### Configuring VPN interfaces on the router
When all files from previous step are in place, execute following commands.

1. Enter configuration mode of router.
```
configure
```
2. Create VPN interface (replace ``<name>`` with name of your file).
```
set interfaces openvpn vtun0 config-file /config/auth/<name>.ovpn
set interfaces openvpn vtun0 description 'VPN'
```
3. Commit, save changes and reboot.
```
commit
save
exit
reboot
```

After these steps you should see in the web interface connected VPN interface.

## Configure NAT
Typically you'll have just 1 IP address from your VPN provider, but whole network behind the router which will send traffic through VPN. For this reason we need to configure NAT.


1. Enter configuration mode of router.
```
configure
```

2. Configure NAT for VPN interface.
```
set service nat rule 5050 description 'OpenVPN Clients'
set service nat rule 5050 log disable
set service nat rule 5050 outbound-interface vtun0
set service nat rule 5050 type masquerade
```

3. Commit and save changes.
```
commit
save
exit
```

## Configure rules for choosing right gateway

1. Enter configuration mode of router.
```
configure
```
2. Delete existing static tables (only in case you didn't modified them manually in past, otherwise solve conflicts yourself).
```
delete protocols static table
```

3. Create table for routing to your ISP (replace ``<IP_of_ISP_gateway>``).
```
set protocols static table 1 interface-route 0.0.0.0/0 next-hop <IP_of_ISP_gateway>
```

3. Create table for routing to VPN.
```
set protocols static table 2 interface-route 0.0.0.0/0 next-hop-interface vtun0
```

4. Configure domains which will be redirected (replace ``<domains>`` by list of domains separated by slash (``/``).
```
set service dns forwarding options ipset=/<domains>/VpnTraffic
```

5. Configure firewall to redirect every request which is going to configured domain to go through VPN. (replace ``<ethernet_port>`` with ethernet port which is connected to your local network).

```
set firewall group address-group VpnTraffic
set firewall modify VpnRule rule 5 action modify
set firewall modify VpnRule rule 5 description 'Target to VPN'
set firewall modify VpnRule rule 5 destination group address-group VpnTraffic
set firewall modify VpnRule rule 5 modify table 2
set interfaces ethernet <ethernet_port> firewall in modify VpnRule
```

6. Commit and save changes.
```
commit
save
exit
```

After this step any traffic aiming for configured domains will be redirected to VPN.

## Configure domains for redirect
At some point you'll get into situation that you want to remove or add more domains to be redirected.

You can do that by executing:
1. Enter configuration mode of router.
```
configure
```

2. Remove currently configured domains
```
delete service dns forwarding options
```

3. Configure domains which will be redirected (replace ``<domains>`` by list of domains separated by slash (``/``).
```
set service dns forwarding options ipset=/<domains>/VpnTraffic
```

4. Commit and save changes.
```
commit
save
exit
```

Changes will take effect immediately.

Note that configuration of domains which are redirected through VPN can be also done through iOS application [RouterWizzard](https://github.com/vladimir-aubrecht/RouterWizzard). It was written based on this tutorial as I was lazy to do these steps manually all the time.
