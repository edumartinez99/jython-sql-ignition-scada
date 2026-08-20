# Sesión 6: Persistencia Transaccional con system.db, Script Transforms y Arquitectura de Eventos

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Prepared Statements y Funciones Dinámicas `system.db` (Tema 12)

* **Conceptos clave a tratar:**
  * Ecosistema de funciones de base de datos en Ignition: comparativa entre `runQuery`, `runPrepQuery`, `runScalarPrepQuery`, `runPrepUpdate` y `runNamedQuery`.
  * Mecánica de _Prepared Statements_ con marcadores de posición `?` y listas de argumentos dinámicos tipados.
  * Procesamiento de retornos: manipulación de `PyDataSet` en lecturas tabulares con `runPrepQuery` y optimización de celdas unitarias con `runScalarPrepQuery`.
  * Ejecución de modificaciones DML con `runPrepUpdate` y validación estricta del número de filas afectadas (`rowsAffected`).
  * Enrutamiento multitabla y multientorno mediante el parámetro `database` para aislar conexiones JDBC.
  * Tratamiento defensivo: gestión de resultados vacíos, `None`, timeouts JDBC y caídas de enlace de red.
  * Criterios arquitectónicos: cuándo recurrir a Prepared Statements dinámicos en la Project Library frente al uso estándar de Named Queries.
  * Eliminación de funciones obsoletas o patrones inseguros de concatenación heredados.
* **Objetivos:**
  * Dominar el uso de sentencias preparadas nativas (`system.db.*Prep*`) para consultas y escrituras dinámicas en scripts.
  * Aprender a validar el impacto de las operaciones DML mediante el control de filas afectadas.
* **Resultado esperado:**
  * El alumno escribe scripts de base de datos seguros, parametrizados y resilientes ante fallos de conexión o respuestas vacías.

***

#### 📘 Bloque Teórico 2: Transacciones ACID, Consistencia y Control de Concurrencia (Tema 13)

* **Conceptos clave a tratar:**
  * Principio de Atomicidad e Integridad Transaccional (_Todo o Nada_) en operaciones críticas de planta.
  * Casos industriales representativos: Cabecera + Líneas de orden, Evento + Detalle de avería, Modificación de parámetro + Auditoría, Declaración de producción + Descuento de stock.
  * Ciclo de vida transaccional en Ignition: `system.db.beginTransaction`, `commitTransaction`, `rollbackTransaction` y `closeTransaction` en bloques `try / except / finally`.
  * Validación rigurosa de precondiciones antes de abrir transacciones: comprobación de existencia, estados operativos válidos y permisos de usuario.
  * Control de concurrencia y mitigación del antipatrón _“leer valor $\rightarrow$ calcular en memoria $\rightarrow$ escribir valor”_.
  * Estrategias de bloqueo lógico (_Optimistic Locking_ con control de versión/timestamp y _Pessimistic Locking_).
  * Separación de responsabilidades: aislamiento de la lógica transaccional en el Gateway / Project Library, desacoplándola del renderizado de interfaz.
* **Objetivos:**
  * Comprender los mecanismos de consistencia transaccional y prevención de colisiones por concurrencia en bases de datos SCADA.
  * Aprender a programar secuencias de escritura atómicas multitabla con soporte integral de rollback.
* **Resultado esperado:**
  * El alumno implementa transacciones industriales blindadas que garantizan que los datos nunca queden a medias ante una falla en mitad del proceso.

***

#### 📘 Bloque Teórico 3: Script Transforms en Bindings y Rendimiento de Interfaz (Tema 14)

* **Conceptos clave a tratar:**
  * Definición y ciclo de vida de un _Script Transform_: recepción del parámetro de entrada `value` y generación de una salida transformada.
  * Comparativa frente a _Expression Transforms_, _Map Transforms_ y _SQL Bindings_: cuándo aporta valor Jython y cuándo genera sobrecarga innecesaria.
  * Casos de uso recomendados: conversión de códigos numéricos a diccionarios de estado (texto, clases CSS, colores, iconos) y conversión de unidades.
  * **Antipatrón crítico de rendimiento:** por qué está terminantemente prohibido ejecutar consultas SQL dentro de Script Transforms en bindings frecuentes.
  * Tratamiento defensivo del parámetro `value`: protección contra `None`, estructuras vacías y anomalías de lectura previa sin romper la vista.
  * Guía de estilo y modularidad: mantenimiento de transforms ligeros (<10 líneas) delegando la lógica compleja en la Project Library.
  * Principio de función pura: los transforms no deben producir efectos secundarios ni ejecutar escrituras en tags o base de datos.
