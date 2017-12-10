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

#### LXC/LXD
LXC permite ejecutar diferentes sistemas Linux (contenedores) aislados en una máquina Linux. Un contenedor es como un entorno virtual, con su propio espacio de procesos y de red. La tecnología LXC utiliza Linux kernel control groups (Cgroups) y Namespaces (espacios de nombres) para proporcionar este aislamiento. Un contenedor tiene su propia visión del sistema operativo, del espacio de IDs de procesos, de la estructura del sistema de ficheros, y de los interfaces de red. Los comandos para interactuar con lxc tienen el formato lxc-<command>

LXD adiciona características sobre LXC tales como una interfaz REST para la gestión de contenedores a través de la red y soporte nativo para OpenStack. Los comandos para interactuar con lxd tienen el formato lxc <command>

Crear el siguiente script install_lxd.sh y ejecutarlo desde un usuario con privilegios
```
#!/bin/sh

USER_ADDED_TO_LXD_GROUP="${USER}"

# LXD/LXC uses lxc-xxx pakcage.
F=https://dl.fedoraproject.org/pub/fedora/linux/releases/25
L=${F}/Everything/source/tree/Packages/l
yum install -y wget
wget -q ${L}/lxc-2.0.5-1.fc25.src.rpm
sudo yum install -y epel-release rpmdevtools rpm-build graphviz libacl-devel
sudo yum-builddep -y lxc-2.0.5-1.fc25.src.rpm
rpmbuild --rebuild lxc-2.0.5-1.fc25.src.rpm

# Install package.
# shellcheck disable=SC2046
sudo yum localinstall -y \
     $(find ~/rpmbuild/RPMS -type f -a ! -name "*debuginfo*")

# Install LXD/LXC.
sudo adduser --system lxd --home /var/lib/lxd/ --shell /bin/false
sudo addgroup --system lxd
sudo mkdir -p /var/log/lxd
sudo chown root:lxd /var/log/lxd
sudo yum install -y git golang sqlite-devel dnsmasq squashfs-tools
export GOPATH=${HOME}/go
export PATH=${GOPATH}/bin/:${PATH}
go get -v -x -tags libsqlite3 \
   github.com/lxc/lxd/lxc \
   github.com/lxc/lxd/lxd
sudo cp "${GOPATH}"/bin/* /usr/bin/

# Create systemd service and socket.
sudo su -c '
cat <<EOF > /usr/lib/systemd/system/lxd.service
[Unit]
Description=LXD - main daemon
After=network.target
Requires=network.target lxd.socket
Documentation=man:lxd(1)

[Service]
EnvironmentFile=-/etc/environment
ExecStart=/usr/bin/lxd --group lxd --logfile=/var/log/lxd/lxd.log
ExecStartPost=/usr/bin/lxd waitready --timeout=600
KillMode=process
TimeoutStartSec=600
TimeoutStopSec=40
Restart=on-failure
LimitNOFILE=infinity
LimitNPROC=infinity

[Install]
Also=lxd.socket
EOF
'

sudo su -c '
cat <<EOF > /usr/lib/systemd/system/lxd.socket
[Unit]
Description=LXD - unix socket
Documentation=man:lxd(1)

[Socket]
ListenStream=/var/lib/lxd/unix.socket
SocketGroup=lxd
SocketMode=0660
Service=lxd.service

[Install]
WantedBy=sockets.target
EOF
'

# Run LXD for initialization.
sudo systemctl --system daemon-reload
sudo systemctl enable lxd
sudo systemctl start lxd

# Initialize LXD.
cat <<EOF | sudo lxd init
yes
default
dir
no
yes
yes
lxdbr0
auto
auto
EOF

# Running container needs user_namespace.enable=1.
sudo yum install -y grub2-tools
. /etc/default/grub
V="$GRUB_CMDLINE_LINUX user_namespace.enable=1"
sudo sed -e "s;^GRUB_CMDLINE_LINUX=.*;GRUB_CMDLINE_LINUX=\"$V\";g" \
-i /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Add user to lxd for running lxc command without privilege.
sudo gpasswd -a "${USER_ADDED_TO_LXD_GROUP}" lxd

# Reboot.
sudo reboot
```

Ejecutar los comandos y verificar que es posible crear un contenedor con Ubuntu 16.04
```
$ lxc launch ubuntu:16.04 ubuntu-1604
$ lxc exec ubuntu-1604 -- find /lib/systemd/system -maxdepth 1 \
-type f -exec sed -e 's/umount\.target//g' -i {} \;
$ lxc exec ubuntu-1604 -- systemctl --system daemon-reload
$ lxc exec ubuntu-1604 reboot
$ uname -a
$ lxc exec ubuntu-1604 -- uname -a
```

