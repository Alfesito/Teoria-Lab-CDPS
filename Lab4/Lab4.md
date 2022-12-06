# PRÁCTICA 4: Introducción a la plataforma IaaS Openstack

Openstack permite la gestión de grandes centros de datos basándose en el paradigma IaaS.

Es un escenario clásico de Openstack formado por:

* **Un nodo controlador**, en el cual corren las principales aplicaciones de control y el interfaz de usuario (Dashboard).
* **Dos nodos de computación**, que dan soporte a la creación de máquinas virtuales utilizando el hipervisor KVM de Linux.

Los nodos poseen varios **interfaces de red** (dos el controlador y los nodos de computación y tres el nodo de red) mediante los cuales se conectan a tres redes distintas:

* **Red de gestión** (10.0.0.0/24), que es una red interna utilizada para todo el tráfico de control entre los nodos que componen la infraestructura Openstack (por ejemplo, para el tráfico de mensajes y llamadas REST entre todos los módulos software).
* **Red de túneles** (10.0.1.0/24), que es la red que soporta todo el tráfico entre máquinas virtuales y entre máquinas virtuales y el nodo de red. Se denomina “de túneles” ya que el tráfico se intercambia mayormente mediante protocolos tales como VXLAN o GRE.
* **Red Exterior** (10.0.10.0/24), que es la red que da acceso al exterior (Internet) y que permite que las máquinas virtuales (a través del nodo de red) tengan acceso o sean accesibles desde el exterior. También se utiliza para el acceso desde el exterior al controlador, mediante el interfaz gráfico o los APIs REST.

En producción los nodos del escenario estarían representados por máquinas físicas (servidores) y las redes que los interconectan por VLANes configuradas en switches Ethernet. Sin embargo, en nuestro caso se creará un escenario virtual corriendo dentro de nuestro host.


## 1. Arranquie del escenario Openstack

1. Creamos un directorio de trabajo, copiamos el escenario virtual de Openstack y lo arrancamos:

		$ cd /mnt/tmp
		$ cp /lab/cdps/p4/get-openstack-tutorial.sh .
		$ chmod +x get-openstack-tutorial.sh
		$ ./get-openstack-tutorial.sh
		$ cd openstack_lab-stein_4n_classic_ovs-v06
		$ sudo vnx -f openstack_lab.xml --create
		
	Una vez creado el escenario debe de aparecer cuatro ventanas de consola:  una del controlador, una del nodo de red y dos de los nodos de computación (sus credenciales son vnx/xxxx y root/xxxx).

2. Configure el escenario:

		$ sudo vnx -f openstack_lab.xml -x start-all
		
3. Configure un NAT que permita la salida a Internet de las máquinas virtuales creadas en OpenStack.

		$ ifconfig -> para sacar el nombre de la interfaz de red asignada a la dirección 138.4.31.X
		$ sudo vnx_config_nat ExtNet <nombre_if_externo>
	
	Nota: la opción "-d" es para desconfigurarel NAT.

4. Completado el arranque y la configuración, acceda al navegador a la URL: *http://10.0.10.11/horizon*. Las credenciales son: sin permisos de admin (Dominio: default; Usuario: demo; Clave: xxxx) y con permisos de admin (Dominio: default; Usuario: admin; Clave: xxxx).

## 2. Creación de una máquina virtual dentro de Openstack

Se va a crear automáticamente un sencillo escenario de red compuesto por una máquina virtual vm1 conectada a una red virtual net0 y con un router r0 que da salida hacia la red exterior.

1. Cargue en Openstack Glance las imágenes de ejemplo necesarias

		$ sudo vnx -f openstack_lab.xml -v -x load-img
2. Compruebe a través del Dashboard (Project->Compute->Images) que las imágenes se han cargado correctamente

3. Cree la MV y red virtual con:

		$ sudo vnx -f openstack_lab.xml -v -x create-demo-scenario
		
4. En la opción "topologia de red" se puede ver un mapa interactivo del escenario creado.


## 4. Creación de una nueva máquina virtual

Cree una nueva máquina virtual desde el Dashboard cuyo nombre sea igual a su login del laboratorio y conéctela a la red net0.

1. Crear una pareja de claves para asociar a la nueva máquina virtual (opción “Project->Compute->Key pairs”. Guarde la clave privada (fichero con extensión .pem) en el host.

2. Crear una nueva instancia con las mismas características que vm1 y asocie las claves creadas anteriormente.

3. Compruebe que la nueva MV tiene conexión al vm1, al host y al exterior.

4. Obtenga el fichero XML de la MV, obtenga el nombre de la interfaz de red y arranque tcpdump para la captura de tráfico de dicha interfaz

		$ virsh dumpxml 1 > vm1.xml -> vemos el nombre de la interfaz creada
		$ tcpdump -i <nombre_interfaz>
		$ ping -c 5 google.com -> desde la MV creada, para ver en la captura de tráfico
		
5. Asigne a vm2 una dirección IP flotante para que tenga conectividad exterior y pruebe a acceder a ella mediante ssh desde el host utilizando las claves privadas.

		$ ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 cirros@10.0.100.X 

	