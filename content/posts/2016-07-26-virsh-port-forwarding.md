---
title: "virsh: port forwarding"
date: 2016-07-26T21:32:47+00:00
url: /2016/07/virsh-port-forwarding/
categories:
  - Linux

---
Source:  
https://www.redhat.com/archives/libvirt-users/2015-June/msg00033.html  
https://www.libvirt.org/hooks.html  
[bash]  
mkdir /etc/libvirt/hooks  
touch /etc/libvirt/hooks/qemu  
chmod 700 /etc/libvirt/hooks/qemu  
[/bash]  
edit /etc/libvirt/hooks/qemu:  
[bash]  
#!/bin/bash  
\# used some from advanced script to have multiple ports: use an equal number of guest and host ports

\# Update the following variables to fit your setup  
Guest_name=node001  
Guest_ipaddr=192.168.122.181  
Host_ipaddr=<external_ip>  
Host_port=( &#8216;63636&#8217; &#8216;5555&#8217; &#8216;5556&#8217; &#8216;5557&#8217; &#8216;5558&#8217; &#8216;5559&#8217; &#8216;5560&#8217; )  
Guest_port=( &#8216;63636&#8217; &#8216;5555&#8217; &#8216;5556&#8217; &#8216;5557&#8217; &#8216;5558&#8217; &#8216;5559&#8217; &#8216;5560&#8217; )

length=$(( ${#Host_port[@]} &#8211; 1 ))  
if [ &#8220;${1}&#8221; = &#8220;${Guest_name}&#8221; ]; then  
if [ &#8220;${2}&#8221; = &#8220;stopped&#8221; ] || [ &#8220;${2}&#8221; = &#8220;reconnect&#8221; ]; then  
for i in \`seq 0 $length\`; do  
iptables -t nat -D PREROUTING -d ${Host\_ipaddr} -p tcp &#8211;dport ${Host\_port[$i]} -j DNAT &#8211;to ${Guest\_ipaddr}:${Guest\_port[$i]}  
iptables -D FORWARD -d ${Guest\_ipaddr}/32 -p tcp -m state &#8211;state NEW -m tcp &#8211;dport ${Guest\_port[$i]} -j ACCEPT  
done  
fi  
if [ &#8220;${2}&#8221; = &#8220;start&#8221; ] || [ &#8220;${2}&#8221; = &#8220;reconnect&#8221; ]; then  
for i in \`seq 0 $length\`; do  
iptables -t nat -A PREROUTING -d ${Host\_ipaddr} -p tcp &#8211;dport ${Host\_port[$i]} -j DNAT &#8211;to ${Guest\_ipaddr}:${Guest\_port[$i]}  
iptables -I FORWARD -d ${Guest\_ipaddr}/32 -p tcp -m state &#8211;state NEW -m tcp &#8211;dport ${Guest\_port[$i]} -j ACCEPT  
done  
fi  
fi  
[/bash]  
Alternative variant:  
[bash]  
#!/bin/bash  
GUESTNAME=&#8221;${1}&#8221;  
COMMAND=&#8221;${2}&#8221;  
Host_ipaddr=95.211.180.232

if [[ $GUESTNAME =~ ^selenium_node[0-9]*$ ]]; then  
NUMBER=${GUESTNAME:13}  
if [ -n &#8220;$NUMBER&#8221; ]; then  
Guest_ipaddr=&#8221;192.168.122.$NUMBER&#8221;  
PORT=&#8221;6$NUMBER&#8221;  
echo $NUMBER $PORT  
for i in {0..1}; do  
if [ $COMMAND = &#8220;stopped&#8221; ] || [ $COMMAND = &#8220;reconnect&#8221; ]; then  
iptables -t nat -D PREROUTING -d ${Host\_ipaddr} -p tcp &#8211;dport ${PORT}${i} -j DNAT &#8211;to ${Guest\_ipaddr}:${PORT}${i}  
iptables -D FORWARD -d ${Guest_ipaddr}/32 -p tcp -m state &#8211;state NEW -m tcp &#8211;dport ${PORT}${i} -j ACCEPT  
fi  
if [ $COMMAND = &#8220;start&#8221; ] || [ $COMMAND = &#8220;reconnect&#8221; ]; then  
iptables -t nat -A PREROUTING -d ${Host\_ipaddr} -p tcp &#8211;dport ${PORT}${i} -j DNAT &#8211;to ${Guest\_ipaddr}:${PORT}${i}  
iptables -I FORWARD -d ${Guest_ipaddr}/32 -p tcp -m state &#8211;state NEW -m tcp &#8211;dport ${PORT}${i} -j ACCEPT  
fi  
done  
fi  
fi

[/bash]