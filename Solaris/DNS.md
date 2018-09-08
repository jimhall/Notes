# DNS Notes

## Date last touched: 

## Proper software installation

Since an "exact-install" was done with pkg:/group/system/solaris-small-server DNS bind packages are not installed. How to determine:

```bash
jhall@s114:~$ pkg search pkg:/service/network/dns/bind
INDEX       ACTION VALUE                                               PACKAGE
group       depend service/network/dns/bind                            pkg:/group/system/solaris-large-server@11.4-11.4.0.0.1.10.0
incorporate depend service/network/dns/bind@9.10.6.1.0-11.4.0.0.1.10.0 pkg:/consolidation/userland/userland-incorporation@11.4-11.4.0.0.1.10.0
incorporate depend service/network/dns/bind@9.10.6.1.0-11.4.0.0.1.11.0 pkg:/consolidation/userland/userland-incorporation@11.4-11.4.0.0.1.11.0
incorporate depend service/network/dns/bind@9.10.6.1.0-11.4.0.0.1.12.0 pkg:/consolidation/userland/userland-incorporation@11.4-11.4.0.0.1.12.0
incorporate depend service/network/dns/bind@9.10.6.1.0-11.4.0.0.1.13.0 pkg:/consolidation/userland/userland-incorporation@11.4-11.4.0.0.1.13.0
incorporate depend service/network/dns/bind@9.10.6.1.0-11.4.0.0.1.14.0 pkg:/consolidation/userland/userland-incorporation@11.4-11.4.0.0.1.14.0
require     depend service/network/dns/bind@9.6.1.2-0.133              pkg:/SUNWbind@9.6.1.2-0.133
pkg.fmri    set    solaris/service/network/dns/bind                    pkg:/service/network/dns/bind@9.10.6.1.0-11.4.0.0.1.10.0
```
Here is a way to check what is in the bind package contents:

```bash
jhall@s114:~$ pkg contents -r pkg:/service/network/dns/bind
PATH
lib/svc/manifest/network/dns/server.xml
lib/svc/method/dns-server
usr/bin/named-rrchecker
usr/sbin/ddns-confgen
usr/sbin/dnssec-checkds
usr/sbin/dnssec-coverage
usr/sbin/dnssec-dsfromkey
...
usr/sbin/rndc-confgen
usr/sbin/tsig-keygen
```

Here is the pkg dry run:

```bash
jhall@s114:~$ pfexec pkg install -nv pkg:/service/network/dns/bind
Re-authentication by jhall is required to use profile:
        Software Installation
(Use ^C to cancel)
Password: 
           Packages to install:         1
            Services to change:         1
     Estimated space available:  14.58 GB
Estimated space to be consumed: 169.40 MB
       Create boot environment:        No
Create backup boot environment:        No
          Rebuild boot archive:        No

Changed packages:
solaris
  service/network/dns/bind
    None -> 9.10.6.1.0-11.4.0.0.1.10.0

Services:
  restart_fmri:
    svc:/system/manifest-import:default

Planning linked: 0/2 done; 1 working: zone:thor
Linked image 'zone:thor' output:
|      Estimated space available:  14.58 GB
| Estimated space to be consumed: 164.91 MB
|           Rebuild boot archive:        No
`
Planning linked: 1/2 done; 1 working: zone:odin
Linked image 'zone:odin' output:
|      Estimated space available:  14.58 GB
| Estimated space to be consumed: 164.91 MB
|           Rebuild boot archive:        No
`
Planning linked: 2/2 done

```
Here is the real installation:

```bash
jhall@s114:~$ pfexec pkg install pkg:/service/network/dns/bind
           Packages to install:  1
            Services to change:  1
       Create boot environment: No
Create backup boot environment: No

Planning linked: 0/2 done; 1 working: zone:thor
Linked image 'zone:thor' output:
`
Planning linked: 1/2 done; 1 working: zone:odin
Linked image 'zone:odin' output:
`
Planning linked: 2/2 done
DOWNLOAD                                PKGS         FILES    XFER (MB)   SPEED
Completed                                1/1         29/29      0.8/0.8  432k/s

Downloading linked: 0/2 done; 1 working: zone:thor
Downloading linked: 1/2 done; 1 working: zone:odin
Downloading linked: 2/2 done
PHASE                                          ITEMS
Installing new actions                         59/59
Updating package state database                 Done 
Updating package cache                           0/0 
Updating image state                            Done 
Creating fast lookup database                   Done 
Executing linked: 0/2 done; 1 working: zone:thor
Executing linked: 1/2 done; 1 working: zone:odin
Executing linked: 2/2 done
Updating package cache                           1/1 
jhall@s114:~$ 
```

## Resources

11.4 Chapter 3 Managing DNS Server and Client Services (https://docs.oracle.com/cd/E37838_01/html/E61011/dnsadmin-1.html#scrolltoc)


