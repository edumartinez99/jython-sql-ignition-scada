# Sesión 5: Escritura Segura en Base de Datos y Arquitectura de Named Queries

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Escrituras Seguras, Auditoría y Control de Persistencia (Tema 10)

* **Conceptos clave a tratar:**
  * Operaciones de modificación de datos en SCADA: registro de eventos con `INSERT`, actualización de estados y recetas con `UPDATE`, y criterios restrictivos para el uso de `DELETE`.
  * Barreras de validación previa: integridad de rangos de ingeniería, verificación de permisos del operador y validación del estado real de la máquina.
  * **Prevención de vulnerabilidades e inyección SQL:** riesgos críticos de la concatenación manual de cadenas en sentencias SQL.
  * Cláusulas `WHERE` blindadas: diseño de condiciones estrictas y unívocas sobre claves primarias (`PK`) para evitar modificaciones o borrados masivos involuntarios.
  * Estrategias de persistencia: diferenciación entre borrado físico destructivo, borrado lógico (`is_deleted = 1`), desactivación de registros y transiciones de estado operativo.
  * Registro de pistas de auditoría (_Audit Trail_): captura obligatoria de usuario, marca de tiempo exacta, valor anterior, valor nuevo y justificación técnica del cambio.
  * Interacción segura con el operador: diálogos de confirmación previa y retroalimentación clara tras la modificación persistente de datos.
  * Estandarización de patrones corporativos de escritura para evitar implementaciones improvisadas.
* **Objetivos:**
  * Aprender a proteger todas las operaciones de inserción, actualización y borrado contra inyecciones y fallos de integridad.
  * Dominar el diseño de mecanismos de borrado lógico y registro estructurado de auditoría en planta.
* **Resultado esperado:**
  * El alumno implementa escrituras seguras con validación de precondiciones, auditoría de cambios y condiciones `WHERE` a prueba de fallos.

***

#### 📘 Bloque Teórico 2: Arquitectura y Gobernanza de Named Queries (Tema 11)

* **Conceptos clave a tratar:**
  * Definición y propósito de las **Named Queries** en Ignition: consultas preconfiguradas, centralizadas en el Gateway y reutilizables en todo el proyecto.
  * Seguridad intrínseca: funcionamiento como _Prepared Statements_ cuando se utilizan parámetros de tipo **Value**, neutralizando riesgos de inyección SQL.
  * Tipología de parámetros: diferencias críticas entre parámetros de tipo **Value** (seguros, tipados) y parámetros de tipo **Query String** (riesgos y restricciones de uso).
  * Clasificación por tipo de operación: _Query_ (lectura tabular), _Scalar Query_ (lectura de celda única) y _Update Query_ (`INSERT`, `UPDATE`, `DELETE`).
  * Organización jerárquica por dominios: estructuración de carpetas en el Designer (`Produccion/`, `Mantenimiento/`, `Calidad/`, `Auditoria/`, `Configuracion/`).
  * Invocación programática mediante `system.db.runNamedQuery`: paso de parámetros mediante diccionarios y gestión de retornos según el scope de ejecución.
  * Gobernanza y control de impacto: gestión del ciclo de vida, versionado de consultas y evaluación de dependencias en pantallas que consumen la Named Query.
  * Creación del estándar interno de diseño para nombres, descripciones, tipado estricto y documentación de parámetros.
* **Objetivos:**
  * Comprender el papel de las Named Queries como la capa oficial y ordenada de acceso a datos en Ignition.
  * Aprender a estructurar, tipar y ejecutar Named Queries seguras desde scripts de Jython.
* **Resultado esperado:**
  * El alumno centraliza todas las consultas SQL del proyecto en un árbol organizado de Named Queries parametrizadas, eliminando consultas dispersas en el código.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 5.1: Patrón de Escritura Segura y Auditoría de Cambios (Tema 10)

* **Objetivos:**
  * Implementar una función de actualización de consignas de proceso con validación previa de límites de ingeniería y rol de usuario.
  * Registrar automáticamente una traza completa en la tabla de auditoría capturando el valor previo, el nuevo valor, el operario y el motivo del cambio.
* **Resultado esperado:**
  * Un script modular probado que actualiza registros de recetas en base de datos solo si se cumplen las precondiciones, generando la entrada de auditoría correspondiente de forma consistente.

***

#### 🧪 Laboratorio 5.2: Borrado Lógico Protegido con Validación Previa de Estado (Tema 10)

* **Objetivos:**
  * Construir un procedimiento seguro de anulación de órdenes de trabajo aplicando borrado lógico (`is_deleted = 1`, `estado = 'CANCELADO'`).
  * Diseñar una cláusula `WHERE` restrictiva con doble comprobación que impida la anulación de órdenes que ya se encuentren en estado de ejecución en la línea.
* **Resultado esperado:**
  * Una función de persistencia que evalúa el número de filas afectadas (`rowsAffected`) y notifica adecuadamente si la operación fue denegada por no cumplir el estado requerido.

***

#### 🧪 Laboratorio 5.3: Creación y Organización Estructurada de Named Queries (Tema 11)

* **Objetivos:**
  * Diseñar y configurar un árbol jerárquico de Named Queries en el Designer para un módulo de inspección de calidad (`Calidad/Inspecciones/*`).
  * Configurar parámetros fuertemente tipados para consultas de lectura tabular, conteo escalar y registro de defectos (_Update Query_).
* **Resultado esperado:**
  * Un conjunto de Named Queries creadas y validadas mediante el panel de pruebas del Designer, documentadas con tipos de datos correctos y libres de parámetros _Query String_ inseguros.

***

#### 🧪 Laboratorio 5.4: Invocación Programática con `system.db.runNamedQuery` (Tema 11)

* **Objetivos:**
  * Invocar las Named Queries creadas desde la Script Console mediante `system.db.runNamedQuery` pasando diccionarios de parámetros estructurados.
  * Procesar las respuestas obtenidas (tabulares y escalares) e implementar captura defensiva ante errores de tipado o parámetros faltantes.
* **Resultado esperado:**
  * Un script de Jython que orquesta consultas y actualizaciones a través de Named Queries, gestionando correctamente los `PyDataSets` devueltos y capturando excepciones de ejecución.
