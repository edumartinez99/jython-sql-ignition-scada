# Sesión 8: Optimización Avanzada de Consultas SQL y Rendimiento de Scripts Jython

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Optimización y Rendimiento en Consultas SQL (Tema 18)

* **Conceptos clave a tratar:**
  * Diagnóstico de cuellos de botella en bases de datos: análisis de filtros ineficientes, cruces (_JOINs_) costosos, volumen de datos y escaneos de tabla completa (_Full Table Scans_).
  * Interpretación de **Planes de Ejecución (**_**Execution Plans / EXPLAIN**_**)** para identificar lecturas de disco innecesarias y cooperar eficazmente con el DBA.
  * **Eliminación del antipatrón `SELECT *`:** impacto del tráfico de red, saturación del buffer JDBC y sobrecarga en la memoria _Heap_ de la JVM de Ignition.
  * Principio de filtrado en origen (_Pushdown_): aplicar filtros y agregaciones en el motor relacional antes de transferir datos al SCADA.
  * Estrategias de acotamiento para UI: límites de registros (`TOP` / `LIMIT`), paginación y ventanas temporales obligatorias.
  * **Indexación y SARGability (**_**Search Argument Able**_**):** diseño de índices compuestos y eliminación de funciones sobre columnas en la cláusula `WHERE`.
  * Segregación arquitectónica: separación de consultas transaccionales rápidas (OLTP) de reportes y analítica pesada (OLAP).
  * Uso de vistas materializadas, tablas intermedias de agregación y definición de acuerdos de nivel de servicio (_SLAs_) de tiempo de respuesta.
* **Objetivos:**
  * Aprender a diagnosticar, indexar y refactorizar consultas SQL lentas para entornos de tiempo real.
  * Dominar el diseño de consultas SARGable que aprovechen los índices B-Tree de la base de datos.
* **Resultado esperado:**
  * El alumno reescribe consultas complejas eliminando escaneos masivos de tabla y reduciendo drásticamente la latencia de respuesta en Ignition.

***

#### 📘 Bloque Teórico 2: Optimización de Scripts Jython, Memoria JVM y Caché (Tema 19)

* **Conceptos clave a tratar:**
  * Detección de sobre-ejecución (_Script Over-triggering_): identificación de scripts en bindings, transforms y eventos de tag que se disparan miles de veces por minuto.
  * Reducción de ineficiencias en la JVM: eliminación de bucles redundantes y conversiones repetitivas de estructuras (`toPyDataSet`, `toDataSet`).
  * **Erradicación del problema N+1:** sustitución del patrón de consultas o lecturas de tag individuales por fila mediante operaciones agrupadas por lotes (_Batching_).
  * Estrategias de almacenamiento en caché en memoria: uso de variables de módulo en la **Project Library** con caducidad por tiempo (_TTL_) y aprovechamiento de la caché nativa en Named Queries.
  * Descarga del hilo de renderizado (_UI Thread_): traslado de cálculos pesados al Gateway o a ejecución asíncrona en segundo plano (`system.util.invokeAsynchronous`).
  * Metodología de instrumentación y medición: registro de tiempos de ejecución con precisión de microsegundos antes y después de refactorizar.
  * Creación y aplicación de la **Checklist Corporativa de Rendimiento** para revisiones de código (_Code Reviews_).
* **Objetivos:**
  * Comprender el impacto del consumo de CPU y memoria Heap de la JVM provocado por scripts ineficientes en Ignition.
  * Aprender a implementar mecanismos de caché en memoria y ejecución por lotes.
* **Resultado esperado:**
  * El alumno identifica y elimina cuellos de botella en scripts de Jython, aplicando técnicas de caché y auditoría de rendimiento estandarizadas.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 8.1: Diagnóstico y Refactorización SARGable de Consultas Lentas (Tema 18)

* **Objetivos:**
  * Analizar una consulta SQL histórica ineficiente que contiene funciones sobre columnas de fecha y texto en la cláusula `WHERE`.
  * Refactorizar la sentencia aplicando condiciones SARGable directas y eliminando columnas innecesarias para optimizar el uso de índices.
* **Resultado esperado:**
  * Una consulta optimizada probada en el _Query Browser_ que reduce drásticamente el tiempo de ejecución sobre tablas con decenas de miles de registros.

***

#### 🧪 Laboratorio 8.2: Eliminación del Antipatrón N+1 en Consultas SCADA (Temas 18 y 19)

* **Objetivo:**
  * Identificar un script heredado que ejecuta una consulta individual dentro de un bucle `for` para cada elemento de una tabla de 50 equipos.
  * Sustituir el proceso por una única consulta relacional (`JOIN` o cláusula `IN (?, ?, ...)`) y mapear los resultados en memoria mediante un diccionario indexado.
* **Resultado esperado:**
  * Un script refactorizado que sustituye 50 peticiones a base de datos por una única llamada en bloque, reduciendo la latencia total a una fracción de tiempo.

***

#### 🧪 Laboratorio 8.3: Implementación de Caché en Memoria con Project Library (Tema 19)

* **Objetivos:**
  * Diseñar un módulo de caché en la Project Library (`app.cache`) con control de expiración temporal (_Time-to-Live_) para catálogos maestros y datos estáticos.
  * Comprobar que múltiples llamadas consecutivas acceden directamente a la memoria RAM de la JVM sin saturar el pool de conexiones JDBC.
* **Resultado esperado:**
  * Un gestor de caché funcional probado en la Script Console que solo consulta la base de datos cuando los datos han expirado o no existen en memoria.

***

#### 🧪 Laboratorio 8.4: Benchmarking y Aplicación de la Checklist de Rendimiento (Tema 19)

* **Objetivos:**
  * Instrumentar un script de transformación de datos con `java.lang.System.nanoTime()` para medir tiempos exactos de procesamiento.
  * Auditar el código aplicando la _Checklist de Rendimiento_ corporativa (eliminación de `SELECT *`, lecturas de tags agrupadas y transforms sin I/O).
* **Resultado esperado:**
  * Un informe de auditoría comparativo antes/después que certifica la optimización del código y el cumplimiento de los estándares de rendimiento para pase a producción.
