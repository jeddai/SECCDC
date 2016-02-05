### Links
- [Domain Name System](#dns)
    - [Named.conf](#named)
        - [ACL](#acl)
        - [Options](#options)
        - [Logging](#logging)
        - [Views](#views)
        - [Final](#final)
    - [Zone Files](#zones)
        - [Internal Zones](#internal zones)
        - [External Zones](#external zones)
    - [Record Files](#records)

<a name="dns"></a>
## DNS Server (172.31.251.72)
We have a domain name system server! How it works is that IP addresses come to it looking for cct.lipscomb.edu, or another domain name that we use, and it redirects them to the right IP address for that domain name. We are using Bind to host our DNS server, and it works using some specific files.

The way Lipscomb's wireless network works is that there are two separate zones. There is a zone for people with an IP address in the domain of `10.0.0.0/8` and `172.16.0.0/12`, and another zone for anyone with a different IP address from any of those. What that allows is for there to be two IP addresses for each server, one internal, and one external. The internal IP addresses are 172.31.251.XX, and the external IP addresses are 74.119.168.XX.

The Bind service runs smoothly normally, but because we have two separate zones, there are multiple zone files and multiple db files. To edit anything in the Bind configuration, you will need to `ssh cctadmin@172.31.251.72` and make yourself the superuser. **The Bind configuration files cannot be edited unless you are the super user.** Once you are the super user, make your way into `/etc/bind`, and there you can edit files. Below are samples and explanations of the files we have set up.

---
<a name="named"></a>
### Named.conf
The Named.conf file is the base of all things Bind. There are different sections of it, with examples shown below.

---
<a name="acl"></a>
#### ACL
First is the Access Control List section. This defines the zone we will be designating as the internal zone. It should look as follows:
```
acl internal { 10.0.0.0/8; 172.16.0.0/12 };
```
Once you have completed that section you can move on to the options section.

---
<a name="options"></a>
#### Options
The options section might just be the most important section. It defines how the Bind server works, and how it interacts with the network. To start off the options file, we have it saying
```
options {
        listen-on port 53 { 127.0.0.1; 172.31.251.72;}; //this is the important line
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { any;};
```
What that line allows is that the DNS service listens on port 53 (the usual port for DNS) and has the IP address of 172.31.251.72 and also the localhost address. The rest of the lines just go there. Don't ask me why, that's just what the internet/Dave Wagner said.

Next is the recursion line. It looks like this:
```
        //recursion no; (I commented this out because I found a better way to do it)
        allow-recursion { internal; localhost; };
```
So remember when we defined the internal ACL up at the top of the named.conf? This makes it so the DNS server itself and anyone on the internal zone can recursively access the DNS server. This is much less of a security risk than having recursion allowed for all clients.

The rest of the Named.conf just kind of goes there and I don't know why exactly, it just does.
```
        dnssec-enable no;
        dnssec-validation no;
        dnssec-lookaside auto;
        bindkeys-file "/etc/named.iscdlv.key";
        managed-keys-directory "/var/cache/bind";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
        notify yes;
        also-notify { 192.168.202.102; };
        allow-transfer { 127.0.0.1; };
```

---
<a name="logging"></a>
#### Logging
Next is the logging section. If anything ever goes wrong, this tells you where it's going to tell you what happened.
```
logging {
        channel default_debug {
                file "/var/log/named/named.run";
                severity dynamic;
        };
};
```

---
<a name="views"></a>
#### Views
Now we get on to the really important section. These view sections are what make the DNS server work for both zones, both internal and external.
```
// INTERNAL
view "internal-view" {
        match-clients { internal; };
        include "/etc/bind/named.internal.zones";
};

// EXTERNAL
view "external-view" {
        match-clients { any; };
        include "/etc/bind/named.external.zones";
};
```
So the `match-clients` line decides which IP addresses each view serves. By declaring the internal view's clients as `internal`, we are again calling on the ACL we created in the first section, meaning the internal view only serves those with the IP addresses within that range.

---
<a name="final"></a>
#### Final Named.conf
Here is what the final Named.conf should look like all put together.
```
acl internal { 10.0.0.0/8; 172.16.0.0/12; };

options {
        listen-on port 53 { 127.0.0.1; 172.31.251.72;};
        #listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { any;};
        //recursion no;
        allow-recursion { internal; localhost; };
        dnssec-enable no;
        dnssec-validation no;
        dnssec-lookaside auto;
        bindkeys-file "/etc/named.iscdlv.key";
        managed-keys-directory "/var/cache/bind";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
        notify yes;
        also-notify { 192.168.202.102; };
        allow-transfer { 127.0.0.1; };
};
logging {
        channel default_debug {
                file "/var/log/named/named.run";
                severity dynamic;
        };
};

// INTERNAL
view "internal-view" {
        match-clients { internal; };
        include "/etc/bind/named.internal.zones";
};

// EXTERNAL
view "external-view" {
        match-clients { any; };
        include "/etc/bind/named.external.zones";
};
```

---
<a name="zones"></a>
### Zone files
So the zone files. How they work is that they are pointed to by the views, and in turn they point to the db files that hold all the records. Here are the internal and external zone files.

---
<a name="internal zones"></a>
#### named.internal.zones
```
zone "cct.lipscomb.edu" {
        type master;
        file "/etc/bind/cct.lipscomb.edu.db";
        check-names fail;
        allow-update { none; };
        allow-query { any; };
};
```

---
<a name="external zones"></a>
#### named.external.zones
```
zone "cct.lipscomb.edu" {
        type master;
        file "/etc/bind/cct.lipscomb.edu.external.db";
        check-names fail;
        allow-update { none; };
        allow-query { any; };
};
```
The only real difference b/w the two is that the external zone points to the external db file.

---
<a name="records"></a>
### Record files
These files hold the actual records that define which domain names point to which IP addresses. Here is the internal record file:
```
$TTL    3600;
$ORIGIN cct.lipscomb.edu.
@               IN      SOA     dns.cct.lipscomb.edu. cahumphreys.mail.lipscomb.edu. (2015170825 86400 3600 604800 10800)
@               NS              dns.cct.lipscomb.edu.
dns             IN      A       172.31.251.72
@               IN      A       172.31.251.72
minecraft       IN      A       172.31.251.74
gitlab          IN      A       172.31.251.75
muse            IN      A       172.31.251.73
mumble          IN      A       172.31.251.74
```
The weird tabbing is important, it helps with understanding what each thing is. For instance, the line `minecraft       IN      A       172.31.251.74` means that minecraft.cct.lipscomb.edu will resolve via A-name record to 172.31.251.74.

The external db file looks like this:
```
$TTL 3600;
$ORIGIN cct.lipscomb.edu.
@               IN      SOA     dns.cct.lipscomb.edu. cahumphreys.mail.lipscomb.edu. (2015170825 86400 3600 604800 10800)
@               NS      dns.cct.lipscomb.edu.
dns             IN      A       74.119.168.72
@               IN      A       74.119.168.72
minecraft       IN      A       74.119.168.74
gitlab          IN      A       74.119.168.75
muse            IN      A       74.119.168.73
mumble          IN      A       74.119.168.74
```
You'll notice the only difference is that the IP addresses for the external file correspond to the external IP addresses the servers have. When changing or adding to the record files, you will need to change both the `cct.lipscomb.edu.db` and `cct.lipscomb.edu.external.db` files accordingly, then run `sudo service bind9 restart` to make your changes valid and running. Whenever you restart Bind, always do `sudo service bind9 status` afterward to make sure your changes didn't break it. A successful status message looks like this:
```
root@dns:/etc/bind# service bind9 status
● bind9.service - BIND Domain Name Server
   Loaded: loaded (/lib/systemd/system/bind9.service; enabled; vendor preset: enabled)
  Drop-In: /run/systemd/generator/bind9.service.d
           └─50-insserv.conf-$named.conf
   Active: active (running) since Fri 2015-09-25 10:11:47 CDT; 4s ago
     Docs: man:named(8)
  Process: 20066 ExecStop=/usr/sbin/rndc stop (code=exited, status=0/SUCCESS)
 Main PID: 20071 (named)
   CGroup: /system.slice/bind9.service
           └─20071 /usr/sbin/named -f -u bind

Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: managed-keys-zone/internal-view: journal file is out of date: removing journal file
Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: managed-keys-zone/internal-view: loaded serial 1266
Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: managed-keys-zone/external-view: journal file is out of date: removing journal file
Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: managed-keys-zone/external-view: loaded serial 1266
Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: zone cct.lipscomb.edu/IN/internal-view: loaded serial 2015170825
Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: zone cct.lipscomb.edu/IN/external-view: loaded serial 2015170825
Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: all zones loaded
Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: running
Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: zone cct.lipscomb.edu/IN/internal-view: sending notifies (serial 2015170825)
Sep 25 10:11:47 dns.cct.lipscomb.edu named[20071]: zone cct.lipscomb.edu/IN/external-view: sending notifies (serial 2015170825)
```
