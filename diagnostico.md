# Informe de diagnóstico - AE6
> Autor: Óscar Regalado León.
> Fecha: Noviembre 2026.
> Autoría: imagen oficial de Ubuntu — https://hub.docker.com/_/ubuntu

Este informe documenta la localización, análisis y resolución de cinco fallos deliberados introducidos en la cadena completa de servicios (DNS, conectividad de puertos, servidor Nginx, backend Django y servicio de directorio OpenLDAP).

## Fallo 1: DNS (servidor autoritativo BIND9)

- **Síntoma observado**: Al intentar resolver el nombre del servicio web desde el contenedor cliente, el comando **nslookup www.dpl.local** devolvió un error de servidor o una IP incorrecta (código NXDOMAIN o resolución nula).

- **Causa**: El registro **A** correspondiente al subdominio **www** dentro del fichero de zona **/etc/bind/db.dpl.local** apuntaba a una dirección IP errónea (una IP huérfana de un contenedor anterior) y el número de serie (serial) no se había incrementado, por lo que BIND9 no recargó la nueva zona.

- **Comprobación**:
~~~bash
sudo docker exec dpl-cliente nslookup www.dpl.local 172.19.0.3
Server:		172.19.0.3
Address:	172.19.0.3#53
** server can't find www.dpl.local: NXDOMAIN
~~~

- **Corrección aplicada**: corrección del fichero **db.dpl.local**, actualización del registro A de www a la IP correcta de Nginx (172.18.0.3) e incremento del serial (ej. de 2024110503 a 2024110504).

~~~bash
sudo docker exec -it dpl-lab nano /etc/bind/db.dpl.local

sudo docker exec dpl-lab rndc reload  #Recarga del servicio de nombres.
~~~

- **Verificación final**: 

~~~bash
sudo docker exec dpl-cliente nslookup www.dpl.local 172.19.0.3        
Server:		172.19.0.3
Address:	172.19.0.3#53

Name:	www.dpl.local
Address: 172.18.0.3
~~~

## Fallo 2: Puerto (red y enrutamiento del proxy)

- **Síntoma observado**: Al realizar una petición HTTP contra el proxy inverso mediante curl -I http://www.dpl.local, el servidor devuelve un código de error 502 Bad Gateway.

~~~bash

~~~

