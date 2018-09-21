# DNS Notes

## Date last touched: _Fri Sep 21 14:01:34 EDT 2018_

# Overview

This is a cookbook to install and configure DNS bind on a Solaris 11.4 system that leverages some of the security enhancements available over time in Solaris 11 -> 11.4. All this work was performed with VirtualBox.

Some key points to the install and configuration:

* The base Solaris install base is pkg:/group/system/solaris-small-server
* It leverages Solaris zones to create multiple virtualized environemnts and to confirm network connectivity between the DNS client and servers
* Solaris Rights management is used  
	* Profiles are created for DNS Start/Stop and DNS Configuration Management
	* Role accounts are created for each profile is assigned to the appropriate role
	* Solaris Audit is configured to log only when the profile is being used. Configuration file changes are audited using the Solaris Audit pfedit(1M) command.
* This is a simple DNS setup: 192.168.1.0/24 private network and norsestuff.com (a bogus domainname) is used.

## Proper software installation

Since an "exact-install" was done with pkg:/group/system/solaris-small-server DNS bind packages are not installed. How to determine:

```bash
backdoor@s114:~$ pkg search pkg:/service/network/dns/bind
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
backdoor@s114:~$ pkg contents -r pkg:/service/network/dns/bind
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
backdoor@s114:~$ pfexec pkg install -nv pkg:/service/network/dns/bind
Re-authentication by backdoor is required to use profile:
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
Here is the real installation (the -r switch is to do the install in the solaris branded non-global zones):

```bash
backdoor@s114:~$ pfexec pkg install -r pkg:/service/network/dns/bind
           Packages to install:  1
            Services to change:  1
       Create boot environment: No
Create backup boot environment: No

Planning linked: 0/2 done; 1 working: zone:thor
Linked image 'zone:thor' output:
| Packages to install: 1
|  Services to change: 1
`
Planning linked: 1/2 done; 1 working: zone:odin
Linked image 'zone:odin' output:
| Packages to install: 1
|  Services to change: 1
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
backdoor@s114:~$ 
```

## Add Role Accounts for DNS management [1]

Zone odin will be the DNS primary and zone thor will be the secondary. Since these zones will eventually be the LDAP servers I am going to create local roles for these services. There is a  a slight modification from the documented procedure and a dedicated role account is created to start/stop services rather than give the authorization directly to an end user. We then assign the role to end users (eg. backdoor).

The DNS domain name will be norsestuff.com (a nonsense domain name) on a private network 192.168.1.0/24.

1. First create the Rights Profiles: One for DNS config file management and one for DNS start/stop. Note that auditing is applied in each profile to keep the amount of additional auditing to a minimum. The DNS Coniguration Management profile uses the solaris.admin.edit authorization so that diff style output is captured in the audit trail for each of the DNS configuration files in the profile. Text fragments for the profiles are created off the root home directory called dnswork and then imported using the profiles(1) command -f option/switch:

```bash
root@odin:~/dnswork# more *.pro*
::::::::::::::
DNS-Configuration-Management.profile
::::::::::::::
set name="DNS Configuration Management"
set desc="Allow audited DNS configuration file changes"
set auths=solaris.admin.edit/etc/named.conf,solaris.admin.edit/var/named/master/master.norsestuff.com,solaris.admin.edit/var/named/master/master.localhost,solaris.admin.edit/v
ar/named/master/localhost.rev,solaris.admin.edit/var/named/master/192.168.1.rev
set always_audit=cusa
set never_audit=no
::::::::::::::
DNS-Service-Management.profile
::::::::::::::
set name="DNS Service Management"
set desc="Ability to start and stop the DNS service"
set auths=solaris.smf.manage.bind
set always_audit=cusa
set never_audit=no

root@odin:~/dnswork# profiles -p "DNS Service Management" -f DNS-Service-Management.profile
root@odin:~/dnswork# profiles -p "DNS Configuration Management" -f DNS-Configuration-Management.profile
```
#### Confirmation that the work has completed as expected.

