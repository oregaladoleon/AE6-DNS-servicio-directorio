# UT6 - Práctica evaluable, DNS y servicio de directorio para el despliegue.

> Autor: Óscar Regalado León  
> Fecha: Noviembre 2026   
> Autoría: imagen oficial de Ubuntu — https://hub.docker.com/_/ubuntu

## Objetivo
Poner nombre y autenticación al despliegue: que **www.dpl.local** y **app.dpl.local** sean resueltos por tu propio **DNS** (BIND9), que el directorio (slapd) **autentique** a sus usuarios vía **LDAP** y que la aplicación sea **accesible y autenticable** por nombre, con un **informe de diagnóstico** de la cadena completa.

~~~mermaid
graph TD
    subgraph dpl-net [Red Docker: dpl-net]
        ClientNet[dpl-cliente <br/> IP: 172.19.0.2]
        DNS[dpl-lab / BIND9 <br/> IP: 172.19.0.3 :53]
    end

    subgraph dpl-app_default [Red Docker: dpl-app_default]
        ClientApp[dpl-cliente <br/> IP: 172.18.0.6]
        Nginx[dpl-nginx <br/> IP: 172.18.0.3]
        Django[dpl-django <br/> IP: 172.18.0.2 :8000]
        LDAP[dpl-lab / OpenLDAP <br/> IP: 172.18.0.7 :389]
    end

    ClientNet -->|1. Consulta DNS dpl.local| DNS
    ClientApp -->|2. HTTP / curl www.dpl.local| Nginx
    ClientApp -->|3. HTTP / curl app.dpl.local:8000| Django
    Django -->|4. Bind LDAP /api/login| LDAP
~~~
Resumen de IPs del entorno:
* **dpl-lab**: 
  * 172.19.0.3 (en la red dpl-net para atender las peticiones DNS de BIND9 en el puerto 53).
  * 172.18.0.7 (en la red dpl-app_default para atender las consultas de OpenLDAP en el puerto 389 desde Django).
* **dpl-cliente**: 
  * 172.19.0.2 (en dpl-net).
  * 172.18.0.6 (en dpl-app_default para alcanzar los servicios web y de aplicación por nombre).
* **dpl-nginx**: 172.18.0.3 (Proxy inverso web para www.dpl.local).
* **dpl-django**: 172.18.0.2 (Servidor de aplicaciones en el puerto 8000 para app.dpl.local).

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

3.Comprobamos las IPs de los contenedores, debido a que, cada vez que recreamos los contenedores las IPs cambian:
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

5. Creación del **fichero de zona** en **dpl-lab**. Debido a que la resolución de nombres de dominio privado **dpl.local** la vamos a gestionar de forma **autoritativa** con **BIND9** que ejecutaremos en el contenedor **dpl-lab**, debemos configurar el **archivo de zona db.dpl.local** que ya habíamos creado en las prácticas guiadas.

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
Donde:
- **SOA** (Start of Authority): Es el registro obligatorio que encabeza cualquier zona DNS autoritativa. Define la autoridad suprema del servidor sobre el dominio. Incluye parámetros críticos como el servidor DNS primario (ns1.dpl.local), el correo de contacto del administrador (admin.dpl.local), el número de serie (serial) —que debe incrementarse en cada modificación para que los clientes reconozcan los cambios— y los temporizadores de transferencia y caché (refresh, retry, expire y minimum TTL).
- **NS** (Name Server): Indica qué servidores están autorizados para responder a consultas sobre la zona dpl.local. En nuestra arquitectura, señala que ns1.dpl.local es el encargado de resolver los registros de este dominio.
- **A** (Address IPv4): Asocia un nombre de host (un subdominio o etiqueta) directamente con una dirección IP de la red de contenedores. Permite traducir nombres legibles por humanos a las IPs numéricas de Docker.

Por otro lado, en una arquitectura web moderna multicontenedor, la separación de responsabilidades se refleja a nivel de red y de nombres mediante distintos puntos de entrada:

- **www.dpl.local** (Proxy Inverso - Nginx): Este registro **A** apunta a la dirección IP del contenedor de Nginx. Nginx actúa como la pasarela web principal (proxy inverso), encargándose de servir los contenidos estáticos (como el build de Vite), gestionar las conexiones de los usuarios y derivar las peticiones hacia el backend. Es la cara pública y accesible del servicio web.

