<p align="center"><a href="https://www.syndeno.com/"><img src="https://syndeno.cloud/assets/images/logo/logo.svg" alt="Syndeno" width="150"/></a></p>
<h2 align="center">Syndeno Monitoring</h2>

#### Introducción
Este repositorio contiene los ficheros de configuración de los servicios para la infraestructura central de Syndeno:
- Prometheus
- AlertManager
- Grafana
- Nginx
- Certbot

Y el `docker-compose.yml` para levantar todas las instancias con un solo comando.

------------
#### Primeros pasos
Para poder levantar los servicios debemos de estar ubicados en donde se encuentra el fichero `docker-compose.yml` y ejecutar el comando `docker-compose up -d`.  Esto levantaría los cinco contenedores pero daría un error, ya que **certbot**  no encontraría los nombres de dominio y no podría crear el certificado. Para crear el certificado debemos de realizar una serie de modificaciones en el fichero `nginx.conf` para que: la primera vez que iniciemos los contenedores, se creen los certificados y nos lance la ruta en donde se han almacenado, posteriormente, modificamos el fichero `nginx.conf` indicando la ruta en donde se encuentra el certificado y otras configuraciones. 

### Paso 1

En el fichero `docker-compose.yml`,en el servicio de **nginx** comentamos la linea en donde este el puerto **443:443** , en el servicio **certbot**, se debe de realizar los siguientes cambios cuando se monta por primera vez en una máquina nueva. 
En la sección **command** por defecto estará así:
```
command: certonly --webroot --webroot-path=/var/www/certbot --email tls.certificates@syndeno.com --agree-tos --force-renewal -d rgmftcv.gc.syndeno.net -d tgxmoas.gc.syndeno.net -d cracbqb.gc.syndeno.net`
```
1. Cambiamos **\--force-renewal** por **\--staging** y cambiamos todos los flags **-d**, dependiendo los nombres de dominio que se vayan a utilizar.
La opción **\--staging** indica que se utilizará el entorno de prueba de Let's Encrypt, emitiendo certificados de prueba para comprobar si la configuración es correcta.
Debería quedar algo tal que así:
```
command: certonly --webroot --webroot-path=/var/www/certbot --email tls.certificates@syndeno.com --agree-tos --staging -d rgmftcv.gc.syndeno.net -d tgxmoas.gc.syndeno.net -d cracbqb.gc.syndeno.net
```

1. En la sección **volumes>web-root>device** asegurarse que la carpeta **/views/** está creada, si no es así, hay que crearla. 
Se define un volumen llamado **web-root** que utiliza el driver local y se mapea al directorio **/views/** del host. Se utiliza para almacenar los archivos que se generan con el comando **certonly** de Certbot.
```
web-root:
    driver: local
    driver_opts:
      type: none
      device: /home/korellana/syndeno-monitoring/Docker/views/
      o: bind
```

### Paso 2
El contenido que se encuentra en el fichero **/nginx/nginx-conf/local-domains.conf** Lo copiamos en el fichero **nginx.conf**. Esto se realiza la primera vez para que certbot pueda comprobar que existe el dominio y crear el certificado. En ese fichero debes cambiar el contenido de  **server_name** y **proxxy_pass**. Indicariamos los nombres de dominio y las IP's+puertos en donde se aloja cada servicio. 
`server_name rgmftcv.gc.syndeno.net;`
`proxy_pass http://172.30.0.2:9090/;`

### Paso 3
Teniendo esto configurado, ejecutar el **docker-compose.yml** con el comando **docker-compose up**. Y buscar en los logs el servicio Certbot que no de ningún error.
Una vez comprobado esto detenemos los contenedores con el comando `docker-compose down`.

### Paso 4
Volver al fichero `docker-compose.yml`, en el servicio **Certbot** apartado **command** volver a poner la configuración que tenía antes de modificar, es decir, quitar **\--staging** y volver a poner **\--force-renewal**. Esta opción indica que se debe forzar la renovación del certificado con los mismos dominios que un certifcado existente (el certificado de prueba del paso 3), incluso si no ha caducado. 
Volvemos a lanzar el `docker-compose up` y comprobamos los logs de Certbot y nos tiene que salir un mensaje como este:
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/rgmftcv.gc.syndeno.net/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/rgmftcv.gc.syndeno.net/privkey.pem
This certificate expires on 2023-08-03.
These files will be updated when the certificate renews.
NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.
```
### Paso 5
Nuevamente detenemos los contenedores con `docker-compose down`.  Borramos la configuración que tenemos en **/nginx/nginx-conf/nginx.conf**, copiamos la configuración que está en **local_conf_ssl.conf** y lo pegamos en **nginx.conf**. Los logs de Certbot nos muestran dos rutas:
```
Certificate is saved at: /etc/letsencrypt/live/rgmftcv.gc.syndeno.net/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/rgmftcv.gc.syndeno.net/privkey.pem
```
En **ssl_certificate** pegamos el certificado **fullchain.pem**. Y en **ssl_certificate_key** pegamos la key **privkey.pem**. Una vez más cambiamos realizamos las modificaciones de **server_name** y **proxy_pass**.

Con estos pasos ya tendríamos Nginx sirviendo los dominios de manera segura.

### Paso 6
##### Renovar Certificados
Los certificados de Let's Encrypt son válidos durante 90 días, por lo que haremos que el contenedor de Certbot se inice cada cierto tiempo para que cree esos certificados. 
Para ello utilizamos **Crontab**, con el comando `crontab -e`, elegimos el editor de texto de preferencia y abajo del todo ponemos lo siguiente:
` * 2 1 * * /usr/bin/docker restart certbot`
Esto quiere decir: ***el uno de cada mes a las dos de la mañana se reinicie el contenedor certbot*** .

Con esto ya tendríamos automatizado los certificados y nuestros dominios ya pueden utulizar HTTPS.
___
#### Ayuda
Con la ayuda de este [enlace](https://www.digitalocean.com/community/tutorials/how-to-secure-a-containerized-node-js-application-with-nginx-let-s-encrypt-and-docker-compose-es "enlace") he podido conseguir crear los certificados y la automatización.
