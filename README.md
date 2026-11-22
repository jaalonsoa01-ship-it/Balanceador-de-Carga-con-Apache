# Balanceador-de-Carga-con-Apache

Cree las 4 maquinas usando Debian-12 Bookworm, 1 Balanceador de Carga llamado JoaquinAlonsoAdamsBalanceador, 2 servidores web en Backend (JoaquinAlonsoAdamsWeb1 y JoaquinAlonsoAdamsWeb2) y Un servidor de BBDD llamado JoaquinAlonsoAdamsBD.

Nota 1: Para facilitarme los procesos de configuración correspondientes, opte por usar putty, asignándole un reenvio de puerto para ssh al adaptador NAT. Porque me permite copiar el código desde el documento, de modo que los puertos serian: 2222 para balanceador, 2223 para BD, 2224 para Web1 y 2225 para Web2

Nota 2: Relacionado a lo anterior, hay que tener en cuenta que todos los comandos puestos con sudo, siempre se tienen que usar si configuramos desde la maquina virtual de forma directa, pero como el usuario original (joaquinub) no es parte de los sudoers, me veo en la obligación de entrar como root, con el fin de poder realizar la actividad.

Establecer las direcciones IP dentro de las 4 maquinas, como voy a entrar como root, no necesito sudo:

nano /etc/network/interfaces 

***Interfaz NAT (Internet)***
```
auto enp0s3
iface enp0s3 inet dhcp
```
***Interfaz Red Interna (IP Estática)***
```
auto enp0s8
iface enp0s8 inet static
    address 192.168.56.10  
    netmask 255.255.255.0
```

Modificaremos la address, usando los siguientes valores para cada uno: .10 para Balanceador, .21 para Web1, .22 para Web2, .30 para DB

sudo systemctl restart networking

Configurando a la maquina WEB1:

Debo instalar el servidor Apache y el cliente de MariaDB para conectarme con la base de datos
```
sudo apt update && sudo apt install apache2 mariadb-client -y
```
Una vez instalado procedo a crear una pagina web sencilla para así asignarla a web1

```
sudo nano /var/www/html/index.html
```

La pagina web en cuestion:
```
<!DOCTYPE html>
<html>
<head><title>JoaquinAlonsoAdamsWeb1</title></head>
<body>
    <h1>Respuesta desde JoaquinAlonsoAdamsWeb1 (Backend 1)</h1>
    <p>Servidor operativo en la Capa 2 (Red Privada).</p>
    <p>Si lees esto, es porque funciona</p>
</body>
</html>
```
### Configurando a la maquina WEB2:

Debo instalar el servidor Apache y el cliente de MariaDB para conectarme con la base de datos
```
apt update && apt install apache2 mariadb-client -y
```
Una vez instalado procedo a crear otra pagina web sencilla con el fin de asignarla a web2
```
sudo nano /var/www/html/index.html
```
La pagina web en cuestion
```
<!DOCTYPE html>
<html>
<head><title>JoaquinAlonsoAdamsWeb2</title></head>
<body>
    <h1>Respuesta desde JoaquinAlonsoAdamsWeb2 (Backend 1)</h1>
    <p>Servidor operativo en la Capa 2 (Red Privada).</p>
    <p>Si lees esto, es porque funciona</p>
</body>
</html>
```
Configurando el servidor BBDD:
```
apt update && apt install mariadb-server -y
```
Con eso instalamos el servidor de MariaDB, a continuación, configuraremos el archivo de red de MariaDB para permitir conexiones de la red privada.
```
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
y cambia bind-address = 127.0.0.1 por bind-address = 0.0.0.0 

Después de esto creamos una base de datos y usuario correspondiente
```
sudo mariadb -u root
```

```
CREATE DATABASE practica;
CREATE USER 'joaquin'@'192.168.56.%' IDENTIFIED BY 'mariadb';
GRANT ALL PRIVILEGES ON practica.* TO 'joaquin'@'192.168.56.%';
FLUSH PRIVILEGES;
EXIT;
```

Reiniciamos el servicio con sudo systemctl restart mariadb, ya tenemos configurado el servidor base de datos. Sin embargo, falta una maquina virtual por modificar
```
mysql -h 192.168.56.30 -u joaquin -p con este comando comprobamos que funciona el servidor de base de datos
```
Configurando balanceador de carga:

Instalar apache 
```
apt update && apt install apache2 -y
```
Habilitar modulos proxy
```
sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests
nano ~/.bashrc
```
agregar esta linea
```
export PATH=$PATH:/usr/sbin
```
Ejecutar este comando para actualizar el PATH source ~/.bashrc

Se instalaron los modulos si todo va bien, para comprobar que funciono, usaremos:
```
systemctl restart apache2
```
Para el usuario final, no sabe que detrás de la pagina web dado por el balanceador, hay varios servidores y la base de datos, siendo que el mismo balanceador actúa como proxy inverso, la primera linea de defensa para evitar ataques a los servidores web 1 y web 2, mucho menos que accedan a la base de datos.

Ahora toca lo más importante y tedioso, configurar el archivo de configuración de apache (000-default.conf) para que este funcione
```
sudo nano /etc/apache2/sites-available/000-default.conf
```

Esta será la configuración que estaré usando dentro del archivo
```
<VirtualHost *:80>
    ServerName localhost
    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
    <Proxy balancer://mycluster>
	#La IP de Web1
        BalancerMember http://192.168.56.21 route=web1
	#La IP de Web2
        BalancerMember http://192.168.56.22 route=web2
        ProxySet lbmethod=byrequests
    </Proxy>
