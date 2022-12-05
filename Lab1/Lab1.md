# Practica 1: Virtualización en Linux

## 1. Gestión de máquinas virtuales KVM con interfaz gráfica

El programa **virt-manager** permite realizar operaciones básicas de gestión de máquinas virtuales. Para arrancar la herramienta en el lab:

		$ HOME=/mnt/tmp sudo virt-manager
		
### 1.1 - Creación de máquinas virtuales KVM con imágenes en formato raw

La configuración de una máquina virtual se almacena internamente en un fichero XML, que suele estar localizado en el directorio /etc/libvirt/qemu.

Actividades a realizar:

* Preparación del entorno

		$ cd /mnt/tmp
		$ mkdir lab1
		$ cd lab1
		
	Copiamos y descomprimimos la imagen
	
		$ cp /lab/cdps/p1/cdps-vm-base-p1.img.bz2 .
		$ bunzip2 cdps-vm-base-p1.img.bz2
		
* Creación de una máquina virtual

	Con el gestor (virt-manager) mediante la opción “Nueva máquina virtual”. Es importante seleccionar la cuarta opción (Importar imagen de disco existente) de la siguiente pantalla y asignamos el hardware respectivo. Con todo lo anterior podemos arrancar la máquina y conectarnos a la VM con virt-manager o con ssh:

		$ ssh cdps@dirección_ip_máquina_virtual

* Creación de otra máquina virtual

	Para crear una nueva máquina virtual de forma análoga a la anterior, utilizaremos el comando:

		$ sudo halt -p
	
	¿Existe ping entre las dos máquinas creadas?

### 1.2 - Crear máquinas virtuales KVM con imágenes en formato qcow2

* El emulador qemu-kvm soporta el formato de imagen de disco qcow2, que proporciona mejoras con respecto al formato raw, tales como un menor tamaño de las imágenes y las opciones de cifrado AES y compresión zlib. Esto se debe a que cuando se duplica una imagen con este formato, no se copia el fichero imagen por completo, si no que la nueva imagen mantiene una referencia a la original.
* Creación de varias máquinas virtuales compartiendo una imagen común

	Crear un nuevo fichero de imagen en formato qcow2 partiendo de la imagen original en formato raw:

		$ qemu-img convert -O qcow2 cdps-vm-base-p1.img cdps-vm-base-p1.qcow2
		
	Crear las imágenes que utilizarán las máquinas virtuales derivadas de cdps-vm-base- p1.qcow2 (este comando se debe de ejecutar por cada MV que se quiera crear):

		$ qemu-img create -f qcow2 -b cdps-vm-base-p1.qcow2 cdps-vm1.qcow2
		
	**Una imagen qcow ahorra espacio respecto a row**
		
	Para conocer la imagen base de un fichero qcow2 se puede utilizar el comando:

		$ qemu-img info --backing-chain <fichero.qcow2>
		
## 2 - Gestión de máquinas virtuales KVM mediante interfaz libvirt

