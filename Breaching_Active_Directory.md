# Breaching Active Directory

This particular module requires immutable to disabled on /etc/resolv.conf. If not the change of resolv-dnsmasq wont take to resolv.conf when systemctl restart dnsmasq is run:

```
Check the permissions 'i' is immutable attribute
# ls -l /etc/resolv.conf && lsattr /etc/resolv.conf
-rw-r--r-- 1 root root 21 Mar 16 11:53 /etc/resolv.conf
----i---------e------- /etc/resolv.conf

#remove immutable attribute here
# chattr -i /etc/resolv.conf 
root@ip-10-48-119-205:~# ls -l /etc/resolv.conf && lsattr /etc/resolv.conf
-rw-r--r-- 1 root root 21 Mar 16 11:53 /etc/resolv.conf
--------------e------- /etc/resolv.conf

#either manually include thmdc ip with nameserver line or run the 'systemctl restart dnsmasq' command 
root@ip-10-48-119-205:~# sudo nano /etc/resolv.conf

root@ip-10-48-119-205:~# nslookup thmdc
;; Got SERVFAIL reply from 10.200.70.101
Server:		10.200.70.101
Address:	10.200.70.101#53

** server can't find thmdc: SERVFAIL

root@ip-10-48-119-205:~# nslookup thmmdt.za.tryhackme.com
Server:		10.200.70.101
Address:	10.200.70.101#53

Name:	thmmdt.za.tryhackme.com
Address: 10.200.70.202

root@ip-10-48-119-205:~# nslookup za.tryhackme.com
Server:		10.200.70.101
Address:	10.200.70.101#53

Name:	za.tryhackme.com
Address: 10.200.70.101

#likely should include nameserver 1.1.1.1 or nameserver 8.8.8.8 in /etc/resolv.conf
#for external tryhackme.com
root@ip-10-48-119-205:~# nslookup tryhackme.com
;; communications error to 10.200.70.101#53: timed out
;; communications error to 10.200.70.101#53: timed out
;; communications error to 10.200.70.101#53: timed out
;; no servers could be reached

root@ip-10-48-119-205:~# ping 10.200.70.101
PING 10.200.70.101 (10.200.70.101) 56(84) bytes of data.
64 bytes from 10.200.70.101: icmp_seq=1 ttl=127 time=1.32 ms
^C
--- 10.200.70.101 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 1.312/1.342/1.390/0.034 ms
root@ip-10-48-119-205:~# 

```
