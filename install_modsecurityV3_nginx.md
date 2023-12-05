# Como instalar y configurar ModSecurity en Ubuntu 22.04 con Nginx

Todos los comandos deben ser ejecutado como super usuario (root)

### Paso 1: Instalar dependencias

```bash
sudo apt update
sudo apt install -y apt-utils autoconf automake build-essential git libcurl4-openssl-dev libgeoip-dev liblmdb-dev libpcre++-dev libtool libxml2-dev libyajl-dev pkgconf wget zlib1g-dev
```

### Paso 2: Descargar y compilar ModSecurity 3.0

```bash
git clone --depth 1 -b v3/master --single-branch https://github.com/SpiderLabs/ModSecurity
cd ModSecurity
git submodule init
git submodule update
./build.sh
./configure
make
make install
cd ..
```

### Paso 3: Descargue el conector NGINX para ModSecurity y compílelo como un módulo dinámico

```bash
git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
```

Determine qué versión de NGINX se está ejecutando en el host donde se cargará el módulo ModSecurity:

```bash
$ nginx -v
nginx version: nginx/1.18.0 (Ubuntu)
```

Descargue el código fuente correspondiente a la versión instalada de NGINX (se requieren las fuentes completas aunque solo se esté compilando el módulo dinámico):

```bash
$ wget http://nginx.org/download/nginx-1.18.0.tar.gz
$ tar zxvf nginx-1.18.0.tar.gz
```

Compile el módulo dinámico y cópielo en el directorio estándar para módulos:

```bash
$ cd nginx-1.18.0
$ ./configure --with-compat --add-dynamic-module=../ModSecurity-nginx
$ make modules
$ cp objs/ngx_http_modsecurity_module.so /usr/lib/nginx/modules/
$ cd ..
```

### Paso 5: Cargue el módulo dinámico del conector NGINX ModSecurity

Agregue la siguiente directiva `load_module` al contexto principal (nivel superior) en `/etc/nginx/nginx.conf`. Le indica a NGINX que cargue el módulo dinámico ModSecurity cuando procesa la configuración:
```
load_module modules/ngx_http_modsecurity_module.so;
```

### Paso 6: Configurar, habilitar y probar ModSecurity

El último paso es habilitar y probar ModSecurity.

Configure el archivo de configuración ModSecurity apropiado. Aquí utilizamos la configuración ModSecurity recomendada proporcionada por TrustWave Spiderlabs, los patrocinadores corporativos de ModSecurity.

```
$ mkdir /etc/nginx/modsec
$ wget -P /etc/nginx/modsec/ https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/modsecurity.conf-recommended
$ mv /etc/nginx/modsec/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf
```

Para garantizar que ModSecurity pueda encontrar el archivo unicode.mapping (distribuido en el directorio ModSecurity de nivel superior del repositorio de GitHub), cópielo en /etc/nginx/modsec

```
cp ModSecurity/unicode.mapping /etc/nginx/modsec
```

Cambie la directiva `SecRuleEngine` en la configuración para cambiar del modo predeterminado "solo detección" a eliminar activamente el tráfico malicioso.

```
sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/nginx/modsec/modsecurity.conf
```

Crea un archivo de configuración para ModSecurity:

```bash
sudo nano /etc/nginx/modsec/main.conf
```

Añade las siguientes líneas:

```nginx
Include "/etc/nginx/modsec/modsecurity.conf"
include "../owasp-modsecurity-crs/rules/*.conf"
```

**NOTA:** lA directiva **"../owasp-modsecurity-crs/rules/*.conf"** agrega todas las reglas existentes en owasp y eso posiblemente sea mucho más de lo realemente necesite, así que le recomiendo que revise en detalle cada una de las reglas ofrecidas en owasp y solo habilitar las que realmente necesita según sus requerimientos.

### Paso 7: Descargar las reglas de OWASP ModSecurity Core Rule Set (CRS)

```bash
git clone --depth 1 -b v3.3/master --single-branch https://github.com/coreruleset/coreruleset.git
mv coreruleset /etc/nginx/owasp-modsecurity-crs
```

### Paso 8: Configurar Nginx con ModSecurity

Abre el archivo de configuración de tu(s) webs (por lo general, en un archivo en `/etc/nginx/sites-available/`).