- **app.dpl.local** (Servidor de Aplicaciones - Django / Gunicorn): Este registro **A** apunta de manera independiente a la IP del contenedor de la aplicación de backend, que escucha habitualmente en el puerto 8000.

Esta diferenciación es necesaria porque desacopla la lógica del servidor web estático del motor de la aplicación. Permite que los servicios internos, scripts de prueba (como los tests E2E o consultas directas a la API) y herramientas de desarrollo alcancen el servidor de aplicaciones por su propio nombre estable, independientemente de que el proxy Nginx esté intermediando o no, facilitando además la escalabilidad horizontal en despliegues reales donde el backend y el servidor web residen en contenedores o nodos separados.

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

7. Lanzar named sin systemd

Debido a que los contenedores no ejecutan systemd, ejecutamos named para ejecutarlo en **foreground**:

~~~bash
docker exec -d dpl-lab named -g   #Se lanza el contenedor en ejecución simulando systemd

docker exec dpl-lab pgrep -a named
30 named -g

docker exec dpl-lab ss -lunp | grep ':53'
UNCONN 0      0         172.19.0.3:53         0.0.0.0:*    users:(("named",pid=30,fd=98)) 
UNCONN 0      0         172.19.0.3:53         0.0.0.0:*    users:(("named",pid=30,fd=101))
UNCONN 0      0         172.19.0.3:53         0.0.0.0:*    users:(("named",pid=30,fd=97)) 
...
~~~

8. Apuntar el **cliente** a la DNS de **dpl-lab** y a la red de la aplicación

~~~bash
docker exec dpl-cliente bash -c "apt-get update && apt-get install -y dnsutils iputils-ping"

docker exec dpl-cliente bash -c "echo 'nameserver 172.19.0.3' > /etc/resolv.conf && echo 'nameserver 1.1.1.1' >> /etc/resolv.conf && echo 'search dpl.local' >> /etc/resolv.conf"  

docker exec dpl-cliente cat /etc/resolv.conf
nameserver 172.19.0.3  #IP dpl-lab
nameserver 1.1.1.1
search dpl.local
~~~

De las prácticas anteriores, debemos confirmar que tanto **dpl-lab** como **dpl-cliente** estén conectadas a la red de docker generada por la aplicación.

~~~bash
sudo docker network ls
NETWORK ID     NAME              DRIVER    SCOPE
570e11cd99ac   bridge            bridge    local
c2436f7eb617   dpl-app_default   bridge    local
8ebb983ae995   dpl-net           bridge    local
85e43dd10ac9   host              host      local
fcf9e9a6cd59   none              null      local

sudo docker network connect dpl-app_default dpl-cliente
~~~

Hasta este punto hemos configurado el **servidor DNS** con BIND9. Ahora procedemos a configurar el **servidor de directorio**.

9. Instalamos el directorio dentro de dpl-lab:

~~~bash
docker exec dpl-lab bash -c "
  echo 'slapd slapd/domain string dpl.local' | debconf-set-selections &&
  echo 'slapd shared/organization string DPL' | debconf-set-selections &&
  echo 'slapd slapd/password1 password CAMBIAR_ESTO' | debconf-set-selections &&
  echo 'slapd slapd/password2 password CAMBIAR_ESTO' | debconf-set-selections &&
  echo 'slapd slapd/backend select MDB' | debconf-set-selections &&
  echo 'slapd slapd/purge_database boolean false' | debconf-set-selections &&
  echo 'slapd slapd/move_old_database boolean true' | debconf-set-selections &&
  apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y slapd ldap-utils"
~~~

11. Lanzamos **slapd** sin systemd y verificamos la base:

Debido a que los contenedores no tienen systemd, **slapd** se lanza en foreground:

~~~bash
docker exec -d dpl-lab slapd -d 0 -h ldap:///

docker exec dpl-lab pgrep -a slapd
215 slapd -d 0 -h ldap:///

docker exec dpl-lab ss -lntp | grep ':389'
LISTEN 0      2048         0.0.0.0:389        0.0.0.0:*    users:(("slapd",pid=215,fd=6)) 
LISTEN 0      2048            [::]:389           [::]:*    users:(("slapd",pid=215,fd=7)) 
~~~

Verificamos la base:

~~~bash
docker exec dpl-lab ldapsearch -x -LLL -b "dc=dpl,dc=local" "objectClass=*" dn
dn: dc=dpl,dc=local

dn: ou=usuarios,dc=dpl,dc=local

dn: uid=alumno,ou=usuarios,dc=dpl,dc=local

dn: uid=maria,ou=usuarios,dc=dpl,dc=local
~~~

12. Preparar el cliente LDAP.

Instalamos las herramientas en **dpl-cliente** que representa otra máquina que autentica contra el directorio.

~~~bash
docker exec dpl-cliente bash -c "apt-get update && apt-get install -y ldap-utils"

#La IP de **dpl-lab** es 172.19.0.3
docker exec dpl-cliente ldapsearch -x -LLL -H ldap://172.19.0.3 -b "dc=dpl,dc=local" "objectClass=*" dn
dn: dc=dpl,dc=local

dn: ou=usuarios,dc=dpl,dc=local

dn: uid=alumno,ou=usuarios,dc=dpl,dc=local

dn: uid=maria,ou=usuarios,dc=dpl,dc=local
~~~

13. Creamos la unidad organizativa y los usuarios (LDIF).

Dentro del directorio del proyecto **~/dpl/AE6/ldif** creamos el fichero **usuarios.ldif**

~~~bash
mkdir -p ldif && touch ~/AE6/ldif/usuarios.ldif
~~~

En el fichero creamos dos usuarios:

~~~
dn: ou=usuarios,dc=dpl,dc=local
objectClass: organizationalUnit
ou: usuarios

dn: uid=alumno,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: alumno
cn: alumno
sn: Alumno DPL
mail: alumno@dpl.local
userPassword: CAMBIAR_ESTO

dn: uid=maria,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: maria
cn: maria
sn: Maria DPL
mail: maria@dpl.local
userPassword: CAMBIAR_ESTO
~~~

Copiamos el fichero a **dpl-cliente** y lo cargamos con el administrador:

~~~bash
docker cp ~/dpl/AE6/ldif/usuarios.ldif dpl-cliente:/tmp/usuarios.ldif

docker exec -it dpl-cliente ldapadd -x -H ldap://172.19.0.3 -D "cn=admin,dc=dpl,dc=local" -W -f /tmp/usuarios.ldif
Enter LDAP Password: 
adding new entry "ou=usuarios,dc=dpl,dc=local"
ldap_add: Already exists (68)
~~~

14. Autenticamos contra el directorio

Desde **dpl-cliente** comprobamos las credenciales de alumno y Maria.

~~~bash
docker exec -it dpl-cliente ldapwhoami -x -H ldap://172.19.0.3 -D "uid=alumno,ou=usuarios,dc=dpl,dc=local" -W
dn:uid=alumno,ou=usuarios,dc=dpl,dc=local

docker exec -it dpl-cliente ldapwhoami -x -H ldap://172.19.0.3 -D "uid=maria,ou=usuarios,dc=dpl,dc=local" -W
dn:uid=maria,ou=usuarios,dc=dpl,dc=local
~~~

15. Consultamos la entrada (equivalente a "buscar quién es")

~~~bash
docker exec dpl-cliente ldapsearch -x -LLL -H ldap://172.19.0.3 -b "ou=usuarios,dc=dpl,dc=local" "(uid=alumno)" cn mail
dn: uid=alumno,ou=usuarios,dc=dpl,dc=local
cn: alumno
mail: alumno@dpl.local
~~~

16. Autenticamos la aplicación Web contra el directorio

La aplicación Web de ejemplo (la de la UT4) trae el endpoint /api/login, que hace el bind LDAP con el usuario y la contraseña recibidos. Se activa con variables de entorno, sin tocar el código:

~~~bash
export LDAP_HOST="ldap://172.18.0.7"     # URL del directorio (obligatoria)
export LDAP_BASE_DN="ou=usuarios,dc=dpl,dc=local"
export LDAP_USER_FILTER="(uid=%s)"
~~~

Para que el contenedor de la aplicación alcance a **dpl-lab**, ambos deben compartir red; conectamos **dpl-lab** a la red del proyecto 

~~~bash
docker network ls 
NETWORK ID     NAME              DRIVER    SCOPE
55e135e5f098   bridge            bridge    local
c2436f7eb617   dpl-app_default   bridge    local
8ebb983ae995   dpl-net           bridge    local
85e43dd10ac9   host              host      local
fcf9e9a6cd59   none              null      local