```bash
root@odin:~/dnswork# profiles -p "DNS Service Management" export
set name="DNS Service Management"
set desc="Ability to start and stop the DNS service"
set auths=solaris.smf.manage.bind
set always_audit=cusa
set never_audit=no
root@odin:~/dnswork# profiles -p "DNS Configuration Management" export
set name="DNS Configuration Management"
set desc="Allow audited DNS configuration file changes"
set auths=solaris.admin.edit/etc/named.conf,solaris.admin.edit/var/named/master/master.norsestuff.com,solaris.admin.edit/var/named/master/master.localhost,solaris.admin.edit/var/named/master/localhost.rev,solaris.admin.edit/var/named/master/192.168.1.rev
set always_audit=cusa
set never_audit=no
```

2. Now create the two roles. This could optionally be consolidated into one role, and the two Rights Profiles above could be directly assinged to a user. These steps are a twist on "How to Run the DNS Service as an Alternative User" [2]. 

#### First create a group:
```bash
/usr/sbin/groupadd -g 1000 dns
```

#### Then create the two role accounts. Note the use of -K roleauth=user which allows a user to authenticate with their own password (rather than the default shared password for Solaris roles). 

```bash
roleadd -c "DNS Administrator Start Stop Role" -d localhost:/export/home/dnsadmin -u 1000 -g dns -m -k /etc/skel -K roleauth=user -P +"DNS Service Management" -s /usr/bin/pfbash dnsadmin

roleadd -c "DNS Configuration File Role" -d localhost:/export/home/dnsconf -u 1001 -g dns -m -k /etc/skel -K roleauth=user -P +"DNS Configuration Management" -s /usr/bin/pfbash dnsconf
```
#### Then lock the role account's password since the account allows for authentication via the user's password (see -K roleauth=user).
```bash
passwd -N dnsadmin
passwd -N dnsconf
```

#### Now add the role to the user so the account can su(1M) and gain access to the role account.

```bash
usermod -R +dnsadmin,dnsconf backdoor
```
#### Validate work regarding role assiginment.

```bash
backdoor@odin:~$ roles
dnsadmin,dnsconf,root
root@odin:~# passwd -as | grep dns
dnsadmin  NL    
dnsconf   NL   
```
3. Begin some of the key DNS work [2]

	* Set service properties for the user.
```
root@odin:~# svccfg -s dns/server:default
svc:/network/dns/server:default> setprop start/user = dnsadmin
svc:/network/dns/server:default> setprop start/group = dns
svc:/network/dns/server:default> exit
```

	* Create a directory for a new process ID file.

```
root@odin:~# mkdir -p /var/named/tmp
root@odin:~# chown dnsadmin:dns /var/named/tmp
```
	* Create an initial named.conf

```bash
dnsconf@odin:~/dnswork$ cat named.conf
options {
directory "/var/named";
pid-file "/var/named/tmp/named.pid";
};
```
	* Use the dnsconf role to create and edit the file
```bash
backdoor@odin:~$ roles
dnsadmin,dnsconf,root
backdoor@odin:~$ su - dnsconf   ### use backdoor's password
Password: 
Oracle Corporation      SunOS 5.11      Solaris_11/11.4/ON/production.build-11.4-29:2018-06-26  June 2018
dnsconf@odin:~/dnswork$ pfedit /etc/named.conf
pfedit: /etc/named.conf has been created.
```
	* Start the DNS SMF service
```
dnsadmin@odin:~$ svcadm enable svc:/network/dns/server:default
dnsadmin@odin:~$ svcs -l svc:/network/dns/server:default
fmri         svc:/network/dns/server:default
name         BIND DNS server
enabled      true
state        online
next_state   none
state_time   September 20, 2018 at  1:41:02 PM EDT
logfile      /var/svc/log/network-dns-server:default.log
restarter    svc:/system/svc/restarter:default
contract_id  494 
manifest     /lib/svc/manifest/network/dns/server.xml
dependency   require_all/none svc:/system/filesystem/local (online)
dependency   require_any/error svc:/network/loopback (online)
dependency   optional_all/error svc:/milestone/network (online)
dnsadmin@odin:~$ cat /var/named/tmp/named.pid 
6323
```

