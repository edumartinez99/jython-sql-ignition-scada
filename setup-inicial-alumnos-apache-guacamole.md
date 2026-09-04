# Setup inicial alumnos - APACHE GUACAMOLE

Accedemos al servidor de Apache Guacamole: [https://lab.eduardo-martinez.es/](https://lab.eduardo-martinez.es/)

* Usuario: `alumno01`
* Password: `PasswordAlumno01!`

La primera vez puede salir este Login:

<figure><img src=".gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

* Usuario: `abc`
* Password: `abc`

Se abre el escritorio.

Abrimos el navegador en el menú horizontal de la parte inferior:

<figure><img src=".gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

Insertamos la URL del Gateway: [http://35.204.54.113:8088/](http://35.204.54.113:8088/)

<figure><img src=".gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

Iniciamos sesión:

* Usuario: `alumno01`
* Password: `AlumnoPass01!`



Pinchamos en Get Designer

<figure><img src=".gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>

\
\
Pinchamos en Download for Linux:

<figure><img src=".gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>



Abrimos el explorador de carpetas en el menú horizontal de la parte inferior.

Nos vamos a Downloads/

<figure><img src=".gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>



Hacemos click derecho y le damos a Open in Terminal here:<br>

<figure><img src=".gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>

Ejecutamos:

* `cd Downloads/`
* `tar -xvf designerlauncher.tar.gz`

<figure><img src=".gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>



Cerramos terminal, abrimos la nueva carpeta y ejecutamos el Designer:

<figure><img src=".gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure>

Abrimos el Designer que viene por defecto e iniciamos sesión:

<figure><img src=".gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

* Usuario: `alumno01`&#x20;
* Password: `AlumnoPass01!`

<figure><img src=".gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

Seleccionamos `prj-alumno01`

<figure><img src=".gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

Comprobamos el acceso a la DB con Tools > Database Query Browser:

<figure><img src=".gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

Accedemos a [https://dbeaver.io/](https://dbeaver.io/)

Descargamos para linux

<figure><img src=".gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

Abrimos el explorador de carpetas en el menú horizontal de la parte inferior.

Nos vamos a Downloads/

Hacemos click derecho y le damos a Open in Terminal here:

<figure><img src=".gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

Ejecutamos:

* `cd Downloads/`
* `tar -xvf dbeaver-ce-26.2.0-linux-x86_64.tar.gz`

Y abrimos dbeaver.

* Elegimos la configuración preferida
* Le damos a Nueva conexión arriba a la izquierda

<figure><img src=".gitbook/assets/image (56).png" alt=""><figcaption></figcaption></figure>

* Elegimos Postgres y añadimos la configuración
  * Host: `postgres-maker-db`
  * Database:   `sandbox_db_01`
  * Username:   `ignition_user`
  * Password: `DBPassword123!`

<figure><img src=".gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

* Test Connection y descargar drivers
* Revisamos que haya datos

<figure><img src=".gitbook/assets/image (58).png" alt=""><figcaption></figcaption></figure>
