# vyos.att
AT&amp;T Fiber RG bypass with VyOS (including IPv6)

Following multiple discussions and instructions on bypassing AT&T residential gateway using PfSense or [AsusWRT-Merlin] (https://github.com/bypassrg/att), this guide is focused on configuring VyOS powered router to be plugged directly to the ONT.
Most of it is applicable to any linux based systems (and should be easier to configure in fact).
Essential part is the native IPv6 support, which was very important to keep for me.

Hardware tested and used - [Protectli Vault FW4B] (https://protectli.com/product/fw4b/), VyOS version - 1.3-rolling-202006xx
AT&T RG - BGW210-700

## Table of Contents
- [Obtaining ATT BGW210 configuration](#obtaining-att-bgw210-configuration)
- [WPA configuration](#wpa-configuration)
- [IPv6 configuration](#ipv6-configuration)
- [Miscellaneous](#miscellaneous)
- [Credits](#credits)

## Obtaining ATT BGW210 configuration
This has been thoroughly covered recently.
Credits go to [reddit.com/user/Streiw/](https://www.reddit.com/r/ATT/comments/g59rwm/bgw210700_root_exploitbypass/). 

Once the access is obtained, the following information needs to be backed up:
- mfg.dat (as described in [https://pastebin.com/SUGLTfv4](https://pastebin.com/SUGLTfv4))
- root certificates, eg:
```
tar cf /www/att/cert.tar /etc/rootcert/
```
- DUID (essential for IPv6)
```
cp /var/etc/dhcp6c_duid /www/att/
```
- [Optional] RG WAN interface MAC address (`ifconfig br2` or HTTP interface, it's also extracted by `mfg_dat_decode`, see the next step)

## WPA configuration
### Certificates
Using `mfg_dat_decode` by [Sergey (devicelocksmith)] (https://www.devicelocksmith.com/2018/12/eap-tls-credentials-decoder-for-nvg-and.html), extract and repackage credentials.
Just copy `mfg.dat` and DER certificates from `cert.tar` into the same directory and run `MacOS X`.

Notes:
- MacOS X didn't work in my 10.15, i've used Linux version instead.
- DER files from cert.tar should be in the same level as `mfg.dat` and `mfg_dat_decoder` (**not** in `./etc/rootcerts`).
Please pay attention to the output, especially errors. For instance, there should be lines:
```
Verifying certificates.. success!
Validating private key.. success!
Found valid AAA server root CA certificates:
```
in the decoder output. In case of:
```
WARNING: No valid server root Certificate Authority DER files found in xxxx
```
it'll likely not work.
### VyOS configuration
- Unpack generated `EAP-TLS_8021x_xxxxxx-xxxxxxxxxxxxxx.tar.gz` and copy 3 pem files along with `wpa_supplicant.conf` to the VyOS router. I've used `/config/auth/att/` directory.
- [Optional] Make sure the RG MAC addresss matches to what has been extracted and configured in `wpa_supplicant.conf`.
- Configure WAN/ONT interface to use the RG MAC address 3 (three) times in the following configuration snippet (AT&T DHCP would not respond to any other MAC address. In addition, the DHCP request should be coming in VLAN 0). Also, please note, the ONT port is `eth3` in my example, but in most cases WAN `eth0` port will be used instead.
```
vyos@vyos# show interfaces ethernet eth3
 hw-id xx:xx:xx:xx:xx:xx
 mac xx:xx:xx:xx:xx:xx
 vif 0 {
     address dhcp
     mac xx:xx:xx:xx:xx:xx
 }
```
- VyOS is not supporting wired 802.1x configuration at this moment. However, it's still supported by the network scripts being used in the environment, and the `/etc/network/interfaces` config file is neither generated/altered by VyOS, nor conflicting with it. Adding the following lines to the file should result in `wpa_supplicant` started and running:
```
auto eth3.0
iface eth3.0 inet dhcp
wpa-driver wired
wpa-conf /config/auth/att/wpa_supplicant.conf
```
- [**NB!**] Direct configuration would persist reboots, but would **not** survive the VyOS upgrade (only VyOS configuration is guaranteed to be saved). All the configuration outside of VyOS configuration shell should be saved and reapplied as needed. This is also applicable to direct configuration of some files in [IPv6 section](#ipv6-configuration). See the list of files, suggested to be changed: [List of directly modified files](#list-of-directly-modified-files).
- NAT configuration is standard:
```
vyos@vyos# show nat source rule 100
 outbound-interface eth3.0
 source {
     address 192.168.0.0/16
 }
 translation {
     address masquerade
 }
```
- [Tips] Occasionally, the interfaces in linux could get renumbered because of the MAC changes, eg. eth3 -> eth4 after the reboot. I recommend to "stick" the interface in udev, eg. creating the `/etc/udev/rules.d/70-persistent-net.rules` file with following content:
```
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="yy:yy:yy:yy:yy:yy", ATTR{type}=="1", KERNEL=="eth*", NAME="eth3"
```
The address should there should be the original interface address, not the one from AT&T RG.
- After committing, the interface (`eth3.0`) should up and configured with DHCP address shortly.
To debug, look into `/var/log/messages` and/or run `wpa_supplicant` manually:
```
sudo /usr/sbin/wpa_supplicant  -Dwired -ieth3.0 -c /config/auth/wpa_supplicant.conf -ddd
```

[Up](#table-of-contents)

## IPv6 configuration
### DUID
DUID configuration is essential to configure IPv6. AT&T would not respond without expected information in DUID. The easiest is to use the AT&T RG `dhcp6c_duid` file. `dhcp6c` is implicitely using, if the file resides in the right place. In VyOS the location is `/var/lib/dhcpv6/dhcp6c_duid`. The size is 29 bytes in my case.
### IA-NA/IA-PD
VyOS is hardcoding values for NA=1, PD=2 in the [template](https://github.com/vyos/vyos-1x/blob/current/data/templates/dhcp-client/ipv6.tmpl) used to generate the `dhcp6c` configuration file (eg. `/run/dhcp6c/dhcp6c.eth3.0.conf`). With these values, AT&T has never responded with IPv6 addresses.
However, NA=0, PD=1 are working flawlessly for me. The following direct change of `/usr/share/vyos/templates/dhcp-client/ipv6.tmpl` file required:
```
vyos@vyos# diff -u <(curl https://raw.githubusercontent.com/vyos/vyos-1x/current/data/templates/dhcp-client/ipv6.tmpl) /usr/share/vyos/templates/dhcp-client/ipv6.tmpl
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   964  100   964    0     0  14830      0 --:--:-- --:--:-- --:--:-- 14830
--- /dev/fd/63	2020-07-07 18:35:58.543053367 -0700
+++ /usr/share/vyos/templates/dhcp-client/ipv6.tmpl	2020-07-03 13:05:52.856329465 -0700
@@ -8,21 +8,21 @@
     information-only;
 {%  endif %}
 {%  if not dhcpv6_temporary %}
-    send ia-na 1; # non-temporary address
+    send ia-na 0; # non-temporary address
 {%  endif %}
 {%  if dhcpv6_pd_interfaces %}
-    send ia-pd 2; # prefix delegation
+    send ia-pd 1; # prefix delegation
 {%  endif %}
 };

 {%  if not dhcpv6_temporary %}
-id-assoc na 1 {
+id-assoc na 0 {
     # Identity association NA
 };
 {%  endif %}

 {%  if dhcpv6_pd_interfaces %}
-id-assoc pd 2 {
+id-assoc pd 1 {
 {%  if dhcpv6_pd_length %}
     prefix ::/{{ dhcpv6_pd_length }} infinity;
 {%  endif %}
```

### dhcp6c configuration
Having proper `dhcp6c` NA/PD configured, the following configuration is straightforward enough:
```
vyos@vyos# show interfaces ethernet eth3
 hw-id xx:xx:xx:xx:xx:xx
 mac xx:xx:xx:xx:xx:xx
 vif 0 {
     address dhcp
     address dhcpv6
     dhcpv6-options {
         prefix-delegation {
             interface eth1 {
                 sla-id 1
                 sla-len 4
             }
             length 60
         }
     }
     mac xx:xx:xx:xx:xx:xx
 }
```
... except `sla-len`. Please note, that AT&T currently allocates /60 prefixes. It leaves 4 bits for SLA. If not configured correctly, the following error would occur:
```
add_ifprefix: invalid prefix length 60 + 16 + 64
```
Note, the configuration above is configuring `eth1` port for the prefix delegation (LAN port). If your setup is different, please adjust.

### RA
Trivial (again, `eth1` for the prefix delegation):
```
vyos@vyos# show service router-advert
 interface eth1 {
     prefix ::/64 {
     }
     reachable-time 0
     retrans-timer 0
 }
```

### Debugging
To debug, `dhcp6c` could be run manually (you can specify any experimental config):
```
sudo /usr/sbin/dhcp6c -D -f -c /run/dhcp6c/dhcp6c.eth3.0.conf eth3.0
```
To verify DUID being properly obtained and used:
```
Jul/02/2020 12:28:53: get_duid: extracted an existing DUID from /var/lib/dhcpv6/dhcp6c_duid: 00:02:00:00:xx:xx............
```
Output for NA:
```
Jul/02/2020 12:28:54: dhcp6_get_options: get DHCP option client ID, len 27
Jul/02/2020 12:28:54:   DUID: 00:02:00:00:xx:xx............
Jul/02/2020 12:28:54: dhcp6_get_options: get DHCP option identity association, len 40
Jul/02/2020 12:28:54:   IA_NA: ID=1, T1=1800, T2=2880
Jul/02/2020 12:28:54: copyin_option: get DHCP option IA address, len 24
Jul/02/2020 12:28:54: copyin_option:   IA_NA address: 2001:xxx:xxxx:xxx::1 pltime=3600 vltime=3600
```
PD:
```
Jul/02/2020 12:28:54: dhcp6_get_options: get DHCP option client ID, len 27
Jul/02/2020 12:28:54:   DUID: 00:02:00:00:xx:xx............
Jul/02/2020 12:28:54: dhcp6_get_options: get DHCP option IA_PD, len 41
Jul/02/2020 12:28:54:   IA_PD: ID=1, T1=1800, T2=2880
Jul/02/2020 12:28:54: copyin_option: get DHCP option IA_PD prefix, len 25
Jul/02/2020 12:28:54: copyin_option:   IA_PD prefix: 2600:xxxx:xxxx:xxx0::/60 pltime=3600 vltime=3600
```
To verify IPv6 from the router:
```
vyos@vyos# show interfaces ethernet eth1 ipv6
 address {
     autoconf
 }
vyos@vyos# ip -6 addr show dev eth1 | egrep inet6 | awk -F ' +|/' '{print $3}' | egrep 2600:
2600:xx:xx:xx...xx
vyos@vyos# ping6 -c 2 -I 2600:xx:xx:xx...xx google.com
PING google.com(sfo03s07-in-x0e.1e100.net (2607:f8b0:4005:808::200e)) from 2600:xx:xx:xx...xx : 56 data bytes
64 bytes from sfo03s07-in-x0e.1e100.net (2607:f8b0:4005:808::200e): icmp_seq=1 ttl=119 time=4.54 ms
64 bytes from sfo03s07-in-x0e.1e100.net (2607:f8b0:4005:808::200e): icmp_seq=2 ttl=119 time=4.48 ms

--- google.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 3ms
rtt min/avg/max/mdev = 4.479/4.507/4.535/0.028 ms
```

[Up](#table-of-contents)

## Miscellaneous
### Firewall
Please do not forget that your router would start to be exposed to the public internet. Please, make sure your firewall is configured as needed and sshd is either not exposed at all, or is using public keys only, or the default password has been changed to very long generated one.

### Tuning/optimizations
This is out of the scope of this guide, but in some cases additional configuration is needed to achieve the RG reference performance.
Example - Protectli FW4B was not able to achieve more than 500-600 Mbit using `eth3` interface for WAN.
The bottleneck happened because it was only using 1 core (out of 4) and by default the `eth2/eth3` ethernet interfaces was only using 1 queue.

Ethernet I210 is supporting 4 combined queues, to permanently (surviving reboots) fix, add `ethtool -L eth3 combined 4` to `/config/scripts/vyos-postconfig-bootup.script`. (`ethtool -G eth3 tx 4096 rx 4096` wouldn't hurt as well).

### List of directly modified files
Besides the VyOS configuration, it's useful to backup:
- `/config/auth/att` - certificates and wpa configuration
- `/etc/network/interfaces` - wired wpa_supplicant
- `/etc/udev/rules.d/70-persistent-net.rules` - to stick interface names
- `/usr/share/vyos/templates/dhcp-client/ipv6.tmpl` - NA/PD values for IPv6
- `/config/scripts/vyos-postconfig-bootup.script` - ethtool etc.

### AT&T public subnet
You can get public /29 subnet for $15/mo. In case you find it appealing, it's only sufficient to configure the translation address in NAT (pick any of 6 available addresses for different rules or use all of them as a range):
```
vyos@vyos# show nat source rule 100
 outbound-interface eth3.0
 source {
     address 192.168.0.0/16
 }
 translation {
     address <YOUR_PUBLIC_IP>
 }
```

## Credits
- [reddit.com/user/Streiw/](https://www.reddit.com/r/ATT/comments/g59rwm/bgw210700_root_exploitbypass/) - all of it could not be possible without the certificates.
- [Sergey (devicelocksmith)](https://www.devicelocksmith.com/2018/12/eap-tls-credentials-decoder-for-nvg-and.html) - excellent decoder and WPA configuration helper.
- [https://github.com/bypassrg/att](https://github.com/bypassrg/att) - for inspiration to clearly document platform specific configuration and share it with community.

[Up](#table-of-contents)