* **Objetivos:**
  * Aprender a diseñar transformaciones de datos eficientes, seguras y ligeras en las propiedades de componentes visuales.
  * Erradicar prácticas que bloquean el hilo de renderizado de la interfaz gráfica.
* **Resultado esperado:**
  * El alumno construye Script Transforms concisos y tolerantes a nulos que delegan la lógica pesada en librerías compartidas.

***

#### 📘 Bloque Teórico 4: Arquitectura de Eventos y Mensajería Desacoplada (Tema 15)

* **Conceptos clave a tratar:**
  * Taxonomía y ámbito de eventos: _Component Events_, _Session Events_ (Perspective), _Client Events_ (Vision), _Gateway Events_ y _Message Handlers_.
  * Diferencias de ejecución: scripts de Gateway (background permanente en servidor) vs scripts de cliente (JVM local en Vision o sub-scope web en Perspective).
  * Diseño de eventos no bloqueantes: prevención del congelamiento del _UI Thread_ ante tareas pesadas de red o reportes.
  * Inicialización de sesiones (_Session Startup Scripts_): carga de contexto del operador, permisos, idioma y variables de arranque.
  * Comunicación asíncrona desacoplada mediante **Message Handlers** (`system.util.sendMessage` / `system.perspective.sendMessage`) entre vistas, páginas y Gateway.
  * Trazabilidad y gobernanza de eventos: documentación de disparadores (_triggers_) y efectos secundarios.
  * Matriz de decisión técnica para ubicar procesos en Gateway, sesión, cliente o Project Library.
* **Objetivos:**
  * Dominar la infraestructura de eventos de Ignition y los mecanismos de comunicación desacoplada.
  * Aprender a coordinar la interacción entre vistas y procesos de fondo sin generar dependencias rígidas.
* **Resultado esperado:**
  * El alumno estructura la lógica de eventos de forma ordenada, utilizando Message Handlers para actualizar interfaces sin recargas forzadas ni acoplamientos.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 6.1: Operaciones Avanzadas con `system.db.*Prep*` y Manejo de Respuestas (Tema 12)

* **Objetivos:**
  * Ejecutar consultas escalares con `system.db.runScalarPrepQuery` y lecturas tabulares parametrizadas con `system.db.runPrepQuery`.
  * Implementar actualizaciones controladas mediante `system.db.runPrepUpdate` evaluando el resultado de `rowsAffected` y gestionando respuestas vacías o nulas.
* **Resultado esperado:**
  * Un script de persistencia probado que ejecuta lecturas y modificaciones directas de forma segura, validando que solo los registros autorizados sean alterados.

***

#### 🧪 Laboratorio 6.2: Transacción Atómica Completa con Rollback y Precondiciones (Tema 13)

* **Objetivo:**
  * Diseñar una función transaccional en Project Library que cierre una orden de fabricación, descuente el stock de materia prima y registre la auditoría como una unidad indivisible.
  * Provocar un error intencionado para verificar que el mecanismo de `rollbackTransaction` anula todas las modificaciones previas, asegurando que ninguna tabla quede modificada.
* **Resultado esperado:**
  * Un procedimiento transaccional robusto en `try / except / finally` que garantiza la consistencia total de la base de datos ante cualquier excepción en tiempo de ejecución.

***

#### 🧪 Laboratorio 6.3: Diseño de Script Transforms Ligeros y Seguros en Perspective (Tema 14)

* **Objetivos:**
  * Crear un _Script Transform_ defensivo sobre una propiedad de estado vinculada a un tag, validando la entrada `value` contra `None` o calidad mala.
  * Delegar la asignación de estilos visuales dinámicos (texto, color de fondo e icono) en una función de la Project Library.
* **Resultado esperado:**
  * Un componente visual en el Designer que actualiza su apariencia en tiempo real de forma fluida, manteniendo un estado neutro controlado cuando el dato de entrada no es válido.

***

#### 🧪 Laboratorio 6.4: Comunicación Desacoplada con Message Handlers y Eventos de Sesión (Tema 15)

* **Objetivos:**
  * Configurar un diálogo modal en Perspective que emita un evento asíncrono mediante `system.perspective.sendMessage` tras confirmar una acción de planta.
  * Configurar un _Message Handler_ en la vista principal del dashboard que capture el mensaje y refresque selectivamente sus datos sin recargar la pantalla completa.
* **Resultado esperado:**
  * Un flujo de comunicación desacoplado entre componentes y vistas independientes que sincroniza la interfaz en tiempo real sin acoplamiento rígido de código.
