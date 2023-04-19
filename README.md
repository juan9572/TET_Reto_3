# TET_Reto_N3
> Repositorio para alojar el proyecto de realizar un despliegue de una infraestructura de una página web de Drupal utilizando una base de datos compartida y un sistema de archivos NFS. La página web se encuentra balanceada por un balanceador de carga y se accede a través de un dominio y protocolo HTTPS.
## Tabla de contenidos:
---

1.  [Información de la asignatura](#información-de-la-asignatura)

2. [Datos del estudiante](#datos-del-estudiante)

3. [Descripción y alcance del proyecto](#descripción-y-alcance-del-proyecto)

4. [Arquitectura de despliegue](#arquitectura-de-despliegue)

6. [Paso a paso del despliegue](#paso-a-paso-del-despliegue)
	 - [Pre-requisitos](#pre-requisitos)
	 - [Configurando Docker](#configurando-docker)
	 - [Base de datos de MARIADB](#base-de-datos-de-mariadb)
	 - [FileServer NFS](#fileserver-nfs)
	 - [Drupal](#drupal)
	 - [Dominio web](#dominio-web)
	 - [Balanceador de carga](#balanceador-de-carga)
7. [Referencias](#referencias) 

## Información de la asignatura
---

 -  **Nombre de la asignatura:** Tópicos especiales en telemática.
-   **Código de la asignatura:**  C2361-ST0263-4528
-   **Departamento:** Departamento de Informática y Sistemas (Universidad EAFIT).
-   **Nombre del profesor:** Juan Carlos Montoya Mendoza.
-  **Correo electrónico del docente:** __[jcmontoy@eafit.edu.co](mailto:jcmontoy@eafit.edu.co)__.

## Datos del estudiante
---

-   **Nombre del estudiante:** Juan Pablo Rincon Usma.
-  **Correo electrónico del estudiante:** __[jprinconu@eafit.edu.co](mailto:jprinconu@eafit.edu.co)__.

## Descripción y alcance del proyecto
---
El proyecto consiste en el despliegue de una página web de Drupal, con una base de datos compartida y un sistema de archivos NFS. Este despliegue se realizará mediante el uso de un balanceador de carga, lo que permitirá un mejor rendimiento y alta disponibilidad de la página. Además, se podrá acceder a la página web a través de un dominio y mediante HTTPS para garantizar la seguridad de la información.

El alcance del proyecto incluye la configuración de un servidor de base de datos MariaDB, donde se almacenarán los datos de la página web de Drupal. Además, se creará un servidor de archivos NFS que permitirá compartir los archivos de la página entre los servidores web de Drupal. Se utilizará un balanceador de carga para distribuir la carga de trabajo entre los servidores web de Drupal, lo que permitirá mejorar el rendimiento y garantizar la alta disponibilidad de la página. Por último, se configurará un servidor web de Apache con soporte para HTTPS y un certificado SSL, lo que garantizará la seguridad de la página web y permitirá el acceso a través de un dominio.


## Arquitectura de despliegue
---
![Arquitectura a desplegar](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Diagrama/Arquitectura.png)
La arquitectura de la solución desplegada se compone de tres capas. La primera capa es la capa de redireccionamiento, la cual implica la utilización de un balanceador de carga de Apache desplegados en Docker para distribuir las solicitudes HTTPS a la segunda capa. La segunda capa es la capa de servidores, que consiste en dos servidores de Drupal desplegados en contenedores Docker, que atienden solicitudes por el puerto 80. La tercera capa consta de dos servidores compartidos entre los servidores de Drupal, y aquí se encuentran los servicios compartidos como la base de datos MariaDB y el servidor de archivos de la página NFS. El balanceador de carga se encarga de distribuir las solicitudes a los servidores de Drupal, mientras que la capa de servidores y la capa compartida aseguran el correcto funcionamiento de los servicios requeridos por la aplicación web. Todo esto se encuentra desplegado en un ambiente de producción y se puede acceder mediante un dominio y HTTPS.
## Paso a paso del despliegue
---

### 1. Pre-requisitos

Cada uno de estos pre-requisitos es fundamental para asegurar el correcto funcionamiento de la solución desplegada, por lo que es importante llevar a cabo estos pasos antes de comenzar con el despliegue de la aplicación web.

 - 5 instancias EC2 en AWS, cada una usando Ubuntu 22.04 como SO y cada una con su propia dirección IP elástica.
- Configurar un proxy en cada instancia para permitir el acceso a internet.
- Abrir el puerto 80 en las instancias que servirán como servidores web de Drupal.
- Abrir el puerto 3306 en la instancia que servirá como base de datos de MariaDB.
- Abrir el puerto 2049 en la instancia que servirá como servidor NFS.
- Abrir los puertos 80 y 443 en la instancia que servirá como balanceador de carga de Apache.
Es importante destacar que aunque en este proyecto se está utilizando AWS, los pre-requisitos necesarios son similares independientemente de la plataforma en la que se despliegue la solución. En este caso, se requieren 5 instancias de Ubuntu 22.04 con direcciones IP elásticas y la configuración del proxy en cada una de ellas. 
Una vez tengamos listo estos requisitos, deberemos tener algo similar a esto:
![Instancias](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Requisitos/Instancias.png)
![Ip elasticas](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Requisitos/Ip%20elasticas.png)
![Proxy](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Requisitos/SecurityGroups.png)

### 2. Configurando Docker
Para instalar Docker en cada instancia, accedemos a cada una de ellas excepto a la del servidor de NFS, y ejecutamos los siguientes pasos. Es importante mencionar que estos pasos están basados en la documentación oficial de Docker para [Ubuntu](https://docs.docker.com/desktop/install/ubuntu/), pero pueden variar dependiendo del sistema operativo utilizado.

A continuación se presentan los pasos para instalar Docker en una instancia Ubuntu:
1. Desinstalar versiones antiguas de Docker usando el siguiente comando:
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```
2. Configurar el repositorio Ubuntu usando los siguientes comandos:
```bash
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg
```
3. Añadir la clave GPG oficial de Docker usando el siguiente comando:
```bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
4. Usar el siguiente comando para configurar el repositorio:
```bash
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  ```
5. Actualizar la lista de paquetes usando el siguiente comando:
```bash
sudo apt-get update
```
6. Si recibe un ERROR GPG al ejecutar `apt-get update`, use los siguientes comandos, sino se presenta el ERROR puede saltarse este paso: 
```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
sudo apt-get update
```
7. Instalar Docker Engine, containerd y Docker Compose usando el siguiente comando:
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
8. Verifique que la instalación de Docker Engine sea exitosa ejecutando la imagen hello-world:
```bash
sudo docker run hello-world
```
Ahora deberías tener Docker listo para ser ejecutado, empezaremos configurando desde los módulos menos dependientes a los más dependientes, es importante tener en cuenta que estos pasos son específicos para Ubuntu, pero los pasos para otras distribuciones de Linux son similares. Se recomienda seguir siempre la documentación oficial para la instalación de Docker en cualquier plataforma.

### 3. Base de datos de MARIADB
Se utilizará MariaDB como base de datos para la aplicación web Drupal. Se explicará cómo instalar MariaDB y cómo configurar las credenciales de acceso para la aplicación web.
1. Descarga la imagen oficial de MariaDB desde Docker Hub ejecutando el siguiente comando:
```bash
sudo docker pull mariadb
```
2. Crea un contenedor de Docker basado en la imagen de MariaDB que acabas de descargar, utilizando el siguiente comando: 
```bash
sudo docker run --name mydb -e MYSQL_ROOT_PASSWORD=password -p 3306:3306 -d --restart=always mariadb
```
3. Vamos a buscar las credenciales y crear la base de datos para nuestro drupal:
```bash
sudo docker exec -it mydb mysql -p
```
Cuando ingresemos haremos lo siguiente:
![ingresoDB](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/MariaDB/MariaDB-1.png)
![createDB](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/MariaDB/MariaDB-2.png)
![userDB](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/MariaDB/MariaDB-3.png)
El paso que acabamos de realizar fue acceder a la consola de MariaDB en el contenedor de la base de datos y crear una base de datos para nuestro sitio web Drupal. También hemos buscado las credenciales del usuario que interactúa con la base de datos.

### 4. FileServer NFS

La configuración del sistema de archivos de red (NFS) es esencial para garantizar que los servidores de Drupal puedan compartir los mismos archivos. NFS es una tecnología que permite compartir sistemas de archivos entre diferentes sistemas operativos a través de una red. En esta sección, se detallará cómo configurar un servidor NFS y cómo montarlo en los servidores de Drupal.

1. Instalamos el paquete **nfs-kernel-server** para obtener el servidor NFS:
```bash
sudo apt-get install nfs-kernel-server -y
```
2. Creamos el directorio compartido para el servidor NFS:
```bash
sudo mkdir /shared_folder
```
3. Configuramos el servidor NFS editando el archivo **/etc/exports** y añadiendo la siguiente línea al final del archivo:
```bash
sudo nano /etc/exports
```
`/shared_folder *(rw,sync,no_subtree_check)`
Se deberá ver algo así:
![nfs-exports](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Nfs/NFS-1.png)
4. Damos permisos a todos para editar en el directorio compartido:
```bash
sudo chmod 777 /shared_folder
```
5. Reiniciamos el servicio del servidor NFS para aplicar los cambios en la configuración:
```bash
sudo systemctl restart nfs-kernel-server
```
### 5. Drupal

En esta sección, explicaremos cómo configurar Drupal en nuestro entorno de AWS, utilizando las instancias EC2 que hemos creado previamente, así como el servidor de base de datos y el NFS.
1. Creamos una carpeta donde tendremos nuestros archivos de Docker con el siguiente comando:
```bash
sudo systemctl restart nfs-kernel-server
```
2. Creamos el archivo docker-compose.yml con el siguiente comando:
 ```bash
sudo nano docker-compose.yml
```
3. Se agrega el siguiente contenido al archivo [docker-compose.yml](https://github.com/juan9572/TET_Reto_3/blob/main/Files/Drupal/docker-compose.yml) para crear el contenedor de Drupal, configurando las credenciales de la base de datos de MariaDB y la conexión con el servidor NFS, cabe recalcar que en el apartado de **volumes** en **nfs-data** en la instrucción **o** se debe colocar la **ip del servidor de nuestro NFS** así como en **device** se coloca la carpeta que se creo en el **NFS**:
 ```yml
version: '3'
services:
  drupal:
    image: drupal:latest
    privileged: true
    ports:
      - "80:80"
    environment:
      - MYSQL_USER=drupal
      - MYSQL_PASSWORD=password
      - MYSQL_DATABASE=drupal
    volumes:
      - drupal-data:/var/www/html
      - nfs-data:/var/www/html/sites/default/nfs_share
    restart: always
volumes:
  drupal-data:
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=54.156.248.189,vers=4,soft
      device: ":/shared_folder"
```
4. Ejecutamos el contenedor de Docker con el siguiente comando:
 ```bash
sudo docker compose up -d
```
5. Accedemos al sitio web de Drupal a través del navegador, ingresando la dirección IP del servidor y el puerto 80.
![AccesDrupal](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Drupal/Drupal-1.png)
Seguiremos dándole continuar, continuar, continuar. Hasta que lleguemos al paso 4 "Configurar base de datos".
6. Configuramos la base de datos utilizando las credenciales obtenidas en la sección anterior.
![ConfigDB](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Drupal/Drupal-2.png)
Como se logra apreciar cuando estemos configurando la base de datos para nuestro Drupal, pondremos los datos de las credenciales que sacamos en el paso de base de datos, además de darle en opciones avanzadas y poner la ip de nuestra base de datos de MariaDB y el puerto 3306 por donde se conectará.
7. Continuamos con la instalación hasta que tengamos acceso al sitio web.
![DrupalHome](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Drupal/Drupal-3.png)
Deberemos ver algo así cuando terminemos la configuración del sitio.
8. Accedemos al contenedor de Docker de Drupal con el siguiente comando:
 ```bash
sudo docker exec -it drupal_drupal_1 bash
```
9. Instalamos el editor de texto nano con el siguiente comando:
 ```bash
apt-get update && apt-get install nano -y
```
10. Modificamos el archivo de configuración de Drupal con el siguiente comando:
 ```bash
nano /var/www/html/sites/default/settings.php
```
11. Buscamos la línea `$settings['file_public_path']` y la cambiamos a `$settings['file_public_path'] = 'sites/default/nfs_share';`
![changeSettings.php](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Drupal/Drupal-5.png)
12. Si revisamos nuestra página deberá verse algo así:
![drupalAfterChange](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Drupal/Drupal-6.png)
Esto ocurre debido a que cambiamos la ruta a la cual Drupal estaba accediendo a los archivos de configuración.
13. Para solucionar el problema de los archivos CSS y JS faltantes, desde el contenedor de Docker vamos a copiar los archivos de nuestra página de Drupal a nuestro servidor NFS con el siguiente comando:
 ```bash
cp -r /var/www/html/sites/default/files/* /var/www/html/sites/default/nfs_share
```
14. Verificamos que los archivos se hayan copiado correctamente.
Si revisamos nuestro server NFS deberán salir allí los archivos y además ahora nuestra página deberá verse así:
![DrupalNfsChange](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Drupal/Drupal-7.png)
Esto significa que ya esta listo el NFS.
15. Para configurar el resto de las instancias de Drupal, sigue prácticamente el mismo proceso que se realizó en los 14 pasos anteriores. Sin embargo, en el **paso 7**, debemos prestar atención a una variación. Una vez que ingresamos los datos para que la instancia de Drupal pueda conectarse a la base de datos, se mostrará una nueva página. En esta página, en lugar de seguir con la parte de "Instalar sitio", veremos una pantalla que dice **Drupal ya está instalado**, algo así.
![DrupalSecondInstance](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/Drupal/Drupal-4.png)
Le daremos **ver sitio existente** y podemos seguir con los demás pasos para poder terminar de instalar y configurar nuestro Drupal.

### 6. Dominio web

Configuraremos un nombre de dominio con un proveedor de DNS y conectaremos ese dominio a nuestra instancia de balanceador de carga que nos redirigirá a los servers de Drupal.

Para esto seleccionamos un proveedor de dominios web, el que queramos en mi caso yo voy a usar **"Namecheap"** y voy a comprar el dominio **"reto3-drupal-distribuido.cfd"**
![Domain](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/SSL%20y%20Https/DOMAIN-1.png)
Cuando tengamos nuestro dominio, procederemos a configurar los DNS de este, para ello buscaremos como poder modificar los registros del DNS, en el caso de **Namecheap**, iremos a la sección __Domain List__ y le damos **"Manage"** a nuestro dominio.
![ManageDomain](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/SSL%20y%20Https/DOMAIN-2.png)
Allí encontramos un sección que dice **"Advance DNS**", cuando entramos allí veremos los registros del DNS (Records DNS), veremos que podemos agregar nuevos Records, llenando los siguientes campos
![DNSRecords](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/SSL%20y%20Https/DOMAIN-3.png)
Vamos a eliminar todos los que estén activos y crearemos 2 tipos de Records:
| Type | Host | Value | TTL |
|--|--|--|--|
| A Record | @ | 3.215.32.87 | Automatic
| CNAME Record | www | reto3-drupal-distribuido.cdf | Automatic |

Cabe recalcar que en el A Record en el campo **Value** se coloca la ip de nuestro balanceador de carga.
 ![RecordsDNS](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/SSL%20y%20Https/DOMAIN-4.png)

### 7. Balanceador de carga

 En esta sección, vamos a configurar un balanceador de carga para distribuir el tráfico de nuestro sitio web entre los servidores que hemos configurado previamente.
 
1.  Creamos una carpeta donde guardaremos todos los datos importantes para nuestro contendor:
 ```bash
sudo mkdir loadBalancer
```
2.  En la carpeta loadBalancer crearemos 2 archivos:
[docker-compose.yml](https://github.com/juan9572/TET_Reto_3/blob/main/Files/LoadBalancer/docker-compose.yml)
 ```bash
sudo nano docker-compose.yml
```
Con el siguiente contenido:
 ```yml
version: "3"

services:
  apache:
    image: httpd:2.4
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./balancer.conf:/usr/local/apache2/conf/balancer.conf
      - /ssl-certbot:/usr/local/apache2/ssl
    restart: always
```
Y creamos el otro:
[balancer.conf](https://github.com/juan9572/TET_Reto_3/blob/main/Files/LoadBalancer/balancer.conf)
 ```bash
sudo nano balancer.conf
```
Con el siguiente contenido:
 ```bash
ServerName reto3-drupal-distribuido.cfd:443

AddType text/css .css

LoadModule ssl_module modules/mod_ssl.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
LoadModule rewrite_module modules/mod_rewrite.so
LoadModule slotmem_shm_module modules/mod_slotmem_shm.so

Listen 443

<VirtualHost *:80>
   Redirect permanent / https://reto3-drupal-distribuido.cfd/
</VirtualHost>

<VirtualHost *:443>
  SSLEngine on
  SSLCertificateFile "/usr/local/apache2/ssl/live/reto3-drupal-distribuido.cfd/fullchain.pem"
  SSLCertificateKeyFile "/usr/local/apache2/ssl/live/reto3-drupal-distribuido.cfd/privkey.pem"

  <Proxy "balancer://drupal-cluster">
    BalancerMember "http://52.71.73.98:80" route=drupal1
    BalancerMember "http://18.235.100.177:80" route=drupal2
    ProxySet stickysession=ROUTEID
  </Proxy>

  ProxyPass "/balancer-manager" !
  ProxyPass "/" "balancer://drupal-cluster/" stickysession=ROUTEID
  ProxyPassReverse "/" "balancer://drupal-cluster/"

  <Location "/balancer-manager">
    SetHandler balancer-manager
    Require host localhost
  </Location>

   <IfModule mod_headers.c>
     Header always edit Set-Cookie (.*) "$1; HttpOnly; Secure; SameSite=Strict"
   </IfModule>
</VirtualHost>
```
Cabe recalcar que en las siguientes líneas se deben poner correspondiente a tus datos:
Las siguientes líneas deberás editarlas en base en tu dominio web.
- ServerName reto3-drupal-distribuido.cfd:443
- Redirect permanent / https://reto3-drupal-distribuido.cfd/

Las siguientes líneas deberás editarlas en base en las direcciones ip's de tus servidores de Drupal.
- BalancerMember "http://52.71.73.98:80" route=drupal1
- BalancerMember "http://18.235.100.177:80" route=drupal2

Las siguientes líneas deberás editarlas en base donde se haya guardado tus certificados SSL que genero Certbot.
- SSLCertificateFile "/usr/local/apache2/ssl/live/reto3-drupal-distribuido.cfd/fullchain.pem" 
- SSLCertificateKeyFile "/usr/local/apache2/ssl/live/reto3-drupal-distribuido.cfd/privkey.pem"
3.  Creamos una carpeta para simular un volumen de docker para guardar el SSL:
 ```bash
sudo mkdir /ssl-certbot
```
4.  Pedimos nuestro certificado SSL y se deberá guardar en el volumen anteriormente creado:
 ```bash
sudo docker run -it --name certbot -v /ssl-certbot:/etc/letsencrypt -p 80:80 -p 443:443 certbot/certbot certonly
```
Seguimos los pasos para pedir el certificado, los cuales se deben ver algo así:
![SSLCertBot](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/SSL%20y%20Https/SSL-1.png)
5.  Ejecutamos nuestro contenedor de docker:
 ```bash
sudo docker compose up -d
```
6.  Vamos al contenedor de docker:
 ```bash
sudo docker exec -it loadbalancer-apache-1 bash
```
7.  Instalamos nano:
 ```bash
apt-get update && apt-get install nano -y
```
8.  Editamos el archivo de configuración de apache:
 ```bash
nano /usr/local/apache2/conf/httpd.conf
```
9.  Al final del archivo ponemos:
 ```bash
Include conf/balancer.conf
```
10. Salimos de contendor y lo reiniciamos para que apache aplique los cambios
 ```bash
exit
sudo docker restart loadbalancer-apache-1
```
11. Revisamos nuestro balanceador de carga
![goingToIp](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/SSL%20y%20Https/SSL-2.png)
![DrupalSSL](https://raw.githubusercontent.com/juan9572/TET_Reto_3/main/Img/SSL%20y%20Https/SSL-4.png)
Ya tenemos configurado el SSL para nuestro balanceador de carga y ya redirige a los servidores de Drupal.

En conclusión, hemos logrado implementar una arquitectura distribuida de Drupal en dos servidores diferentes utilizando Docker y Docker Compose. Gracias a esto, hemos mejorado la escalabilidad y disponibilidad de nuestro sitio web, permitiendo que más usuarios puedan acceder a él sin afectar su rendimiento.

Además, hemos agregado una capa de seguridad importante al utilizar SSL para cifrar las comunicaciones entre el usuario y el servidor, gracias al certificado SSL generado con Certbot y utilizado en nuestro balanceador de carga.

Este proyecto nos ha permitido aprender sobre diferentes tecnologías, desde la creación de contenedores de Docker y la configuración de un balanceador de carga hasta la implementación de SSL en nuestro sitio web, lo que nos brinda una base sólida para seguir aprendiendo y mejorando nuestras habilidades en el mundo de la tecnología.
## Referencias
---

- [How to Configure Apache Load Balancer](https://www.inmotionhosting.com/support/server/apache/apache-load-balancer/)
- [Get Certbot (Docker)](https://eff-certbot.readthedocs.io/en/stable/install.html#alternative-1-docker)
- [Setup a NFS Server With Docker](https://blog.ruanbekker.com/blog/2020/09/20/setup-a-nfs-server-with-docker/)
- [Install Docker Desktop on Ubuntu](https://docs.docker.com/desktop/install/ubuntu/)
