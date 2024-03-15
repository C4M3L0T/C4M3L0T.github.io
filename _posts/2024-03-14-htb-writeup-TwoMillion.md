---
layout: single
title: TwoMillion - Hack The Box
excerpt: "Esta máquina fácil será mi primera publicación en esta página, mi objetivo es crear Writeups en español y documentar mi avance como pentester al jugar CTF." 
date: 2024-03-14
classes: wide
header:
  teaser: /assets/images/htb-writeup-twoMillion/twoMillion_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - infosec
tags:  
  - JavaScript
  - API
  - Inyección de comandos
  - Enumeracion de sistema
  - CVE-2024-0386
---

![](/assets/images/htb-writeup-twoMillion/twoMillion_logo.png)

Bueno, empezamos con esta máquina fácil de HackTheBox, la descripción de la máquina nos dice que fue lanzada para celebrar los 2 millones de usuarios en la página. La máquina es una versión antigüa de la misma página que incluye una invitación hackeable. Nos dice que después de hackear la invitación una cuenta puede crearse en la plataforma. La cuenta puede ser usada para enumerar varios API, y solo uno puede ser usado para elevar el usuario a admin. Con accesos administrativos el usuario puede ejecutar un comando de inyección en la VPN del administarador, lo cual nos puede permitir ingresar a una shell. y un archivo .env, este archivo contiene las credenciales de la base de datos y asi podemos ingresar como user admin dentro de la máquina. El kernel del sistema se encuentra desactualizado y el exploit CVE-2023-0386 puede ser usado para ganar el shell de root. Bueno, con esto comenzamos.

## Escaneo de puerto

El siguiente comando nos ayuda a escanear los puertos abiertos de la máquina víctima y los exporta a un archivo llamado "escaneo" y como podemos observar tenemos el 80 y el 22 abiertos a los que corresponden a los servicios http y ssh 

```
   1   │ # Nmap 7.94SVN scan initiated Thu Mar 14 22:53:06 2024 as: nmap -p- --open -
       │ sS --min-rate 5000 -n -Pn -vvv -oN escaneo 10.10.11.221
   2   │ Nmap scan report for 10.10.11.221
   3   │ Host is up, received user-set (0.11s latency).
   4   │ Scanned at 2024-03-14 22:53:06 CST for 14s
   5   │ Not shown: 65533 closed tcp ports (reset)
   6   │ PORT   STATE SERVICE REASON
   7   │ 22/tcp open  ssh     syn-ack ttl 63
   8   │ 80/tcp open  http    syn-ack ttl 63
   9   │ 
  10   │ Read data files from: /usr/bin/../share/nmap
  11   │ # Nmap done at Thu Mar 14 22:53:20 2024 -- 1 IP address (1 host up) scanned 
       │ in 13.62 seconds

```



## Página web 

Al poner la IP de la máquina víctima podemos observar que nos redirige hacia https://2million.htb 

![](/assets/images/htb-writeup-twoMillion/website1.png)

Lo que podemos hacer para poder observar la página es agregar el vHost a nuestro archvio hosts de nuestra máquina con el siguiente comando 

```
echo '10.10.11.221 2million.htb' | sudo tee -a /etc/hosts

```

Y después de ingresar el vHost podemos ver la versión antigüa de la página de HackTheBox

![](/assets/images/htb-writeup-twoMillion/website2.png)

Podemos observar que la página tiene habilitada la parte de "join". Después de hacer click en el botón "Join HTB" nos redirige a /invite 

![](/assets/images/htb-writeup-twoMillion/website3.png)

Esto parece ser la versión antigüa de invitación de HTB. lo que podemos hacer es ver el código fuente de la página.

![](/assets/images/htb-writeup-twoMillion/website4.png)

Al parecer la segunda función del script JS se llama cuando se da click en el botón y envía una petición POST a "/api/v1/invite/verify", para después validar si el código ingresado es valido o no. Al parecer también hay otro script llamado "inviteapi.min.js" siendo cargado en la página, al parecer esta pfiscadp, sin embargo lo podemos ingresar en "de4js" para que sea un código leible  

![](/assets/images/htb-writeup-twoMillion/website5.png)


![](/assets/images/htb-writeup-twoMillion/website6.png)

```
function verifyInviteCode(code) {
    var formData = {
        "code": code
    };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function (response) {
            console.log(response)
        },
        error: function (response) {
            console.log(response)
        }
    })
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function (response) {
            console.log(response)
        },
        error: function (response) {
            console.log(response)
        }
    })
}
```
Podemos obervar dos funciones, la primera es similar a la que vimos anteriormente que hace una petición POST pero la segunda se ve mas interesante para nuestro proposito, por lo que se ve, es que se puede hacer una petición POST a "/api/v1/invite/how/to/generate". Este endpoint se ve que puede ser muy interesante, entonces para acceder lo que podemos hacer es llamar esta función de JS desde la consola usando "curl"

