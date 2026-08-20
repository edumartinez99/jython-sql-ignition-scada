# Sesión 1: Arquitectura de Scripting, Jython y Fundamentos SCADA

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Del Dato de Planta al Script Útil (Tema 1)

* **Conceptos clave a tratar:**
  * Ubicación de los puntos de scripting en Ignition Designer sin entrar en desarrollo visual complejo.
  * Trazabilidad del dato: recorrido desde el tag OPC/base de datos hasta la vista, tabla o componente.
  * Transformación de datos brutos en información de valor para operación (estados, colores, alarmas).
  * Ejecución de consultas simples a tablas de pruebas y conversión a estructuras manejables en Jython.
  * Fronteras de responsabilidad: qué pertenece al motor SQL, qué a Jython y qué al contexto nativo de Ignition.
  * Criterios de calidad: por qué un script que “funciona” no siempre es seguro, rápido ni mantenible.
* **Objetivos:**
  * Comprender el flujo end-to-end de una señal industrial dentro del motor de Ignition.
  * Identificar con precisión qué parte de un problema debe resolverse con SQL, cuál con Jython y cuál con configuración nativa.
* **Resultado esperado:**
  * El alumno identifica visualmente todos los puntos de scripting en Designer y distingue con claridad las responsabilidades de cada capa tecnológica.

***

#### 📘 Bloque Teórico 2: Tipos de Scripts, Scopes y Arquitectura de Ejecución (Tema 2)

* **Conceptos clave a tratar:**
  * Diferenciación entre maquetación/diseño SCADA y scripting de lógica, consultas y automatización.
  * Puntos de ejecución: Property Bindings, Transforms, Component Events, Session Events, Gateway Events y Project Library.
  * Scopes de ejecución: diferencias críticas entre _Gateway Scope_, _Perspective Session Scope_ y _Vision Client Scope_.
  * Frecuencia y disparadores: scripts únicos, por evento, recurrentes y scripts que se disparan en exceso de forma inadvertida.
  * Consecuencias de una mala ubicación: saturación del Gateway, pérdida de rendimiento en UI y errores difíciles de depurar.
  * Matriz de herramientas: Expressions vs Expression Bindings vs Script Transforms vs SQL Bindings vs Named Queries vs Project Scripts.
  * Regla de diseño: lógica repetible en Project Library, consultas en Named Queries y transforms ligeros.
* **Objetivos:**
  * Dominar el concepto de _Scope_ y su impacto directo en el acceso a recursos y rendimiento del sistema.
  * Aprender a seleccionar la herramienta nativa adecuada antes de escribir código Jython innecesario.
* **Resultado esperado:**
  * El alumno dispone de un criterio estructurado (Guía de Decisión) para determinar exactamente en qué scope y punto de Ignition debe residir cada lógica.

***

#### 📘 Bloque Teórico 3: Jython en Ignition, Diferencias con Python 3 y Límites (Tema 3)

* **Conceptos clave a tratar:**
  * Fundamentos de Jython como intérprete de Python 2.7 sobre la JVM y diferencias con CPython moderno (FastAPI, Python 3).
  * Diferencias sintácticas y de comportamiento: `print`, división entera vs flotante, Unicode (`u'...'`), ausencia de _f-strings_ y _async/await_.
  * Incompatibilidad con librerías nativas en C (ej. `numpy`, `pandas`) y adaptación de hábitos de programación.
  * Capacidades añadidas de Jython: importación y uso directo de clases Java (`java.lang`, `java.util`, `java.text`).
  * Integración con el ecosistema de Ignition a través del módulo `system.*`.
  * Clasificación y buenas prácticas: scripts de laboratorio vs producción, legibilidad, indentación y modularidad en funciones pequeñas.
* **Objetivos:**
  * Asimilar las limitaciones y ventajas de Jython 2.7 sobre la JVM para evitar errores de sintaxis y compatibilidad.
  * Conocer los mecanismos de interoperabilidad directa con clases nativas de Java.