```
dnsconf@odin:~/dnswork$ logout
backdoor@odin:~$ 
backdoor@odin:~$ su - dnsadmin ### use backdoor's password
Password: 
Oracle Corporation      SunOS 5.11      Solaris_11/11.4/ON/production.build-11.4-29:2018-06-26  June 2018
dnsadmin@odin:~$

4. In parallel, the steps to monitor the resulting audit activity.

	* Watch the audit activity via praudit(8) (Note: The default policy is to send all audit records to the global zone so name resolution will not be done because dnsconf is not added to the global zone). The only events audited at this point for the session are:
		* the backdoor account starts is the ssh login by backdoor, 
		* the su(8) to dnsconf, 
		* The commands that invoke the "DNS Configuration Management" profile (In this case the pfedit(8) of named.conf and to start the DSN service). pfedit(8) uses the $EDITOR, so in this case it was a vim(1) session. 
		* A logout of dsnconf is done and and a su(8) to dnsadmin is done. 
		* Then the svc:/network/dns/server:default is started via SMF using the "DNS Service Management" profile. 

	* In the global zone cd(1) to /var/audit and choose the appropriate audit file. For example:

```bash
tail -0f 20180915140628.not_terminated.s114 | praudit
```

	* Here are the events that were generated by the steps to create the /etc/named.conf and start the DNS service. First the login by the backdoor account:

```
header,85,2,login - ssh,,s114,2018-09-20 12:56:08.483-04:00
subject,backdoor,backdoor,staff,backdoor,staff,6220,2453822926,44394 22 s114
return,success,0
```
	* Then the su(8) to dnsconf:
```
header,85,2,role login,,s114,2018-09-20 12:57:10.265-04:00
subject,backdoor,1001,1000,1001,1000,6230,2453822926,171 3 s114
return,success,0
```
	* Then the diff(1) audit record created by pfedit(8) for the creation of /etc/named.conf:
```
header,365,2,create administrative file,,s114,2018-09-20 12:58:29.136-04:00
subject,backdoor,root,1000,1001,1000,6240,2453822926,171 3 s114
path,/etc/named.conf
use of authorization,solaris.admin.edit/etc/named.conf
text,--- /etc/named.conf        2018-09-20 12:58:14.400878006 -0400
     +++ /etc/named.conf.pfedit.q.TrOb  2018-09-20 12:58:29.118152703 -0400
     @@ -0,0 +1,4 @@
     +options {
     +directory "/var/named";
     +pid-file "/var/named/tmp/named.pid";
     +};
     
