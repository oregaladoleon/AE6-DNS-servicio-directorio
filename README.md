# UT6 - Práctica evaluable, DNS y servicio de directorio para el despliegue.

> Autor: Óscar Regalado León  
> Fecha: Noviembre 2026   
> Autoría: imagen oficial de Ubuntu — https://hub.docker.com/_/ubuntu

## Objetivo
Poner nombre y autenticación al despliegue: que **www.dpl.local** y **app.dpl.local** sean resueltos por tu propio **DNS** (BIND9), que el directorio (slapd) **autentique** a sus usuarios vía **LDAP** y que la aplicación sea **accesible y autenticable** por nombre, con un **informe de diagnóstico** de la cadena completa.

## Índice

- [Instalación](#Instalación)
- [Configuración](#configuración)
- [Comprobación](#comprobación)
- [Problemas encontrados](#problemas-encontrados)
- [Enlace al repositorio remoto](https://github.com/oregaladoleon/AE6-DNS-servicio-directorio.git)

## Instalación
1. Creamos un nuevo repositorio  en la web de **Github** y lo clonamos en nuestro equipo anfitrión dentro del directorio de trabajo **~/dpl/AE6**:

2. Levantamos los contenedores existentes:
~~~bash
sudo docker ps -a

sudo docker start dpl-lab

sudo docker start dpl-cliente

cd ../AE4/

chmod +x deploy.sh

sudo ./deploy.sh 
~~~

3.Comprobamos las IPs de los contenedores debido a que, cada vez que recreamos los contenedores las IPs cambian:
~~~bash
sudo docker inspect dpl-lab --format '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}: {{$v.IPAddress}}
{{end}}'
bridge: 172.17.0.3
dpl-app_default: 172.18.0.7
dpl-net: 172.19.0.3

sudo docker inspect dpl-cliente --format '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}: {{$v.IPAddress}}
{{end}}'
bridge: 172.17.0.2
dpl-app_default: 172.18.0.6
dpl-net: 172.19.0.2

sudo docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dpl-nginx
172.18.0.3

sudo docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dpl-django
172.18.0.2
~~~

4. Instalamos **bind9** en **dpl-lab**:

~~~bash
docker exec dpl-lab bash -c "apt-get update && apt-get install -y bind9"

docker exec dpl-lab named -v  
BIND 9.18.39-0ubuntu0.24.04.9-Ubuntu (Extended Support Version) <id:>
~~~

## Configuración

5. Creación del **fichero de zona** en **dpl-lab**:

~~~bash
docker exec dpl-lab touch /etc/bind/db.dpl.local  

docker exec -it dpl-lab bash   

 # nano /etc/bind/db.dpl.local
~~~
El fichero tendrá la siguiente configuración:

~~~
$TTL 3600
$ORIGIN dpl.local.
@       IN  SOA ns1.dpl.local. admin.dpl.local. (
        2024110504 ; serial    #incrementamos en una unidad al último serial realizado 
        3600       ; refresh
        900        ; retry
        604800     ; expire
        86400 )    ; minimum TTL
@       IN  NS  ns1.dpl.local.
ns1     IN  A   172.19.0.3    #IP dpl-lab
www     IN  A   172.18.0.3    #IP dpl-nginx
app     IN  A   172.18.0.2    #IP dpl-django
~~~

6. Declarar la zona y validar.

~~~bash
nano /etc/bind/named.conf.local
~~~

Añadimos la siguiente configuración al fichero:

~~~
zone "dpl.local" {
    type master;
    file "/etc/bind/db.dpl.local";
};
~~~

Validamos la configuración y el fichero de zona:

~~~bash
docker exec dpl-lab named-checkconf

docker exec dpl-lab named-checkzone dpl.local /etc/bind/db.dpl.local
zone dpl.local/IN: loaded serial 2024110504
OK
~~~


