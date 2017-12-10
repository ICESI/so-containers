### Introducción a contenedores en Linux
Universidad ICESI  
Curso: Sistemas Operativos  
Docente: Daniel Barragán C.  
Tema: Introducción al aislamiento de recursos empleando Linux namespaces
Correo: daniel.barragan at correo.icesi.edu.co

### Objetivos
* Conocer los fundamentos de las tecnologías de contenedores tipo LXC y LXD en Linux
* Crear entornos aislados para la ejecución de procesos a nivel del sistema operativo

### Introducción
En esta guía se introducen conceptos acerca de la tecnología de contenedores virtuales y sus fundamentos. Además se presentan
ejemplos a través de los cuales se podrá experimentar con cada una de dichas tecnologías. La información presentada es resultado de la recopilación de distintas fuentes de información, las referencias se presentan al final de la guía.

### Desarrollo

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

### Actividades

* Investigue en que consiste una bomba fork y como prevenirla usando las herramientas vistas en esta guía.

**Nota:** NO ejecute este comando en una consola hasta investigar primero que hace
```
# :(){ :|: & };:
```

### Referencias
http://man7.org/linux/man-pages/man7/namespaces.7.html  
https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces  
https://securityinside.info/contenedores-linux-seguridad/  
