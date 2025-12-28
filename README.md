# WebDAV con Apache en Docker

Resumen
-------
Este proyecto despliega un servidor WebDAV (Apache HTTPD 2.4) en Docker para que varios usuarios se conecten a su carpeta personal. La conexión se expone a través de Dokploy y pasa por Cloudflare (DNS/CDN/proxy).

Arquitectura (resumen)
- Cloudflare: gestiona DNS, CDN y proxy/reverse-proxy (opcional TLS/HTTPS).
- Dokploy: herramienta/servicio que despliega la aplicación y la hace accesible públicamente.
- Contenedor Docker `httpd:2.4`: ejecuta Apache con la configuración en `httpd.conf`.
- Volúmenes montados:
  - `${DIR_USER}` -> `/var/webdav` : almacenamiento de usuarios.
  - `${HTTPD_CONF}` -> `/usr/local/apache2/conf/httpd.conf` : configuración de Apache (solo lectura).
  - `${PASS_USERS}` -> `/usr/local/apache2/conf/htpasswd` : archivo `htpasswd` con usuarios/contraseñas (solo lectura).
  - `${ERROR_FILES}` -> `/usr/local/apache2/htdocs/site` : páginas de error personalizadas (solo lectura).

Archivos clave
- `docker-compose.yml` : define el servicio `webdav` usando la imagen `httpd:2.4` y monta los volúmenes.
- `.env` : variables usadas por `docker-compose` (`DIR_USER`, `HTTPD_CONF`, `PASS_USERS`, `ERROR_FILES`).
- `httpd.conf` : configuración de Apache (viene adjunta). Puntos importantes:
  - `Alias /storage "/var/webdav"` mapea la ruta WebDAV al volumen de usuarios.
  - `<Directory "/var/webdav">` define permisos y opciones sobre el almacenamiento.
  - `SetEnvIf Request_URI "^/storage/([^/]+)" REQ_FOLDER=$1` captura la carpeta solicitada (primer segmento después de `/storage/`).
  - `<LocationMatch "/storage">` activa `DAV On` y configura `AuthType Basic` con `AuthUserFile` apuntando a `htpasswd`.
  - `Require expr "%{ENV:REQ_FOLDER} == %{REMOTE_USER}"` garantiza que cada usuario solo acceda a la carpeta que coincide con su nombre de usuario.
  - `DavLockDB "/tmp/DavLock"` controla los locks de WebDAV.

Cómo funciona la autenticación y el aislamiento de usuarios
- El cliente se conecta utilizando winscp (Windows) o Davs: (linux) a `<dominio o subdominio>/storage/<usuario>`.
- Apache extrae `<usuario>` desde la URL y lo guarda en la variable de entorno `REQ_FOLDER`.
- Apache solicita autenticación básica contra `/usr/local/apache2/conf/htpasswd`.
- Solo si `REMOTE_USER` (usuario autenticado) coincide con `REQ_FOLDER`, Apache da acceso; así cada usuario solo puede ver/editar su propia carpeta.

Pasos rápidos para levantar el servicio
1. Verifica y ajusta las variables en `.env` (rutas absolutas en el host).
2. Crea las carpetas de usuarios dentro de la ruta indicada por `DIR_USER`, por ejemplo `/home/clients/alice`.
3. Añade usuarios a `htpasswd` (el nombre de usuario debe coincidir con la carpeta):

```bash
# ejemplo usando la utilidad apache2-utils
htpasswd -B /home/configDav/htpasswd alice
```

4. Asegúrate de permisos: el proceso Apache dentro del contenedor corre como `www-data`; ajusta el propietario/permiso de las carpetas montadas para permitir lectura/escritura.

```bash
chown -R 33:33 /home/clients/alice
# o según UID/GID de www-data en tu entorno
```

5. Inicia el contenedor:

```bash
docker compose up -d
```

Comprobación y logs
- Ver logs del servicio:

```bash
docker compose logs -f webdav
```

- Revisar que Apache carga `httpd.conf` correctamente y que `AuthUserFile` apunta al `htpasswd` correcto.

Consideraciones de despliegue (Dokploy + Cloudflare)
- Dokploy: se encarga del despliegue del contenedor y del enrutado hacia el servicio. Asegúrate de que Dokploy enruta el tráfico hacia la máquina/container donde corre Docker.
- Cloudflare: si usas Cloudflare en modo proxy, puede manejar TLS front-end. Para evitar problemas con autenticación o WebDAV, preferible usar "Full (strict)" y que el backend tenga un certificado válido, o ajustar según tu flujo de TLS.

Seguridad y recomendaciones
- Protege el archivo `htpasswd` y evita montarlo con permisos de escritura desde el contenedor.
- Usa TLS entre cliente y Cloudflare; y TLS entre Cloudflare y el origen si es posible (Full strict recomendado).
- Revisa los permisos de las carpetas de usuario para evitar que un proceso del host pueda modificar datos sin control.
- Considera límites y monitoring: WebDAV puede dejar archivos lockeados; `DavLockDB` debe estar en una ubicación estable y persistente si necesitas reinicios.

Problemas comunes
- Error 403/401: verifica `htpasswd`, rutas y la expresión `Require expr` (la carpeta solicitada debe coincidir con el usuario del htpasswd).
- Permisos de archivos: el contenedor necesita permisos adecuados sobre `/var/webdav` (UID/GID de `www-data`).
- Destino cambiado (Cloudflare): si Cloudflare altera cabeceras o usa HTTPS, la directiva `RequestHeader edit Destination ^https: http:` en `httpd.conf` intenta ajustar cabeceras relacionadas; revisa si es necesaria en tu flujo.



