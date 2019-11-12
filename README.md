# Pasos para crear una entidad de certificadora.
Una entidad certificadora o autoridad de certificación es una entidad de confianza encargada de emitir y revocar los certificados digítales.

## Entidad certificadora
El primer paso que haremos es organizar el escenario. Esto preparar una estructura de directorios que nos permitallevar el seguimiento de los certificados emitidos.

Para esto vamos a crear una carpeta en la cual vamos a guardar la entidad y los certificados emitidos y revocados. Para todo este proceso es preferible tener privilegios de super usuario.

> `sudo -i`<br>
> `mkdir /root/ca`

posteriormente vamos a ingresar a la carpeta y crearemos todos los directorios necesarios.
> `cd /root/ca` <br>
> `mkdir certs crl newcerts private` 

En este proceso estaremos cambiando los permisos de los archivos que vamos creando.
> `chmod 700 private` <br>
> `touch index.txt` <br>
> `echo 1000 > serial` <br>

Una vez términada la estructura de directorios, procederemos a la creación del archivo de configuración de openssl. Para este se puede copiar y pegar la configuración que se encuetra en el archivo que llamaremos ***openssl.cnf***

[openssl.cnf](./openssl.cnf)

Lugo se debe crear la clave privada para la entidad certificadora
>`openssl genrsa -aes256 -out private/ca.key.pem 4096`

Al ejecutar el comando aparecera un mensaje pidiendo la frase o palabra de seguridad para la clave.

> `Enter pass phrase for ca.key.pem:` *********<br>
> `Verifying - Enter pass phrase for ca.key.pem:` *********<br>

Por último editamos los privilegios de los archivos que acabamos de crear.

> `chmod 400 private/ca.key.pem`

