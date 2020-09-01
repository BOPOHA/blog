---
title: "install selfhosted internal mailinator"
date: 2020-09-01T19:54:50+00:00
url: /2020/09/install-selfhosted-internal-mailinator/
categories:
  - Linux
  - Golang
  - named
  - inbucket

---
# Setup domain
 
 - Log in https://my.freenom.com/
 - buy yourdomain.here for $0/year
 - set glue records:
   ```
   KLEY.yourdomain.here - xxx.yyy.zzz.abc
   YELK.yourdomain.here - xxx.yyy.zzz.abc
   ```
 - Use custom nameservers:
   ```
   KLEY.yourdomain.here
   YELK.yourdomain.here
   ```

# Setup DNS service

```shell
sudo -i
dnf install bind bind-utils
cat > /etc/named.conf << TOLIK
options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 {  none; };

        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        recursion no;
        dnssec-enable no;
        dnssec-validation no;
        managed-keys-directory "/var/named/dynamic";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

zone "yourdomain.here" {
        type master;
        file "yourdomain.here.hosts";
        allow-transfer {
                127.0.0.1;
                localnets;
                };
        };

TOLIK


cat > /var/named/yourdomain.here.hosts << TOLIK
\$ttl 38400
@  IN  SOA  yourdomain.here. root.yourdomain.here. (
            1461722457
            10800
            3600
            604800
            38400 )
yourdomain.here.  IN  A  xxx.yyy.zzz.abc
kley.yourdomain.here. IN      A       xxx.yyy.zzz.abc
yelk.yourdomain.here. IN      A       xxx.yyy.zzz.abc
yourdomain.here.  IN  MX  5 yourdomain.here.
@  IN  NS  kley.yourdomain.here.
@  IN  NS  yelk.yourdomain.here.

TOLIK


named-checkconf /etc/named.conf
named-checkzone  yourdomain.here  /var/named/yourdomain.here.hosts 
# zone yourdomain.here/IN: loaded serial 1461722457
# OK

```

# Setup MAIL service

```shell
mkdir /srv/inbucket
chown daemon:daemon /srv/inbucket
dnf install -y https://github.com/inbucket/inbucket/releases/download/v3.0.0-beta3/inbucket_3.0.0-beta3_linux_amd64.rpm

mkdir /etc/systemd/system/inbucket.service.d/

cat > /etc/systemd/system/inbucket.service.d/override.conf << TOLIK
[Service]
Environment=INBUCKET_SMTP_ADDR=0.0.0.0:25
Environment=INBUCKET_SMTP_DOMAIN=yourdomain.here
Environment=INBUCKET_SMTP_MAXRECIPIENTS=2000
Environment=INBUCKET_SMTP_DEFAULTACCEPT=false
Environment=INBUCKET_SMTP_ACCEPTDOMAINS=yourdomain.here
Environment=INBUCKET_POP3_DOMAIN=yourdomain.here
Environment=INBUCKET_POP3_ADDR=127.0.0.1:1100
Environment=INBUCKET_WEB_ADDR=0.0.0.0:80
Environment=INBUCKET_WEB_MAILBOXPROMPT=@yourdomain.here
Environment=INBUCKET_WEB_MONITORHISTORY=1000
Environment=INBUCKET_STORAGE_TYPE=file
Environment=INBUCKET_STORAGE_PARAMS=path:/srv/inbucket
Environment=INBUCKET_STORAGE_RETENTIONPERIOD="8760h"
Environment=INBUCKET_STORAGE_RETENTIONSLEEP="3600s"
Environment=INBUCKET_STORAGE_MAILBOXMSGCAP="0"

NonBlocking=true
NoNewPrivileges=yes
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
LimitNOFILE=500000
LimitNPROC=500000

TOLIK

systemctl start inbucket
systemctl status inbucket
systemctl enable inbucket

```
