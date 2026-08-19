---
description: Setup y Validación del Entorno de Trabajo
---

# Setup inicial alumnos

**Curso:** SQL y Python/Jython para Scripting en Ignition (SCADA)\
**Objetivo de la sesión:** Configurar tu puesto local de desarrollo, conectar el **Designer Launcher** al Gateway de formación en la nube, crear tu proyecto individual aislado y validar el correcto funcionamiento del intérprete **Jython 2.7**, la base de datos **PostgreSQL (`SandboxDB`)** y el motor de **Tags**.

***

### 📌 1. Parámetros de Conexión y Credenciales

| Parámetro                     | Valor de Configuración                                                         |
| ----------------------------- | ------------------------------------------------------------------------------ |
| **URL del Gateway**           | `http://34.7.33.184:8088`                                                      |
| **Base de Datos por Defecto** | `SandboxDB` (PostgreSQL 15)                                                    |
| **Tag Provider por Defecto**  | `default`                                                                      |
| **Usuario Asignado**          | `alumnoXX` _(ejemplo: `alumno01`, `alumno02`, ... `alumno20`)_                 |
| **Contraseña Asignada**       | `Password_XX!` _(ejemplo: `Password_01!`, `Password_02!`, ... `Password_20!`)_ |

> ⚠️ **Nota:** Sustituye siempre `XX` por el número de alumno de dos dígitos que te haya asignado el formador.

***

### 🚀 2. Paso a Paso de Configuración del Entorno

#### Paso 2.1: Descargar e Instalar el Designer Launcher

Si no dispones del **Ignition Designer Launcher** instalado en tu ordenador:

1. Abre tu navegador web y entra en: **`http://34.7.33.184:8088`**
2. En la página de inicio del Gateway, haz clic en el botón azul **Download Designer Launcher** _(o dirígete a la sección Home $\rightarrow$ View Projects $\rightarrow$ Download Designer Launcher)_.
3. Descarga la versión adecuada para tu sistema operativo (Windows, macOS o Linux).
4. Ejecuta el instalador y completa el asistente con las opciones predeterminadas.

***

#### Paso 2.2: Vincular el Gateway de Formación en el Launcher

1. Abre la aplicación **Designer Launcher** en tu equipo.
2. Haz clic en el botón **Add Designer** (o **Manually Add Gateway** en la esquina superior derecha).
3.  En el campo **Gateway URL**, introduce exactamente:

    ```
    http://34.7.33.184:8088
    ```
4. Haz clic en **Add Gateway**.
5. Comprueba que el Gateway aparece en la lista de conexiones con el indicador en verde (**Available / Online**).

***

#### Paso 2.3: Iniciar Sesión y Crear tu Proyecto Individual

Cada alumno trabajará sobre un proyecto propio para evitar colisiones de código:

1. En el Designer Launcher, selecciona el Gateway `34.7.33.184` y pulsa **Open Designer**.
2. En la ventana de autenticación, introduce tus credenciales:
   * **Username:** `alumnoXX` _(ejemplo: `alumno03`)_
   * **Password:** `Password_XX!` _(ejemplo: `Password_03!`)_
3. En el selector de proyectos, pulsa en el botón **New Project (+)**.
4. Cumplimenta los campos del proyecto con los siguientes valores exactos:
   * **Project Name:** `Proyecto_Alumno_XX` _(ejemplo: `Proyecto_Alumno_03`)_
   * **Title:** `Formacion Scripting - Alumno XX`
   * **Project Template:** `Blank Project`
   * **Default Database:** Selecciona **`SandboxDB`** en el menú desplegable.
   * **Default Tag Provider:** Selecciona **`default`** en el menú desplegable.
5. Haz clic en **Create New Project**. El entorno de desarrollo del Designer se abrirá en pantalla.

***

### 🧪 3. Batería de Tests de Validación Técnica

Realiza los siguientes tres tests en tu Designer para certificar que tu entorno dispone de permisos de ejecución de scripts, conexión SQL operativa y acceso al motor de tags.

***

#### 🧪 Test 3.1: Validación de Script Console y Motor Jython sobre la JVM

* **Objetivo:**
  * Verificar que el entorno de desarrollo local (Designer) tiene acceso operativo a la **Script Console** y que el intérprete **Jython 2.7** ejecuta instrucciones correctamente sobre la Máquina Virtual de Java (JVM).
* **Pasos a ejecutar:**
  1. En el menú superior del Designer, ve a: **Tools $\rightarrow$ Script Console**.
  2.  En el panel izquierdo de edición, pega el siguiente script:

      ```python
      # Verificación de Entorno Jython y JVM
      import sys
      import java.lang.System as JSystem

      print "=== TEST 1: ENTORNO JYTHON ==="
      print "Version Python/Jython:", sys.version
      print "JVM Vendor:", JSystem.getProperty("java.vendor")
      print "JVM Version:", JSystem.getProperty("java.version")
      print "Resultado: [OK]"
      ```
  3. Haz clic en el botón **Execute** (esquina inferior derecha).
* **Resultado esperado:**
  *   El panel derecho de salida muestra la versión de Jython (2.7.x), los datos del fabricante de la JVM y el mensaje de confirmación sin errores de sintaxis:

      ```
      === TEST 1: ENTORNO JYTHON ===
      Version Python/Jython: 2.7.x (...)
      JVM Vendor: (...)
      JVM Version: 11.x / 17.x (...)
      Resultado: [OK]
      ```