* **Resultado esperado:**
  * El alumno escribe código compatible con la JVM de Ignition, evitando vicios de Python 3 y aprovechando las librerías Java cuando proceda.

***

#### 📘 Bloque Teórico 4: Fundamentos de Lenguaje y Lógica Defensiva SCADA (Tema 4 - Parte 1)

* **Conceptos clave a tratar:**
  * Declaración y nomenclatura estandarizada de variables para estados de máquina, contadores, timestamps y alarmas.
  * Tipos de datos primitivos y colecciones: enteros, decimales, cadenas, booleanos, fechas, listas, tuplas y diccionarios.
  * Control de flujo con `if / elif / else` para enclavamientos, límites de proceso y lógica operativa.
  * Uso seguro de bucles `for` y precauciones críticas con `while` en entornos SCADA.
  * Patrones de acumulación: sumatorios, conteos, medias, máximos, mínimos y estados agregados.
  * Programación defensiva: tratamiento estricto de `None`, cadenas vacías, listas vacías y datos nulos de base de datos o tags en calidad `Bad`.
  * Conversión y casting seguro entre tipos procedentes de tags, formularios y tablas SQL.
* **Objetivos:**
  * Escribir algoritmos industriales defensivos que no fallen ante señales corruptas o nulas.
  * Dominar el uso de estructuras de control y acumuladores optimizados para datos de planta.
* **Resultado esperado:**
  * El alumno implementa algoritmos SCADA robustos, con validación de rangos y manejo garantizado de nulos y excepciones de tipo.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 1.1: Recorrido del Dato de Planta y Primer Script Transform (Tema 1)

* **Objetivos:**
  * Rastrear una señal industrial desde su origen hasta la visualización en pantalla.
  * Implementar una transformación de datos que convierta valores analógicos y códigos brutos en información estructurada para el operador.
* **Resultado esperado:**
  * Un componente de interfaz vinculado a un tag mediante un _Script Transform_ funcional que valida calidades, realiza conversión de escala y devuelve un objeto formateado con texto de estado y color visual.

***

#### 🧪 Laboratorio 1.2: Auditoría de Scopes y Guía de Ubicación de Scripts (Tema 2)

* **Objetivos:**
  * Experimentar las diferencias operativas y de contexto ejecutando código en Script Console, eventos de cliente y eventos de Gateway.
  * Identificar el sobre-disparo (_over-triggering_) de scripts en bindings de alta frecuencia.
* **Resultado esperado:**
  * Un documento de referencia ("Guía de Ubicación") validado experimentalmente, reconociendo qué funciones del API `system.*` están permitidas en cada scope y evitando bloqueos de interfaz.

***

#### 🧪 Laboratorio 1.3: Compatibilidad Jython 2.7 e Integración con Clases Java (Tema 3)

* **Objetivos:**
  * Refactorizar scripts con errores típicos de migración desde Python 3 (cadenas Unicode, formatos de texto y división).
  * Importar y consumir paquetes nativos de Java dentro de la Script Console para operaciones de alta precisión y formateo temporal.
* **Resultado esperado:**
  * Un conjunto de scripts corregidos que operan sin excepciones sobre Jython 2.7, integrando clases Java (`java.lang.Math`, `java.text.SimpleDateFormat`) de manera nativa.

***

#### 🧪 Laboratorio 1.4: Lógica SCADA Defensiva, Tratamiento de Nulos y Acumuladores (Tema 4 - Parte 1)

* **Objetivos:**
  * Procesar un lote heterogéneo de lecturas de máquina con presencia de datos corruptos, valores fuera de rango y campos `None`.
  * Construir un algoritmo de acumulación para calcular totales de producción, disponibilidad y clasificación de paradas.
* **Resultado esperado:**
  * Un script modular ejecutado en consola que procesa la colección completa de datos de prueba sin lanzar excepciones, generando un resumen consolidado con métricas de rendimiento y avisos formateados para el operario.