return,success,0
```
	* The logout of dnsconf:
```
header,85,2,role logout,,s114,2018-09-20 13:14:01.092-04:00
subject,backdoor,1001,1000,1001,1000,6230,2453822926,171 3 s114
return,success,0
```

	* The su(8) to dnsadmin:

```
header,85,2,role login,,s114,2018-09-20 13:31:07.209-04:00
subject,backdoor,1000,1000,1000,1000,6296,2453822926,171 3 s114
return,success,0
```

	* Enabling the DNS service via SMF svcadm(8)

```
header,203,2,change service instance property,,s114,2018-09-20 13:41:00.799-04:00
subject,backdoor,1000,1000,1000,1000,6309,2453822926,171 3 s114
use of authorization,solaris.smf.manage.bind
fmri,svc:/network/dns/server:default/:properties/restarter_actions/auxiliary_tty
text,b
text,"1"
return,success,0
header,227,2,change service instance property,,s114,2018-09-20 13:41:00.804-04:00
subject,backdoor,1000,1000,1000,1000,6309,2453822926,171 3 s114
use of authorization,solaris.smf.manage.bind
fmri,svc:/network/dns/server:default/:properties/restarter_actions/auxiliary_fmri
text,s
text,"svc:/network/ssh:default"
return,success,0
header,175,2,persistently enable service instance,,s114,2018-09-20 13:41:00.821-04:00
subject,backdoor,1000,1000,1000,1000,6309,2453822926,171 3 s114
use of authorization,solaris.smf.manage.bind
fmri,svc:/network/dns/server:default/:properties/general/enabled
return,success,0
header,187,2,change service instance property,,s114,2018-09-20 13:41:00.821-04:00
subject,backdoor,1000,1000,1000,1000,6309,2453822926,171 3 s114
use of authorization,solaris.smf.manage.bind
fmri,svc:/network/dns/server:default/:properties/general/enabled
text,b
text,"1"
return,success,0
```

5. Now create a more complete DNS configuration so that named controls the norsestuff.com domain.

	* Create necessary sub-directories under /var/named:
```bash
root@odin:/var/named# mkdir master dump stats
root@odin:/var/named# chown dnsadmin:dns *
root@odin:/var/named# ls -l
total 12
drwxr-xr-x   2 dnsadmin dns            2 Sep 20 14:34 dump
drwxr-xr-x   2 dnsadmin dns            2 Sep 20 14:34 master
drwxr-xr-x   2 dnsadmin dns            2 Sep 20 14:34 stats
drwxr-xr-x   2 dnsadmin dns            3 Sep 20 13:41 tmp
```
	* pfedit(8) /etc/named.conf

```
header,1254,2,edit administrative file,,s114,2018-09-20 15:01:12.585-04:00
subject,backdoor,root,1000,1001,1000,6428,2453822926,171 3 s114
path,/etc/named.conf
use of authorization,solaris.admin.edit/etc/named.conf
text,--- /etc/named.conf        2018-09-20 12:58:29.118152703 -0400
     +++ /etc/named.conf.pfedit.6DVBUb  2018-09-20 15:01:12.567433072 -0400
     @@ -1,4 +1,37 @@
      options {
     -directory "/var/named";
     -pid-file "/var/named/tmp/named.pid";
     +        directory "/var/named";
     +        pid-file "/var/named/tmp/named.pid";
     +        dump-file       "/var/named/dump/named_dump.db";
     +        statistics-file "/var/named/stats/named.stats";
     +        forwarders { 192.168.1.1; };
     +};
     +
     +
     +// zone clause - master for example.com
     +zone "norsestuff.com" in{
     +  type master;
     +  file "master/master.norsestuff.com";
     +  allow-update {none;};
     +};
     +// required local host domain
     +zone "localhost" in{
     +  type master;
     +  file "master/master.localhost";
     +  allow-update {none;};
     +};
     +
     +// REVERSE LOOK-UPS
     +
     +// localhost reverse map
     +zone "0.0.127.IN-ADDR.ARPA" in{
     +  type master;
     +  file "master/localhost.rev";
     +  allow-update{none;}; // optional
     +};
     +// reverse map for local addresses at example.com
     +// uses 10.146.44.0 for illustration
     +zone "1.168.192.IN-ADDR.ARPA" in{
     +  type master;
     +  file "192.168.1.rev";
     +  allow-update {none;};
      };
     
