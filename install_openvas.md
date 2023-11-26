# Como instalar OpenVAS en Ubuntu

OpenVAS (Open Vulnerability Assessment System): OpenVAS es un escáner de vulnerabilidades de código abierto que forma parte de GVM. Realiza escaneos de seguridad en sistemas y redes para identificar posibles vulnerabilidades. OpenVAS utiliza una base de datos de pruebas de seguridad y está constantemente actualizado para incluir información sobre nuevas vulnerabilidades.

## Administrador de vulnerabilidades de Greenbone

Greenbone es el proveedor de gestión de vulnerabilidades de código abierto más utilizado del mundo. Su misión es ayudarle a detectar vulnerabilidades antes de que puedan ser explotadas, reduciendo el riesgo y el impacto de los ciberataques. OpenVAS es un escáner de vulnerabilidades con todas las funciones. Sus capacidades incluyen pruebas no autenticadas, pruebas autenticadas, varios protocolos industriales y de Internet de alto y bajo nivel, ajuste de rendimiento para escaneos a gran escala y un potente lenguaje de programación interno para implementar cualquier tipo de prueba de vulnerabilidad.

**Links de interes:**
- [Sitio web de GVM](https://www.greenbone.net/en/)
- [Sitio web OpenVAS](https://www.openvas.org/)
- [GitHub](https://github.com/greenbone)
- [Documentos oficiales de GVM](https://greenbone.github.io/docs/latest/)

---

### Comience a instalar las dependencias para GVM 22.4.x.

Instalamos dependencias
```
sudo apt update && \
sudo apt -y upgrade && \
sudo apt install -y build-essential && \
sudo apt install -y cmake pkg-config gcc-mingw-w64 git curl \
libgnutls28-dev libxml2-dev libssh-gcrypt-dev libunistring-dev \
libldap2-dev libgcrypt20-dev libpcap-dev libglib2.0-dev libgpgme-dev libradcli-dev libjson-glib-dev \
libksba-dev libical-dev libpq-dev libsnmp-dev libpopt-dev libnet1-dev gnupg gnutls-bin \
libmicrohttpd-dev redis-server libhiredis-dev openssh-client xsltproc nmap \
bison postgresql postgresql-server-dev-all smbclient fakeroot sshpass wget \
heimdal-dev dpkg rsync zip rpm nsis socat libbsd-dev snmp uuid-dev curl gpgsm \
python3 python3-paramiko python3-lxml python3-defusedxml python3-pip python3-psutil python3-impacket \
python3-venv python3-setuptools python3-packaging python3-wrapt python3-cffi python3-redis python3-gnupg \
xmlstarlet texlive-fonts-recommended texlive-latex-extra perl-base xml-twig-tools \
libpaho-mqtt-dev python3-paho-mqtt mosquitto xmltoman doxygen
```

### Configurar rutas de instalación definidas por el usuario de GVM

Los servicios proporcionados por Greenbone Community Edition deben ejecutarse como un usuario y grupo dedicado. Por lo tanto se creará un usuario y un grupo con el mismo nombre `gvm`.

Cree el usuario GVM y agréguelo al grupo sudoers sin iniciar sesión. También agregue su usuario sudo actual al grupo GVM para poder ejecutar gvmd.

```
sudo useradd -r -M -U -G sudo -s /usr/sbin/nologin gvm && \
sudo usermod -aG gvm $USER && su $USER
```

### Definición de variables

A continuación, defina los directorios base, fuente, compilación e instalación.

```
export PATH=$PATH:/usr/local/sbin && export INSTALL_PREFIX=/usr/local && \
export SOURCE_DIR=$HOME/source_gvm && mkdir -p $SOURCE_DIR && \
export BUILD_DIR=$HOME/build_gvm && mkdir -p $BUILD_DIR && \
export INSTALL_DIR=$HOME/install_gvm && mkdir -p $INSTALL_DIR
```

### Importar clave de firma GVM

Para validar la integridad de los archivos fuente descargados, se utiliza GnuPG. Requiere descargar la clave pública de Greenbone Community Signing e importarla al llavero del usuario actual.
```
curl -f -L https://www.greenbone.net/GBCommunitySigningKey.asc -o /tmp/GBCommunitySigningKey.asc && \
gpg --import /tmp/GBCommunitySigningKey.asc
```

Para comprender el resultado de validación de la herramienta gpg, es mejor marcar la clave de firma de la comunidad de Greenbone como totalmente confiable. Edite la clave de firma de GVM para confiar en última instancia.
```
echo "8AE4BE429B60A59B311C2E739823FAA60ED1E580:6:" | gpg --import-ownertrust
```

### Construcción e instalación de los componentes GVM

Descargue y cree las bibliotecas [gvm-libs](https://github.com/greenbone/gvm-libs)

gvm-libs es una biblioteca C que proporciona funciones básicas como análisis XML y comunicación en red. Se utiliza en openvas-scanner , gvmd , gsad y pg-gvm.

```
export GVM_LIBS_VERSION=22.7.3 && \
curl -f -L https://github.com/greenbone/gvm-libs/archive/refs/tags/v$GVM_LIBS_VERSION.tar.gz -o $SOURCE_DIR/gvm-libs-v$GVM_LIBS_VERSION.tar.gz && \
curl -f -L https://github.com/greenbone/gvm-libs/releases/download/v$GVM_LIBS_VERSION/gvm-libs-v$GVM_LIBS_VERSION.tar.gz.asc -o $SOURCE_DIR/gvm-libs-v$GVM_LIBS_VERSION.tar.gz.asc
```

### Verificando el archivo fuente

```
gpg --verify $SOURCE_DIR/gvm-libs-v$GVM_LIBS_VERSION.tar.gz.asc $SOURCE_DIR/gvm-libs-v$GVM_LIBS_VERSION.tar.gz
```

El resultado del último comando debería ser similar a:

```
gpg: Signature made Fri Apr 16 08:31:02 2021 UTC
gpg:                using RSA key 9823FAA60ED1E580
gpg: Good signature from "Greenbone Community Feed integrity key" [ultimate]
```

Una vez que haya confirmado que la firma es buena, proceda a instalar las bibliotecas GVM.

```
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/gvm-libs-v$GVM_LIBS_VERSION.tar.gz && \
mkdir -p $BUILD_DIR/gvm-libs && cd $BUILD_DIR/gvm-libs && \
cmake $SOURCE_DIR/gvm-libs-$GVM_LIBS_VERSION \
  -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX \
  -DCMAKE_BUILD_TYPE=Release \
  -DSYSCONFDIR=/etc \
  -DLOCALSTATEDIR=/var && \
make DESTDIR=$INSTALL_DIR install && \
sudo cp -rv $INSTALL_DIR/* / && \
rm -rf $INSTALL_DIR/*
```

### Construya el administrador de vulnerabilidades de Greenbone (gvmd)
El demonio de gestión de vulnerabilidades de Greenbone (gvmd) es el servicio principal de Greenbone Community Edition. Maneja autenticación, gestión de escaneo, información de vulnerabilidad, informes, alertas, programación y mucho más. Como backend de almacenamiento, utiliza una base de datos PostgreSQL.

A continuación descargue, verifique y cree [Greenbone Vulnerability Manager (GVM)](https://github.com/greenbone/gvmd)

```
export GVMD_VERSION=23.1.0 && \
curl -f -L https://github.com/greenbone/gvmd/archive/refs/tags/v$GVMD_VERSION.tar.gz -o $SOURCE_DIR/gvmd-v$GVMD_VERSION.tar.gz && \
curl -f -L https://github.com/greenbone/gvmd/releases/download/v$GVMD_VERSION/gvmd-$GVMD_VERSION.tar.gz.asc -o $SOURCE_DIR/gvmd-$GVMD_VERSION.tar.gz.asc && \
gpg --verify $SOURCE_DIR/gvmd-$GVMD_VERSION.tar.gz.asc $SOURCE_DIR/gvmd-v$GVMD_VERSION.tar.gz
```
El resultado del último comando debería ser similar a:

```
gpg: Signature made Fri Apr 16 08:31:02 2021 UTC
gpg:                using RSA key 9823FAA60ED1E580
gpg: Good signature from "Greenbone Community Feed integrity key" [ultimate]
```

Extraiga el archivo GVMD descargado y continúe con la instalación.

```
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/gvmd-v$GVMD_VERSION.tar.gz && \
mkdir -p $BUILD_DIR/gvmd && cd $BUILD_DIR/gvmd && \
cmake $SOURCE_DIR/gvmd-$GVMD_VERSION \
  -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX \
  -DCMAKE_BUILD_TYPE=Release \
  -DLOCALSTATEDIR=/var \
  -DSYSCONFDIR=/etc \
  -DGVM_DATA_DIR=/var \
  -DOPENVAS_DEFAULT_SOCKET=/run/ospd/ospd-openvas.sock \
  -DGVM_FEED_LOCK_PATH=/var/lib/gvm/feed-update.lock \
  -DSYSTEMD_SERVICE_DIR=/lib/systemd/system \
  -DPostgreSQL_TYPE_INCLUDE_DIR=/usr/include/postgresql \
  -DLOGROTATE_DIR=/etc/logrotate.d && \
make DESTDIR=$INSTALL_DIR install && \
sudo cp -rv $INSTALL_DIR/* / && \
rm -rf $INSTALL_DIR/*
```

### Construya el asistente de PostgreSQL pg-gvm

Continúe con la descarga y compilación del último asistente de PostgreSQL. PG-GVM es una extensión del servidor PostgreSQL que agrega varias funciones utilizadas por gvmd , por ejemplo, iCalendar y evaluación de rango de host. En versiones anteriores, estas funciones eran administradas directamente por gvmd mientras que pg-gvm usa la administración de extensiones integrada en PostgreSQL.

```
export PG_GVM_VERSION=22.6.1 && \
curl -f -L https://github.com/greenbone/pg-gvm/archive/refs/tags/v$PG_GVM_VERSION.tar.gz -o $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION.tar.gz && \
curl -f -L https://github.com/greenbone/pg-gvm/releases/download/v$PG_GVM_VERSION/pg-gvm-$PG_GVM_VERSION.tar.gz.asc -o $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION.tar.gz.asc && \
gpg --verify $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION.tar.gz.asc $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION.tar.gz
```

Continúe con la instalación del asistente de PostgreSQL.

```
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION.tar.gz && \
mkdir -p $BUILD_DIR/pg-gvm && cd $BUILD_DIR/pg-gvm && \
cmake $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION \
  -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX \
  -DCMAKE_BUILD_TYPE=Release \
  -DPostgreSQL_TYPE_INCLUDE_DIR=/usr/include/postgresql && \
make DESTDIR=$INSTALL_DIR install && \
sudo cp -rv $INSTALL_DIR/* / && \
rm -rf $INSTALL_DIR/*
```

### Instalar NodeJS y Yarn

Puede usar el manejador de versión [nvm](https://github.com/nvm-sh/nvm) para instalar node
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
```

Al ejecutar el comando anteriores, se descarga un script y lo ejecuta. El script clona el repositorio nvm en `~/.nvm` e intenta agregar las líneas fuente del siguiente fragmento al archivo de perfil correcto (`~/.bash_profile`, `~/.zshrc`, `~/.profile`, or `~/.bashrc`). En nuestro casoq ue estamos haciendo esta guía en Ubuntu 22 sería en `~/.bashrc` así que para que surga efecto la modificación tenemos que recargar dicho fichero con el siguiente comando:
```
source ~/.bashrc
```

Luego de eso procedemos a instalar node18 y luego yarn
```
nvm install 18 && npm i -g yarn
```

### Construya el asistente de seguridad Greenbone
Continúe con la descarga y compilación de [Greenbone Security Assistant (GSA)](https://github.com/greenbone/gsa)

```
export GSA_VERSION=22.9.1 && \
curl -f -L https://github.com/greenbone/gsa/archive/refs/tags/v$GSA_VERSION.tar.gz -o $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz && \
curl -f -L https://github.com/greenbone/gsa/releases/download/v$GSA_VERSION/gsa-dist-$GSA_VERSION.tar.gz.asc -o $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz.asc && \
gpg --verify $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz.asc $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz
```

Proceder con la instalación de GSA.

```
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz && \
cd $SOURCE_DIR/gsa-$GSA_VERSION && rm -rf build && \
yarn && yarn build && \
sudo mkdir -p $INSTALL_PREFIX/share/gvm/gsad/web/ && \
sudo cp -r build/* $INSTALL_PREFIX/share/gvm/gsad/web/
```

### Construya el demonio asistente de seguridad de Greenbone

Continúe con la descarga y compilación del [demonio asistente de seguridad de Greenbone (GSAD)](https://github.com/greenbone/gsad)

El servidor web gsad está escrito en el lenguaje de programación C. Sirve contenido estático como imágenes y proporciona una API para la aplicación web. Internamente se comunica con gvmd mediante GMP.

```
export GSAD_VERSION=22.8.0 && \
curl -f -L https://github.com/greenbone/gsad/archive/refs/tags/v$GSAD_VERSION.tar.gz -o $SOURCE_DIR/gsad-$GSAD_VERSION.tar.gz && \
curl -f -L https://github.com/greenbone/gsad/releases/download/v$GSAD_VERSION/gsad-$GSAD_VERSION.tar.gz.asc -o $SOURCE_DIR/gsad-$GSAD_VERSION.tar.gz.asc && \
gpg --verify $SOURCE_DIR/gsad-$GSAD_VERSION.tar.gz.asc $SOURCE_DIR/gsad-$GSAD_VERSION.tar.gz
```

### Construya el módulo OpenVAS Samba

openvas-smb es un módulo auxiliar para openvas-scanner. Incluye bibliotecas (openvas-wmiclient/openvas-wincmd) para interactuar con los sistemas Microsoft Windows a través de la API de Instrumental de administración de Windows y un binario winexe para ejecutar procesos de forma remota en ese sistema.

Es una dependencia opcional de openvas-scanner pero es necesaria para escanear sistemas basados ​​en Windows.

Descargue y cree el módulo [OpenVAS SMB](https://github.com/greenbone/openvas-smb)

```
export OPENVAS_SMB_VERSION=22.5.6 && \
curl -f -L https://github.com/greenbone/openvas-smb/archive/refs/tags/v$OPENVAS_SMB_VERSION.tar.gz -o $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION.tar.gz && \
curl -f -L https://github.com/greenbone/openvas-smb/releases/download/v$OPENVAS_SMB_VERSION/openvas-smb-v$OPENVAS_SMB_VERSION.tar.gz.asc -o $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION.tar.gz.asc && \
gpg --verify $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION.tar.gz.asc $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION.tar.gz
```

### Construya el escáner OpenVAS

Descargue y cree el [escáner openvas (OpenVAS)](https://github.com/greenbone/openvas-scanner)

```
export OPENVAS_SCANNER_VERSION=22.7.7 && \
curl -f -L https://github.com/greenbone/openvas-scanner/archive/refs/tags/v$OPENVAS_SCANNER_VERSION.tar.gz -o $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz && \
curl -f -L https://github.com/greenbone/openvas-scanner/releases/download/v$OPENVAS_SCANNER_VERSION/openvas-scanner-v$OPENVAS_SCANNER_VERSION.tar.gz.asc -o $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz.asc && \
gpg --verify $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz.asc $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz
```

Proceder con la instalación de OpenVAS.

```
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz && \
mkdir -p $BUILD_DIR/openvas-scanner && cd $BUILD_DIR/openvas-scanner && \
cmake $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION \
  -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX \
  -DCMAKE_BUILD_TYPE=Release \
  -DSYSCONFDIR=/etc \
  -DLOCALSTATEDIR=/var \
  -DOPENVAS_FEED_LOCK_PATH=/var/lib/openvas/feed-update.lock \
  -DOPENVAS_RUN_DIR=/run/ospd && \
make DESTDIR=$INSTALL_DIR install && \
sudo cp -rv $INSTALL_DIR/* / && \
rm -rf $INSTALL_DIR/*
```

### Construir ospd-openvas

ospd-openvas es una implementación de servidor OSP que permite a gvmd controlar de forma remota un escáner openvas. Se ejecuta como un demonio y espera solicitudes OSP entrantes de gvmd.

Proceda a descargar [ospd-openvas](https://github.com/greenbone/ospd-openvas)

```
export OSPD_OPENVAS_VERSION=22.6.2 && \
curl -f -L https://github.com/greenbone/ospd-openvas/archive/refs/tags/v$OSPD_OPENVAS_VERSION.tar.gz -o $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION.tar.gz && \
curl -f -L https://github.com/greenbone/ospd-openvas/releases/download/v$OSPD_OPENVAS_VERSION/ospd-openvas-v$OSPD_OPENVAS_VERSION.tar.gz.asc -o $SOURCE_DIR/ospd-openvas-v$OSPD_OPENVAS_VERSION.tar.gz.asc && \
gpg --verify $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION.tar.gz.asc $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION.tar.gz
```

Extraiga archivos e inicie la instalación.

```
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION.tar.gz && \
cd $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION && \
mkdir -p $INSTALL_DIR/ospd-openvas && \
python3 -m pip install --root=$INSTALL_DIR/ospd-openvas --no-warn-script-location . && \
sudo cp -rv $INSTALL_DIR/ospd-openvas/* /
```

### Construir notus-scanner

notus-scanner se utiliza para detectar productos vulnerables mediante la evaluación de la información interna del sistema recopilada por openvas-scanner. Se comunica con openvas-scanner y ospd-openvas a través de MQTT. Se ejecuta como un demonio.

Primero descargue y verifique el nuevo notus-scanner.

```
export NOTUS_VERSION=22.6.2 && \
curl -f -L https://github.com/greenbone/notus-scanner/archive/refs/tags/v$NOTUS_VERSION.tar.gz -o $SOURCE_DIR/notus-scanner-$NOTUS_VERSION.tar.gz && \
curl -f -L https://github.com/greenbone/notus-scanner/releases/download/v$NOTUS_VERSION/notus-scanner-v$NOTUS_VERSION.tar.gz.asc -o $SOURCE_DIR/notus-scanner-$NOTUS_VERSION.tar.gz.asc && \
gpg --verify $SOURCE_DIR/notus-scanner-$NOTUS_VERSION.tar.gz.asc $SOURCE_DIR/notus-scanner-$NOTUS_VERSION.tar.gz
```

Una vez verificado, proceda e instale.

```
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/notus-scanner-$NOTUS_VERSION.tar.gz && \
cd $SOURCE_DIR/notus-scanner-$NOTUS_VERSION && \
mkdir -p $INSTALL_DIR/notus-scanner && \
python3 -m pip install --root=$INSTALL_DIR/notus-scanner --no-warn-script-location . && \
sudo cp -rv $INSTALL_DIR/notus-scanner/* /
```

### Instalar GVM tools

Las herramientas de gestión de vulnerabilidades de Greenbone, o gvm-tools para abreviar, son una colección de herramientas que ayudan a controlar las instalaciones de Greenbone Community Edition o los dispositivos Greenbone Enterprise de forma remota.

Básicamente, las herramientas ayudan a acceder a los protocolos de comunicación Greenbone Management Protocol (GMP) y Open Scanner Protocol (OSP).

gvm-tools son opcionales y no son necesarios para una pila GVM funcional.

La última versión de gvm-tools se puede instalar para cada usuario a través de la herramienta de instalación estándar de Python pip.

Alternativamente, para instalarlo en todo el sistema sin ejecutar pip como usuario root, se pueden usar los siguientes comandos:

```
mkdir -p $INSTALL_DIR/gvm-tools && \
python3 -m pip install --root=$INSTALL_DIR/gvm-tools --no-warn-script-location gvm-tools && \
sudo cp -rv $INSTALL_DIR/gvm-tools/* /
```

### Configurar Redis

En cuanto a la arquitectura , el escáner ( openvas-scanner y ospd-openvas ) utiliza el almacenamiento de clave/valor de Redis para manejar la información de VT y los resultados del escaneo.

A continuación, configure redis para la instalación predeterminada de GVM.

```
sudo cp $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION/config/redis-openvas.conf /etc/redis/ && \
sudo chown redis:redis /etc/redis/redis-openvas.conf && \
echo "db_address = /run/redis-openvas/redis.sock" | sudo tee -a /etc/openvas/openvas.conf
```

Agregue redis al grupo GVM.

```
sudo usermod -aG redis gvm
```

Inicie el servidor redis y habilítelo como servicio de inicio.

```
sudo systemctl start redis-server@openvas.service && \
sudo systemctl enable redis-server@openvas.service
```

Proceda a configurar los permisos correctos.

```
sudo mkdir -p /var/lib/notus && \
sudo mkdir -p /run/gvmd && \
sudo mkdir -p /var/lib/notus && \
sudo mkdir -p /run/notus-scanner && \
sudo mkdir -p /run/gvmd && \
sudo chown -R gvm:gvm /var/lib/gvm && \
sudo chown -R gvm:gvm /var/lib/openvas && \
sudo chown -R gvm:gvm /var/lib/notus && \
sudo chown -R gvm:gvm /var/log/gvm && \
sudo chown -R gvm:gvm /run/gvmd && \
sudo chmod -R g+srw /var/lib/gvm && \
sudo chmod -R g+srw /var/lib/openvas && \
sudo chmod -R g+srw /var/log/gvm
```

También debe ajustar los permisos para la sincronización del feed.

```
sudo chown gvm:gvm /usr/local/sbin/gvmd && \
sudo chmod 6750 /usr/local/sbin/gvmd
```

Validación de feeds.

```
export GNUPGHOME=/tmp/openvas-gnupg && \
mkdir -p $GNUPGHOME && \
export OPENVAS_GNUPG_HOME=/etc/openvas/gnupg && \
sudo mkdir -p $OPENVAS_GNUPG_HOME && \
sudo cp -r /tmp/openvas-gnupg/* $OPENVAS_GNUPG_HOME/ && \
sudo chown -R gvm:gvm $OPENVAS_GNUPG_HOME
```

OpenVAS se lanzará desde un proceso ospd-openvas. Actualice la ruta segura en el archivo sudoers en consecuencia.

```
sudo visudo
```

El comando anterior abríra un editor de texto y debemos agregar al final lo siguiente:
```
# allow users of the gvm group run openvas
%gvm ALL = NOPASSWD: /usr/local/sbin/openvas
```

### Configurar Mosquitto broker

El corredor Mosquitto MQTT se utiliza para la comunicación entre ospd-openvas, openvas-scanner y notus-scanner.

```
sudo systemctl start mosquitto.service && \
sudo systemctl enable mosquitto.service && \
echo "mqtt_server_uri = localhost:1883\ntable_driven_lsc = yes" | sudo tee -a /etc/openvas/openvas.conf
```

### Configurar la base de datos PostgreSQL
Para obtener información adicional, consulte la referencia greenbone/gvmd [INSTALL.md](https://github.com/greenbone/gvmd/blob/main/INSTALL.md). Primero asegúrese de que se hayan instalado las dependencias requeridas (consulte Requisitos previos ). Antes de que podamos agregar el usuario de PostgreSQL, asegúrese de que el servicio esté en funcionamiento.

```
sudo systemctl start postgresql@15-main.service
```

Proceda a crear un usuario y una base de datos de Postgres.

```
sudo -u postgres bash
cd
createuser -DRS gvm
createdb -O gvm gvmd
psql gvmd
```

Configure los permisos correctos y cree extensiones de base de datos.

```
# Una vez dentro de la CLI de postgresql ejecutar los siguientes comandos
create role dba with superuser noinherit;
grant dba to gvm;
create extension "uuid-ossp";
create extension "pgcrypto";
exit
exit
```

### Crear administrador de GVM

Para acceder y configurar los datos de vulnerabilidad, es necesario crear un usuario administrador. Este usuario puede iniciar sesión a través de la interfaz web de Greenbone Security Assistant (GSA). Tendrán acceso a todos los datos y luego se configurarán para actuar como propietario de importación de feeds.

Antes de crear el administrador, asegúrese de salir de la sesión de Postgres y de volver a cargar la caché del cargador dinámico.

```
sudo ldconfig
```

Una vez que haya recargado la caché del cargador dinámico, continúe con la creación del usuario.

```
/usr/local/sbin/gvmd --create-user=admin --password=admin
```

> No olvides cambiar la contraseña más tarde.


### Configuración del propietario de la importación de feeds

Ciertos recursos que anteriormente formaban parte del código fuente de gvmd ahora se envían a través del feed. Un ejemplo es la configuración de escaneo “Completo y Rápido”.

Actualmente, cada recurso necesita un propietario para aplicar los permisos y gestionar el acceso a los recursos.

Por lo tanto, gvmd solo creará estos recursos si se configura un propietario de importación de feeds. Aquí, el usuario administrador creado previamente se utilizará como propietario de importación de feeds.

```
/usr/local/sbin/gvmd --modify-setting 78eceaec-3385-11ea-b237-28d24461215b --value `/usr/local/sbin/gvmd --get-users --verbose | grep admin | awk '{print $2}'`
```

### Instalar greenbone-feed-sync

Instale el nuevo greenbone-feed-sync que reemplazó el antiguo enfoque de sincronizar los datos (VT, SCAP, CERT y GVMD) individualmente.

```
mkdir -p $INSTALL_DIR/greenbone-feed-sync && \
python3 -m pip install --root=$INSTALL_DIR/greenbone-feed-sync --no-warn-script-location greenbone-feed-sync && \
sudo cp -rv $INSTALL_DIR/greenbone-feed-sync/* /
```

### Sincronización de Greenbone Feed

```
sudo /usr/local/bin/greenbone-feed-sync
```

### Configurar sistema
Cree el script de servicio systemd para ospd-openvas. Para Ubuntu, establezca la dirección específica del intermediario mqtt para el host del servidor.

```
cat << EOF > $BUILD_DIR/ospd-openvas.service
[Unit]
Description=OSPd Wrapper for the OpenVAS Scanner (ospd-openvas)
Documentation=man:ospd-openvas(8) man:openvas(8)
After=network.target networking.service redis-server@openvas.service
Wants=redis-server@openvas.service
ConditionKernelCommandLine=!recovery

[Service]
Type=exec
User=gvm
Group=gvm
RuntimeDirectory=ospd
RuntimeDirectoryMode=2775
PIDFile=/run/ospd/ospd-openvas.pid
ExecStart=/usr/local/bin/ospd-openvas --foreground --unix-socket /run/ospd/ospd-openvas.sock --pid-file /run/ospd/ospd-openvas.pid --log-file /var/log/gvm/ospd-openvas.log --lock-file-dir /var/lib/openvas --socket-mode 0o770 --mqtt-broker-address 192.168.0.1 --mqtt-broker-port 1883 --notus-feed-dir /var/lib/notus/advisories
SuccessExitStatus=SIGKILL
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF
```

Ahora copie el script de inicio en el directorio del administrador de su sistema.

```
sudo cp $BUILD_DIR/ospd-openvas.service /etc/systemd/system/
```

Cree el script de servicio systemd para notus-scanner.

```
cat << EOF > $BUILD_DIR/notus-scanner.service
[Unit]
Description=Notus Scanner
Documentation=https://github.com/greenbone/notus-scanner
After=mosquitto.service
Wants=mosquitto.service
ConditionKernelCommandLine=!recovery

[Service]
Type=exec
User=gvm
RuntimeDirectory=notus-scanner
RuntimeDirectoryMode=2775
PIDFile=/run/notus-scanner/notus-scanner.pid
ExecStart=/usr/local/bin/notus-scanner --foreground --products-directory /var/lib/notus/products --log-file /var/log/gvm/notus-scanner.log
SuccessExitStatus=SIGKILL
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF
```

Finalmente copie el último script de inicio en el directorio del administrador de su sistema.

```
sudo cp $BUILD_DIR/notus-scanner.service /etc/systemd/system/
```
Luego configure los scripts de inicio. Primero configure el script de inicio de Greenbone Manager.

```
cat << EOF > $BUILD_DIR/gvmd.service
[Unit]
Description=Greenbone Vulnerability Manager daemon (gvmd)
After=network.target networking.service postgresql.service ospd-openvas.service
Wants=postgresql.service ospd-openvas.service
Documentation=man:gvmd(8)
ConditionKernelCommandLine=!recovery

[Service]
Type=exec
User=gvm
Group=gvm
PIDFile=/run/gvmd/gvmd.pid
RuntimeDirectory=gvmd
RuntimeDirectoryMode=2775
ExecStart=/usr/local/sbin/gvmd --foreground --osp-vt-update=/run/ospd/ospd-openvas.sock --listen-group=gvm
Restart=always
TimeoutStopSec=10

[Install]
WantedBy=multi-user.target
EOF
```

Copie el script de inicio de la carpeta de compilación al directorio del administrador del sistema.

```
sudo cp $BUILD_DIR/gvmd.service /etc/systemd/system/
```

Una vez guardado el primer script de inicio, proceda a crear el script para Greenbone Security Assistant (GSA). Recuerde definir su dirección IP para GSA.

```
cat << EOF > $BUILD_DIR/gsad.service
[Unit]
Description=Greenbone Security Assistant daemon (gsad)
Documentation=man:gsad(8) https://www.greenbone.net
After=network.target gvmd.service
Wants=gvmd.service

[Service]
Type=exec
User=gvm
Group=gvm
RuntimeDirectory=gsad
RuntimeDirectoryMode=2775
PIDFile=/run/gsad/gsad.pid
ExecStart=/usr/local/sbin/gsad --foreground --listen=192.168.0.1 --port=9392 --http-only
Restart=always
TimeoutStopSec=10

[Install]
WantedBy=multi-user.target
Alias=greenbone-security-assistant.service
EOF
```

Copie el script de inicio al directorio del sistema.

```
sudo cp $BUILD_DIR/gsad.service /etc/systemd/system/
```

### Habilitar e iniciar servicios
Para habilitar los scripts de inicio creados, vuelva a cargar el demonio de control del sistema.

```
sudo systemctl daemon-reload
```

Una vez que hayas recargado el demonio procede a habilitar cada uno de los servicios.

```
sudo systemctl enable notus-scanner
sudo systemctl enable ospd-openvas
sudo systemctl enable gvmd
sudo systemctl enable gsad
```

Luego inicie cada servicio.

```
sudo systemctl start ospd-openvas
sudo systemctl start notus-scanner
sudo systemctl start gvmd
sudo systemctl start gsad
```
