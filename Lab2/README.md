# Practica 2. Configuración de RAID y configuración de un escenario de red virtual con balanceador de tráfico

## 1. Configuración de sistemas RAID locales por software

### 1.1.- Creación de MV
Para empezar, es importante crear una carpeta en el directorio /mnt/tmp y copiamos las imágenes qcow2 y la plantilla xml modificandola.

	$ sudo virsh define s0.xml
	$ sudo virsh start s0
	
### 1.2.- Creación del RAID

La herramienta **mdadm** de Linux permite gestionar metadispositivos configurados sobre dispositivos (discos) físicos que se pueden configurar de acuerdo con determinadas configuraciones de RAID.

Accedemos a virt-manager y creamos 5 discos nuevos de 0,2 Gb, del tipo “Virtio disk”, con nombre dX.img (siendo X=[1-5]) y localizados en el directorio de la práctica.

Arrancamos la VM con `$ sudo virsh start s0` y nos metemos en la máquina con ssh `$ ssh -X cdps@<ip_vm>`

Una vez en la consola de la VM podemos cer que se han creado las 5 discos con el comando `$ sudo fdisk -l`. Para poder usar los 5 discos es necesario particionarlos con el comando *fdisk*. Para cada disco definiremos una única partición (que cubra todo el disco) y la etiquetaremos del tipo Linux RAID. Ejecutamos las siguientes instrucciones para cada disco:

	1) $ sudo fdisk /dev/vdX
	2) Comando g: crea una nueva tabla de particiones del tipo GPT
	3) Comando n: crea una nueva partición con todo el contenido del disco
	4) Comando t: cambia el tipo de la partición a Linux RAID
	5) Comando w: escribe los cambios en el disco

Comprobaremos que el módulo del kernel necesario para crear la configuración RAID5 está cargado en el kerner:

	$ lsmod | grep raid

La configuración RAID propuesta es un RAID 5 con el comando mdadm:

	$ sudo mdadm --create /dev/md0 --level=raid5 --raid-devices=5 /dev/vdb1 /dev/vdc1 /dev/vdd1 /dev/vde1 /dev/vdf1 
	
**¿Qué contenido se muestra al leer este fichero?** El fichero */proc/mdstat* contiene información sobre las configuraciones RAID presentes en el sistema.

	cdps@cdps:~$ cat /proc/mdstat
	Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] 
	[raid10] md0 : active raid5 vdf1[5] vde1[3] vdd1[2] vdc1[1] vdb1[0]
	825344 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
	
**¿Qué tamaño (neto) tendrá el grupo RAID recién creado?** 845.15 MB
	
	Comprobamos el espacio disponible
		cdps@cdps:~$ df -k
	Vemos en array size el tamaño neto
		cdps@cdps:~$ sudo mdadm --detail /dev/md0

Creado el grupo RAID será necesario crear un sistema de ficheros sobre él, para que pueda ser utilizado por el sistema operativo con:

	$ sudo mkfs.ext4 /dev/md0
	
Teniendo un sistema de ficheros podemos montarlo en el sistema operativo

	$ sudo mount /dev/md0 /
	
### 1.3.- Comandos básicos con RAID (mdadm)

Vemos que existe un nuevo dispositivo/disco

	$ fdisk -l
	
Consultar el estado el estado del grupo RAID
	
	$ mdadm --query /dev/md0
	
Vemos los detalles del RAID
	
	$ mdadm --detail /dev/md0

**¿Qué estado presenta el grupo?** State : clean

Consultas sobre cualquiera de los componentes de ese grupo

	$ mdadm --query /dev/vdb1 mdadm --examine /dev/vdb1
	
Comprobar constantemente el estado del raid

	$ watch mdadm --detail /dev/md0
	
### 1.4.- Simulamos un fallo en un disco del RAID
	
Simular el fallo de un disco

	$ mdadm --fail /dev/md0 /dev/vdb1
	
**El estado del grupo ha cambiado. ¿Cuál es?** State : clean, degraded

Simulación del reemplazo del disco fallido:

	Limpiar la información que del grupo RAID pueda tener la partición /dev/vdb1 que es la que ha salido del grupo RAID
		$ mdadm --remove /dev/md0 /dev/vdb1 mdadm --zero-superblock /dev/vdb1
	Volvemos a crear el RAID
		$ mdadm --add /dev/md0 /dev/vdb1
		
**¿Qué se observa ahora si se consulta el estado del disco?** Se observa que hay *Raid Devices : 5* y *Total Devices : 4*

	cdps@cdps:~$ cat /proc/mdstat
	cdps@cdps:~$ sudo mdadm --detail /dev/md0
	
### 1.5.- Simulamos un fallo en dos discos del RAID
	
	$ mdadm --fail /dev/md0 /dev/vdc1
	$ mdadm --fail /dev/md0 /dev/vdd1
	