docker network connect dpl-app_default dpl-lab

cd ~/dpl/ae4 && docker compose up -d      # recrea la app con LDAP_HOST exportado
[+] up 4/4
 ✔ Container dpl-pgadmin Running                                                                                      0.0s
 ✔ Container dpl-django  Healthy                                                                                      1.1s
 ✔ Container dpl-nginx   Running                                                                                      0.0s
 ✔ Container dpl-db      Healthy 
~~~


~~~bash
curl -s -X POST -d "usuario=alumno&password=CAMBIAR_ESTO" http://localhost:8000/api/login
{"autenticado": true, "usuario": "alumno", "perfil": {"cn": "alumno", "mail": "alumno@dpl.local"}}

curl -s -o /dev/null -w '%{http_code}\n' -X POST -d "usuario=alumno&password=incorrecta" http://localhost:8000/api/login
401
~~~
Debido a problemas al realizar las comprobaciones con curl, he tenido que acceder a AE4/compose.yml y descomentar las líneas de las variables de entorno.
~~~
app:
    build: ./app               # contexto app/ (usa a>
    container_name: dpl-django
    environment:
      # Configuración por variables de entorno (sin c>
      APP_MENSAJE: "Hola desde el servidor de aplicac>
      APP_DATA_DIR: /data
      # Capa de datos: la app usa PostgreSQL si DB_HO>
      DB_HOST: db
      DB_NAME: travelroad
      DB_USER: alumno
      DB_PASSWORD: ${DB_PASSWORD:-CAMBIAR_ESTO}
      # Directorio LDAP (UT5): si LDAP_HOST está defi>
      # contra el servicio de directorio. Valores de >
      #   export LDAP_HOST="ldap://IP_DPL_LAB"
      LDAP_HOST: ldap://172.19.0.3  #Tuve que descomentarla
      LDAP_BASE_DN: "ou=usuarios,dc=dpl,dc=local"   #Tuve que descomentarla
      LDAP_USER_FILTER: "(uid=%s)"   #Tuve que descomentarla
    volumes:
~~~

17. Volcamos el directorio con **slapcat**.

Con **slapcat** exportamos la base directamente desde el servidor en formato LDIF.

~~~bash
docker exec dpl-lab slapcat
~~~

## Comprobación.

Creamos el fichero **~/dpl/AE6/scripts/comprobar.sh**:

~~~bash
nano ~/dpl/AE6/scripts/comprobar.sh
~~~

Le añadimos el script propuesto en el guión de la práctica.

~~~
#!/usr/bin/env bash
# comprobar.sh — comprobación de DNS y directorio (AE6).
# Uso: IP_DPL_LAB=172.19.0.3 ./comprobar.sh
set -u

IP_DPL_LAB="${IP_DPL_LAB:?Define IP_DPL_LAB con la IP de dpl-lab en dpl-net}"
BASE_DN="dc=dpl,dc=local"
USER_DN="${USER_DN:-uid=alumno,ou=usuarios,${BASE_DN}}"
USER_PASSWORD="${USER_PASSWORD:-CAMBIAR_ESTO}"
ZONE_FILE="/etc/bind/db.dpl.local"
FALLOS=0

ok()   { echo "[OK]   $1"; }
fallo(){ echo "[FALLO] $1"; FALLOS=$((FALLOS + 1)); }

echo "== DNS =="
docker exec dpl-lab named-checkconf && ok "named-checkconf" || fallo "named-checkconf"
docker exec dpl-lab named-checkzone dpl.local "$ZONE_FILE" | grep -q OK \
  && ok "named-checkzone" || fallo "named-checkzone"
docker exec dpl-lab pgrep -a named >/dev/null && ok "named activo" || fallo "named no activo"

for nombre in www.dpl.local app.dpl.local; do
  ip="$(docker exec dpl-cliente dig +short "@${IP_DPL_LAB}" "$nombre" A)"
  [ -n "$ip" ] && ok "$nombre -> $ip" || fallo "$nombre no resuelve"
done

echo "== Directorio =="
docker exec dpl-lab pgrep -a slapd >/dev/null && ok "slapd activo" || fallo "slapd no activo"
docker exec dpl-lab ldapsearch -x -LLL -b "$BASE_DN" "(uid=alumno)" dn | grep -q "dn:" \
  && ok "usuario alumno en el directorio" || fallo "usuario alumno ausente"