Finalmente, creamos el certificado para la entidad autoritaria.
> `openssl req -config openssl.cnf \`<br>
      `-key private/ca.key.pem \` <br>
      `-new -x509 -days 7300 -sha256 -extensions v3_ca \`<br>
      `-out certs/ca.cert.pem`
      
Al ejecutar el comando se hacen algunas preguntas como la ubicación, Localidad, Nombre de la compañia entre otras. Luego editamos los privilegios del archivo resultante.
> ` chmod 444 certs/ca.cert.pem`

## Certificado intermedio

Al finalizar la creación de la entidad certificadora, se necesita un certificado intermedio adicional. Esto con el objetivo de brindar más seguridad. Si la clave intermedia esta comprometida, la entidad emisora de los certificados puede revocar el intermedio y crear uno nuevo.

Como se hizo anteriormente, aquí tambien es necesario organizar una estructura de directorios, el cual se puede lograr ejecutando los siguientes comandos.

> `mkdir /root/ca/intermediate`<br>
> `cd /root/ca/intermediate`<br>
> `mkdir certs crl csr newcerts private`<br>
> `chmod 700 private` <br>
> `touch index.txt` <br>
> `echo 1000 > serial` <br>

Una vez creada la estructura de directorios, procedemos a crear el archivo configuración para openssl en la entidad intermedia. Esto se hace creando un archivo ***openssl.cnf*** dentro de la carpeta creada anteriormete ***intermediate***.

[intermediate/openssl.cnf](./intermediate/openssl.cnf)

las unicas diferencias entre este archivo y el creado en la entidad certificadora radica en estas 5 lineas.
> `dir             = /root/ca/intermediate`<br>
> `private_key     = $dir/private/intermediate.key.pem`<br>
> `certificate     = $dir/certs/intermediate.cert.pem`<br>
> `crl             = $dir/crl/intermediate.crl.pem`<br>
> `policy          = policy_loose`<br>

Finalizado el archivo de configuración, vamos a crear la calve privada para la entidad intermedia. Esta se crea de manera muy similar al anterior y también se le cambia los permisos del archivo resultantes.

> `cd /root/ca/`<br>
> `openssl genrsa -aes256 \` <br>
      `-out intermediate/private/intermediate.key.pem 4096`
> `chmod 400 intermediate/private/intermediate.key.pem`

Al crear la llave privada, se puede proceder a la creación del certificado para la entidad intermedia.
> `openssl req -config intermediate/openssl.cnf -new -sha256 \`<br>
      `-key intermediate/private/intermediate.key.pem \` <br>
      `-out intermediate/csr/intermediate.csr.pem`

Este certificado debe ser firmado y debe ser valido por un periodo más corto que el certificado de la entidad raiz. Esto se hace mdeiante el siguiente comando.

> `openssl ca -config openssl.cnf -extensions v3_intermediate_ca \` <br>
      `-days 3650 -notext -md sha256 \`<br>
      `-in intermediate/csr/intermediate.csr.pem \`<br>
      `-out intermediate/certs/intermediate.cert.pem`<br>
> `chmod 444 intermediate/certs/intermediate.cert.pem`

Al ejecutar el comando nos pedira confirmación para firmar el certificado.
Para verificar que todo lo que se ha hecho esta bien, se procede a correr el siguiente comando:

> `openssl verify -CAfile certs/ca.cert.pem \`<br>
      `intermediate/certs/intermediate.cert.pem`

lo cual deberá tener una salida como esta.
> `intermediate.cert.pem: OK`

**Nota:** Se debe tener en cuenta que los datos que se ingresaron al crear el certificado de la entidad certificadora y la intermedia deben ser iguales.

## Solicitar certificado desde un servidor web

Desde una máquina diferente que hará las veces de servidor web, crearemos una estructura de directorios parecida a la vista anteriormente.

> `cd /root/ca` <br>
> `mkdir certs crl newcerts private`<br>
> `chmod 700 private` <br>
> `touch index.txt` <br>
> `echo 1000 > serial` <br>

Cuando se haya terminado de crear la estructura de directorios, procedemos a crear un archivo de configuración parecido al que creamos anteriormente.

[openssl.cnf](./server/openssl.cnf)

Luego generamos la clave privada:

>`openssl genrsa -aes256 -out private/prueba.key.pem 4096`<br>
>`chmod 400 private/ca.key.pem`

Una vez tenemos la clave privada creamos una petición de certificado con el siguiente comando.
> `openssl req -new -key private/prueba.key.pem -out csr/prueba.csr.pem`

Luego lo mandamos al servidor de la autoridad certificadora. Normalmente este se envía por un mensaje de correo electrónico, sin mebargo, este debe estar firmado y encriptado con GPG.
Para el proposito de este documento se va a hacer la transferencia mediante scp, que es el método de transferencia de ssh.

## Crear el certificado desde la entidad con autoridad

Una vez la petición se ecuntre en el servidor de la entidad autoritaria, generamos el certificado de la siguiente manera:

> `openssl ca -config intermediate/openssl.cnf \`<br>
      `-extensions server_cert -days 375 -notext -md sha256 \`<br>
      `-in intermediate/csr/prueba.csr.pem \`<br>
      `-out intermediate/certs/prueba.cert.pem`<br>
> `chmod 444 intermediate/certs/prueba.cert.pem`

Posteriormente se envía el certificado resultante al servidor.

## Implementación

Una vez tengamos el certificado en nuestra máquina que hará las veces de servidor, se procede a verificar que el certificado recibido sea correspondiente a la clave privada que poseemos. Se ejecutan los siguientes comandos y el la salida de ambos deben ser iguales.

> `openssl rsa -in /root/ca/private/prueba.key.pem -noout -modulus`
> `openssl x509 -in prueba.cert.pem -noout -modulus`

Antes de comenzar movemos los archivos ***prueba.cert.pem*** y ***prueba.key.pem*** a la dirección ***/etc/apache2/cert/***.

Al hacer la verificación, podemos proceder a hacer la configuración del apache. Primeros activamos el modulo ssl en apache con el siguiente comando.

> `sudo a2enmod ssl`<br>

Luego editamos la configuración del archivo apache que se suele encontrar en ***/etc/apache2/sites-available/default-ssl.cnf***. Editamos estas líneas:
> `ServerAdmin email@example.net`

añadimos el dominio. De estar haciendo este proceso de manera local, entonces se colocara la ip con la que está trabajando.

> `ServerName ADD_YOUR_IP_OR_DOMAIN_NAME_HERE`

Por último editmos estas lineas:

> `SSLCertificateFile    /etc/apache2/cert/apache.crt`<br>
> `SSLCertificateKeyFile /etc/apache2/cert/apache.key`

por último añadimos la configuración al apache y reiniciamos el servicio:

>`a2ensite default-ssl.conf`<br>
>`service apache2 restart`
