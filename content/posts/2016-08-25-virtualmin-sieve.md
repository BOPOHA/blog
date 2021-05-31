---
title: "virtualmin: sieve"
date: 2016-08-25T07:59:56+00:00
url: /2016/08/virtualmin-sieve/
categories:
  - Linux
  - virtualmin
tags:
  - centos
  - sieve
  - virtualmin

---
```shell 
yum install dovecot-pigeonhole -y

cp -a /etc/procmailrc{,.orig}

echo '  
DELIVER=/usr/libexec/dovecot/deliver  
LOGFILE=/var/log/procmail.log  
TRAP=/etc/webmin/virtual-server/procmail-logger.pl  
:0wi  
VIRTUALMIN=|/etc/webmin/virtual-server/lookup-domain.pl $LOGNAME  
EXITCODE=$?  
:0  
* ?/bin/test "$EXITCODE" = "73"  
/dev/null  
EXITCODE=0  
:0  
* ?/bin/test "$VIRTUALMIN" != ""  
{  
INCLUDERC=/etc/webmin/virtual-server/procmail/$VIRTUALMIN  
}  
DEFAULT=$HOME/Maildir/  
ORGMAIL=$HOME/Maildir/  
DROPPRIVS=yes  
:0w  
| $DELIVER  
:0  
$DEFAULT  
' > /etc/procmailrc

sed -i "s|protocol lda {|protocol lda {\n mail\_plugins = \$mail\_plugins sieve|" /etc/dovecot/conf.d/15-lda.conf 

sed -i "s|^protocols =.*$|protocols = imap sieve|" /etc/dovecot/dovecot.conf

echo '  
service managesieve-login {  
    inet_listener sieve {  
        port = 4190  
    }  
} ' >> /etc/dovecot/conf.d/20-managesieve.conf

service dovecot restart  
```