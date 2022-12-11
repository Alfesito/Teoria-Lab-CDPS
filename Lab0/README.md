# PRÁCTICA 0: Comandos básicos de Linux
Recuerda que usando man <comando>, puede ver instrucciones más detalladas del comando en cuestión.

## 1. Comandos imprescindibles

	$ history -c
	$ pwd
	$ ls -l
	$ mkdir dirDatos
	$ cd dirDatos
	$ touch datos
	$ cat datos
	$ wc -lc datos -> Contar palabras o ver cuánto ocupa el archivo
	$ cp datos ejemplos/ej1.c
	$ diff datos ejemplos/ej1.c -> Mira las diferencias entre los dos archivos
	$ mv ejemplos/ej1.c ejemplos/ej2.java
	$ ln -s ../otro dir/cosas/ejercicios/ej2.txt ej2_s -> Como un enlace directo en Windows
	$ rmdir mi dirDatos -> Elimina directorio
	$ rm -r dirDatos -> Elimina recursivamente
	
## 2. Redireccionamiento de salidas

	$ ls -l > datos -> Si había algo en el archivo, lo borra
	$ echo 'hola' >> datos -> Empieza a escribir en la última línea
	
## 3. Gestión de permisos
	
	$ chmod <permisos> <nf>
	
Los permisos son rwx, se pueden añadir de forma hexadecimal o con +rwx.

## 4. Búsqueda y ordenación en el sistema de ficheros

	$ df -h -> Conocer el espacio disponible
	$ find / -type f -name *.txt 2>/dev/null
	$ grep -i "HOLA" -> Case insensitive
	$ grep -E "thm|tryhackme|200"
	$ grep -r "helloword" -> Recirsive
	$ sort saludos -> Ordena el contenido de saludos
	
## 5. Tuberías (pipes)

	$ cat gestiona-pc1.json | grep num_serv | awk '{print$NF}'
	
## 6. Gestión de procesos

	$ ps -> Información de los procesos que se est ́an ejecutando
	$ kill <pid> -> mata un proceso
	