docker exec dpl-cliente ldapwhoami -x -H "ldap://$IP_DPL_LAB" -D "$USER_DN" -w "$USER_PASSWORD" \
  | grep -q "dn:" && ok "bind correcto" || fallo "bind correcto rechazado"
docker exec dpl-cliente ldapwhoami -x -H "ldap://$IP_DPL_LAB" -D "$USER_DN" -w "incorrecta" >/dev/null 2>&1 \
  && fallo "un bind incorrecto no debería funcionar" || ok "bind incorrecto rechazado"

echo "== Integración =="
docker exec dpl-cliente curl -s -o /dev/null -w '%{http_code}' http://www.dpl.local \
  | grep -q '^200$' && ok "Web por nombre (200)" || fallo "Web por nombre"
docker exec dpl-cliente curl -s -o /dev/null -w '%{http_code}' -X POST \
  -d "usuario=alumno&password=${USER_PASSWORD}" http://www.dpl.local/api/login \
  | grep -q '^200$' && ok "login LDAP (200)" || fallo "login LDAP"

echo
if [ "$FALLOS" -eq 0 ]; then
  echo "Resultado: todas las comprobaciones han pasado."
else
  echo "Resultado: $FALLOS comprobación(es) fallida(s)."
  exit 1
fi
~~~

Lo lanzamos:

~~~bash
sudo IP_DPL_LAB=172.19.0.3 ./comprobar.sh
== DNS ==
[OK]   named-checkconf
[OK]   named-checkzone
[OK]   named activo
[OK]   www.dpl.local -> 172.18.0.3
[OK]   app.dpl.local -> 172.18.0.2
== Directorio ==
[OK]   slapd activo
[OK]   usuario alumno en el directorio
[OK]   bind correcto
[OK]   bind incorrecto rechazado
== Integración ==
[OK]   Web por nombre (200)
[OK]   login LDAP (200)

Resultado: todas las comprobaciones han pasado.
~~~

## Problemas encontrados.

- Debido a problemas al realizar las comprobaciones con curl del servicio de directorio, hemos tenido que acceder a AE4/compose.yml y descomentar las líneas de las variables de entorno.
~~~
app:
    build: ./app               # contexto app/ (usa a>
    container_name: dpl-django
    environment:
      # Configuración por variables de entorno (sin c>
      APP_MENSAJE: "Hola desde el servidor de aplicac>
      APP_DATA_DIR: /data
      # Capa de datos: la app usa PostgreSQL si DB_HO>
      DB_HOST: db
      DB_NAME: travelroad
      DB_USER: alumno
      DB_PASSWORD: ${DB_PASSWORD:-CAMBIAR_ESTO}
      # Directorio LDAP (UT5): si LDAP_HOST está defi>
      # contra el servicio de directorio. Valores de >
      #   export LDAP_HOST="ldap://IP_DPL_LAB"
      LDAP_HOST: ldap://172.19.0.3  #Tuve que descomentarla
      LDAP_BASE_DN: "ou=usuarios,dc=dpl,dc=local"   #Tuve que descomentarla
      LDAP_USER_FILTER: "(uid=%s)"   #Tuve que descomentarla
    volumes:
~~~

Una vez modificado, hemos podido realizar las pruebas de manera satisfactoria.

~~~bash
curl -s -X POST -d "usuario=alumno&password=CAMBIAR_ESTO" http://localhost:8000/api/login
{"autenticado": true, "usuario": "alumno", "perfil": {"cn": "alumno", "mail": "alumno@dpl.local"}}

curl -s -o /dev/null -w '%{http_code}\n' -X POST -d "usuario=alumno&password=incorrecta" http://localhost:8000/api/login
401
~~~

- Al realizar las comprobaciones del servidor DNS:
~~~bash
docker exec dpl-cliente curl -I http://www.dpl.local
docker exec dpl-cliente curl -s http://app.dpl.local:8000/api/mensaje   
~~~

Los dos últimos comandos nos han dado fallo debido a que dpl-cliente no está conectada a la red que ha creado compose  **dpl-app_default**, por lo tanto hemos tenido que añadirla.

~~~bash
sudo docker network connect dpl-app_default dpl-cliente
~~~