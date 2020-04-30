# Nginx Cheat Sheet

(Comprensión del servidor Nginx y algoritmos de selección de bloque de ubicación - https://goo.gl/YyzshP)

* [Índice alfabético de directivas](http://nginx.org/en/docs/dirindex.html)
* [Índice alfabético de variables](http://nginx.org/en/docs/varindex.html)
* [Documentación](http://nginx.org/en/docs/)
* [Nginx en docker](./docker)

```
server {
	location {

	}
}
```

## Bloques: servidor

Prioridad

1. escucha
2. nombre del servidor

### Directiva: listen

La directiva de *escucha* se puede establecer en:

* Una combinación de dirección IP/Puerto.
* Una dirección IP solitaria que luego escuchará en el puerto predeterminado 80.
* Un puerto solitario que escuchará cada interfaz en ese puerto.
* La ruta a un socket Unix.

Cuando estan *incompletas* se comportan de la siguiente manera:

* Un bloque sin directiva de escucha utiliza el valor 0.0.0.0:80.
* Un bloque configurado en una dirección IP 111.111.111.111 sin puerto se convierte en 111.111.111.111:80
* Un bloque configurado en el puerto 8888 sin dirección IP se convierte en 0.0.0.0:8888

#### Opciones por defecto: default_server

```
server {
    listen      80 default_server;
    server_name example.net www.example.net;
    ...
}
```

### Directiva: server_name

Nginx los evalúa utilizando la siguiente fórmula:

* Nginx primero intentará encontrar un bloque de servidor con un nombre de servidor que coincida exactamente con el valor en el encabezado "Host" de la solicitud.
* Busque un bloque de servidor con un nombre_servidor que coincida con un comodín inicial (indicado por un * al comienzo del nombre en la configuración).
* Si no se encuentra ninguna coincidencia con un comodín inicial, Nginx busca un bloque de servidor con un nombre_servidor que coincida con un comodín final (indicado por un nombre de servidor que termina con un * en la configuración).
* Si no se encuentra ninguna coincidencia utilizando un comodín final, Nginx evalúa los bloques del servidor que definen el nombre_servidor utilizando expresiones regulares (indicadas con un ~ antes del nombre).
* Si no se encuentra una coincidencia de expresión regular, Nginx selecciona el bloque de servidor predeterminado para esa dirección IP y puerto.

```
server {
    listen 80;
    server_name example.com;
    ...
}
server {
    listen 80;
    server_name ~^(www|host1).*\.example\.com$;
    ...
}
server {
    listen 80;
    server_name ~^(subdomain|set|www|host1).*\.example\.com$;
    ...
}
server {
    listen 80;
    server_name  ~^(?<user>.+)\.example\.net$;
    ...
}
```

## Bloque: location

```
location optional_modifier location_match {
	...
}
```

### Opciones: optional_modifier

* (ninguno): la ubicación se interpreta como una coincidencia de prefijo. Esto significa que la ubicación dada se comparará con el comienzo del URI de solicitud para determinar una coincidencia.
* =: Este bloque se considerará una coincidencia si el URI de la solicitud coincide exactamente con la ubicación dada.
* ~: Esta ubicación se interpretará como una coincidencia de expresión regular entre mayúsculas y minúsculas .
* ~ *: El bloque de ubicación se interpretará como una coincidencia de expresión regular que no distingue entre mayúsculas y minúsculas .
* ^ ~: Si este bloque se selecciona como la mejor coincidencia de expresión no regular, la coincidencia de expresión regular no tendrá lugar.

### Directiva: index

```
index index.$geo.html index.0.html /index.html;
autoindex on | off;
```

### Directiva:  try_files

```
root /var/www/main;
try_files $uri $uri.html $uri/ /fallback/index.html;
```

Si se realiza una solicitud para / blahblah, la primera ubicación recibirá inicialmente la solicitud. Intentará encontrar un archivo llamado blahblah en el directorio / var / www / main. Si no puede encontrar uno, seguirá buscando un archivo llamado blahblah.html.

### Directiva:  rewrite

```
rewrite ^/rewriteme/(.*)$ /$1 last;
```

Una solicitud de / rewriteme / hello será manejada inicialmente por el primer bloque de ubicación. Se reescribirá en / hello y se buscará una ubicación.

### Directives:  error_page

```
error_page 404             /404.html;
error_page 500 502 503 504 /50x.html;
```

------

## Examples

### Simple configuracion para un sitio PHP

```
server {
    listen      80;
    server_name example.org www.example.org;
    root        /data/www;

    location / {
        index   index.html index.php;
    }

    location ~* \.(gif|jpg|png)$ {
        expires 30d;
    }

    location ~ \.php$ {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME
                      $document_root$fastcgi_script_name;
        include       fastcgi_params;
    }
}
```

### App server (Redmine)

```
server {
    listen 80;
    server_name 107.170.165.117 myproject.com www.myproject.com;

    root /srv/redmine/public;
    passenger_enabled on;

    client_max_body_size 10m;
}
```

### App server (Jenkins)

```
upstream app_server {
    server 127.0.0.1:8080 fail_timeout=0;
}

server {
    listen 80;
    listen [::]:80 default ipv6only=on;
    server_name ci.yourcompany.com;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;

        if (!-f $request_filename) {
            proxy_pass http://app_server;
            break;
        }
    }
}
```