**¿Qué instrucciones podrían utilizarse para simular el fallo de dos discos? ¿Qué le ocurriría al grupo RAID?** No se puede hacer fallar un disco debido a la versión de Ubuntu en la que estamos trabajando. Pero **no debería poderse recuperar, ya que ha fallado más de un disco**.

Al no ser posible la recuperación del RAID, esnecesario pararlo y volverlo a crear.

	$ umount /export mdadm --stop /dev/md0
	
## 2. Configuración de un escenario de red virtual con balanceador de tráfico

Se va a configurar un escenario de red pensado para evaluar y probar un paquete software con funcionalidades de balanceo de tráfico con **HAProxy**.

### 2.1.- Creación del escenario de balanceo de carga

1. Creamos las MV que formarán el escenario, el comando por cada máquina que queramos crear:

		$ qemu-img create -f qcow2 -b cdps-vm-base-p2.qcow2 lb.qcow2

2. Creamos el fichero XML de cada MV editando la plantilla.

3. Creamos los nombres de los bridges que soportan cada una de lass LAN:

		$ sudo brctl addbr LAN1
		$ sudo brctl addbr LAN2
		$ sudo ifconfig LAN1 up
		$ sudo ifconfig LAN2 up
	
4. Arrancamos las máquinas virtuales y accedemos a su consola

		$ sudo virsh define s1.xml
		$ sudo virsh start s1
		$ sudo virsh console s1	 	ó	  $ xterm -e sudo 'virsh console s1' &
	
5. Cambiamos el nombre modificando el fichero */etc/hostname*
	
		$sudo bash -c "echo s1 > /etc/hostname"
	
6. Modificamos el fichero /etc/hosts para cambiar la entrada asociada a la dirección 127.0.1.1 con el nombre de la VM. Y hacemos roboot y el nombre de las MV debería haber cambiado.

7. Configuramos la red de cada una de las MV

	* _No permanente_:
	
		- 	Para el balanceador de tráfico (lb):
		
				$ sudo ifconfig eth0 10.10.1.1/24
				$ sudo ifconfig eth1 10.10.2.1/24
				$ sudo bash –c "echo 1 > /proc/sys/net/ipv4/ip_forward" -> Para que el balanceador trabaje como un router
	
		- 	Para s1:
			
				$ sudo ifconfig eth0 10.10.2.11/24
				$ sudo ip route add default via 10.10.2.1
			
		-  Para c1:
	
				$ sudo ifconfig LAN1 10.10.1.3/24
				$ sudo ip route add 10.10.0.0/16 via 10.10.1.1
				
	* _Permanente_, editando los ficheros */etc/network/interfaces*

8. Modificamos la página web de los servidores:

		$ sudo bash -c "echo S1 > /var/www/html/index.html"

### 2.2.- Pruebas de conectividad y captura de tráfico

Verificamos que se pueden hacer ping entre máqunas y el comando `$ curl <ip_servidor>` 

**Cómo se realiza la conectividad entre las máquinas virtuales**

	alfesito@l123:/mnt/tmp/fer$ brctl show
	
**¿Cómo se realiza la conexión del host al escenario virtual?** `alfesito@l123:/mnt/tmp/fer$ ifconfig` -> Vemos que la conexión del host se hace con el bridge virbr0.

### 2.3.- Configuración del balanceador de carga

Lea la sección ['Configure Load Balancing'](https://www.linode.com/docs/guides/how-to-use-haproxy-for-load-balancing/#configure-load-balancing) del tutorial de HAProxy.

Para poner en marcha el balanceador:

1. Pare el servidor web que esta corriendo en lb
		
		$ service apache2 stop
		
2. Edite el fichero de configuración /etc/haproxy/haproxy.cfg, añadiendo al final: 

		frontend lb
		      bind *:80
		      mode http
		       default_backend webservers
		backend webservers
		      mode http
		      balance roundrobin
		      server s1 10.10.2.11:80 check
		      server s2 10.10.2.12:80 check
		      server s3 10.10.2.13:80 check

	Se pone al balanceador a escuchar en el puerto 80 de lb, define tres servidores activos (10.10.2.11:80, 10.10.2.12:80 y 10.10.2.13:80).

3. Rearrancamos HAproxy:
	
		$ sudo service haproxy restart
		
4. Realizamos peticiones continuas desde el host a un servidor

		$ while true; do curl 10.10.1.1; sleep 0.1; done
	
5. Configure el acceso a la pagina web de estadísticas de HAProxy siguiendo las [indicaciones](https://www.linode.com/docs/guides/how-to-use-haproxy-for-load-balancing/#configure-load-balancing).

6. Reinicie el balanceador para que cambien las configuraciones y acceda a las estadísticas del balanceador (http://10.10.1.1:8001). Además, genere un tráfico hacia los servidores para comprobar que se actualizan.
	
7. Vea que si deshabilitamos una interfaz de un servidor con `$ ifconfig eth0 down`, las peticiones se redirigen a los que quedan activos.
	