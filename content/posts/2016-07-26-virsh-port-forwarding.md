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
```shell 
mkdir /etc/libvirt/hooks  
touch /etc/libvirt/hooks/qemu  
chmod 700 /etc/libvirt/hooks/qemu  
```  
edit /etc/libvirt/hooks/qemu:  
```shell 
#!/bin/bash  
# used some from advanced script to have multiple ports: use an equal number of guest and host ports

# Update the following variables to fit your setup  
Guest_name=node001  
Guest_ipaddr=192.168.122.181  
Host_ipaddr=<external_ip>  
Host_port=( '63636' '5555' '5556' '5557' '5558' '5559' '5560' )  
Guest_port=( '63636' '5555' '5556' '5557' '5558' '5559' '5560' )

length=$(( ${#Host_port[@]} -- 1 ))  
if [ "${1}" = "${Guest_name}" ]; then  
if [ "${2}" = "stopped" ] || [ "${2}" = "reconnect" ]; then  
for i in `seq 0 $length`; do  
iptables -t nat -D PREROUTING -d ${Host\_ipaddr} -p tcp --dport ${Host\_port[$i]} -j DNAT --to ${Guest\_ipaddr}:${Guest\_port[$i]}  
iptables -D FORWARD -d ${Guest\_ipaddr}/32 -p tcp -m state --state NEW -m tcp --dport ${Guest\_port[$i]} -j ACCEPT  
done  
fi  
if [ "${2}" = "start" ] || [ "${2}" = "reconnect" ]; then  
for i in `seq 0 $length`; do  
iptables -t nat -A PREROUTING -d ${Host\_ipaddr} -p tcp --dport ${Host\_port[$i]} -j DNAT --to ${Guest\_ipaddr}:${Guest\_port[$i]}  
iptables -I FORWARD -d ${Guest\_ipaddr}/32 -p tcp -m state --state NEW -m tcp --dport ${Guest\_port[$i]} -j ACCEPT  
done  
fi  
fi  
```  
Alternative variant:  
```shell 
#!/bin/bash  
GUESTNAME="${1}"  
COMMAND="${2}"  
Host_ipaddr=95.211.180.232

if [[ $GUESTNAME =~ ^selenium_node[0-9]*$ ]]; then  
NUMBER=${GUESTNAME:13}  
if [ -n "$NUMBER" ]; then  
Guest_ipaddr="192.168.122.$NUMBER"  
PORT="6$NUMBER"  
echo $NUMBER $PORT  
for i in {0..1}; do  
if [ $COMMAND = "stopped" ] || [ $COMMAND = "reconnect" ]; then  
iptables -t nat -D PREROUTING -d ${Host\_ipaddr} -p tcp --dport ${PORT}${i} -j DNAT --to ${Guest\_ipaddr}:${PORT}${i}  
iptables -D FORWARD -d ${Guest_ipaddr}/32 -p tcp -m state --state NEW -m tcp --dport ${PORT}${i} -j ACCEPT  
fi  
if [ $COMMAND = "start" ] || [ $COMMAND = "reconnect" ]; then  
iptables -t nat -A PREROUTING -d ${Host\_ipaddr} -p tcp --dport ${PORT}${i} -j DNAT --to ${Guest\_ipaddr}:${PORT}${i}  
iptables -I FORWARD -d ${Guest_ipaddr}/32 -p tcp -m state --state NEW -m tcp --dport ${PORT}${i} -j ACCEPT  
fi  
done  
fi  
fi
```