***

#### 🧪 Test 3.2: Validación de Conexión JDBC a Base de Datos (`SandboxDB`)

* **Objetivo:**
  * Comprobar la conectividad bidireccional entre Ignition y el motor **PostgreSQL** a través del pool de conexiones JDBC `SandboxDB`, asegurando permisos de consulta para el usuario de formación.
* **Pasos a ejecutar:**
  1.  En la misma **Script Console**, borra el contenido anterior y pega el siguiente script:

      ```python
      # Verificación de Conexión JDBC a PostgreSQL
      db_conn = "SandboxDB"

      try:
          query = "SELECT current_database() AS db, current_user AS usuario, version() AS version_pg"
          resultado = system.db.runQuery(query, database=db_conn)
          
          print "=== TEST 2: BASE DE DATOS POSTGRESQL ==="
          print "Base de Datos Conectada:", resultado[0]["db"]
          print "Usuario JDBC:", resultado[0]["usuario"]
          print "Motor:", resultado[0]["version_pg"][:40] + "..."
          print "Resultado: [OK - CONEXION ESTABLECIDA]"
      except Exception as e:
          print "Error de conexion a DB:", unicode(e)
      ```
  2. Haz clic en **Execute**.
* **Resultado esperado:**
  *   La consulta SQL retorna una fila con los metadatos de la base de datos sin lanzar excepciones JDBC (`SQLException`):

      ```
      === TEST 2: BASE DE DATOS POSTGRESQL ===
      Base de Datos Conectada: sandbox_db
      Usuario JDBC: ignition_user
      Motor: PostgreSQL 15.x...
      Resultado: [OK - CONEXION ESTABLECIDA]
      ```

***

#### 🧪 Test 3.3: Validación del Tag Provider y Creación de Tags (`default`)

* **Objetivo:**
  * Validar los permisos de creación y modificación de tags en el proveedor `default`, comprobando la lectura programática mediante la función nativa `system.tag.readBlocking` y la inspección del objeto `QualifiedValue`.
* **Pasos a ejecutar:**
  1. En el panel izquierdo del Designer, selecciona la pestaña **Tag Browser**.
  2. Despliega el proveedor `[default]`.
  3. Haz clic derecho sobre `default` $\rightarrow$ **New Folder** $\rightarrow$ Nómbrala con tu identificador: `Alumno_XX` _(ejemplo: `Alumno_03`)_.
  4. Haz clic derecho sobre tu carpeta creada `Alumno_XX` $\rightarrow$ **New Tag $\rightarrow$ Memory Tag**:
     * **Name:** `Tag_Test_Conexion`
     * **Data Type:** `Float4`
     * **Value:** `100.0`
  5. Pulsa **Apply** y luego **OK**.
  6.  En la **Script Console**, pega y ejecuta el siguiente script _(asegúrate de sustituir `XX` por tu número)_:

      ```python
      # Sustituye XX por tu numero de alumno asignado:
      tag_path = "[default]Alumno_XX/Tag_Test_Conexion"

      qv = system.tag.readBlocking([tag_path])[0]

      print "=== TEST 3: TAG ENGINE ==="
      print "Tag Path:", tag_path
      print "Valor:", qv.value
      print "Calidad:", qv.quality
      print "Timestamp:", qv.timestamp
      print "Resultado: [OK - TAG OPERATIVO]"
      ```
* **Resultado esperado:**
  *   El script lee el tag en tiempo real, confirmando el valor numérico, calidad confiable (`Good`) y la marca temporal actual:

      ```
      === TEST 3: TAG ENGINE ===
      Tag Path: [default]Alumno_XX/Tag_Test_Conexion
      Valor: 100.0
      Calidad: Good
      Timestamp: [Fecha y hora actual]
      Resultado: [OK - TAG OPERATIVO]
      ```

***

### 📋 4. Checklist Final de Conformidad

Antes de dar por concluida la Sesión 0, valida que cumples con todos los puntos requeridos:

* [ ] **Acceso al Gateway:** Puedes acceder a la web `http://34.7.33.184:8088` sin problemas de firewall o red.
* [ ] **Designer Launcher:** Instalado localmente y el Gateway `34.7.33.184` figura en estado verde (_Online_).
* [ ] **Proyecto Creado:** Has creado tu proyecto `Proyecto_Alumno_XX` vinculado a `SandboxDB` y `default`.
* [ ] **Test 3.1 Superado:** La Script Console ejecuta código Jython 2.7 correctamente.
* [ ] **Test 3.2 Superado:** La conexión `SandboxDB` responde a consultas SQL sobre PostgreSQL.
* [ ] **Test 3.3 Superado:** Tu carpeta `[default]Alumno_XX` y el tag `Tag_Test_Conexion` están creados y son legibles por script.
* [ ] **Puesto de Trabajo:** Dispones de **dos pantallas** (una para el Designer y otra para Zoom/guías del curso) y auriculares con micrófono configurados.

> 💾 **Paso Final:** Guarda los cambios en tu proyecto pulsando en la barra superior **File $\rightarrow$ Save** (o atajo `Ctrl + S`). ¡Tu entorno está 100% preparado para la **Sesión 1**!
