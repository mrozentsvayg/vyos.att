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
- Configure WAN/ONT interface to use the RG MAC address 3 (three) times in the following configuration snippet (AT&T DHCP would not respond to any other MAC address). Also, please note, the ONT port is `eth3` in my example, but in most cases WAN `eth0` port will be used instead.
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

[Up]([#table-of-contents)

## IPv6 configuration

## Miscellaneous

## Credits
- [reddit.com/user/Streiw/](https://www.reddit.com/r/ATT/comments/g59rwm/bgw210700_root_exploitbypass/) - all this could not be possible without certificates
- [Sergey (devicelocksmith)](https://www.devicelocksmith.com/2018/12/eap-tls-credentials-decoder-for-nvg-and.html) - excellent WPA configuration helper