Añade las siguientes líneas dentro del bloque `server`:

```nginx
modsecurity on;
modsecurity_rules_file /etc/nginx/modsec/main.conf;
```

Por ejemplo:

```nginx
server {
    listen 80;
    server_name tu_domino.com;

    location / {
        # Configuración adicional de tu servidor
    }

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
}
```

### Paso 9: Reiniciar Nginx

```bash
service nginx restart
```

Con esto, ModSecurity debería estar instalado y configurado en tu servidor Nginx en Ubuntu 22.04 Ten en cuenta que estas instrucciones pueden variar dependiendo de tu configuración específica y de las actualizaciones futuras de los paquetes. Asegúrate de revisar la documentación oficial de ModSecurity y Nginx para obtener la información más reciente y específica para tu caso.

# Como verificar que ModSecurity está correctamente instalado y funcionando

puedes seguir estos pasos:

1. **Revisar los logs de Nginx y ModSecurity:**
   - Verifica los logs de error de Nginx (`/var/log/nginx/error.log`) y los logs de ModSecurity (`/var/log/modsec_audit.log`).
   - Si ModSecurity encuentra y bloquea una solicitud que considera maliciosa, deberías ver entradas en el archivo `modsec_audit.log`.

2. **Configurar una regla de prueba:**
   - Puedes crear una regla de prueba en ModSecurity para ver si está bloqueando solicitudes correctamente. Por ejemplo, puedes agregar la siguiente regla en tu archivo de configuración de ModSecurity (`/etc/nginx/modsec/main.conf`):

     ```nginx
     SecRule ARGS:testparam "@contains test" \
     "id:1,phase:2,deny,status:403,msg:'Test rule blocking request'"
     ```

   - Esta regla bloqueará cualquier solicitud que contenga el parámetro `testparam` con el valor "test". Después de agregar la regla, reinicia Nginx y envía una solicitud con ese parámetro para ver si ModSecurity la bloquea.

   Emita el siguiente comando `curl`. El código 403 de estado confirma que la regla está funcionando.
   ```
   $ curl localhost?testparam=test
   <html>
   <head><title>403 Forbidden</title></head>
   <body bgcolor="white">
   <center><h1>403 Forbidden</h1></center>
   <hr><center>nginx/1.13.1</center>
   </body>
   </html>
   ```

# Manten actualizadas las reglas de OWASP CRS (Core Rule Set) en ModSecurity

### 1. Clonar el Repositorio de OWASP CRS:

OWASP CRS se mantiene en un repositorio de GitHub. Puedes clonar este repositorio para obtener las últimas actualizaciones:

```bash
cd /etc/nginx
sudo git clone --depth 1 -b v3.3/master --single-branch https://github.com/coreruleset/coreruleset.git owasp-modsecurity-crs
```

### 2. Configurar las Actualizaciones Automáticas:

Puedes configurar un cronjob para que actualice automáticamente las reglas de OWASP CRS en intervalos regulares. Edita el archivo crontab con el siguiente comando:

```bash
sudo crontab -e
```

Añade la siguiente línea al archivo crontab para actualizar las reglas, por ejemplo, cada día a las 2 AM:

```bash
0 2 * * * cd /etc/nginx/owasp-modsecurity-crs && git pull
```

Guarda y cierra el archivo. Esto configurará la actualización automática de las reglas de OWASP CRS todos los días a las 2 AM.

### 3. Reiniciar Nginx:

Después de actualizar las reglas, reinicia Nginx para aplicar los cambios:

```bash
sudo service nginx restart
```

### 4. Verificar las Actualizaciones:

Puedes verificar si las reglas se han actualizado revisando el registro de ModSecurity y el log de auditoría. También puedes verificar la fecha de la última actualización en el directorio de reglas de OWASP CRS:

```bash
ls -l /etc/nginx/owasp-modsecurity-crs/rules
```

Estos pasos asegurarán que mantengas actualizadas las reglas de OWASP CRS en tu entorno ModSecurity. Ten en cuenta que, antes de aplicar actualizaciones automáticas en producción, es recomendable probarlas en un entorno de desarrollo o prueba para garantizar que no afecten negativamente a tu aplicación.

**Feliz codificación** ✌💻