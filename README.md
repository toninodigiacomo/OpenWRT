# OpenWrt Project
The OpenWrt Project is a Linux operating system targeting embedded devices. Instead of trying to create a single, static firmware, OpenWrt provides a fully writable filesystem with package management. This frees us from the application selection and configuration provided by the vendor and allows us to customize the device through the use of packages to suit any application.

## 
Installation source : https://firmware-selector.openwrt.org
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
> [!IMPORTANT]
> Line with ```option ipaddr '192.168.1.2'```is the important one!
 
## Installing Software
The default OpenWrt installation is extremely bare, so we need to install some basic packages from the web interface or at the command line with the OpenWrt opkg package management tool:
```
opkg update && opkg install nano luci-app-ttyd bash python3 python3-pip
```

## Securing OpenWrt
By default, OpenWrt has no password set for the default root user, and whilst on a local network that might not always be critical, it’s best to set a password and take a few steps to keep things secure.

### About HTTPS on OpenWrt
The main advantage of HTTPS is a standardized protocol for securing HTTP connection. To access an HTTPS page is just typing https://openwrt.lan/ instead of http://openwrt.lan/. It's simple. Just make sure that luci-ssl and its dependencies are installed.
There are some disadvantages, though. Here are some of them.
1. External libraries makes a bloated installation.
 - On systems with just 4 MB of flash, it's not possible to enable HTTPS for LuCI web interface. Why? Because with TLS libraries integrated, the resulting image doesn't fit 4 MB of flash.
2. Browser warning on non-properly signed certificate.
 - Well, this is a good browser feature. Unless the self signed root CA has been imported to the browser, this warning creeps us out! Why bother with commercial CA when our need is just securing our own router management interface for our own use?
 - Of course, we can just buy a properly signed certificate for our own openwrt.lan domain and ip address to get rid of the annoying browser warning. We can also just import the self-signed root CA used for certificate creation to browser certificate store. 

### How to get rid of LuCI HTTPS certificate warnings
With these instructions, we can generate our own self-signed certificate, which browser will accept as valid. 
One new headache was that, browsers usually only look at one key part of a self-signed certificate, the CN (common name). However, starting with Chrome version 58, it not only looks at the CN (common name) in the certificate, but also at the SAN (subject alt name or DNS name), which makes generating a certificate more complicated than before. We may have even had a certificate we made ourself, that worked until recently, stop working when Chrome 58 was released and most likely automatically updated and installed.

So, to get rid of the annoying “Warning, this is an insecure site, do you want to proceed?” warning messages, and other similar messages from other browsers, we will proceed with the following.

I know it looks long, but it's easy and goes fast. Should take about 10 minutes tops. 

### Create & Install
1. Connect via SSH
  Install the ```openssl-util``` and LuCI ```uhttpd``` packages. This is required to generate a new certificate in the way we want it to be, and to be able to easily tell LuCI how to use it.
```
opkg update && opkg install openssl-util luci-app-uhttpd
```
2. Create ```/etc/ssl/myconfig.conf``` with the following content:
```
nano /etc/ssl/myconfig.conf
```

#### myconfig.conf
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

3. We can edit the values for C (country), ST (state), L (location), O (organization), OU (organization unit) to whatever we want.
 - It's extremely important the values for ***CN*** and ***DNS.1*** match, and also that ***IP.1*** has the correct private IP address for the device.
> [!CAUTION]
 We might have a different IP, or might access it via a hostname; the hostname should go in both ***CN*** and ***DNS.1*** fields. The correct private IP address should go into ***IP.1***.

4. Save the file and navigate to /etc/ssl ```cd /etc/ssl```
   
5. Then issue the following command:
```
openssl req -x509 -nodes -days 397 -newkey rsa:2048 -keyout mycert.key -out mycert.crt -config myconfig.conf
```
This will create two files, ```mycert.key``` and ```mycert.crt```
Alternatively we can create ECDSA certificate (to speedup key exchange phase) with the following command:
```
openssl req -x509 -nodes -days 397 -newkey ec:<(openssl ecparam -name prime256v1) -keyout mycert.key -out mycert.crt -config myconfig.conf
```
> [!NOTE]
> In the commands above, the validity of the certificate was set to 13 months (397 days, the “-days” option), so the process would need to be repeated when the period lapses. Some (all) browsers do not accept longer validity:
> https://github.com/cabforum/servercert/blob/90a98dc7c1131eaab01af411968aa7330d315b9b/docs/BR.md?plain=1#L175

