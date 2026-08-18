# Sesión 4: Logging Profesional, Gestión de Excepciones y SQL Esencial para SCADA

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Logging Profesional y Gestión de Excepciones Jython/Java (Tema 8)

* **Conceptos clave a tratar:**
  * Uso defensivo de `try / except / finally` evitando capturas genéricas que silencian fallos e introducen datos corruptos en planta.
  * Clasificación de errores: recuperables, de validación de datos, de conectividad (DB/PLC), de permisos y de lógica algorítmica.
  * Captura de excepciones nativas de Java (`java.lang.Exception`, `java.sql.SQLException`) dentro del entorno Jython.
  * Sistema de trazabilidad con `system.util.getLogger`: configuración de loggers por contexto (`app.modulo.*`) y jerarquía de niveles (`DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL`).
  * Enriquecimiento de logs con metadatos contextuales: función de origen, parámetros recibidos, usuario en sesión, tag o consulta afectada.
  * Diseño de _Wrappers_ funcionales para blindar operaciones críticas de base de datos y escrituras de tags.
  * Desacoplamiento de audiencias: logs técnicos detallados para diagnóstico en el Gateway vs mensajes comprensibles y orientados a la acción para el operador en interfaz.
  * Pautas de higiene para evitar la saturación de los registros del Gateway (_Gateway Console logs_).
* **Objetivos:**
  * Dominar el tratamiento robusto de excepciones mixtas (Python y Java) en Ignition.
  * Aprender a estructurar un sistema de logging corporativo trazable, limpio y contextualizado.
* **Resultado esperado:**
  * El alumno diseña bloques de captura específicos y emite logs contextuales en el Gateway sin exponer errores técnicos a los operadores de planta.

***

#### 📘 Bloque Teórico 2: SQL Esencial y Analítica en Base de Datos para SCADA (Tema 9)

* **Conceptos clave a tratar:**
  * Estructuración de consultas industriales con `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY` y `HAVING`.
  * Funciones de agregación (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`) aplicadas a producción, consumos energéticos, lotes y métricas de turno.
  * **Principio de delegación en base de datos:** por qué realizar cálculos masivos en el motor relacional y evitar transferir miles de filas hacia la JVM de Ignition.
  * Relaciones y enriquecimiento de datos con `INNER JOIN` y `LEFT JOIN`: combinación de eventos operativos con catálogos de productos, operarios y estados.
  * Lógica condicional directa en SQL mediante `CASE WHEN` para semáforos, categorización de paradas y etiquetas visuales.
  * Filtros compuestos por ventanas temporales, líneas, lotes y rangos de ingeniería.
  * Particularidades sintácticas entre motores relacionales comunes (Microsoft SQL Server, PostgreSQL, MySQL / MariaDB).
  * Creación de la _Guía de Consultas Base_ para estandarizar operaciones recurrentes en el proyecto.
* **Objetivos:**
  * Diseñar consultas analíticas optimizadas que resuelvan cálculos de KPIs directamente en el motor de base de datos.
  * Integrar catálogos maestros y lógica condicional `CASE WHEN` para alimentar tablas y cuadros de mando en Ignition.
* **Resultado esperado:**
  * El alumno escribe consultas SQL relacionales complejas, legibles y eficientes, adaptadas al motor de base de datos de la planta.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 4.1: Sistema de Logging Estructurado y Captura de Excepciones Mixtas (Tema 8)

* **Objetivos:**
  * Configurar un logger modular mediante `system.util.getLogger` y registrar eventos en diferentes niveles de severidad.
  * Implementar un bloque de captura que intercepte tanto excepciones de Jython como excepciones de la JVM (`java.lang.Exception`), extrayendo el contexto del error.
* **Resultado esperado:**
  * Un script probado en consola que genera trazas estructuradas visibles en la consola de logs del Gateway ante fallos provocados de datos o red.

***

#### 🧪 Laboratorio 4.2: Creación de un Wrapper Defensivo para Operaciones Críticas (Tema 8)

* **Objetivos:**
  * Diseñar una función envoltorio (_wrapper_) en la Project Library que ejecute operaciones de base de datos o tags con protección ante caídas.
  * Asegurar el retorno garantizado de valores de reserva (_fallbacks_) y la notificación amigable al operador en caso de fallo.
* **Resultado esperado:**
  * Un wrapper genérico reutilizable que previene que las pantallas de interfaz se bloqueen o muestren errores no controlados cuando se produce una desconexión.

***

#### 🧪 Laboratorio 4.3: Consultas de Agregación, KPIs y Clasificación con CASE WHEN (Tema 9)

* **Objetivos:**
  * Construir una consulta SQL analítica sobre registros históricos de producción para calcular piezas totales, mermas y porcentaje de calidad agrupados por máquina y turno.
  * Incorporar una clasificación condicional mediante `CASE WHEN` para categorizar la eficiencia operativa en semáforos de color.
* **Resultado esperado:**
  * Una consulta SQL validada en el _Query Browser_ que retorna un resumen consolidado de KPIs listo para su consumo directo en componentes SCADA.

***

#### 🧪 Laboratorio 4.4: Cruces Relacionales (JOINs) y Dialectos SQL en Ignition (Tema 9)

* **Objetivos:**
  * Combinar tablas operativas de órdenes de trabajo con maestros de productos y tablas de paradas mediante `LEFT JOIN`.
  * Adaptar y documentar las diferencias de sintaxis (funciones de fecha, tratamiento de nulos y límites) para SQL Server, PostgreSQL y MySQL.
* **Resultado esperado:**
  * Una consulta relacional multicapa documentada en la _Guía de Consultas Base_, asegurando compatibilidad y rendimiento óptimo en el motor de base de datos de pruebas.