Cuando tenemos que gestionar muchas MVs, el interfaz grafico no es suficiente. Por ello utilizamos el comando **[virsh](https://computingforgeeks.com/virsh-commands-cheatsheet/)**, que es un intérprete de libvirt. Esta librería emplea los conceptos de *nodo* (máquina física), *hipervisor* (capa de software que permita virtualizar un nodo y crear un conjunto de máquinas virtuales) y *dominio* (instancia de un sistema operativo que ejecuta en una máquina virtual, soportada por un hipervisor).

* Creación de una máquina virtual

	Arrancar una de las máquinas virtuales previamente creadas y extraer el fichero XML:

		$ sudo virsh dumpxml <id_maquinavirtual> > nombre_fichero.xml

	El <id_maquinavirtual> se obtendrá mediante la ejecución del comando “sudo virsh list”.

	Interpretar el archivo XML:

	Una vez modificado el archivo XML, podemos crear una máquina virtual nuevo con:

		$ sudo virsh create nombre_fichero.xml

	Y arrancamos la máquina.

		$ $ sudo virsh start <nombreVM>

## 3 - Gestión de máquinas virtuales con LXD

LXC permite crear contenedores (virtualización ligera) en Linux, de forma que se les puede asignar recursos y aislar el software que se ejecuta en ellos del resto del sistema. LXD no es necesaria una capa de virtualización o hipervisor, lo que permite una gran eficiencia en su ejecución.

LXD se compone de los siguientes elementos:

* **Imágenes**: Es el sistema de ficheros de la máquina virtual ligera. Se proporcionan diversas imágenes predefinidas, aunque es posible crear nuevas imágenes, desde cero o partiendo de las anteriores. La descarga de las imágenes se asegura mediante el uso de identificadores únicos firmados mediante resúmenes sha256. Las imágenes se utilizan para crear contenedores a partir de ellas. Internamente se utiliza COW para crear varios contenedores partiendo de una misma imagen.
* **Contenedores**: Es una instancia de una imagen. Incluye un conjunto de recursos, como un sistema de fichero (rootfs), configuraciones sobre los recursos (entorno de ejecución, opciones de seguridad, etc.), los perfiles aplicados, estado de ejecución y otras propiedades. La configuración de un contenedor se hace al ejecutar un conjunto de perfiles o ejecutar órdenes de configuración de un parámetro concreto.
* **Perfiles**: Definen conjuntos de parámetros de la configuración de un contenedor. Un conjunto de perfiles se puede aplicar sobre un contenedor en secuencia al crearlo.
* **Instantáneas**: Es una foto de la imagen de un contenedor realizada en un instante concreto (contenedor inmutable). No se puede modificar.
* **Acceso remoto**: Es posible ejecutar órdenes de LXD a máquinas remotas. LXD es un demonio de la red, por lo que puede recibir operaciones invocadas de una máquina remota.
* **Seguridad**: Proporciona un conjunto de mecanismos para garantizar la seguridad de los contenedores (aislamiento entre ellos, limitación en acceso al host, etc.).

### 3.1 – Iniciación para ejecutar operaciones de LXD

* Añadir al usuario ($USER) en el grupo LXD

		$ sudo usermod -a -G lxd $USER
		$ su - $USER
Comprobar que el usuario pertenece al grupo lxd mediante:
		
		$ groups

	Nota: Este método permite usar lxd sólo desde el terminal en el que se han ejecutado los comandos.

* Iniciar LXD

	LXD requiere configurar e inicializar los recursos que utilizarán los contenedores, para ello usaremos la configuración automática mediante la ejecución del comando

		$ lxd init --auto
	
### 3.2 - Introducción a la creación de imágenes
El sistema de LXD proporciona, por defecto, un conjunto de imágenes de contenedores basados en las distribuciones más habituales de Linux, utilazando el siguiente comado, se crearía un contenedor con la versión 18.04 de Ubuntu (la imagen se descarga desde un repositorio remoto, muy parecido a Docker):

	$ lxc launch ubuntu:20.04 cdps-vm-lxd
	
Si ya tenemos la imagen de la MV en local, seguiremos estos pasos:

* Creamos una imagen a partir de la versión local

		$ lxc image import /lab/cdps/p1/ubuntu2004 /lab/cdps/p1/ubuntu2004.root --alias ubuntu2004

* Comprobamos que la imagen se ha descargado correctamente, listando las imagenes disponibles en el pc

		$ lxc image list
		
	**Las _imágenes_ son una plantilla para crear _contenedores_**
### 3.3 – Creación de contenedores virtuales LXD
* Creamos un contenedor
	
		$ lxc launch ubuntu2004 cdps-vm1-lxd

* Para acceder a la consola

		$ lxc console cdps-vm1-lxd

*  Para borrar imagenes

		$ lxc image delete ubuntu1804

### 3.4 – Gestión del ciclo de vida de un contenedor

*  Arrancar un contenedor ya creado

		$ lxc start cdps-vm-lxd

*  Parar la ejecución de un contenedor

		$ lxc stop cdps-vm-lxd –-force

* Para pausar la ejecución
	
		$ lxc pause cdps-vm-lxd

* Borrar contenedor

		$ lxc delete cdps-vm-lxd

### 3.5 – Gestión de imágenes de LXD

Opciones de gestionar imágenes

	$ lxc image --help
	
### 3.6 – Ejecutar ordenes (comandos) dentro de un contenedor LXD

	$ lxc exec cdps-vm-lxd -- ls -la
	
	$ lxc exec cdps-vm-lxd bash
	
### 3.7 – Gestión de ficheros de un contenedor de LXD

* Obtenga un fichero de un contendor y almacenarlo un fichero local

		$ lxc file pull cdps-vm-lxd/root/.bashrc datos

* Cargue un fichero local en un contenedor

		$ lxc file push datos cdps-vm-lxd/root/datos

* Edite un fichero en un contenedor

		$ lxc file edit cdps-vm-lxd/root/datos
		
Los componentes de un lxd, como imágenes, contenedores o dispositivo virtual de almacenamiento (storage pool) se almacenan en /var/lib/lxd

### 3.8 – Configurar el hardware del contenedor

* Mostramos la información del conenedor

		$ lxc info cdps-vm-lxd
		
* Configure el límite de hardware

		$ lxc config set cdps-vm-lxd <haardware> <num de recursos>
		
## 4 - Ejecución remota de aplicaciones gráficas

La imagen de máquina virtual proporcionada no dispone de interfaz gráfica. Es posible arrancar aplicaciones gráficas en la máquina virtual que utilicen como pantalla de una máquina remota, utilizando un **sistema de ventanas X**. Para ello utilizaremos **xterm**, que es una aplicación que emula un terminal del sistema de ventanas X.

* Actualizar la lista de paquetes software

		$ sudo apt-get update
		
* Instalar “xterm” en la máquina virtual

		$ sudo apt-get install xterm xauth x11-apps
		
* Abrir una sesión remota en la máquina anfitriona, habilitando el envío de elementos de las X

		$ slogin -X <nombre_usuario>@<dirección_ip_máquina_virtual>
		
* Ejecutar xterm en la MV

		$ xterm