## API

![](/assets/images/htb-writeup-twoMillion/api1.png)

Bueno, para esto tuve que investigar tipos de encriptación, ya que si leemos el JSON result podemos observar que nos da una pista. por lo que investigue la salida de "data" esta encriptado en "ROT13" o también conocido como encriptación César, básicamente lo que hace esta encriptación es recorrer el abecedario 13 lugares. Ya he tenido experiencia con esta encriptación gracias a OverTheWire y hay una página que lo desencripta

![](/assets/images/htb-writeup-twoMillion/api2.png)

Pues el siguiente paso es fácil, hacer lo que nos pide la API, haremos la petición correspondiente 

![](/assets/images/htb-writeup-twoMillion/api3.png)

Como podemos observar ahora no esta encriptado, ahora esta códificado y la codificación es en Base64, entonces lo decodificamos. Y al parecer ya tenemos nuestro código de invitación y podemos ponerlo en la página.

![](/assets/images/htb-writeup-twoMillion/api4.png)

Ingresamos el código en la página correspondiente y como podemos observar, nos muestra una página de registro, lo que haremos ahora es poner nuestros datos, un correo falso y la contraseña que queramos.

![](/assets/images/htb-writeup-twoMillion/api5.png)

Ahora vemos la página de login, ingresamos nuestros datos anteriores

![](/assets/images/htb-writeup-twoMillion/api6.png)

Y listo, ya ingresamos a la página /home, al revisar, pude ver que hay una página interesante, "Access", vemaos que podemos hacer.

![](/assets/images/htb-writeup-twoMillion/api7.png)

Ingresamos a Access y podemos ver que nos deja descargar y regenerar su VPN para poder ingresar a HTB, Pues es hora de usar burpsuite e interceptar y ver lo que hace el botón de "Connection Pack"


![](/assets/images/htb-writeup-twoMillion/api8.png)

```
GET /api/v1/user/vpn/generate HTTP/1.1
Host: 2million.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://2million.htb/home/access
DNT: 1
Connection: close
Cookie: PHPSESSID=jtfkmq8glgfqbqpm4dokukah86
Upgrade-Insecure-Requests: 1
```
Después de dar click en el botón se crea una petición GET hacia "/api/v1/users/vpn/generate" y retorna un archivo VPN para que nuestro usuario lo descargue.

Probemos que podemos hacer si hacemos un request a la URL de /api


![](/assets/images/htb-writeup-twoMillion/api9.png)

```
Usamos -v para poder ver mas detalles sobre la respuesta del servidor, asi como la respuesta del código
```

Al parecer obtenemos el status 401 que es cuando no estas autorizado, 

![](/assets/images/htb-writeup-twoMillion/api10.png)

Pues utilice la cookie que previamente nos dio el burpsuite para poder ingresar y nos lanzó un código 200, que significa Ok.

Ahora intentemos con /api/v1


![](/assets/images/htb-writeup-twoMillion/api11.png)

Y obtenemos otras URL's con las cuales trabajar, probremos con las de admin.


![](/assets/images/htb-writeup-twoMillion/api12.png)

Pues no era ninguna sorpresa que no estamos autorizados para poder usar este endpoint, intentemos con otra 


![](/assets/images/htb-writeup-twoMillion/api13.png)

Y otra vez obtenemos un código de error 404 intentemos con el ultimo, y como podemos observar, es un PUT.

![](/assets/images/htb-writeup-twoMillion/api14.png)

Ahora no nos da ese error, pero nos marca que el contenido no es valido. Cuando usamos API's es usual que usemos JSON para enviar y recibir datos, y ahora mismo sabemos que la API nos contesta con JSON, entonces intentemos poner el "Content-type" header en JSON e intentemos otra vez

![](/assets/images/htb-writeup-twoMillion/api15.png)

Interesante, ahora nos dice que nos hace falta el parametro "is_admin". Pues pongámoslo también 


![](/assets/images/htb-writeup-twoMillion/api16.png)

Nos vuelve a pedir otro paremetro, esta vez, is_admin tiene que ser 1 o 0, a chambear.

![](/assets/images/htb-writeup-twoMillion/api17.png)

¡Cool!, esto solo significa que ahora nuestro usuario "camelot" es ahora un administrador, ya que este endpoint, lo que hace es actualizar el estado de nuestro usuario, comprovemoslo con el EndPoint para autenticar administradores 

![](/assets/images/htb-writeup-twoMillion/api18.png)

Estabamos en lo correcto, ahora obtenemos un true, confirmando lo anterior.