- **Causa**: El fichero de configuración de Nginx (construido desde el contexto del proyecto AE4) tiene mal definido el destino del proxy_pass, apuntando a un puerto erróneo del backend (http://dpl-django:8080 en lugar de http://dpl-django:8000).

~~~
#Fichero ~/dpl/AE4/nginx/default.conf
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://dpl-django:8080;   #modifico de 8000 a 8080 para probocar el fallo
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
~~~

~~~bash
sudo docker restart dpl-nginx  #recargamos la configuración específica de dpl-nginx

sudo docker exec dpl-cliente curl -I http://www.dpl.local
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0   1HTTP/1.1 502 Bad Gateway      0      0 --:--:-- --:--:-- --:--:--     0 #Se produce el error
Server: nginx/1.27.5
Date: Sun, 15 Nov 2026 10:10:34 GMT
Content-Type: text/html
Content-Length: 157
Connection: keep-alive
57    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
~~~

- **Corrección aplicada**: modificación de la directiva proxy_pass en el fichero de configuración de nginx del proyecto local (dentro de la estructura de AE4):

~~~
#Fichero ~/dpl/AE4/nginx/default.conf
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://dpl-django:8000;   #modifico de 8080 a 8000 para subsanar el fallo
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
~~~

~~~bash
docker restart dpl-nginx
~~~

- **Verificación final**:
~~~bash
sudo docker exec dpl-cliente curl -I http://www.dpl.local
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
HTTP/1.1 200 OK     #Conexión subsanada.
Server: nginx/1.27.5
Date: Sun, 15 Nov 2026 10:16:28 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive
~~~

## Fallo 3: Servidor Web (nginx y ficheros estáticos)

- **Síntoma observado**: al acceder mediante curl -I http://www.dpl.local, el servidor devuelve un código 403 Forbidden o 404 Not Found, indicando que nginx no encuentra los ficheros estáticos del build.

- **Causa**: la directiva root en el archivo de configuración del virtual host de nginx apunta a una ruta vacía o inexistente dentro del contenedor web, o el volumen del build de Vite no se ha copiado correctamente.

~~~
server {
    listen 80;
    server_name _;

    #La raíz global apunta al directorio de estáticos donde hemos generado la ruta falsa
    root /usr/share/nginx/ruta_falsa; 
    index index.html;

    #Peticiones a la API forzamos un endpoint específico
    location /api/ {
        proxy_pass http://dpl-django:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # El resto intenta cargar ficheros estáticos o devuelve 404
    location / {
        try_files $uri $uri/ =404;
    }
}
~~~

~~~bash
sudo docker restart dpl-nginx

sudo docker exec dpl-cliente curl -s -o /dev/null -w '%{http_code}\n' http://www.dpl.local
404
~~~
- **Corrección aplicada**: modificación de la directiva proxy_pass en el fichero de configuración de nginx del proyecto local (dentro de la estructura de AE4) para el funcionamiento de nginx exclusivamente como proxy inverso.

~~~
#Fichero ~/dpl/AE4/nginx/default.conf
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://dpl-django:8000;  
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
~~~

- **Verificación final**:

~~~bash
docker restart dpl-nginx

sudo docker exec dpl-cliente curl -s -o /dev/null -w '%{http_code}\n' http://www.dpl.local
200
~~~

## Fallo 4: 

- **Síntoma observado**: Al realizar una petición HTTP contra el servicio web http://www.dpl.local, el proxy inverso responde con un código de error 502 Bad Gateway, indicando que la infraestructura web funciona pero no puede comunicarse con el backend.

- **Causa**: El contenedor encargado de ejecutar el servidor de aplicaciones Django (dpl-django) se encuentra detenido (Exited), interrumpiendo el flujo de peticiones.

- **Comprobación**: Simulamos el fallo deteniendo el contenedor de la aplicación:

~~~bash
sudo docker stop dpl-django
dpl-django

sudo docker exec dpl-cliente curl -s -o /dev/null -w '%{http_code}\n' http://www.dpl.local
502  #Fallo generado
~~~
- **Corrección aplicada**: iniciamos de nuevo el contenedor dpl-django:
~~~bash
sudo docker start dpl-django
~~~
- **Verificación final**: 
~~~bash
sudo docker exec dpl-cliente curl -s -o /dev/null -w '%{http_code}\n' http://www.dpl.local
200
~~~

## Fallo 5: autenticación

- **Síntoma**: El inicio de sesión en la aplicación web con un usuario válido del dominio devuelve 401 Unauthorized (fallo de autenticación).

- **Causa**: El servicio slapd en el contenedor dpl-lab está detenido.

~~~bash
sudo docker exec dpl-lab pkill slapd

sudo docker exec dpl-lab pgrep slapd
~~~

~~~bash
sudo docker exec dpl-cliente ldapwhoami -x -H ldap://172.18.0.6 -D "uid=alumno,ou=usuarios,dc=dpl,dc=local" -w CAMBIAR_ESTO
ldap_sasl_bind(SIMPLE): Can't contact LDAP server (-1)
~~~

- **Corrección aplicada**: volvemos a arrancar el servicio.

~~~bash
sudo docker exec -d dpl-lab slapd -d 0
~~~

- **Verificación final**: 

~~~bash
sudo docker exec dpl-cliente ldapwhoami -x -H ldap://172.18.0.6 -D "uid=alumno,ou=usuarios,dc=dpl,dc=local" -w CAMBIAR_ESTO
dn:uid=alumno,ou=usuarios,dc=dpl,dc=local
~~~


