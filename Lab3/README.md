# Práctica 3: Gestión de servicios

En esta práctica se va a configurar un servicio web Apache en una máquina virtual como en la nube. Tanto manualmente en la nube y en una máquina física, como por medio de un script en python.

## 1. Configuración manual en la nube

### 1.1.- Instalación de VM en una nube pública

Accediendo a [Google Cloud](https://cloud.google.com). Accedemos al menú y la opción de Compute Engine y seleccionamos VM instaces. Y vamos seleccionando las opciones requeridas. Una vez terminada la configuracón de la MV, dodemos arrancarlas y conectarnos a ello por medio de ssh.

### 1.2.- Instalación de paquetes en la VM

Una vez dentro de la MV, prodemos a la instalación de paquetes necesarios para instalar un servidor Apache.

	$ sudo apt-get update
	$ sudo apt-get install apache2
	$ sudo apt-get install lynx
	$ sudo apt-get install curl
	
Instalados los paquetes, vemos si Apache está funcionando:
	
	$ sudo systemctl status apache2
	$ sudo service --all-status
	
En el caso de que no esté activo el servicio Apache, debemos arrancarlo:
	
	$ sudo service apache2 start

### 1.3.- Funcionamiento del servicio Apache
	
Con lynx o curl, podemos ver si Apache funciona correctamente (dentro de la VM):

	$ lynx http://localhost
	$ curl http://127.0.0.1
	
Nota: en el caso que queramos ver el funcionamiento del servicio desde otra máquina, es necesario conocer la IP pública de la VM que sostiene el servicio Apache.

## 2. Configuración manual en máquina física

### 2.1.- Copiar y crear la MV

De la misma forma que en otras practicas con la imagen de la MV y con virt-maneger la instalamos

	$ cp /lab/cdps/p1/cdps-vm-base-p1.img.bz2 . 
	$ bunzip2 cdps-vm-base-p1.img.bz2
	$ HOME=/mnt/tmp sudo virt-manager -> pra configurar la MV
	
Arrancamos la máquina y nos metemos en su terminal

	$ sudo virsh start <nombre de la maquina>
	$ sudo virsh console <nombre de la maquina>

### 2.2.- Configuaciones de red

Para modificar la dirección de red de la máquina virtual, debemos editar el fichero */etc/network/interfaces*:

	$ sudo vi /etc/network/interfaces	(Incluyendo al final de la linea lo siguiente)
		iface eth0 inet static
		address 192.168.122.241
		netmask 255.255.255.0
		gateway 192.168.122.1
		dns-nameservers 192.168.122.1
		
Guarde los cambios y haga reboot: `$ sudo reboot now`

### 2.3.- Comprobación de la conexión

Desde una terminal del host verificamos que hay conexión entre el host y la VM

	$ ping -c 1 192.168.122.241
	
### 2.4.- Instalación de servicios

Mismo procedimiento que se ha hecho anteriormente en la nube.

Observe que el directorio */etc/apache2/sites-available* contiene los ficheros de configuración del servidor. Y en el directorio */var/www/html* se encuentran los ficheros HTML que se van a ver al usar Apache.


## 3. Automatización de la conficuración con Python

* **Instalapache.py**: nstalará en una máquina el servidor apache y realiza la configuración inicial.

		from subprocess import STDOUT, check_os.system
		import os
		
		print("\n[!]Tienes que ser root para poder ejecutar este script\n")
		print("Instalando...\n")
		check_os.system(['apt-get', 'install', '-y', 'apache2'], stdout=open(os.devnull, 'wb'), stderr=STDOUT)
		check_os.system(['apt-get', 'install', '-y', 'lynx'], stdout=open(os.devnull, 'wb'), stderr=STDOUT)
		check_os.system(['apt-get', 'install', '-y', 'wget'], stdout=open(os.devnull, 'wb'), stderr=STDOUT)
		check_os.system(['apt-get', 'install', '-y', 'curl'], stdout=open(os.devnull, 'wb'), stderr=STDOUT)
		print("La instalación ha sido completada\n")

* **Addwebhost.py**: añadirá una nueva web, admitiendo el nombre de dicha web como parámetro.

		import sys  # Leer y modificar ficheros
		from subprocess import call  # Llamar a un comando
		from subprocess import STDOUT, check_call
		import os
		
		param = str(sys.argv[1])
		print('NOTA: Para ejecutar este script es necesario ser root con "$su root". De otra forma no se ejecutará correctamente el script.\n')
		
		if len(sys.argv) == 2:
		    print("\nAñadiendo webhost y usuario...\n")
		
		    # Iniciamos el servicio apache2
		    check_call(['service', 'apache2', 'start'],
		               stdout=open(os.devnull, 'wb'), stderr=STDOUT)
		
		    # Copiamos la configuración inical y le añadimos dos líneas necesarias para la creacción de un nuevo dominio
		    fin = open('/etc/apache2/sites-available/000-default.conf', 'r')  # in file
		    fout = open('/etc/apache2/sites-available/'+param+'.conf', 'w')  # out file
		    for line in fin:
		        if "DocumentRoot /var/www/html" in line:
		            fout.write("        DocumentRoot /var/www/html\n")
		            fout.write("        ServerName "+param+".cdps\n")
		            fout.write("        ServerAlias www."+param+".cdps\n")
		
		        else:
		            fout.write(line)
		    fin.close()
		    fout.close()
		
		    check_call(['a2ensite', param+'.conf'],
		               stdout=open(os.devnull, 'wb'), stderr=STDOUT)
		    check_call(['service', 'apache2', 'reload'],
		               stdout=open(os.devnull, 'wb'), stderr=STDOUT)
		
		    # Modificamos el archivo /etc/hosts para asignar un dominio a una dirección ip
		    os.system('echo 192.168.122.241 '+param +
		              '   www.'+param+'.cdps>>/etc/hosts')
		    check_call(['mkdir', '/var/www/'+param],
		               stdout=open(os.devnull, 'wb'), stderr=STDOUT)
		    check_call(['cp', '/var/www/html/index.html', '/var/www/'+param +
		               '/index.html'], stdout=open(os.devnull, 'wb'), stderr=STDOUT)
		    print('Webhost añadido correctamente\n')
		    
		    # Añadimos un usuario con el grupo cdps y le asignamos un directorio
		    os.system('useradd '+param+' -s /bin/bash')
		    os.system('mkdir /home/'+param)
		    os.system('chown '+param+':'+param+' -R /home/'+param)
		    os.system('chmod 755 -R /home/'+param)
		    os.system('usermod -d /home/'+param)
		    print('Usuario con nombre '+param+' se ha añadido correctamente')
		    print('[+] Escribe la contraseña para el usuario creado')
		    os.system('passwd '+param)
		    
		
		else:
		    print("\nNo se ha pasado un parámetro al script\n")
		    print("Se debe pasar como parámetro el nombre del dominio\n")

