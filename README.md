# OpenWrt Project
The OpenWrt Project is a Linux operating system targeting embedded devices. Instead of trying to create a single, static firmware, OpenWrt provides a fully writable filesystem with package management. This frees us from the application selection and configuration provided by the vendor and allows us to customize the device through the use of packages to suit any application.

## 
Installation source :
link : https://firmware-selector.openwrt.org
## Changing fixed IP address
```
vi /etc/config/network
```
- Press insert to enter then new value.
- Press Esc to stop editing.
- Press ```:wq!```to save the modifications.

## Securing OpenWrt
By default, OpenWrt has no password set for the default root user, and whilst on a local network that might not always be critical, it’s best to set a password and take a few steps to keep things secure.

### About HTTPS on OpenWrt
The main advantage of HTTPS is a standardized protocol for securing HTTP connection. To access an HTTPS page is just typing https://openwrt.lan/ instead of http://openwrt.lan/. It's simple. Just make sure that luci-ssl and its dependencies are installed.
There are some disadvantages, though. Here are some of them.
1. External libraries makes a bloated installation.
  On systems with just 4 MB of flash, it's not possible to enable HTTPS for LuCI web interface. Why? Because with TLS libraries integrated, the resulting image doesn't fit 4 MB of flash.
2. Browser warning on non-properly signed certificate.
  Well, this is a good browser feature. Unless the self signed root CA has been imported to the browser, this warning creeps you out! Why bother with commercial CA when our need is just securing our own router management interface for our own use?
  Of course, you we just buy a properly signed certificate for our own openwrt.lan domain and ip address to get rid of the annoying browser warning. We can also just import the self-signed root CA used for certificate creation to your browser certificate store. 

### How to get rid of LuCI HTTPS certificate warnings