Para entrar al contenedor
```
lxc exec ubuntu-1604 bash
# exit
```

Para obtener información del contenedor
```
$ lxc info ubuntu-1604
Name: ubuntu-1604
Remote: unix://
Architecture: x86_64
Created: 2017/10/14 20:50 UTC
Status: Running
Type: persistent
Profiles: default
Pid: 20473
```

Linux provee mecanismos para leer información del kernel. Uno de estos mecanismos es el directorio **/proc**
```
$ cat /proc/meminfo
```

En el directorio /proc/cgroups se encuentran los controladores definidos en el kernel, por ejemplo el controlador **cpuset**.
```
$ cat /proc/cgroups
```

Para obtener información de los cgroups para el contenedor LXC/LXD
```
$ cat /proc/20473/cgroup
```

El directorio **/sys** permite manipular datos del kernel
```
$ mount | grep cgroup
```

Para obtener el número máximo de procesos que se pueden ejecutar en el contenedor
```
$ cat /sys/fs/cgroup/pids/lxc/ubuntu-1604/pids.max
max
```

Método directo para modificar los valores empleados anteriormente
```
$ lxc config set ubuntu-1604 limits.processes 1000
$ lxc config unset ubuntu-1604 limits.processes
```

Es posible comprobar la efectividad de ajustar un límite a la cantidad de procesos que se pueden crear en el contenedor LXC/LXD a través de una bomba fork
```
$ sudo su -c 'echo "1000" >> /sys/fs/cgroup/pids/lxc/ubuntu-1604/pids.max'
$ cat /sys/fs/cgroup/pids/lxc/ubuntu-1604/pids.max
1000
$ lxc info ubuntu-1604 | grep Processes
Processes: 26
$ lxc exec ubuntu-1604 -- perl -e 'fork while fork' \&
$ lxc info ubuntu-1604 | grep Processes
Processes: 26
```

#### Kernel Capabilities, SELinux y AppArmor
El demonio de docker requiere de privilegios de root para su ejecución, por lo tanto hay recomendaciones que se deben tener en cuenta para reducir los vectores de vulnerabilidades al emplear tecnologías de contenedores virtuales.

#### Docker y Algo más
Docker es un proyecto de código abierto que automatiza el despliegue de aplicaciones dentro de contenedores de software, proporcionando una capa adicional de abstracción y automatización de virtualización a nivel de sistema operativo. Docker utiliza características de aislamiento de recursos del kernel de Linux, tales como cgroups y espacios de nombres (namespaces) para permitir que procesos independientes se ejecuten dentro de una sola instancia de Linux.

Docker inicialmente usaba la tecnología LXC para la virtualización de recursos, la cual se encarga del manejo de los namespaces, cgroups, capabilities y controles de acceso al filesystem para los procesos que se ejecutaban en contenedores virtuales. Desde la versión 0.9 de docker, la herramienta para virtualización por defecto es el componente opensource libcontainer. libcontainer fue inicialmente escrita en Go, pero tiempo después proveedores de servicios en la nube la portaron a otros lenguajes como C, C++ y ASP.Net.

Aunque docker es el estándar de facto en la industria para el despliegue de contenedores virtuales, ya se pueden encontrar
proyectos similares en curso, como por ejemplo rocket de CoreOS.

**Nota:** Es posible emplear docker con LXC iniciando el demonio de docker con el comando **docker -e -x lxc**

#### Hypervisores
Es posible ejecutar contenedores de docker o rocket de forma nativa sobre un sistema operativo anfitrión ó tambien sobre hypervisores tipo 1 (nativo, unhosted o bare metal) ó tipo 2 (hosted)

#### Virtualización (state of art)

![][1]  
**Figura 1.** Estado del arte en virtualización

### Actividades

* Investigue en que consiste una bomba fork y como prevenirla usando las herramientas vistas en esta guía.

**Nota:** NO ejecute este comando en una consola hasta investigar primero que hace
```
# :(){ :|: & };:
```

### Referencias
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
http://kyleolivo.com/dev/2016/08/21/lxd-and-cgroups/

https://medium.com/@tiffanyfayj/docker-1-11-et-plus-engine-is-now-built-on-runc-and-containerd-a6d06d7e80ef
https://godoc.org/github.com/opencontainers/runc/libcontainer
http://www.zdnet.com/article/docker-libcontainer-unifies-linux-container-powers/
https://github.com/opencontainers/runc/tree/master/libcontainer
https://sysdig.com/blog/monitoring-greedy-containers-part-1/

https://lwn.net/Articles/676831/

https://www.ibm.com/developerworks/community/blogs/powermeup/entry/Docker_Virtualization_and_Hypervisors?lang=en

[1]: images/modern_virtualization.png
