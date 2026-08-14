# Requisitos del sistema

- Minimo 4 CPUs
- Minimo 16 GB RAM

# Directorios y ficheros del proyecto

- `config_files/`: ficheros de configuracion de arranque. Tambien incluye directorios para imagenes, logos y favicon de la interfaz del proveedor.
- `sp-remote-xml/`: metadatos XML de los servicios a los que se quiere unir el IdP.
- `init_script.sh`: script de automatizacion de despliegue.
- `docker-compose.yaml`: despliegue estandar (HTTP en contenedor, SSL en proxy externo).
- `docker-compose_https.yaml`: despliegue alternativo con Nginx en HTTPS dentro del contenedor.

# Pasos para desplegar el proveedor de identidad

## 1) Descargar repositorio

Opciones:

- `git clone https://github.com/rediris-es/IdPnube_onprem_docker_ldap.git`
- Descargar el ZIP del proyecto.

## 2) Rellenar ficheros de configuracion

Segun el modo de despliegue elegido:

- Modo estandar (`docker-compose.yaml`):
    - `config_files/config_back.env`
    - `config_files/config_front.env`
- Modo HTTPS integrado (`docker-compose_https.yaml`):
    - `config_files/config_back_https.env`
    - `config_files/config_front_https.env`

## 3) Ejecutar script de inicializacion

Ejecutar `init_script.sh` para el despliegue de la instancia Docker y la posible generacion de certificados de firma.

Parametros requeridos:

`<nombre_corto_inst> <C> <ST> <L> <O> <OU> <CN> <email>`

Ejemplo:

`sh init_script.sh 'RedIRIS' 'ES' 'Madrid' 'Madrid' 'RedIRIS' 'Middleware' 'rediris.idpnube.rediris.es' 'admin@rediris.es'`

## 4) Levantar contenedores

### Opcion A: despliegue estandar (sin SSL en contenedor)

```bash
docker compose -f docker-compose.yaml up -d
```

En este modo, se debe configurar Apache/Nginx externo para terminar SSL y redirigir al servicio en `http://127.0.0.1:8082`.

### Opcion B: despliegue HTTPS integrado (SSL dentro de Docker)

```bash
docker compose -f docker-compose_https.yaml up -d
```

Este modo levanta directamente Nginx con SSL en el contenedor:

- Publica `443:443` en el host.
- Lee los certificados desde `config_files/certs`.
- Usa `config_files/config_front_https.env` y `config_files/config_back_https.env`.

Requisitos especificos para esta opcion:

- Definir `SERVER_NAME_SSO` con el dominio real del IdP en ambos ficheros de configuracion HTTPS.
- Colocar los certificados en `config_files/certs` con los nombres:
    - `cert_web.crt`
    - `cert_web.key`
- Tener libre el puerto 443 en el host.

Validaciones recomendadas:

```bash
docker compose -f docker-compose_https.yaml ps
curl -k https://127.0.0.1/simplesaml
```

URL esperada de acceso:

- `https://<SERVER_NAME_SSO>/simplesaml`

Nota: con `docker-compose_https.yaml` no es necesario terminar SSL en un proxy externo para este servicio.

# Ejemplos de proxy externo (solo modo estandar)

Los `ProxyPass` del ejemplo son correctos; el resto de parametros debe adaptarse al entorno.

## Configuracion Apache

```plain
<VirtualHost *:443>
        ServerName
        CustomLog /log/apache/access.log combined
        ErrorLog /log/apache/error.log

        SSLEngine               on
        SSLCertificateFile
        SSLCertificateKeyFile
        SSLProtocol             all -SSLv2 -SSLv3
        SSLCipherSuite          ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS
        SSLHonorCipherOrder     on
        SSLProxyEngine on

        ProxyPreserveHost On

        ProxyPass / http://127.0.0.1:8082/
        ProxyPassReverse / http://127.0.0.1:8082/

</VirtualHost>
```

## Configuracion Nginx

```plain
client_max_body_size 0;
server {
    listen 80;
    server_name nombreDelServicio;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name nombreDelServicio;

    ssl_certificate /ruta/al/certificado;
    ssl_certificate_key /ruta/a/la/clavePrivada;
    ssl_trusted_certificate /ruta/al/certificado/de/la/CA;
    access_log /ruta/access/logs;
    error_log /ruta/error/logs;

    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;

        proxy_redirect off;
        proxy_pass http://127.0.0.1:8082;
    }
}
```
