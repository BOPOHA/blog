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
[bash]  
yum install dovecot-pigeonhole -y

cp -a /etc/procmailrc{,.orig}

echo &#8216;  
DELIVER=/usr/libexec/dovecot/deliver  
LOGFILE=/var/log/procmail.log  
TRAP=/etc/webmin/virtual-server/procmail-logger.pl  
:0wi  
VIRTUALMIN=|/etc/webmin/virtual-server/lookup-domain.pl $LOGNAME  
EXITCODE=$?  
:0  
* ?/bin/test &#8220;$EXITCODE&#8221; = &#8220;73&#8221;  
/dev/null  
EXITCODE=0  
:0  
* ?/bin/test &#8220;$VIRTUALMIN&#8221; != &#8220;&#8221;  
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
&#8216; > /etc/procmailrc

sed -i &#8220;s|protocol lda {|protocol lda {\n mail\_plugins = \$mail\_plugins sieve|&#8221; /etc/dovecot/conf.d/15-lda.conf 

sed -i &#8220;s|^protocols =.*$|protocols = imap sieve|&#8221; /etc/dovecot/dovecot.conf

echo &#8216;  
service managesieve-login {  
inet_listener sieve {  
port = 4190  
}  
} &#8216; >> /etc/dovecot/conf.d/20-managesieve.conf

service dovecot restart  
[/bash]