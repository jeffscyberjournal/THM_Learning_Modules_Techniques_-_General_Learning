# Hacking With Powershell

Note ICMP blocked, ping wont work, use namp -Pn <ip> to check connection is present if for any reason RDP fails.

```
┌──(hacktopuser㉿hacktop)-[/mnt/VBoxShare/openvpn-troubleshooting-2025-infra-upgrades]
└─$ nmap -Pn 10.49.143.197 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-10 21:31 +1000
Nmap scan report for 10.49.143.197
Host is up (0.43s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT     STATE SERVICE
3389/tcp open  ms-wbt-server

Nmap done: 1 IP address (1 host up) scanned in 26.79 seconds
```                                                                  

Connect when sure its present.
```
$ xfreerdp /u:Administrator /p:BHN2UVw0Q /v:10.49.143.197
```

## Task 2: What is Powershell

Very light overview of powershell Nouns and Verbs and commands. 

## Task 3: Basic Powershell Commands

- Intro to 'Get-Help Command-Name' command.