return,success,0
```
	* Create (using pfedit) the following files:

		* /var/named/master/192.168.1.rev (reverse lookups)
		* /var/named/master/localhost.rev (localhost reverse lookup)
		* /var/named/master/master.localhost
		* /var/named/master/master.norsestuff.com

```
header,1254,2,edit administrative file,,s114,2018-09-20 15:01:12.585-04:00
subject,backdoor,root,1000,1001,1000,6428,2453822926,171 3 s114
path,/etc/named.conf
use of authorization,solaris.admin.edit/etc/named.conf
text,--- /etc/named.conf        2018-09-20 12:58:29.118152703 -0400
     +++ /etc/named.conf.pfedit.6DVBUb  2018-09-20 15:01:12.567433072 -0400
     @@ -1,4 +1,37 @@
      options {
     -directory "/var/named";
     -pid-file "/var/named/tmp/named.pid";
     +        directory "/var/named";
     +        pid-file "/var/named/tmp/named.pid";
     +        dump-file       "/var/named/dump/named_dump.db";
     +        statistics-file "/var/named/stats/named.stats";
     +        forwarders { 192.168.1.1; };
     +};
     +
     +
     +// zone clause - master for example.com
     +zone "norsestuff.com" in{
     +  type master;
     +  file "master/master.norsestuff.com";
     +  allow-update {none;};
     +};
     +// required local host domain
     +zone "localhost" in{
     +  type master;
     +  file "master/master.localhost";
     +  allow-update {none;};
     +};
     +
     +// REVERSE LOOK-UPS
     +
     +// localhost reverse map
     +zone "0.0.127.IN-ADDR.ARPA" in{
     +  type master;
     +  file "master/localhost.rev";
     +  allow-update{none;}; // optional
     +};
     +// reverse map for local addresses at example.com
     +// uses 10.146.44.0 for illustration
     +zone "1.168.192.IN-ADDR.ARPA" in{
     +  type master;
     +  file "192.168.1.rev";
     +  allow-update {none;};
      };
     
return,success,0





header,1229,2,create administrative file,,s114,2018-09-20 15:06:58.912-04:00
subject,backdoor,root,1000,1001,1000,6433,2453822926,171 3 s114
path,/var/named/master/192.168.1.rev
use of authorization,solaris.admin.edit/var/named/master/192.168.1.rev
text,--- /var/named/master/192.168.1.rev        2018-09-20 15:06:26.880612101 -0400
     +++ /var/named/master/192.168.1.rev.pfedit.mL.vHd  2018-09-20 15:06:58.896825752 -0400
     @@ -0,0 +1,17 @@
     +; simple reverse mapping zone file for norsestuff.com$TTL 2d    ; default TTL for zone
     +$ORIGIN 1.168.192.IN-ADDR.ARPA.
     +; Start of Authority record defining the key characteristics of the zone (domain)
     +@         IN      SOA   ns1.norsestuff.com. hostmaster.norsestuff.com. (
     +                        2003080800 ; sn = serial number
     +                        12h         ; refresh
     +                        15m        ; retry
     +                        3w         ; expiry
     +                        2h         ;nxdomainttl
     +                        )
     +; name servers Resource Records for the domain
     +          IN      NS      odin.norsestuff.com.
     +;         IN      NS      ns2.norsestuff.net.
     +; PTR RR maps an IPv4 address to a host name
     +31       IN      PTR     odin.norsestuff.com.
     +32       IN      PTR     thor.norsestuff.com.
     +20       IN      PTR     s114.norsestuff.com.
     
return,success,0
header,192,2,privileged execution,,s114,2018-09-20 15:06:58.912-04:00
path,/usr/bin/pfedit
path,/home/dnsconf/dnswork
exec_args,2,pfedit,/var/named/master/192.168.1.rev
use of privilege,successful use of priv,file_dac_write
subject,backdoor,root,1000,1001,1000,6433,2453822926,171 3 s114
return,success,0


header,767,2,create administrative file,,s114,2018-09-20 15:07:30.233-04:00
subject,backdoor,root,1000,1001,1000,6436,2453822926,171 3 s114
path,/var/named/master/localhost.rev
use of authorization,solaris.admin.edit/var/named/master/localhost.rev
text,--- /var/named/master/localhost.rev        2018-09-20 15:07:18.280724308 -0400
     +++ /var/named/master/localhost.rev.pfedit.3NbvKc  2018-09-20 15:07:30.219051484 -0400
     @@ -0,0 +1,11 @@
     +$TTL 86400 ; 24 hours
     +
     +; could use $ORIGIN 0.0.127.IN-ADDR.ARPA.
     +@       IN      SOA     localhost. hostmaster.localhost.  (
     +                        1997022700 ; Serial
     +                        3h      ; Refresh
     +                        15      ; Retry
     +                        1w      ; Expire
     +                        3h )    ; Minimum
     +        IN      NS      localhost.
     +1       IN      PTR     localhost.
     
