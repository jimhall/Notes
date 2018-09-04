# Test
This is a test of the emergency blah blah

Here is a second modificaiton of the readme file

And here is the third

And here is a way to see a diff

Wondering if I remember how to do this

```bash
root@dnszone:~# svccfg -s dns/server:default
svc:/network/dns/server:default> setprop options/ip_interfaces = "IPv4"
svc:/network/dns/server:default> quit
root@dnszone:~# svcadm refresh network/dns/server:default
root@dnszone:~# svcadm enable network/dns/server:default
root@dnszone:~# svccfg -s dns/server:default listprop options/ip_interfaces
options/ip_interfaces astring     IPv4
root@dnszone:~# useradd -c "Trusted DNS administrator user" -s /usr/bin/pfbash -A solaris.smf.manage.bind dnsadmin
root@dnszone:~# auths dnsadmin
solaris.admin.wusb.read,solaris.mail.mailq,solaris.network.autoconf.read,solaris.smf.manage.bind
root@dnszone:~# svccfg -s dns/server:default
svc:/network/dns/server:default> setprop start/user = dsnadmin
svc:/network/dns/server:default> setprop start/group = dns
svc:/network/dns/server:default> exit
root@dnszone:~#  /usr/sbin/groupadd -g 1002 dns
root@dnszone:~# cat /etc/group
```