</VirtualHost>
```
Hay que validar que no haya errores en el archivo de configuración, el mas mínimo error lo puede dañar

El comando siguiente sirve para probar las capacidades del apache2
```
sudo apache2ctl configtest
```
Como se ha realizado una configuración a apache, siempre hay que reiniciar el apache
```
sudo systemctl restart apache2
```
Probar el balanceador de carga, debo usar el navegador conectándome con localhost y usar el reenvio de puertos (8080), cuando entramos, intercambiara entre web 1 y web 2 para evitar que se sobrecarge la pagina, sin embargo, su conexión no es segura ya que se conecta mediante http://, por lo tanto necesitaremos obtener la certificación

Instalar un certificado valido con certbot 
```
sudo apt install certbot python3-certbot-apache -y
```
Despues de instalar, volvemos a reiniciar apache

El error de conexión segura (ERR_SSL_PROTOCOL_ERROR o similar) al intentar entrar por https://localhost:8080 se debe a dos motivos técnicos principales en mi entorno local con VirtualBox:

El puerto de red no coincide: El reenvío de puertos que configuraste en VirtualBox mapea el puerto anfitrión 8080 hacia el puerto 80 (HTTP) de la máquina virtual, no al puerto 443 (HTTPS). Por lo tanto, estás intentando hablar por HTTPS en un puerto configurado para tráfico plano HTTP.

Limitación de Certbot: Certbot y Let's Encrypt no pueden emitir certificados SSL válidos para localhost ni para IPs privadas, ya que requieren un dominio público accesible desde Internet para validar la seguridad.

### Crear la lista de control de acceso:

Configuración para el balancadr de cargas:
```
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

Configurar JoaquinAlonsoAdamsWeb1 & JoaquinAlonsoAdamsWeb2:
```
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.56.10 to any port 80 proto tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

Configurar JoaquinAlonsoAdamsBD:
```
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.56.21 to any port 3306 proto tcp
sudo ufw allow from 192.168.56.22 to any port 3306 proto tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

Capa 1 (Balanceador de Carga): Es la única expuesta al exterior. Debe permitir tráfico público de Internet en los puertos web (80 para HTTP y 443 para HTTPS), además del puerto de administración (22 para SSH).

Capa 2 (Servidores Web en Backend): Deben estar completamente protegidos del exterior. Su grupo de seguridad bloquea el tráfico de Internet y solo permite peticiones HTTP (80) procedentes exclusivamente de la IP del balanceador.

Capa 3 (Base de Datos): Es el nivel más crítico y aislado. Bloquea cualquier conexión general y solo permite tráfico en su puerto específico (por ejemplo, el 3306 de MariaDB) procedente exclusivamente de los servidores web de la Capa 2.

Le añadiremos un certificado DNS mediante DuckDNS, como podran ver en la siguiente imagenes:


