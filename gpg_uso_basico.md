# GPG
GPG (GNU Privacy Guard) es una implementación gratuita y de código abierto del estándar OpenPGP (Pretty Good Privacy), que es un sistema de cifrado y autenticación de datos. GPG se utiliza comúnmente para cifrar y firmar correos electrónicos, archivos y otros datos para proteger la privacidad y la seguridad de la información.

Aquí hay algunas funciones clave de GPG:

1. **Cifrado de Correos Electrónicos y Archivos:** GPG permite cifrar correos electrónicos y archivos de manera que solo las personas autorizadas, que poseen la clave de descifrado correspondiente, puedan acceder al contenido.

2. **Firma Digital:** GPG permite firmar digitalmente mensajes y archivos para garantizar su autenticidad. La firma digital confirma que el contenido no ha sido alterado y proviene de la persona que afirma ser el remitente.

3. **Verificación de Identidad:** Las claves GPG están asociadas con identidades de usuario, generalmente en forma de direcciones de correo electrónico. Cuando recibes datos cifrados o firmados, puedes verificar la identidad del remitente utilizando las claves GPG.

4. **Administración de Claves Públicas y Privadas:** GPG utiliza un par de claves: una pública y una privada. La clave pública se comparte con otros, mientras que la clave privada se mantiene en secreto. Las claves se pueden gestionar utilizando el software GPG.

5. **Red de Confianza:** GPG utiliza un sistema de red de confianza en el que los usuarios pueden firmar las claves públicas de otros para indicar que confían en la autenticidad de esas claves. Esto crea una red de confianza que puede ayudar a verificar la identidad de usuarios desconocidos.

GPG es una herramienta poderosa y ampliamente utilizada para la seguridad de la información y la protección de la privacidad en línea. Se utiliza en diversos contextos, desde la comunicación por correo electrónico segura hasta la verificación de la integridad de software descargado.

## Comandos principales

#### Crear una clave GPG Privada/publica
```
gpg --full-generate-key
```

Este comando te va a realizar una serie de preguntas o te solicitara que elijas entre algunas opciones antes de generar tu clave, alguna de esas preguntas u opciones son las siguientes:
  
  - **Seleccione qué tipo de clave desea** Acá puedes elegir la primera (por defecto) que es RSA
  
  - **Las claves RSA pueden tener entre 1024 y 4096 bits de longitud. ¿Qué tamaño de clave quieres?** por defecto es 3072 se recomienda que sea 4096

  - **Especifique durante cuánto tiempo debe ser válida la clave.** Esto depende de la utilidad que le quieras dar a la clave, si va a ser tu clave por defecto para uso diario puede elegir 0 que significa que no se caduca nunca

#### Subir tu clave publica a un servidor de claves GPG
Una vez que has generado tu clave GPG, puedes subirla a un servidor de claves GPG para que otros puedan encontrar y verificar tu clave pública.

**Encuentra tu ID de clave GPG:**
Antes de subir tu clave, necesitas conocer la ID de tu clave GPG. Puedes encontrarla ejecutando el siguiente comando en la terminal:
```
gpg -k
```

El comando anterior va a imprimir algo similar a lo siguiente:
```
root@server_gpg:~# gpg -k
/root/.gnupg/pubring.kbx
------------------------
pub   rsa4096 2023-12-28 [SC]
      99BA7C8339F7B23A859EB8DB6E52FABD4A81E25B
uid           [ultimate] User Demo (Clave para realizar pruebas) <user.demo@gmail.com>
sub   rsa4096 2023-12-28 [E]
```

Si tienes más de una clave vas a ver una salida más amplia, de la salida anterior el ID de la clave sería `99BA7C8339F7B23A859EB8DB6E52FABD4A81E25B` el cual llamaremos `TU_ID_DE_CLAVE` ese ID es el que podemos compartir con otros ya que es el publio

**Sube tu clave al servidor:**
Utiliza el siguiente comando para subir tu clave al servidor de claves GPG:

```
gpg --keyserver keys.openpgp.org --send-keys TU_ID_DE_CLAVE
```

Una vez que enviar tu clave a **keys.openpgp.org** te debería de llegar un mail con un link al cual tienes que ingresar para poder activar tu clave en el servidor de claves para que otras personas puedan descargarse tu clave pública

#### Descargar claves públicas de otros usuarios
Pide a otros usuarios su ID de clave pública y utiliza el siguiente comando:
```
gpg --recv-keys ID_DE_CLAVE
```

### Firmar claves públicas en las que confias
Verifica la identidad: Antes de firmar una clave, asegúrate de que la persona sea quien dice ser. Verifica de la forma más fiable posible que el el ID de la clave pública de la
otra persona es realmente esa. Chequea todo el ID.

```
gpg --local-user ID_DE_CLAVE_CON_QUE_FIRMAS --sign-key ID_DE_CLAVE
```

**NOTA:** se usa la bandera `--local-user` en caso de que tengas varias claves en su sistemas y queras especificar con cual de ellas quieres firmar la clave de terceros

**Opcional: Subir la Clave Firmada:**
Si deseas compartir tu firma con otros, puedes subir la clave firmada a un servidor de claves GPG:

```
gpg --keyserver keys.openpgp.org --send-keys ID_DE_CLAVE
```

#### Como agregar nuevos Users IDs (correos) a una clave GPG existente
Antes de agregar nuevos User IDs, puedes visualizar la información actual de la clave utilizando el comando `gpg --list-keys`. Esto te proporcionará la ID de la clave que estás a punto de modificar.

**Editar la Clave:**
Utiliza el comando `gpg --edit-key` para editar la clave. Reemplaza `ID_DE_LA_CLAVE` con la ID de la clave que deseas modificar.

```
gpg --edit-key ID_DE_LA_CLAVE
```

**Agregar un Nuevo User ID:**
En el entorno de edición, puedes agregar un nuevo User ID utilizando el comando `adduid`

Sigue las instrucciones para proporcionar la información del nuevo User ID, que generalmente incluirá el nombre, la dirección de correo electrónico y cualquier comentario adicional.

**Confirma los Cambios:**
Después de agregar el nuevo User ID, confirma los cambios utilizando el comando `save`

Esto guardará los cambios y saldrá del entorno de edición.

**Opcional: Firmar el User ID:**
Si deseas, puedes firmar el nuevo User ID para indicar que confías en su autenticidad. Dentro del entorno de edición, selecciona el User ID utilizando el comando `uid` el cual le listara todos los User id existente, luego de eso use el comando nuevamente indicando el número del User ID el cual quiere firma la identidad utilizando el comando `sign`
```
uid NUMERO_DE_SECUENCIA
sign
save
```

#### Como exportar mi clave privada
Antes de exportar, puedes visualizar la información actual de tu clave utilizando el comando `gpg -K`. Esto te proporcionará los IDs de las claves privadas que tienes en tu sistema.

Para exportar tu clave GPG, puedes utilizar el siguiente comando:
```
gpg -o mi_clave_privada.gpg -a --export-secret-keys ID_DE_LA_CLAVE
```

El comando anterior va a generar un fichero llamado `mi_clave_privada.gpg` en el directorio actual que va a contener tu clave privada. Asegurate de guardar bien ese fichero ya que contiene tu clave privada y es sumamente sensible, no puedes compartirlo con nadie.