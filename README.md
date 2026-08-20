---
description: Setup y Validación del Entorno de Trabajo LOC
---

# Setup inicial alumnos - LOCAL

## Guía del Alumno — Sesión 0: Setup y Validación del Entorno de Trabajo

* **Curso:** SQL y Python/Jython para Scripting en Ignition (SCADA)
* **Objetivo de la sesión:** Configurar, replicar y validar el entorno de laboratorio (tanto en local mediante contenedores como en el servidor Cloud de prácticas) con la base de datos PostgreSQL poblada con los esquemas reales del proyecto.

***

### 1. Herramientas Necesarias

Antes de comenzar, asegúrate de tener instalado en tu equipo:

1. **Docker Desktop** (con Docker Compose v2) habilitado y en ejecución (en caso de trabajar en local). [https://docs.docker.com/desktop/setup/install/windows-install/](https://docs.docker.com/desktop/setup/install/windows-install/)\
   ![](.gitbook/assets/image.png)<a href="https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe?utm_source=docker&#x26;utm_medium=webreferral&#x26;utm_campaign=docs-driven-download-win-amd64&#x26;_gl=1*o16p7h*_gcl_au*MTM0MDM3MDU3My4xNzg3MjM2Mjcx*_ga*Nzg5MTk5MzU3LjE3ODcyMzYyNzE.*_ga_XJWPQMJYHQ*czE3ODcyMzYyNzAkbzEkZzEkdDE3ODcyMzYyOTUkajM1JGwwJGgw" class="button secondary">Docker Desktop for Windows - x86_64</a>\
   ejecutamos el exe\
   ![](<.gitbook/assets/image (1).png>)\
   per user\
   esperamos a que instale
2. **Cliente de Base de Datos** (opcional pero recomendado: DBeaver, pgAdmin o extensión de VS Code) para inspeccionar PostgreSQL.\
   ![](<.gitbook/assets/image (3).png>)\
   descargar la versión zip\
   extraer todo\
   ![](<.gitbook/assets/image (4).png>)\
   el dbeaver.exe ya nos ejectua el cliente\
   seguimos la configuración inicial a nuestro gusto\
   ![](<.gitbook/assets/image (5).png>)

***

### 2. Opción A: Despliegue del Entorno Completo en Local (Docker)

Esta opción te permite tener una réplica exacta del servidor de formación en tu propia máquina para desarrollar y practicar fuera de las sesiones online.

```
                      ARQUITECTURA DEL ENTORNO LOCAL
                      
 ╔═══════════════════════════════════════════════════════════════════════════╗
 ║                         TU ORDENADOR (HOST LOCAL)                         ║
 ║                                                                           ║
 ║  [ Ignition Designer Launcher ]                                           ║
 ║             │                                                             ║
 ║             ▼ (http://localhost:8088)                                     ║
 ║  ╔═════════════════════════════════════════════════════════════════════╗  ║
 ║  ║                   DOCKER ENGINE / DOCKER COMPOSE                    ║  ║
 ║  ║                                                                     ║  ║
 ║  ║  ┌──────────────────────────────┐  ┌─────────────────────────────┐  ║  ║
 ║  ║  │      ignition-gateway        │  │         postgres-db         │  ║  ║
 ║  ║  │  (Ignition 8.1.33 / Jython)  │  │  (PostgreSQL 15 - Alpine)   │  ║  ║
 ║  ║  │  Puerto: 8088 / 8043         │  │  Puerto: 5432               │  ║  ║
 ║  ║  └──────────────┬───────────────┘  └──────────────┬──────────────┘  ║  ║
 ║  ║                 │                                 │                 ║  ║
 ║  ║                 └──── JDBC: SandboxDB ────────────┘                 ║  ║
 ║  ╚═════════════════════════════════════════════════════════════════════╝  ║
 ╚═══════════════════════════════════════════════════════════════════════════╝
```

***

#### Paso A.1: Crear el archivo `docker-compose.yml`

Crea una carpeta en tu ordenador llamada `ignition-local-lab` y dentro de ella un archivo denominado `docker-compose.yml` con el siguiente contenido:\
\
![](<.gitbook/assets/image (18).png>)

\
\
<img src=".gitbook/assets/image (19).png" alt="" data-size="original">\
\
\
![](<.gitbook/assets/image (20).png>)\
\
\
\
\
\
\
![](<.gitbook/assets/image (21).png>)

instalar drives

test connection

<figure><img src=".gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

```

      - IGNITION_LICENSE_KEY=2K22-F3NR
      - IGNITION_ACTIVATION_TOKEN=eyJrdHkiOiJSU0EiLCJraWQiOiI4NWIwYmI0Yy1hYTNlLTQzNzEtODI3OC0wMDNiOTRkOTNhNzgiLCJ1c2UiOiJzaWciLCJhbGciOiJSUzI1NiIsIm4iOiJ0S3hXdzU3Y1p6cGVBdWFnTFFJRnBFaHJwd1RfLXJVSmF5b3NHbm5ZOW1uZTRzV1pZWFVaOEx2cGhLc3FXTEJUeTktYlVyaUtXdzBweGtuZzIwYTV1cXZMbDFEVEtkRlpzWVZnRlp6U1dZdjZ1Z0NidWV3ZzBHZXBRYlNsdlh0LXE3QjdtYkNwSVdOODhmdllQc3haRHlQcTMwTFpEU1NPdkxibi1hbDI4SHdCZjRGTjltam0xSVY5OV9pYlppS05sZTBkQ05HSFBXTXZJcEltQVVNbXo5eTRYdEdFZDRsdHpiQmtzOWhHLWo1YnloX21NOWdabE5oS2p1cXBQSGEyZFdWWEFBMmV6WFk0bmFyaVgwb0N6cHNKdmFPLVhhNUZieUZSeUw0X3NnNHdxSllYR2trNDlPWTJobUMwVDMwb2JNcVhPRGo0U0dJalRZc29lZ2NTWnciLCJlIjoiQVFBQiIsImQiOiJRVXN4RmdSQTZldDlrelhPdUVWTlRhaEFnMzA2bDF1NnVpaGxCUEo1RnVCX0JOeGRpSXQ5NHZLLU8weHlndGZDaU9EWUxNWjFkZDlsY193NkQzVFpaVjExdDhzbGJTeno2Y1JtUzVrX3FjMzlRQ0E5SnkxdlZISEtDTzFrb254UmRISFUwVi0zVjlJeEV2Q3RuMUJSZFRmSlI0Tk1sc1BaODloME1GVnVud3pPT0d2Zi1abkpNN0lZdjhsUzlkcWdUclpmZDQzR29ETVYtUDN6NXFYcWZUeUJYcWJBeFhzSTlSUXk2Q2JKMHNRSzFRaXl3b2NiNzFsdlRqUG0xaTRFU0pHN2lkcFE1ZEllSkhKbUhGclQ4elBuM2djVUlWcXJicDZETFdCMnV0Y1k0OXhJTlczNy1ISDgzMjZLQk5aVVhtSDA1aGVBdTJTbGcweVVFbzZsZ1EiLCJwIjoiMktHUF81OF80MkdFemtvZktMYTdyMWVIbWdYWk11ZFBYdTZfQTFCcy1mT2hiTkxlV0RHcW5SakFVU1JiTHItam1wNDlkVFY5al83d2VkcTVOVmVCYlc0VzktZGk1TGxYNTg3c0d4SS02cGNubzVpM0lXOHI2T2JsMzlsOWI1bEVRbDBMNXMyLTNHS2xkaV9uYS1paDBiREJqWHhZM1BtRTRBUkktcENHZnVjIiwicSI6IjFZSGppQWhRR3dCdl9BOTVLcFA1UXB3MWpTYW9iZDNsMzh4eld0aTVFTlpCSXRiV1NDOGI5Z2laN2tiLUFXdl9INlppbi1sMDB5SmJIYkZ6YnpFUjBMeXBJd2Y2MFNva0hzLWRhVWljeHZINm95MDFzenNhTERtR0FDc1AyUnBHNEtrSW41UUliTnJfRHNMM3RUM3MtSmZsNnRZbkRxaFVaME9CM0VzeDRJRSIsImRwIjoiRXpBMFhoTVFDS2NCcVhnZFRIRHJMUHZXMGdqRWxXS3h3Qm5ycDNKX1JLQ1U0dHZHd0E4ZUtxNGZrdEJpbDBCNFVHREYxdFQzRzBNY3I5NTAyMG0xLUNoeE5tSXplMGtEaFVfcHpfZ014S0RBN1JmQTJPQk5CbU0xWjEtUFljdzBwS0F6UnExZ3c0cWxWMU9rN3dUN0dHVE1zQ2ljZ201RG04Z2xZclJjaFc4IiwiZHEiOiJtR0pmT2Z6czU0aTFaSXJLcVJmNTVJX0hMTm8xaGt6RXY2bVZmM2FGQjc1VHVRRHE2WlF0LWJrRDNHdnc2S1RpN3Z6N0VUVTN1MldlOEo5eFN5QVRuZzY1RFJhcDdsV01lQzBvSlRlOUpjVVpaUk5rYTJxNGNHNFI5TmJITmVXcVJyaC1QaDhTc0ZiUmlnQ2ZlVTBjY0FWQ0JRMFp6VDFaR0dhM0xickJlNEUiLCJxaSI6ImlKcHVzVm5OX2toS0lub2VGNmZhSkxlSkJxUGlQSlRHdUhJOEdHa0hEelF3NzBLNWNyeVFkbl9kQV9uUW9Ud3pyVmhfUERLZU1RVVFuTXp2VnJTWjJlSWEtc21pWTJ2VnZDVEdDbGNWTmdhaEp0RjFhYzRyMTdUWjRmck93elRmZDJUWk9kZEo4elhSYzAtTWJjV2l4aXlSNmJkdWYxdGd3Z3p3R1VCb1VRZyJ9
   
```

[https://drive.google.com/file/d/1UOb6wr7\_f6C7qTcrCv5iAT8betSdxtMA/view?usp=sharing](https://drive.google.com/file/d/1UOb6wr7_f6C7qTcrCv5iAT8betSdxtMA/view?usp=sharing)

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

inductiveautomation.com\
![](<.gitbook/assets/image (11).png>)\
\
![](<.gitbook/assets/image (13).png>)\
\
![](<.gitbook/assets/image (14).png>)\
\
![](<.gitbook/assets/image (15).png>)\
\
![](<.gitbook/assets/image (16).png>)\
\
![](<.gitbook/assets/image (24).png>)\
\
\
\
<br>

***

#### Paso A.2: Script de Inicialización de Base de Datos (`init.sql`)

![](<.gitbook/assets/image (23).png>)\
Dentro de la carpeta `ignition-local-lab`, crea una subcarpeta llamada `init-db` y dentro de ella un archivo llamado `init.sql`.

Este script genera las tablas requeridas para simular exactamente los casos de producción del proyecto (**Gestor Documental, Enclavamientos, Registro de Actuaciones, Histórico de Consignas, Notas y Limpieza de Operadores**):

[https://drive.google.com/file/d/1lnm7bsZsbh9\_B5EBxlRXQa9Xc1O0dDF0/view?usp=sharing](https://drive.google.com/file/d/1lnm7bsZsbh9_B5EBxlRXQa9Xc1O0dDF0/view?usp=sharing)

***

#### Paso A.3: Levantar el Entorno Local

Abre una terminal en la carpeta `ignition-local-lab` y ejecuta:

```bash
docker compose up -d
```

Espera aproximadamente 60 segundos. Verifica que ambos contenedores estén en estado `healthy` o `Up`:

```bash
docker compose ps
```

***

#### Paso A.4: Configurar la Conexión de Base de Datos en el Gateway Local

1. Abre en tu navegador: `http://localhost:8088`
2. Inicia sesión con el usuario `admin` y contraseña `AdminMasterPass123!`.
3. Dirígete a **Config $\rightarrow$ Databases $\rightarrow$ Connections**.
4. Pulsa en **Create new Database Connection...**
5. Selecciona **PostgreSQL JDBC Driver** y pulsa **Next**.
6. Introduce los siguientes parámetros:
   * **Name:** `SandboxDB` _(respetar mayúsculas/minúsculas)_
   * **Connect URL:** `jdbc:postgresql://postgres:5432/sandbox_db`
   * **Username:** `ignition_user`
   * **Password:** `DBPassword123!`
7. Pulsa **Create New Database Connection**.
8. Confirma que el estado en la columna _Status_ figure como **`Valid`**.\
   ![](<.gitbook/assets/image (25).png>)

<figure><img src=".gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

get designer\
![](<.gitbook/assets/image (27).png>)\
\
![](<.gitbook/assets/image (28).png>)

<figure><img src=".gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

login

<figure><img src=".gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

\
<br>

***

### 3. Opción B: Conexión al Servidor de Formación Cloud (GCP)

Si vas a realizar las prácticas conectado directamente a la máquina virtual del curso:

* **IP Pública del Servidor:** `34.7.33.184`
* **URL del Gateway Web:** `http://34.7.33.184:8088`
* **Conexión de Base de Datos preconfigurada:** `SandboxDB`
* **Tag Provider por defecto:** `default`
* **Tus credenciales individuales de diseñador:**
  * Usuario: `alumnoXX` (ej. `alumno01`, `alumno02`, ..., `alumno20`)
  * Contraseña: `Password_XX!` (ej. `Password_01!`, `Password_02!`, ..., `Password_20!`)

***

### 4. Configuración del Ignition Designer Launcher

1. Abre la aplicación **Ignition Designer Launcher** en tu ordenador.
2. Pulsa en **Add Gateway**.
3. Añade la dirección correspondiente:
   * Para entorno local: `http://localhost:8088`
   * Para entorno Cloud: `http://34.7.33.184:8088`
4. Selecciona el Gateway añadido y pulsa **Open Designer**.
5. Autentícate con tus credenciales asignadas (`alumnoXX` o `admin` en local).
6. Crea tu proyecto personal de prácticas con el nombre: `Proyecto_Alumno_XX`.

***

### 5. Batería de Tests de Validación del Entorno

Realiza los siguientes 5 tests desde tu Designer para certificar que tu entorno está listo para los laboratorios del curso.

***

#### 🧪 Test 1: Conectividad y Autenticación en Designer

* **Objetivo:** Validar la comunicación entre el Designer Launcher y el Gateway de Ignition, confirmando permisos de diseño.
* **Ejecución:**
  1. Abrir el proyecto personal `Proyecto_Alumno_XX` en el Designer.
  2. Verificar que en la esquina inferior derecha el estado de comunicación marque **Connected** con icono verde.
* **Resultado esperado:**
  * El Designer abre el espacio de trabajo sin errores de autenticación ni alertas de desconexión.

***

#### 🧪 Test 2: Validación de la Conexión JDBC `SandboxDB`

* **Objetivo:** Comprobar desde la consola de scripting que la conexión con PostgreSQL está activa y accesible por el motor de scripts.
*   **Ejecución:**

    1. En el Designer, abre el menú superior **Tools $\rightarrow$ Script Console**.
    2. Ejecuta el siguiente script:

    ```python
    res = system.db.runScalarPrepQuery("SELECT 1", [], database="SandboxDB")
    print "Respuesta de PostgreSQL:", res
    ```
* **Resultado esperado:**
  * La consola imprime: `Respuesta de PostgreSQL: 1`.

***

#### 🧪 Test 3: Validación del Esquema de Datos de Laboratorio

* **Objetivo:** Verificar que las tablas necesarias para los casos de estudio (gestor documental, enclavamientos, notas y operadores) contienen datos de prueba.
*   **Ejecución:**

    1. En la **Script Console**, pega y ejecuta:

    ```python
    q = """
    SELECT 
        (SELECT COUNT(*) FROM documents_metadata) AS total_docs,
        (SELECT COUNT(*) FROM enclavaments) AS total_enclavaments,
        (SELECT COUNT(*) FROM notes WHERE Readed = 0) AS notas_pendientes,
        (SELECT COUNT(*) FROM REG_OPERADORS) AS total_operadores
    """
    pyds = system.db.runPrepQuery(q, [], database="SandboxDB")
    for row in pyds:
        print "Documentos:", row["total_docs"]
        print "Enclavamientos:", row["total_enclavaments"]
        print "Notas pendientes:", row["notas_pendientes"]
        print "Registros operadores:", row["total_operadores"]
    ```
* **Resultado esperado:**
  * La consola muestra recuentos válidos (Documentos $\ge 0$, Enclavamientos $\ge 2$, Notas pendientes $\ge 2$, Registros operadores $\ge 3$).

***

#### 🧪 Test 4: Validación del Tag Provider y Lectura en Bloque

* **Objetivo:** Certificar que el Tag Provider `default` permite la lectura agrupada y responde con objetos `QualifiedValue` correctos.
*   **Ejecución:**

    1. En la **Script Console**, ejecuta:

    ```python
    system.tag.writeBlocking(["[default]Test_Signal"], [100.0])
    qv = system.tag.readBlocking(["[default]Test_Signal"])[0]
    print "Valor Tag:", qv.value
    print "Calidad:", qv.quality.toString()
    print "Es Buena Calidad?:", qv.quality.isGood()
    ```
* **Resultado esperado:**
  * La consola imprime `Valor Tag: 100.0`, `Calidad: Good` y `Es Buena Calidad?: True`.

***

#### 🧪 Test 5: Simulación de Named Query con Parámetros Tipados

* **Objetivo:** Validar la capacidad de ejecución de consultas de inserción y lectura parametrizada sobre la tabla de actuaciones del operador.
*   **Ejecución:**

    1. En la **Script Console**, ejecuta una inserción y comprobación parametrizada:

    ```python
    # Inserción de prueba
    ins_q = "INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari) VALUES (?, ?, ?)"
    system.db.runPrepUpdate(ins_q, ["BOMBA_01", "Test de validación Sesión 0", "Alumno_Test"], database="SandboxDB")

    # Lectura de comprobación
    sel_q = "SELECT Descripcio, Usuari FROM ins_Registre_Actuacions WHERE Equip = ? ORDER BY id DESC LIMIT 1"
    resultado = system.db.runPrepQuery(sel_q, ["BOMBA_01"], database="SandboxDB")
    print "Última actuación registrada:", resultado[0]["Descripcio"], "| Por:", resultado[0]["Usuari"]
    ```
* **Resultado esperado:**
  * La consola imprime: `Última actuación registrada: Test de validación Sesión 0 | Por: Alumno_Test`.

***

### 6. Checklist de Conformidad Final (Pase a Sesión 1)

Marca cada casilla para certificar que tu entorno está 100% operativo:

* [ ] **Designer Launcher** instalado y comunicando con el Gateway (`localhost:8088` o `34.7.33.184:8088`).
* [ ] Conexión de base de datos **`SandboxDB`** en estado **`Valid`**.
* [ ] Base de datos PostgreSQL con todas las tablas creadas y datos de prueba cargados.
* [ ] Los 5 tests de validación ejecutados en la **Script Console** con resultado satisfactorio.
* [ ] Proyecto de Designer `Proyecto_Alumno_XX` creado y guardado.
