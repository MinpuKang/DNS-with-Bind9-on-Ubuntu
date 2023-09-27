##	Purpose

This document is used to build DNS with bind9 service on ubuntu.

## Bind9 Installation

### With Internet Connection

If internet is reachable, using below CLI:

```apt-get install bind9```

### Without Internet Connection

If no internet, using dpkg to install all dependency packages one by one until bind9 can be installed.

Download SW, here is a summary for the dependency package for bind9 v9.16:

 https://github.com/MinpuKang/DNS-with-Bind9-on-Ubuntu/tree/main/ubuntu-bind9-dependancy-packages-u

Or if above one is not available, can download one by one from https://pkgs.org/

Install CLI:

```dpkg -i <package name>```


###	Package Installed Status

Installed Status check:

```
dpkg -l | grep -i <string>

e.g.:

#  dpkg -l | grep bind9

ii  bind9                                 1:9.16.37-1~deb11u1                   amd64        Internet Domain Name Server
ii  bind9-dnsutils                        1:9.16.1-0ubuntu2.12                  amd64        Clients provided with BIND 9
ii  bind9-doc                             1:9.18.16-1~deb12u1                   all          Documentation for BIND 9
ii  bind9-host                            1:9.16.1-0ubuntu2.12                  amd64        DNS Lookup Utility
ii  bind9-libs:amd64                      1:9.16.37-1~deb11u1                   amd64        Shared Libraries used by BIND 9
ii  bind9-utils                           1:9.16.37-1~deb11u1                   amd64        Utilities for BIND 9
iU  bind9utils                            1:9.18.16-1~deb12u1                   all          Transitional package for bind9-utils

The first three columns of the output show the desired action, the package status, and errors, in that order.
          Desired action:
            u = Unknown
            i = Install
            h = Hold
            r = Remove
            p = Purge

          Package status:
            n = Not-installed
            c = Config-files
            H = Half-installed
            U = Unpacked
            F = Half-configured
            W = Triggers-awaiting
            t = Triggers-pending
            i = Installed

          Error flags:
            <empty> = (none)
            R = Reinst-required

```

More detail of status: https://linuxprograms.wordpress.com/2010/05/11/status-dpkg-list/


Detail of dpkg https://manpages.ubuntu.com/manpages/trusty/man1/dpkg.1.html


## Configuration Files

Note: super user required, contact server/DNS owner to update.

###	Global Configuration

This part configuration files are stored in /etc/bind/.

Three main files as below may be required to update:

•	named.conf
```
main file which includes other files.

No need to be updated.
```

•	named.conf.options

Configure:
-	zone file directory.
-	Listen port
-	Forwarders

An example as below without forward to other DNSs:
```
acl "trusted" {
        localhost;
};
options {
        directory "/var/cache/bind";

        listen-on { any; };
        allow-transfer { none; };

        //forwarders {
        //    1.1.1.1;
        //    8.8.8.8;
        //};

};
```
•	named.conf.local

Define the zone and file mapping, and file need to be created under the directory configured in named.conf.options, like in above example, zone files need to be created in folder /var/cache/bind.

Below is an example for named.conf,local:
```
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "hk314.top" {
    type master;
    file "db.hk314.top";
};
zone "mnc001.mcc666.3gppnetwork.org" {
    type master;
    file "db.mnc001.mcc666.3gppnetwork.org";
};
```
###	Zone File Configuration
Zone file includes detail of DNS resolve, zone files need to be stored in the directory configured in named.conf.options, like in above example, zone files need to be created in folder /var/cache/bind.

And zone file names need to be aligned with each one defined in name.conf.local
```
# cd /var/cache/bind/
# ls -l
total 28
-rw-r--r-- 1 root root 7571 Sep 26 06:57 db.hk314.top
-rw-r--r-- 1 root root 5524 Sep 21 12:07 db.mnc001.mcc666.3gppnetwork.org
-rw-r--r-- 1 bind bind  297 Sep 26 06:58 managed-keys.bind
-rw-r--r-- 1 bind bind 3400 Sep 26 06:58 managed-keys.bind.jnl
#
```

Zone file includes the TTL, ORIGIN(zone) and detail A/AAAA Record, NAPTR record or SRV record, an example as below:
```
# cd /var/cache/bind/        ->based on directory in named.conf.options
# vi db.hk314.top            ->based on zone file in named.conf.local
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     localhost. root.localhost. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
@       IN      A       127.0.0.1
@       IN      AAAA    ::1

;$ORIGIN    hk314.top.

;===========================================================
;DNS configuration: services including all are examples here
;===========================================================

test.hk314.top.                                 IN A   1.1.1.10
```

## How to Add new Record

Note: super user required, contact server/DNS owner to update.

###	Record with configured zone

If the zone is defined in named.conf.local, then add the new record in mapped file, after that, restart bind9 service, step as below:

1.	Check the mapped file path based on named.conf.local(zone file) and name.conf.options(directory).

2.	Add the new record in mapped file of zone:

The record format is as below for A Record, if zone is followed in the record, the last dot must be required.
```
test1.hk314.top.                                 IN A   1.1.1.1
```

NAPTR record as below, if zone is followed in the record, the last dot must be required.
```
*.tac.epc.mnc001.mcc666.3gppnetwork.org.        IN NAPTR 10 10 "a"  "x-3gpp-sgw:x-s5-gtp:x-s8-gtp" "" topon.sgw-s5s8.sgw.epc.mnc001.mcc666.3gppnetwork.org.
```

3.	Restart bind9 service:
```
# Check status
systemctl status bind9.service

# Restart bind9
systemctl restart bind9.service

# Check status after restart
systemctl status bind9.service
```

### Record with new zone 

If record zone is not defined in named.conf.local, need to add zone-file mapping in named.conf.local, then add record in file, after that restart bind9 service, step as below:

1.	Add zone-file maping in named.conf.local
```
# vi named.conf.local 
………………………
zone "example.com" {
    type master;
    file "db.example.com";
};
```

2.	Add record in zone file, if zone is followed in the record, the last dot must be required.
```
# cd /var/cache/bind/       ->based on directory in named.conf.options
# vi db.example.com         ->based on zone file in named.conf.local
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     localhost. root.localhost. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
@       IN      A       127.0.0.1
@       IN      AAAA    ::1

;$ORIGIN    example.com.

;===========================================================
;DNS configuration: services including all are examples here
;===========================================================

test.example.com.                                 IN A   2.2.2.2
```

3.	Restart bind9 service:
```
# Check status
systemctl status bind9.service

# Restart bind9
systemctl restart bind9.service

# Check status after restart
systemctl status bind9.service
```

## Test and Tips

Monitor if dns service works well:
```
netstat -apn | grep ":53"
OR
ss -lnpa | grep -i ":53"
```

DNS Record can be tested with dig of nslookup, dig can use to test natrp record with DNS destination:
```
nslookup <FQDN>

dig <FQDN>

dig @<dns server IP> -b <source IP> <FQDN> -t NAPTR
```


If dnsmasq is used, must stop dnsmasq firstly, then start bind9 and also need disable dnsmasq to disable auto-start when system reboot.
```
systemctl status dnsmasq.service
systemctl stop dnsmasq.service
systemctl disable dnsmasq.service
systemctl status dnsmasq.service
```

A Windows SW for EPC/5GC DNS Configure Generation as assets with below link can be used to generate EPC/5GC DNS record:

https://github.com/MinpuKang/DNS-Config-Generator