return,success,0
header,192,2,privileged execution,,s114,2018-09-20 15:07:30.234-04:00
path,/usr/bin/pfedit
path,/home/dnsconf/dnswork
exec_args,2,pfedit,/var/named/master/localhost.rev
use of privilege,successful use of priv,file_dac_write
subject,backdoor,root,1000,1001,1000,6436,2453822926,171 3 s114
return,success,0


header,727,2,create administrative file,,s114,2018-09-20 15:08:05.610-04:00
subject,backdoor,root,1000,1001,1000,6439,2453822926,171 3 s114
path,/var/named/master/master.localhost
use of authorization,solaris.admin.edit/var/named/master/master.localhost
text,--- /var/named/master/master.localhost     2018-09-20 15:07:52.536617550 -0400
     +++ /var/named/master/master.localhost.pfedit.Ljf57d       2018-09-20 15:08:05.596717272 -0400
     @@ -0,0 +1,12 @@
     +$TTL 86400 ; 24 hours could have been written as 24h or 1d
     +$ORIGIN localhost.
     +@  1D  IN    SOA @ hostmaster (
     +             2004022401 ; serial
     +             12h ; refresh
     +
     +15m ; retry
     +             1w ; expiry
     +             3h ; minimum
     +     )
     +@  1D  IN  NS @ ; localhost is the name server
     +   1D  IN  A  127.0.0.1 ; always returns the loop-back address
     
return,success,0
header,195,2,privileged execution,,s114,2018-09-20 15:08:05.610-04:00
path,/usr/bin/pfedit
path,/home/dnsconf/dnswork
exec_args,2,pfedit,/var/named/master/master.localhost
use of privilege,successful use of priv,file_dac_write
subject,backdoor,root,1000,1001,1000,6439,2453822926,171 3 s114
return,success,0


header,1882,2,create administrative file,,s114,2018-09-20 15:10:09.094-04:00
subject,backdoor,root,1000,1001,1000,6442,2453822926,171 3 s114
path,/var/named/master/master.norsestuff.com
use of authorization,solaris.admin.edit/var/named/master/master.norsestuff.com
text,--- /var/named/master/master.norsestuff.com        2018-09-20 15:09:57.433379094 -0400
     +++ /var/named/master/master.norsestuff.com.pfedit.VeU8va  2018-09-20 15:10:09.077510219 -0400
     @@ -0,0 +1,31 @@
     +; IPv4 zone file for norsestuff.com
     +$TTL 2d    ; default TTL for zone
     +$ORIGIN norsestuff.com. ; base domain-name
     +; Start of Authority record defining the pey characteristics
     +; of the zone (domain)
     +@         IN      SOA   odin.norsestuff.com. hostmaster.norsestuff.com. (
     +                        2003080800 ; se = serial number
     +                        12h         ; ref = refresh
     +                        15m        ; ret = refresh retry
     +                        3w         ; ex = expiry
     +                        2h         ; nx = nxdomainttl
     +                        )
     +; name servers Resource Records for the domain
     +              IN      NS      odin.norsestuff.com.
     +; the second name server is
     +; external to this zone (domain).
     +;              IN      NS      ns2.norsestuff.net.
     +; mail server Resource Records for the zone (domain)
     +; value 10 denotes it is the most preferred
     +;     3w       IN      MX  10  mail.norsestuff.com.
     +; the second mail server has lower preference (20) and is
     +; external to the zone (domain)
     +;              IN      MX  20  mail.norsestuff.net.
     +; domain hosts includes NS and MX records defined previously
     +; plus any others required
     +odin                IN      A       192.168.1.31
     +thor                IN      A       192.168.1.32
     +s114                IN      A       192.168.1.20
     +; aliases ftp (ftp server) to an external location
     +primary             IN      CNAME   odin.norsestuff.com.
     +gzone               IN      CNAME   s114.norsestuff.com.
     
