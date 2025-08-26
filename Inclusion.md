# Inclusion

![Image](https://github.com/user-attachments/assets/0731e8c6-f67e-44a2-b233-0e62aca3237e)

## Reconocimiento

Como siempre, tras comprobar que tenemos conectividad con la máquina procedemos a escanear los puertos que están abiertos con **Nmap**:

![Image](https://github.com/user-attachments/assets/6a3fe688-f0ec-4952-9110-36dffd34a118)

En este primer escaneo podemos observar, por un lado que se trata de una máquina Linux (debido al TTL de 64) y por otro que tiene tanto el servicio HTTP activo por el puerto 80 como el servicio SSH por el puerto 22.

![Image](https://github.com/user-attachments/assets/d8612241-17d8-49fb-96d1-a3d834400ace)

En este segundo escaneo obtenemos más información sobre este servicio, como la versión de Apache o la de OpenSSH.

Un primer vistazo a la web nos confirma que esta URL no tiene nada de interés, puesto que se trata del clásico archivo index de apache, por lo que podemos buscar directorios ocultos:

![Image](https://github.com/user-attachments/assets/9f8a5e6b-c074-4f21-ae6c-43459e07bbf9)

Utilizando **gobuster** vemos que hay un directorio /shop:

![Image](https://github.com/user-attachments/assets/18453b87-3979-48a1-92cc-0396949dd850)

En este mismo directorio hay una pista muy clara que no está indicando que tenemos que explotar un LFI(Local File Inclusion):

![Image](https://github.com/user-attachments/assets/6a696489-b1ab-4a22-9ae3-30b7195d64f6)

Para confirmar el LFI hacemos un par de pruebas con **wfuzz**:

![Image](https://github.com/user-attachments/assets/55ba7a87-4a26-4251-ad9e-3fd9048824d5)

## Explotación

Tras ver que podemos acceder a archivos del sistema leemos el /etc/passwd para obtener posibles usuarios con los que poder acceder a la máquina a través de SSH. Obtenemos dos usuarios: seller y manchi.

Utilizamos hydra con ambos y obtenemos las credenciales de manchi:

![Image](https://github.com/user-attachments/assets/40f52147-1e74-4341-9864-503e8c87c17b)

## Escalada de privilegios

Una vez estamos dentro procedemos a la escalada de privilegios, para ello probamos distintos vectores (sudoers, archivos SUID, tareas Cron, posibles binarios mal configurados...) Tras un tiempo sin encontrar nada procedemos a intentar fuerza bruta para obtener la contraseña del usuario seller. Para ello, transferimos nuestro script y el diccionario desde nuestra máquina atacante a la máquina víctima. Al no tener wget, nc o curl instalados utilizamos descriptores de archivo para la transferencia:

![Image](https://github.com/user-attachments/assets/9fc6c8a2-2a37-4bb2-b907-40e82e13ef09)

Obtenemos la contraseña de este nuevo usuario y accedemos:

![Image](https://github.com/user-attachments/assets/aed30e21-c16b-45a3-9e7b-74a84d630d27)

En este caso podemos escalar a través de una mala configuración de /etc/sudoers:

![Image](https://github.com/user-attachments/assets/923aae64-3b69-4a51-96e4-10bd888c5ff7)

Ejecutamos el comando para obtener una nueva bash y accedemos como root:

![Image](https://github.com/user-attachments/assets/2549930a-89c4-467e-978b-e464451f6aa5)