6. In ***LuCI***, go to ***Services → uHTTPd***
 - In the field for ***HTTPS Certificate***, select the file ```/etc/ssl/mycert.crt```
 - In the field for ***HTTPS Private Key***, select the file ```/etc/ssl/mycert.key```
 - Hit save and apply.

7. Restart ***uHTTPd***
```
/etc/init.d/uhttpd restart
```

## Resize storage on OpenWrt Raspberry Pi
1. To resize storage on Raspberry Pi, internet access is required to download additional software.
2. **ssh** in the openwrt device ```ssh root@192.168.1.2```
3. Install the required packages
```
opkg update && opkg install cfdisk resize2fs tune2fs lsblk parted
```
4. Resize the partition:
 - ```cfdisk /dev/mmcblk0```
 *Now Resize the ***/dev/mmcblk0p2*** partition (enter desired space). After this write the changes and quit.*
 - Reboot
5. Remount root as RO (if fails, reboot and remount as ro again)
```
mount -o remount,ro /
```
6. Remove reserved GDT blocks
```
tune2fs -O^resize_inode /dev/mmcblk0p2
fsck.ext4 /dev/mmcblk0p2 
```
(This might probably fail, doesn't seem to affect anything!)
 - Reboot
7. Resize the **f2fs** filesystem
```
resize2fs /dev/mmcblk0p2
```
8. Check new root partition size with ```lsblk```

## OpenWrt as Docker container host
1. Install ***dockerd***
```
opkg update && opkg install dockerd
```
> [!NOTE]
> This daemon provides the Docker Engine API and manages Docker objects such as images, containers, networks, and volumes.
2. Install ***docker***
```
opkg install docker
```
> [!NOTE]
> This client is command line based.
3. For a LuCI web client install ***luci-app-dockerman***
```
opkg install luci-app-dockerman
```
> [!NOTE]
> This package will also install dockerd and docker-compose as dependencies. It can work with dockerd on local and remote hosts. The default folder for docker in dockerman is **/opt/docker/** so mount the storage at **/opt** or change the folder in **Docker > Overview > Docker Root Dir** then restart the dockerd service.

## AdGuard Home
AdGuard Home (AGH) is a free and open source network-wide advertising and trackers blocking DNS server. It operates as a DNS server that re-routes tracking domains to a “black hole”, thus preventing devices from connecting to those servers. It is based on software used with public AdGuard DNS servers.
In addition, AdGuard Home also offers DNS encryption features such as DNS over TLS (DoT) and DNS over HTTPS (DoH) built-in without any additional packages needed. 

#### DNS latency/performance
For the best performance and lowest latency on DNS requests, AGH should be the primary DNS resolver in the DNS chain. If we currently have dnsmasq or unbound installed, we should move these services to an alternative port and have AGH use DNS port 53 with upstream DNS resolvers of the choice configured.

This wiki recommends keeping dnsmasq/unbound as the local/PTR resolver for Reverse DNS.

The rationale for this is due to resolvers like dnsmasq forking each DNS request when AGH is set as an upstream, this will have an impact on DNS latency which is can be viewed in the AGH dashboard.

We will also not benefit from being able to see the DNS requests made by each client if AGH is not the primary DNS resolver as all traffic will appear from the router.

> [!IMPORTANT]
> The install script in the setup section will move dnsmasq to port 54 and set it for AGH to use as local PTR / reverse DNS lookups.

#### Flash/storage space requirements
The compiled AdGuardHome binary has grown since the 0.107.0 release. For many routers this will be quite a significant amount of storage taken up in the overlay filesystem. In addition, features like statistics and query logging will also require further storage space when being written to the working directory. 
For routers with less flash space, it is highly recommended to use USB or an external storage path to avoid filling up the overlay filesystem. If we have low flash space, we may want to use the custom installation method and have all of the AdGuard Home installation stored outside of the flash storage. Alternatively we can also perform an exroot configuration.

Currently (May 2022 edge build 108) a full install to the /opt folder requires about **100mb** of space.
 - *(70mb) 35mb x2 for the AGH binary and again for when it backups and upgrades. (that's in the agh-backup folder)*
 - *20mb for filters*
 - *2mb - 90 days of statistics*
 - *53mb - 7 days of query logs*
We can tweak logging to keep things smaller if required.

#### Query/statistics logging
One of the main benefits of AGH is the detailed query and statistics data provided, however for many routers having long retention periods for this data can cause issues (see flash/storage space requirements). 

If we are using the default tmpfs storage, we should set a relatively short retention period or disable logging altogether. If we want to have longer retention periods for query/statistics data, consider moving the storage directory to outside the routers flash space.

### Installation
Since 21.02, there is a official AdGuard Home **package** which can be installed through opkg.

Required dependencies (ca-bundle) are automatically resolved and installed when using the official package.
```
opkg update && opkg install adguardhome
```

The official OpenWrt package uses the following paths and directories by default:
 - *The AdGuardHome application will be installed to ***/usr/bin/AdGuardHome***.*
 - *The main ***adguardhome.yaml*** configuration file is stored at ***/etc/adguardhome.yaml***.*
 - *The default working directory is ***/var/adguardhome*** (By default /var is a symlink to /tmp).*
 - *The working directory can be configured in ***/etc/config/adguardhome****
 - *An ***init.d*** script is provided at ***/etc/init.d/adguardhome***.*
    
The default configured working directory will mean query logs and statistics will be lost on a reboot. To avoid this we should configure a persistent storage path such as /opt or /mnt with external storage and update the working directory accordingly.

To have AdGuard Home automatically start on boot and to start the service:
```
service adguardhome enable
service adguardhome start
```

### Setup
After installing the opkg package, run the following commands through **SSH** to prepare for making AGH the primary DNS resolver. These instructions assume you are using dnsmasq. This will demote dnsmasq to an internal DNS resolver only.

The ports chosen are either well known alternate ports or reasonable compromises. Check with https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers for reserved ports.
```
# Get the first IPv4 and IPv6 Address of router and store them in following variables for use during the script.
NET_ADDR=$(/sbin/ip -o -4 addr list br-lan | awk 'NR==1{ split($4, ip_addr, "/"); print ip_addr[1]; exit }')
NET_ADDR6=$(/sbin/ip -o -6 addr list br-lan scope global | awk '$4 ~ /^fd|^fc/ { split($4, ip_addr, "/"); print ip_addr[1]; exit }')
echo "Router IPv4 : ""${NET_ADDR}"
echo "Router IPv6 : ""${NET_ADDR6}"
 
# 1. Move dnsmasq to port 54.
# 2. Set local domain to "lan".
# 3. Add local '/lan/' to make sure all queries *.lan are resolved in dnsmasq;
# 4. Add expandhosts '1' to make sure non-expanded hosts are expanded to ".lan";
# 5. Disable dnsmasq cache size as it will only provide PTR/rDNS info, making sure queries are always up to date (even if a device internal IP change after a DHCP lease renew).
# 6. Disable reading /tmp/resolv.conf.d/resolv.conf.auto file (which are your ISP nameservers by default), you don't want to leak any queries to your ISP.
# 7. Delete all forwarding servers from dnsmasq config.
uci set dhcp.@dnsmasq[0].port="54"
uci set dhcp.@dnsmasq[0].domain="lan"
uci set dhcp.@dnsmasq[0].local="/lan/"
uci set dhcp.@dnsmasq[0].expandhosts="1"
uci set dhcp.@dnsmasq[0].cachesize="0"
uci set dhcp.@dnsmasq[0].noresolv="1"
uci -q del dhcp.@dnsmasq[0].server
 
# Delete existing config ready to install new options.
uci -q del dhcp.lan.dhcp_option
uci -q del dhcp.lan.dns
 
# DHCP option 3: Specifies the gateway the DHCP server should send to DHCP clients.
uci add_list dhcp.lan.dhcp_option='3,'"${NET_ADDR}"
 
# DHCP option 6: Specifies the DNS server the DHCP server should send to DHCP clients.
uci add_list dhcp.lan.dhcp_option='6,'"${NET_ADDR}" 
 
# DHCP option 15: Specifies the domain suffix the DHCP server should send to DHCP clients.
uci add_list dhcp.lan.dhcp_option='15,'"lan"
 
# Set IPv6 Announced DNS
uci add_list dhcp.lan.dns="$NET_ADDR6"
 
uci commit dhcp
service dnsmasq restart
service odhcpd restart
exit 0
```





## Adding a USB Drive and Creating Samba Shares






## Ressources and links
- https://openwrt.org
- https://www.makerspace-online.com/openwrt-on-a-pi
- https://thepihut.com/blogs/raspberry-pi-tutorials/installing-openwrt-on-raspberry-pi-5
- https://paul-mackinnon.medium.com/openwrt-raspberry-pi-docker-vlan-project-9cb1db10684c