return,success,0
header,200,2,privileged execution,,s114,2018-09-20 15:10:09.094-04:00
path,/usr/bin/pfedit
path,/home/dnsconf/dnswork
exec_args,2,pfedit,/var/named/master/master.norsestuff.com
use of privilege,successful use of priv,file_dac_write
subject,backdoor,root,1000,1001,1000,6442,2453822926,171 3 s114
return,success,0
```

6. Check the configuration files with named-checkconf and fix any syntax errors. Restart the DNS service [3]

	* run named-checkconf
	* svcadm restart svc:/network/dns/server:default using the dnsadmin account

```
backdoor@odin:~$ su - dnsadmin
Password: 
Oracle Corporation      SunOS 5.11      Solaris_11/11.4/ON/production.build-11.4-29:2018-06-26  June 2018
dnsadmin@odin:~$ svcadm restart svc:/network/dns/server:default
dnsadmin@odin:~$ svcs -l svc:/network/dns/server:default
fmri         svc:/network/dns/server:default
name         BIND DNS server
enabled      true
state        online
next_state   none
state_time   September 20, 2018 at  3:16:01 PM EDT
logfile      /var/svc/log/network-dns-server:default.log
restarter    svc:/system/svc/restarter:default
contract_id  495 
manifest     /lib/svc/manifest/network/dns/server.xml
dependency   require_all/none svc:/system/filesystem/local (online)
dependency   require_any/error svc:/network/loopback (online)
dependency   optional_all/error svc:/milestone/network (online)
```

7. Now configure a DNS client on zone thor to test [4]:

```
backdoor@thor:~$ su
Password: 
root@thor:~# svccfg -s dns/client
svc:/network/dns/client> setprop config/search = \
> astring: ("norsestuff.com")
svc:/network/dns/client> setprop config/nameserver = \
> net_address: (192.168.1.31)
svc:/network/dns/client> quit
root@thor:~# svccfg -s dns/client
svc:/network/dns/client> select network/dns/client:default
svc:/network/dns/client:default> refresh
svc:/network/dns/client:default> quit
root@thor:~# svccfg -s system/name-service/switch
svc:/system/name-service/switch> setprop config/host = astring: "files dns"
svc:/system/name-service/switch> select system/name-service/switch:default
svc:/system/name-service/switch:default> refresh
svc:/system/name-service/switch:default> quit
root@thor:~# svcadm enable network/dns/client
root@thor:~# svcadm enable system/name-service/switch
root@thor:~# ping odin
root@thor:~# ping odin
odin is alive
root@thor:~# ping odin.norsestuff.com
odin.norsestuff.com is alive
root@thor:~# dig odin.norsestuff.com

; <<>> DiG 9.10.6-P1 <<>> odin.norsestuff.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53446
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;odin.norsestuff.com.		IN	A

;; ANSWER SECTION:
odin.norsestuff.com.	172800	IN	A	192.168.1.31

;; AUTHORITY SECTION:
norsestuff.com.		172800	IN	NS	odin.norsestuff.com.

;; Query time: 0 msec
;; SERVER: 192.168.1.31#53(192.168.1.31)
;; WHEN: Thu Sep 20 15:55:39 EDT 2018
;; MSG SIZE  rcvd: 78
```


## Resources

[1] 11.4 Chapter 3 Managing DNS Server and Client Services (https://docs.oracle.com/cd/E37838_01/html/E61011/dnsadmin-1.html#scrolltoc)

[2] https://docs.oracle.com/cd/E37838_01/html/E61011/dnsref-38.html#scrolltoc

[3] https://docs.oracle.com/cd/E37838_01/html/E61011/dnsref-37.html#scrolltoc

[4] https://docs.oracle.com/cd/E37838_01/html/E61011/dnsref-36.html#scrolltoc
