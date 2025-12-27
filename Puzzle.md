# Puzzle

![Image](https://github.com/user-attachments/assets/943cbb42-5414-4db8-b595-6703723fc04c)

## Reconocimiento

Como siempre, tras comprobar que tenemos conectividad con la máquina procedemos a escanear los puertos que están abiertos con **Nmap**:

![Image](https://github.com/user-attachments/assets/30ff9ef6-7a43-4735-805e-abd9275eb26b)

En este primer escaneo podemos observar, por un lado que se trata de una máquina Linux (debido al TTL de 64) y por otro que tiene tanto el servicio HTTP activo por el puerto 80 como el servicio SSH por el puerto 22.

![Image](https://github.com/user-attachments/assets/afe44881-ce67-47d5-b7c2-b5325a8eec79)

En este segundo escaneo obtenemos más información sobre este servicio, como la versión de Apache o la de OpenSSH.

## Explotación

En un primer vistazo a la web se nos explica como esta configurado el CTF y vemos un panel de login:

![Image](https://github.com/user-attachments/assets/f4b8dfe5-64a9-41d4-9688-b91207307da3)

Utilizando **gobuster** vemos que hay un robots.txt con ciertas rutas y al descargarlo en nuestro equipo para crear un nuevo diccionario para hacer fuzzing vemos que al final hay unas credenciales y la primera pieza:

![Image](https://github.com/user-attachments/assets/72d963c7-a7b8-4fdd-900d-3e30032a5059)

Accedemos a un directorio con informacion del usuario. En la URL vemos que se esta utilizando un parametro username, probamos con usuarios comunes como administrator y admin. Con el segundo conseguimos acceder a unas nuevas credenciales confirmando la presencia de un IDOR. Accedemos como este nuevo usuario y encontramos la segunda pieza:

![Image](https://github.com/user-attachments/assets/da3d6acd-54f2-4b10-b763-240849935309)

Ahora tenemos un nuevo panel de administracion con una adivinanza y al introducir la respuesta (SQLINJECTION) conseguimos la tercera pieza:

![Image](https://github.com/user-attachments/assets/86687137-ecbe-42f3-8bef-231503a90620)

Con esta pista ya podemos deducir que el filtro que tenemos en ese mismo panel es vulnerable a SQLi. Haciendo unas pruebas lo confirmamos en su comportamiento a los errores. Tratando de listar las bases de datos con el siguiente comando obtenemos la cuarta pieza (antes de utilizar esta query se debe hacer otras como la confirmacion del número de columnas):

- `')union select 1,2, schema_name from information_schema.schemata-- -`

![Image](https://github.com/user-attachments/assets/12e295f3-5482-4255-b620-68960289ef9c)

Para agilizar el proceso de obtener información podemos utilizar sqlmap. Tras listar bases de datos, tablas y columnas encontramos que hay una larga lista de usuarios y contraseñas con hash bcrypt. Podemos intentar crackear dichas contraseñas de usuarios interesantes como admin o paco para obtener credenciales para SSH pero es un camino sin salida.

Como hemos obtenido las cuatro piezas nos fijamos en que se trata de base64 por lo que juntamos las cuatro piezas y tras decodificarlo obtenemos credenciales para un nuevo usuario:

![Image](https://github.com/user-attachments/assets/152d4ce5-1e72-416c-aba6-52f8fd5c3a4d)

## Escalada de privilegios

Obtenemos acceso por SSH y encontramos la primera flag correspondiente al usuario sin privilegios. Solo queda obtener acceso a root por lo que investigamos distintos directorios y permisos. Observamos que podemos listar capabilities y el binario de Python3 tiene una explotable:

![Image](https://github.com/user-attachments/assets/7da0136b-afd0-4aaf-8bb1-7be6cafce01f)

Adaptamos el comando a la ruta del binario y obtenemos acceso como root:

![Image](https://github.com/user-attachments/assets/491d2749-5b39-4a43-81b5-e35be55ab6a9)

Ya simplemente nos quedaría ir al directorio del usuario root para encontrar la última flag correspondiente al usuario privilegiado.
