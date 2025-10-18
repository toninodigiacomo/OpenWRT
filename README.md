# OpenWrt Project
The OpenWrt Project is a Linux operating system targeting embedded devices. Instead of trying to create a single, static firmware, OpenWrt provides a fully writable filesystem with package management. This frees us from the application selection and configuration provided by the vendor and allows us to customize the device through the use of packages to suit any application.

## 
Installation source :
link : https://firmware-selector.openwrt.org
## Changing fixed IP address
```
vi /etc/config/network
```
- Press insert to enter a new value.
- Press Esc to stop editing.
- Press ```:wq!```to save the modifications.

```
config interface 'lan'
        option device 'br-lan'
        option proto 'static'
        option ipaddr '192.168.1.2'
        option netmask '255.255.255.0'
        option ip6assign '60'
        option gateway '192.168.1.1'
        list dns '192.168.1.1'
        option broadcast '192.168.1.255'
```

## Securing OpenWrt
By default, OpenWrt has no password set for the default root user, and whilst on a local network that might not always be critical, it’s best to set a password and take a few steps to keep things secure.

### About HTTPS on OpenWrt
The main advantage of HTTPS is a standardized protocol for securing HTTP connection. To access an HTTPS page is just typing https://openwrt.lan/ instead of http://openwrt.lan/. It's simple. Just make sure that luci-ssl and its dependencies are installed.
There are some disadvantages, though. Here are some of them.
1. External libraries makes a bloated installation.
 - On systems with just 4 MB of flash, it's not possible to enable HTTPS for LuCI web interface. Why? Because with TLS libraries integrated, the resulting image doesn't fit 4 MB of flash.
2. Browser warning on non-properly signed certificate.
 - Well, this is a good browser feature. Unless the self signed root CA has been imported to the browser, this warning creeps you out! Why bother with commercial CA when our need is just securing our own router management interface for our own use?
 - Of course, you we just buy a properly signed certificate for our own openwrt.lan domain and ip address to get rid of the annoying browser warning. We can also just import the self-signed root CA used for certificate creation to your browser certificate store. 

### How to get rid of LuCI HTTPS certificate warnings
With these instructions, we can generate our own self-signed certificate, which your browser will accept as valid. 
One new headache was that, browsers usually only look at one key part of a self-signed certificate, the CN (common name). However, starting with Chrome version 58, it not only looks at the CN (common name) in the certificate, but also at the SAN (subject alt name or DNS name), which makes generating a certificate more complicated than before. We may have even had a certificate we made ourself, that worked until recently, stop working when Chrome 58 was released and most likely automatically updated and installed.

So, to get rid of the annoying “Warning, this is an insecure site, do you want to proceed?” warning messages, and other similar messages from other browsers, we will proceed with the following.

I know it looks long, but it's easy and goes fast. Should take about 10 minutes tops. 

#### Create & Install
1. Connect via SSH
  Install the ```openssl-util``` and LuCI ```uhttpd``` packages. This is required to generate a new certificate in the way we want it to be, and to be able to easily tell LuCI how to use it.
```
opkg update && opkg install openssl-util luci-app-uhttpd
```
2. Create ```/etc/ssl/myconfig.conf``` with the following content:
```
nano /etc/ssl/myconfig.conf
```
<details>
<summary>myconfig.conf</summary>myconfig.conf
```
[req]
distinguished_name  = req_distinguished_name
x509_extensions     = v3_req
prompt              = no
string_mask         = utf8only
     
[req_distinguished_name]
C                   = US
ST                  = VA
L                   = SomeCity
O                   = OpenWrt
OU                  = Home Router
CN                  = luci.openwrt

[v3_req]
keyUsage            = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage    = serverAuth
subjectAltName      = @alt_names
basicConstraints    = CA:true

[alt_names]
DNS.1               = luci.openwrt
IP.1                = 192.168.1.1

```
</details>

3. We can edit the values for C (country), ST (state), L (location), O (organization), OU (organization unit) to whatever we want.
 - It's extremely important the values for ***CN*** and ***DNS.1*** match, and also that ***IP.1*** has the correct private IP address for the device.
> [!CAUTION]
 We might have a different IP, or might access it via a hostname; the hostname should go in both ***CN*** and ***DNS.1*** fields. The correct private IP address should go into ***IP.1***.



