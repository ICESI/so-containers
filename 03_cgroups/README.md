### Introducción a contenedores en Linux
Universidad ICESI  
Curso: Sistemas Operativos  
Docente: Daniel Barragán C.  
Tema: Introducción a la asignación de recursos empleando cgroups 
Correo: daniel.barragan at correo.icesi.edu.co

### Objetivos
* Conocer los fundamentos de las tecnologías de contenedores tipo LXC y LXD en Linux
* Crear entornos aislados para la ejecución de procesos a nivel del sistema operativo

### Introducción
En esta guía se introducen conceptos acerca de la tecnología de contenedores virtuales y sus fundamentos. Además se presentan
ejemplos a través de los cuales se podrá experimentar con cada una de dichas tecnologías. La información presentada es resultado de la recopilación de distintas fuentes de información, las referencias se presentan al final de la guía.

### Desarrollo

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

### Actividades

* Investigue en que consiste una bomba fork y como prevenirla usando las herramientas vistas en esta guía.

**Nota:** NO ejecute este comando en una consola hasta investigar primero que hace
```
# :(){ :|: & };:
```

### Referencias
https://www.certdepot.net/rhel7-get-started-cgroups/
http://epistolatory.blogspot.com.co/2014/07/first-sysadmin-impressions-on-rhel-7.html
https://www.freedesktop.org/wiki/Software/systemd/MyServiceCantGetRealtime/
https://blog.docker.com/2013/08/containers-docker-how-secure-are-they/  
https://elpuig.xeill.net/Members/vcarceler/articulos/introduccion-a-los-grupos-de-control-cgroups-de-linux
