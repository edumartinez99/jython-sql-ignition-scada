---
description: Optimización Avanzada de Consultas SQL y Rendimiento de Scripts Jython
---

# Sesión 8

### 1. Repaso Inicial y Resolución de Dudas de la Sesión 7

#### Objetivos

* Consolidar el uso de operaciones masivas por lotes (_Batch Operations_) con `system.tag.readBlocking` y `writeBlocking`.
* Revisar la lógica de cálculo de calendarios industriales, días operativos y turnos nocturnos que cruzan la medianoche.

#### Contenidos

* Repaso de las consultas sobre la validación obligatoria de `QualityCode` en objetos `QualifiedValue`.
* Revisión de las dudas en la delimitación de ventanas temporales con objetos `java.util.Date` en Named Queries.
* Comprobación del correcto filtrado por rangos de fecha en las tablas del sandbox.

#### Resultado esperado

* Fijación de las directrices de acceso eficiente a tags y manejo de fechas, asegurando la base técnica para abordar la optimización de recursos y rendimiento del Gateway.

***

### 2. Tema 18: Optimización Avanzada de Consultas SQL Industriales

#### Objetivos

* Comprender el impacto en la memoria JVM y el pool JDBC derivado del uso indiscriminado de `SELECT *` y escaneos completos de tabla.
* Aplicar el principio de SARGability (_Search Argument Able_) en cláusulas `WHERE` para forzar el uso de índices B-Tree en la base de datos.
* Diseñar estrategias de indexación compuesta y separación de capas entre consultas transaccionales rápidas y consultas analíticas agregadas.

#### Contenidos

**1. Proyección Controlada, Serialización JDBC y Presión de Memoria en la JVM**

* **El peligro crítico del anti-patrón `SELECT *` en entornos SCADA:**
  * En plantas industriales, las tablas de telemetría, eventos de proceso (`REG_OPERADORS`, `ins_Registre_Actuacions`), alarmas y recetas contienen habitualmente decenas de columnas (campos de texto libre, trazas de depuración, metadatos JSON y configuraciones de ingeniería).
  * *Sobrecarga de red y driver JDBC:* El comodín `SELECT *` fuerza al motor SQL a leer la totalidad de las páginas de datos en disco, transferir bloques innecesarios a través de la red de control y serializar cada campo celda a celda en el driver JDBC hacia el Gateway.
  * *Presión destructiva sobre el Garbage Collector de la JVM:* Cada fila transferida se desenvuelve como un array de objetos en la memoria heap de Ignition. Si una consulta histórica recupera 50.000 filas con 20 columnas, se instancian más de 1.000.000 de objetos temporales en memoria RAM. Esto dispara pausas mayores de recolección de basura (*Stop-the-World GC Pauses*), generando micro-congelamientos en las pantallas de los operadores y retardos transitorios en el Gateway.
* **El principio de proyección estricta y mínima necesaria:**
  * Regla de diseño: Proyectar únicamente las columnas indispensables para la lógica de cálculo o el renderizado visual (`SELECT id, Data, Operador, Accio FROM REG_OPERADORS`).
  * *Habilitación de consultas cubiertas (*Covering Indexes*):* Si todas las columnas requeridas en el `SELECT` forman parte de un índice B-Tree, el motor RDBMS resuelve la consulta íntegramente leyendo el índice en memoria RAM sin acceder a las páginas de datos en disco (*Index-Only Scan*), multiplicando la velocidad de respuesta por órdenes de magnitud.

**2. SARGability e Índices B-Tree: Anatomía del Acceso Quirúrgico a Datos**

* **El concepto de SARGability (*Search Argument Able*):**
  * Un predicado o filtro en la cláusula `WHERE` es *sargable* cuando el optimizador de base de datos puede emplear un índice de árbol B (*B-Tree*) para acotar directamente las filas coincidentes sin evaluar toda la tabla.
  * *Operadores sargables nativos:* `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `IS NULL` y `LIKE 'patron%'` (búsqueda por prefijo exacto).
  * *Operadores no sargables habituales:* `LIKE '%patron'`, `<>`, `!=`, `NOT IN` y, de forma crítica, **cualquier función matemática, de cadena o de fecha aplicada sobre una columna indexada**.
* **El colapso por funciones sobre columnas temporales e indexadas:**
  * Análisis de los anti-patrones industriales más comunes en consultas históricas:
    ```sql
    -- ANTI-PATRONES: Inutilizan índices B-Tree y fuerzan Full Table Scan
    WHERE EXTRACT(YEAR FROM Data) = 2020
    WHERE DATE_FORMAT(Data, '%Y-%m-%d') = '2026-10-22'
    WHERE CAST(id_equipo AS VARCHAR) = 'BOMBA_01'
    WHERE UPPER(Operador) = 'JUAN'
    ```
  * *Mecánica interna en el motor SQL:* Para evaluar `EXTRACT(YEAR FROM Data) = 2020`, el optimizador desconoce el resultado de la función hasta que procesa cada fila de forma individual. Por tanto, se ve forzado a leer todas las páginas de la tabla desde disco, aplicar la función sobre cada registro (ej. 2.000.000 de filas) y descartar las que no cumplan la condición.
  * *Consecuencia en planta:* Escaneo completo de tabla (*Full Table Scan*), saturación de la E/S de disco, picos de CPU del 100% en el servidor de base de datos y retención excesiva de conexiones en el pool JDBC.
* **El estándar sargable por rangos binarios:**
  * En lugar de calcular o formatear el valor de la columna, se transforman los límites de comparación en literales o parámetros fijos evaluados en tiempo de consulta:
    ```sql
    -- PATRÓN SARGABLE ÓPTIMO: Búsqueda quirúrgica por índice (Index Seek)
    WHERE Data >= '2020-01-01 00:00:00' AND Data < '2021-01-01 00:00:00'
    ```
  * *Mecánica del optimizador:*
    * La columna permanece pura (`Data`), permitiendo una búsqueda binaria directa por índice (*Index Seek* o *Index Range Scan*).
    * El motor desciende por el árbol B hasta la primera clave que cumple `>= '2020-01-01 00:00:00'` y lee secuencialmente hasta el límite estricto `< '2021-01-01 00:00:00'`, recuperando miles de filas en pocos milisegundos con un coste mínimo de lectura de páginas.
    * El uso del operador menor estricto `<` con el primer instante del período siguiente evita desajustes de milisegundos (cubre exactamente hasta el microsegundo `2020-12-31 23:59:59.999999`).

**3. Estrategias de Indexación Compuesta y Separación Transaccional/Analítica**

* **Índices compuestos B-Tree y la regla del prefijo más a la izquierda (*Leftmost Prefix Rule*):**
  * En registros de planta, las consultas operativas habitualmente combinan el identificador de máquina y una ventana temporal.
  * *Definición de índice compuesto:* `CREATE INDEX idx_reg_equip_data ON REG_OPERADORS (Equip, Data);`.
  * *Jerarquía de columnas:* La columna filtrada por igualdad exacta (`Equip = 'EDAR_BOMBA_01'`) debe situarse en la primera posición del índice, seguida de la columna de rango temporal (`Data >= ...`). Esto permite acotar primero el sub-árbol del equipo y luego barrenar el intervalo temporal.
  * *Ordenación gratuita en disco:* Como las filas dentro del índice B-Tree ya están físicamente ordenadas por `Data`, la cláusula `ORDER BY Data DESC` se resuelve directamente sin necesidad de algoritmos de ordenación en memoria RAM (*Sort Operations*).
* **Separación de cargas transaccionales y analíticas en SCADA (OLTP vs. OLAP):**
  * *El conflicto de concurrencia en planta:* Paneles de supervisión mensual, cálculos de OEE o auditorías analíticas ejecutando consultas pesadas sobre tablas transaccionales en las que los PLCs y autómatas insertan telemetría en tiempo real.
  * *Bloqueos y contención de hilos:* Consultas analíticas extensas adquieren bloqueos compartidos (*Shared Locks*), reteniendo a los hilos transaccionales que intentan adquirir bloqueos exclusivos (*Exclusive Locks*).
  * *Estrategias arquitectónicas de desacoplamiento:*
    * *Tablas de agregación histórica precalculada (Rollup / Summary Tables):* Procesos desatendidos en el Gateway (Gateway Scheduled Scripts mediante CRON en cambio de turno o fin de jornada) calculan los acumulados diarios y los insertan en tablas de resumen (`summary_daily_kpis`). Los cuadros de mando consultan la tabla agregada (cientos de filas) en menos de 1 ms, sin degradar la tabla transaccional de millones de filas.
    * *Réplicas de lectura (*Read Replicas*):* Configuración de conexiones de base de datos secundarias en el Gateway dedicadas a consultas analíticas pesadas, aislando el motor transaccional primario.

```mermaid
flowchart TD
    subgraph Consulta_No_Sargable [Consulta No Sargable: Invalida Indices]
        FuncFilter["WHERE EXTRACT(YEAR FROM Data) = 2020"]
        TableScan["Full Table Scan: Lee Millones de Filas de Disco -> Alta Latencia"]
        FuncFilter --> TableScan
    end

    subgraph Consulta_Sargable [Consulta Sargable: Optimizada]
        DirectRange["WHERE Data >= '2020-01-01 00:00:00' AND Data < '2021-01-01 00:00:00'"]
        IndexSeek["Index Seek: Acceso Directo por Arbol B-Tree -> Milisegundos"]
        DirectRange --> IndexSeek
    end
```

#### Resultado esperado

* Capacidad para diagnosticar consultas lentas, reescribir filtros SQL garantizando el uso de índices y diseñar estructuras de datos optimizadas para reporting industrial.

***

### 3. Laboratorio 8.1: Optimización de Consultas SQL con Named Queries, Query Binding y Medición de Latencia en Perspective

#### Objetivos

* Construir y contrastar dos _Named Queries_ en Ignition: una consulta histórica no optimizada (con `SELECT *` y función `EXTRACT` en el `WHERE`) frente a una consulta optimizada bajo estándares sargables con rango de fechas e índices B-Tree.
* Alimentar el componente Table de Perspective mediante un **Query Binding nativo** hacia la Named Query optimizada, erradicando scripts innecesarios en la interfaz.
* Implementar un selector de modo (*Legacy No Optimizada* vs *Optimizada Sargable*) y un **Label de telemetría de latencia** que mida en caliente el tiempo de ejecución en milisegundos (`ms`) mediante `System.nanoTime()`.

#### Resultado esperado

* Dos Named Queries operativas sobre `ins_Registre_Actuacions` en `DB_Admin`.
* Vista en Perspective con una tabla que visualiza los registros y un Label interactivo que constata en tiempo real cómo la consulta sargable reduce la latencia en más de un 80% frente al escaneo completo de tabla (*Full Table Scan*).

#### Paso a Paso para la Realización

**Paso 1: Creación de las 2 Named Queries en Ignition Designer**

1. En el **Project Browser**, navegar a **Server Scripting -> Named Queries**.
2. Crear la carpeta `Actuacions` (clic derecho -> *New Folder...*).
3. **Named Query 1: `Actuacions/Consulta_NoOptimizada`**
   * **Database:** `DB_Admin`
   * **Query Type:** *Query*
   * **Parameters:** Añadir parámetro `targetYear` (Tipo: `Integer`, valor de prueba: `2026`), `targetMonth` (Tipo: `Integer`, valor de prueba: `09`), `targetDay` (Tipo: `Integer`, valor de prueba: `28`).
   * **SQL Query:**
     ```sql
     -- ANTI-PATRÓN: SELECT * y función EXTRACT en WHERE (Full Table Scan)
     SELECT * 
     FROM ins_Registre_Actuacions 
     WHERE EXTRACT(DAY FROM DataHora) = :targetDay
     AND EXTRACT(MONTH FROM DataHora) = :targetMonth
     AND EXTRACT(YEAR FROM DataHora) = :targetYear
     ORDER BY DataHora DESC
     ```
4. **Named Query 2: `Actuacions/Consulta_Optimizada`**
   * **Database:** `DB_Admin`
   * **Query Type:** *Query*
   * **Parameters:**
     * `startDate` (Tipo: `DateTime`, valor de prueba: `2026-09-28 00:00:00`)
     * `endDate` (Tipo: `DateTime`, valor de prueba: `2026-09-29 00:00:00`)
   * **SQL Query:**
     ```sql
     -- ESTÁNDAR SARGABLE: Proyección estricta + Rango temporal limpio sobre índice B-Tree
     SELECT id, Equip, Descripcio, Usuari, DataHora 
     FROM ins_Registre_Actuacions 
     WHERE DataHora >= :startDate AND DataHora < :endDate
     ORDER BY DataHora DESC
     ```

**Paso 2: Comprobación en la pestaña Testing del Designer**
1. En cada Named Query, abrir la pestaña **Testing**.
2. Pulsar **Execute**.
3. Observar el mensaje inferior del Designer: `Query executed in XX ms`, comprobando la diferencia de tiempos entre la versión con `EXTRACT` y la versión con rango sargable.

**Paso 3: Creación de la Vista en Perspective (`Views/Laboratorios/Lab8_1_SQLBenchmark`)**
1. En **Perspective -> Views**, crear la vista `Laboratorios/Lab8_1_SQLBenchmark` (`Flex Container`, `column`).
2. Configurar las propiedades personalizadas en la vista (**custom**):
   * `custom.queryMode`: `"OPTIMIZADA"` (string: `"NO_OPTIMIZADA"` o `"OPTIMIZADA"`)
   * `custom.startDate`: `"2026-09-28 00:00:00"` (string o Date)
   * `custom.endDate`: `"2026-09-29 00:00:00"` (string o Date)
   * `custom.targetYear`: `2026` (integer)
   * `custom.targetMonth`: `09` (integer)
   * `custom.targetDay`: `28` (integer)
   * `custom.latencyLabel`: `"Selecciona un modo y pulsa 'Ejecutar Consulta'"` (string)

**Paso 4: Configuración de la Tabla con Query Binding Nativo**
1. Añadir el componente **Table** (`table_Actuaciones`).
2. En la propiedad `props.data`, abrir la ventana de binding (**cadena**).
3. Seleccionar **Query -> Named Query**:
   * **Path:** `Actuacions/Consulta_Optimizada`
   * **Parameters:**
     * `startDate` $\rightarrow$ Binding a `{view.custom.startDate}`
     * `endDate` $\rightarrow$ Binding a `{view.custom.endDate}`
4. Pulsa **OK**. La tabla carga directamente los registros desde base de datos sin ejecutar una sola línea de script en el cliente.

**Paso 5: Controles de Prueba y Medición de Latencia en el Label**
1. Añadir un contenedor horizontal sobre la tabla con:
   * **Dropdown / Radio Group (`drop_Modo`):**
     * `props.options`:
       ```json
       [
         {"value": "NO_OPTIMIZADA", "label": "Modo Legacy (SELECT * + EXTRACT)"},
         {"value": "OPTIMIZADA", "label": "Modo Optimizado (Sargable + Índices)"}
       ]
       ```
     * `props.value`: Binding bidireccional (**Bidirectional**) a `view.custom.queryMode`.
   * **Botón Ejecutar (`btn_Ejecutar` -> `events -> onActionPerformed`):**
     ```python
     from java.lang import System

     mode = self.view.custom.queryMode

     if mode == "NO_OPTIMIZADA":
         params = {"targetYear": self.view.custom.targetYear,
         "targetMonth": self.view.custom.targetMonth,
         "targetDay": self.view.custom.targetDay}
         query_path = "Actuacions/Consulta_NoOptimizada"
     else:
         params = {"startDate": self.view.custom.startDate, "endDate": self.view.custom.endDate}
         query_path = "Actuacions/Consulta_Optimizada"

     # 1. Medir latencia de ejecucion con precision de nanosegundos en la JVM
     t0 = System.nanoTime()
     ds = system.db.runNamedQuery(query_path, params)
     t1 = System.nanoTime()

     elapsed_ms = round((t1 - t0) / 1000000.0, 2)
     filas = ds.getRowCount() if ds is not None else 0

     # 2. Actualizar datos de la tabla y telemetria en el Label
     self.getSibling("table_Actuaciones").props.data = ds

     if mode == "NO_OPTIMIZADA":
         self.view.custom.latencyLabel = "⚠️ Legacy: {} ms (Full Table Scan | {} filas)".format(elapsed_ms, filas)
     else:
         self.view.custom.latencyLabel = "⚡ Optimizada: {} ms (Index Seek B-Tree | {} filas)".format(elapsed_ms, filas)
     ```
   * **Label de Latencia (`lbl_Latencia`):**
     * `props.text`: Binding a `view.custom.latencyLabel`.
     * `props.style`: Tamaño de fuente destacado (`16px`, negrita).

**Paso 6: Verificación Interactiva**
1. Abrir la vista en Preview (`Ctrl+P`).
2. Seleccionar **Modo Legacy** y pulsar **Ejecutar Consulta**: el Label reporta la latencia elevada (~30-50 ms o superior) debido al escaneo completo de tabla.
3. Conmutar a **Modo Optimizado** y pulsar **Ejecutar Consulta**: el Label reporta inmediatamente un tiempo drásticamente inferior (~2-4 ms), demostrando en vivo la eficacia de la indexación sargable sobre `ins_Registre_Actuacions`.

***

### 4. Tema 19: Optimización y Rendimiento de Scripts Jython en Ignition

#### Objetivos

* Detectar "puntos calientes" (_Hotspots_) de ejecución excesiva en bindings, Script Transforms y eventos reactivos.
* Eliminar conversiones redundantes de tipos en bucles para reducir la presión sobre el recolector de basura de la JVM (_Garbage Collector_).
* Implementar patrones de caché en memoria de Gateway con tiempo de expiración (TTL) para datos de baja volatilidad.

#### Contenidos

**1. Detección y Mitigación de Scripts Sobrecargados y Puntos Calientes (*Hotspots*)**

* **La naturaleza del motor Jython en la arquitectura de Ignition:**
  * Jython 2.7 ejecuta código Python compilándolo dinámicamente a bytecode sobre la Máquina Virtual de Java (JVM) del Gateway o del cliente.
  * A diferencia de Java puro compilado estáticamente, el tipado dinámico de Python requiere resolución en tiempo de ejecución. Los bucles masivos o conversiones redundantes tienen un coste en ciclos de CPU mucho más elevado.
* **Puntos calientes (*Hotspots*) habituales en sistemas SCADA:**
  * *Script Transforms en Bindings de alta frecuencia:* Transformadores asociados a tags que cambian cada 100 ms o a propiedades que se recalculan ante cualquier evento de interfaz en Perspective. Si un transform contiene llamadas bloqueantes o lógica compleja, satura los núcleos de CPU del servidor multiplicando la carga por cada cliente web conectado.
  * *Gateway Timer Scripts mal dimensionados:* Tareas cíclicas configuradas en el pool compartido (*Shared Pool*) ejecutando scripts cuya duración excede el intervalo de muestreo (ej. script de 800 ms corriendo cada 500 ms), provocando el encolamiento y congelamiento de todos los temporizadores del Gateway.
  * *Tag Change Scripts con flancos no filtrados:* Scripts de cambio de tag que omiten la validación `if not initialChange:`, disparando cientos de ejecuciones concurrentes cada vez que el Gateway arranca o se guarda el proyecto en el Designer.
* **Perfilado de rendimiento de alta precisión en Jython:**
  * Medición de tiempos de ejecución utilizando las APIs nativas de la JVM:
    * `java.lang.System.nanoTime()`: Medición de intervalos ultracortos con precisión de nanosegundos ($10^{-9}\text{ s}$), ideal para medir funciones de transformación de datos y micro-benchmarking en memoria RAM.
    * `java.lang.System.currentTimeMillis()`: Reloj de época para cálculo de duraciones de proceso, control de tiempos de expiración y almacenamiento de marcas temporales en milisegundos.
  * *Cálculo de latencia en milisegundos con alta resolución:*
    $$\Delta t_{\text{ms}} = \frac{t_1 - t_0}{1.000.000,0}$$

**2. Optimización de Memoria, Bucles e Iteraciones en la JVM**

* **El coste oculto de las conversiones redundantes de datasets:**
  * *Análisis del anti-patrón de manipulación:*
    ```python
    # ANTI-PATRÓN: Múltiples conversiones encadenadas que saturan el Heap
    raw_ds = system.db.runQuery("SELECT ...")
    py_ds = system.dataset.toPyDataSet(raw_ds)
    lista_dicts = []
    for row in py_ds:
        lista_dicts.append({"id": row["id"], "nombre": row["nombre"]})
    final_ds = system.dataset.toDataSet(headers, [list(d.values()) for d in lista_dicts])
    ```
  * *Presión sobre la memoria Heap:* Cada conversión intermedia (`Dataset` -> `PyDataSet` -> `dict` -> `list` -> `Dataset`) crea nuevos objetos efímeros en la memoria de la JVM. En bucles de procesamiento histórico o telemetría, esto fragmenta la memoria y provoca una alta frecuencia de ciclos del recolector de basura (*GC pressure*), ralentizando el rendimiento general del Gateway.
* **El estándar de acceso directo con `PyDataSet`:**
  * `system.dataset.toPyDataSet` (o el retorno directo de `system.db.runPrepQuery`) envuelve el dataset nativo de Java en un adaptador ligero sin duplicar los datos en memoria.
  * Ofrece acceso indexado directo por nombre de columna (`row["columna"]`) o índice ordinal (`row[0]`) con complejidad O(1).
  * Si el objetivo es iterar para renderizar una tabla o evaluar condiciones lógicas, debe iterarse directamente sobre el `PyDataSet` sin conversiones intermedias ni generación de diccionarios auxiliares innecesarios.
* **Comprensiones de listas (*List Comprehensions*):**
  * Las comprensiones de listas (`[expr for item in iterable]`) se ejecutan internamente a nivel de bytecode en la JVM, siendo entre un 30% y un 50% más rápidas en Jython que los bucles `for` tradicionales con llamadas continuas al método `.append()`.

**3. El Patrón de Caché en Memoria del Gateway con TTL (*Time-To-Live*)**

* **La problemática de las consultas repetitivas de datos de baja volatilidad:**
  * En entornos industriales, multitud de tablas almacenan datos que rara vez cambian: catálogos maestros de equipos, listas de recetas, parámetros de tolerancias de calidad, tablas de operadores autorizados y descripciones de averías.
  * *El anti-patrón de consulta recurrente:* Decenas de clientes SCADA o scripts de Gateway ejecutando continuamente consultas SQL (`SELECT * FROM machine_setpoints`) para leer exactamente la misma información estática. Cada petición abre conexiones JDBC, transmite datos por la red y consume CPU del motor de base de datos.
* **Arquitectura del patrón de Caché en Memoria RAM del Gateway:**
  * Implementación de un almacén en memoria a nivel de `Project Library` (diccionario global `_CACHE_STORE`).
  * En Ignition, los módulos de `Project Library` mantienen su estado en memoria dentro del contexto del Gateway entre ejecuciones de scripts, lo que permite conservar variables globales de caché de forma persistente mientras el servicio de Gateway esté activo.
  * *Mecánica del patrón `get_or_set`:*
    1. Se calcula una clave unívoca para el recurso (`cache_key`).
    2. Se comprueba si la clave existe en memoria y si su tiempo transcurrido no supera el TTL configurado:
       $$\text{now\_ms} - \text{timestamp} < \text{TTL\_ms}$$
    3. **Cache Hit (Acierto de Caché):** Se devuelve de forma inmediata el dato residente en memoria RAM. Tiempo de respuesta: **< 0,05 ms** (cero consultas SQL, cero consumo de conexiones JDBC y cero latencia de red).
    4. **Cache Miss (Fallo o Expiración):** Se invoca la función original de extracción (`fetch_function`), se almacena el nuevo resultado junto con la marca temporal actual (`System.currentTimeMillis()`) y se retorna el dato fresco. Tiempo de respuesta: ~15 a 50 ms (únicamente en la primera ejecución o tras la caducidad del TTL).
* **Estrategias de invalidación y mantenimiento de caché:**
  * *Expiración automática por tiempo (TTL):* Refresco periódico desatendido garantizado (ej. cada 60 segundos o 10 minutos).
  * *Invalidación proactiva manual (*Cache Invalidation*):* Invocación de `clear_cache(cache_key)` inmediatamente después de que un operador guarde una modificación en la pantalla de recetas o consignas, forzando a que la siguiente lectura recargue los datos frescos de la base de datos sin demoras.
  * *Pre-calentamiento desatendido mediante Gateway Timer Scripts (*Cache Warm-Up*):* Para erradicar la penalización por fallo de caché (*Cache Miss Penalty*) que sufre el primer operador tras expirar el TTL, un temporizador en el Gateway ejecuta periódicamente la consulta en segundo plano con un hilo dedicado (`Dedicated Thread Pool`), manteniendo los datos permanentemente calientes en RAM con latencia cero (< 0.05 ms) para todos los clientes conectados.
  * *Invalidación reactiva guiada por eventos (*Event-Driven Invalidation* con Tag Change Scripts):* Cuando un autómata PLC, sistema MES o proceso externo modifica las consignas en base de datos, la variación de un tag disparador (`[default]Config/ReloadCatalogs` o cambio de versión de receta) desencadena un *Gateway Tag Change Script* que purga al instante la entrada obsoleta de la caché sin requerir reinicios del sistema.


```mermaid
flowchart TD
    Req[Peticion de Datos: Receta / Catalogo] --> CheckCache{¿Existe en Memoria y TTL Vigente?}
    
    CheckCache -->|Cache Hit: Valido| ReturnRAM[Retorno Inmediato desde Memoria RAM: 0 ms de BD]
    
    CheckCache -->|Cache Miss: Expirado o Inexistente| FetchDB[Consulta SQL a Base de Datos]
    FetchDB --> UpdateCache[Almacena en Diccionario con Timestamp Actual]
    UpdateCache --> ReturnFresh[Retorno de Datos Actualizados]
```

#### Resultado esperado

* Dominio de las técnicas de perfilado de scripts, eliminación de redundancias de memoria y diseño de sistemas de caché en el Gateway para maximizar la velocidad de respuesta del sistema SCADA.

***

### 5. Laboratorio 8.2: Gestor de Caché Genérico con TTL en Memoria de Gateway y Monitor en Perspective

#### Objetivos

* Construir un módulo en `Project Library` (`project.util.cache`) que implemente un almacén de caché en memoria RAM con control de expiración por TTL e invalidación manual.
* Diseñar un monitor interactivo en Perspective (`Views/Laboratorios/Lab8_2_CacheMonitor`) con scripts de 1 o 2 líneas para visualizar la aceleración de Cache Hit frente a Cache Miss.
* Implementar la automatización desatendida en el Gateway:
  * **Gateway Timer Script** para pre-calentamiento periódico en background (*Cache Warm-Up*).
  * **Gateway Tag Change Script** para invalidación reactiva inmediata ante pulsos de tags disparadores de recarga de recetas.

#### Resultado esperado

* Módulo de caché validado ante accesos repetitivos y expiración.
* Monitor en Perspective donde la primera consulta indica `CACHE MISS (SQL - ~20 ms)` y las siguientes `CACHE HIT (RAM - < 0.05 ms)`, con botón para invalidar manualmente la memoria.
* Gateway autónomo donde un Timer Script mantiene la caché caliente cada 60 s y un Tag Change Script sobre `[default]Config/ReloadCatalogs` purga la memoria reactivamente ante cambios en planta.

#### 4. Paso a Paso para la Realización

**Paso 1: Creación del Módulo en Project Library**

1. En **Ignition Designer**, navegar a **Server Scripting -> Project Library**.
2. Crear el script: `util.cache` (carpeta `util`, script `cache`).

**Paso 2: Implementación del Gestor de Caché con TTL y Funciones UI**

Copiar y guardar el siguiente código en `util.cache`:

```python
"""
Modulo: util.cache
Descripcion: Gestor de cache en memoria RAM de Gateway con control de TTL (Time-To-Live),
             invalidacion manual y funciones ligeras para Perspective.
Autor: Equipo SCADA
"""
from java.lang import System

# Almacen persistente en el contexto del modulo en la JVM
# Estructura interna: {'clave': {'data': objeto, 'timestamp': epoch_ms, 'hits': int}}
_CACHE_STORE = {}

def get_or_set(cache_key, fetch_function, ttl_seconds=60, *args, **kwargs):
    """
    Recupera un dato de la memoria RAM si es valido o ejecuta la funcion proveedora
    para refrescarlo y almacenarlo con timestamp actual.
    
    Args:
        cache_key (str): Identificador unico del dato en cache (ej. 'cat_setpoints').
        fetch_function (callable): Funcion proveedora si hay cache miss.
        ttl_seconds (int): Tiempo de vida del dato en segundos.
        
    Returns:
        any: El dato recuperado de memoria RAM o generado por la funcion proveedora.
    """
    global _CACHE_STORE
    logger = system.util.getLogger("SCADA.Cache")
    
    now_ms = System.currentTimeMillis()
    ttl_ms = ttl_seconds * 1000
    
    # 1. Comprobar si la clave existe y el TTL sigue vigente (Cache Hit)
    if cache_key in _CACHE_STORE:
        entry = _CACHE_STORE[cache_key]
        elapsed = now_ms - entry["timestamp"]
        
        if elapsed < ttl_ms:
            entry["hits"] += 1
            logger.debug("Cache HIT para '{}' (Hits: {}, Edad: {} ms)".format(cache_key, entry["hits"], elapsed))
            return entry["data"]
            
    # 2. Si no existe o expiro el TTL (Cache Miss): Ejecutar funcion proveedora
    logger.info("Cache MISS para '{}'. Consultando origen de datos...".format(cache_key))
    fresh_data = fetch_function(*args, **kwargs)
    
    # 3. Guardar en memoria con timestamp actual
    _CACHE_STORE[cache_key] = {
        "timestamp": now_ms,
        "data": fresh_data,
        "hits": 0
    }
    
    return fresh_data


def clear_cache(cache_key=None):
    """
    Invalida una clave especifica o vacia la totalidad de la memoria cache.
    """
    global _CACHE_STORE
    logger = system.util.getLogger("SCADA.Cache")
    
    if cache_key:
        if cache_key in _CACHE_STORE:
            del _CACHE_STORE[cache_key]
            logger.info("Clave de cache eliminada: '{}'".format(cache_key))
    else:
        total = len(_CACHE_STORE)
        _CACHE_STORE.clear()
        logger.info("Cache total limpiada ({} entradas eliminadas)".format(total))


def get_cache_stats():
    """
    Retorna un diccionario con el estado actual y metricas de la memoria cache.
    """
    global _CACHE_STORE
    now_ms = System.currentTimeMillis()
    stats = {}
    
    for k, v in _CACHE_STORE.items():
        stats[k] = {
            "age_seconds": round((now_ms - v["timestamp"]) / 1000.0, 1),
            "total_hits": v["hits"]
        }
    return stats


# ==============================================================================
# FUNCIONES AUXILIARES PARA PERSPECTIVE (Mantiene los scripts de UI en 1-2 lineas)
# ==============================================================================

def fetch_catalog_setpoints():
    """
    Funcion proveedora real: consulta la tabla machine_setpoints en SANDBOX_DB.
    """
    sql = "SELECT equip, flow_sp, pressure_sp, is_active FROM machine_setpoints WHERE is_active = TRUE"
    return system.db.runQuery(sql, database="SANDBOX_DB")


def query_setpoints_cached_ui(ttl_seconds=10):
    """
    Consulta el catalogo de consignas midiendo el tiempo exacto y detectando
    si la respuesta se sirvio desde RAM (HIT) o desde Base de Datos (MISS).
    
    Returns:
        dict: {'data': dataset, 'source': str, 'elapsed_ms': float, 'hits': int}
    """
    global _CACHE_STORE
    key = "cat_setpoints"
    now_ms = System.currentTimeMillis()
    ttl_ms = ttl_seconds * 1000
    
    # Pre-comprobacion para etiquetar HIT vs MISS
    is_hit = False
    if key in _CACHE_STORE:
        if (now_ms - _CACHE_STORE[key]["timestamp"]) < ttl_ms:
            is_hit = True
            
    t0 = System.nanoTime()
    data = get_or_set(key, fetch_catalog_setpoints, ttl_seconds=ttl_seconds)
    t1 = System.nanoTime()
    
    elapsed = round((t1 - t0) / 1000000.0, 4)
    hits = _CACHE_STORE.get(key, {}).get("hits", 0)
    
    return {
        "data": data,
        "source": "MEMORIA RAM (CACHE HIT)" if is_hit else "BASE DE DATOS SQL (CACHE MISS)",
        "elapsed_ms": elapsed,
        "hits": hits
    }


def format_setpoints_table(dataset_data):
    """
    Transforma el dataset retornado en filas estructuradas para el componente Table.
    """
    if dataset_data is None:
        return []
        
    pyds = system.dataset.toPyDataSet(dataset_data)
    rows = []
    for r in pyds:
        rows.append({
            "Equipo": r["equip"],
            "Consigna_Caudal": "{:.1f} m3/h".format(r["flow_sp"]),
            "Consigna_Presion": "{:.1f} bar".format(r["pressure_sp"]),
            "Estado": "ACTIVO" if r["is_active"] else "INACTIVO"
        })
    return rows


def format_cache_stats_table(stats_dict):
    """
    Formatea las estadisticas de memoria cache para una tabla de Perspective.
    """
    if not stats_dict or not isinstance(stats_dict, dict):
        return []
        
    rows = []
    for k, v in stats_dict.items():
        rows.append({
            "Clave": k,
            "Edad_Segundos": "{} s".format(v.get("age_seconds", 0)),
            "Total_Hits_RAM": v.get("total_hits", 0)
        })
    return rows


# ==============================================================================
# FUNCIONES PARA GATEWAY EVENT SCRIPTS (TIMER Y TAG CHANGE)
# ==============================================================================

def warmup_catalog_cache(ttl_seconds=120):
    """
    Ejecutado periodicamente por un Gateway Timer Script.
    Pre-calienta y renueva la memoria en background antes de que expire el TTL,
    garantizando que ningun operador de planta experimente la latencia de un Cache Miss.
    """
    logger = system.util.getLogger("SCADA.Cache.Timer")
    logger.debug("Ejecutando warm-up de catalogo de consignas en background...")
    return get_or_set("cat_setpoints", fetch_catalog_setpoints, ttl_seconds=ttl_seconds)


def invalidate_on_tag_trigger(tag_path, reset_flag=True):
    """
    Ejecutado por un Gateway Tag Change Script ante actualizacion de recetas.
    Invalida reactivamente la cache y opcionalmente resetea el tag booleano a False.
    """
    logger = system.util.getLogger("SCADA.Cache.TagChange")
    clear_cache("cat_setpoints")
    logger.info("Cache 'cat_setpoints' invalidada reactivamente desde tag: {}".format(tag_path))
    if reset_flag:
        system.tag.writeBlocking([tag_path], [False])
```

**Paso 3: Guardar el Proyecto**

1. Guardar en el Designer (**File -> Save** o `Ctrl+S`).

**Paso 4: Pruebas y Validación desde la Script Console**

1. Abrir la **Script Console** (**Tools -> Script Console**).
2. Ejecutar la validación completa del gestor de caché:

```python
print "=== VALIDACION LABORATORIO 8.2: GESTOR DE CACHE CON TTL ==="

# Limpiar cache previa
util.cache.clear_cache("cat_setpoints")

# 1. Primera Llamada: Cache Miss (debe consultar la BD)
res1 = util.cache.query_setpoints_cached_ui(ttl_seconds=5)
print "\n1. Primera Llamada:"
print " - Origen:     ", res1["source"]
print " - Tiempo:     ", res1["elapsed_ms"], "ms"
print " - Hits en RAM:", res1["hits"]

# 2. Segunda Llamada Inmediata: Cache Hit (debe resolver desde RAM)
res2 = util.cache.query_setpoints_cached_ui(ttl_seconds=5)
print "\n2. Segunda Llamada Inmediata:"
print " - Origen:     ", res2["source"]
print " - Tiempo:     ", res2["elapsed_ms"], "ms"
print " - Hits en RAM:", res2["hits"]

# 3. Formateo para Perspective Table
rows = util.cache.format_setpoints_table(res2["data"])
print "\n3. Filas formateadas para Perspective Table:", len(rows)
```

**Paso 5: Salida Esperada en Consola**

```
=== VALIDACION LABORATORIO 8.2: GESTOR DE CACHE CON TTL ===

1. Primera Llamada:
 - Origen:      BASE DE DATOS SQL (CACHE MISS)
 - Tiempo:      22.45 ms
 - Hits en RAM: 0

2. Segunda Llamada Inmediata:
 - Origen:      MEMORIA RAM (CACHE HIT)
 - Tiempo:      0.038 ms
 - Hits en RAM: 1

3. Filas formateadas para Perspective Table: 4
```

***

**Paso 6: Arquitectura y Creación del Monitor de Caché en Perspective (`Lab8_2_CacheMonitor`)**

El monitor en Perspective permite visualizar en tiempo real la drástica diferencia entre el acceso a disco SQL y la lectura en memoria RAM. Todos los componentes de Perspective se mantienen en **1 o 2 líneas**, delegando en `util.cache`.

```mermaid
flowchart TD
    subgraph Vista_Perspective [Perspective: Views/Laboratorios/Lab8_2_CacheMonitor]
        direction TB
        subgraph UI_Controls [Controles: 1-2 Líneas]
            NumTTL[num_TTL: Tiempo de Vida en Segundos]
            BtnFetch[btn_Consultar: onActionPerformed -> 2 Líneas]
            BtnClear[btn_Invalidar: onActionPerformed -> 2 Líneas]
        end
        subgraph UI_KPIs [Banner KPI de Rendimiento]
            CardSource[kpi_Origen: Origen RAM vs SQL]
            CardLatency[kpi_Latencia: Tiempo de Respuesta en ms]
            CardHits[kpi_Hits: Contador de Cache Hits]
        end
        subgraph UI_Tables [Tablas de Datos: 1 Línea]
            TableCatalogo[table_Catalogo: props.data Transform -> 1 Línea]
            TableStats[table_Stats: props.data Transform -> 1 Línea]
        end
    end

    subgraph Gateway_Events [Gateway Event Scripts: Automatización Desatendida]
        TimerWarmup[Gateway Timer Script: warmup_catalog_cache cada 60s]
        TagTrigger[Gateway Tag Change Script: [default]Config/ReloadCatalogs]
    end

    subgraph Backend_Library [Project Library: util.cache]
        BtnFetch -->|query_setpoints_cached_ui ttl| CoreCache[query_setpoints_cached_ui / get_or_set]
        BtnClear -->|clear_cache key| ClearLib[clear_cache]
        TimerWarmup -->|Pre-calentamiento periódico en background| CoreCache
        TagTrigger -->|Invalidación reactiva por flanco de tag| ClearLib
        CoreCache -->|Cache Miss| DBPostgres[(machine_setpoints en SANDBOX_DB)]
        CoreCache -->|Cache Hit| JVMRAM[Memoria RAM JVM: _CACHE_STORE]
        TableCatalogo -->|format_setpoints_table value| FormatCat[format_setpoints_table]
        TableStats -->|format_cache_stats_table stats| FormatStats[format_cache_stats_table]
    end
```

1. En el **Project Browser**, navegar a **Perspective -> Views**.
2. Crear la vista `Laboratorios/Lab8_2_CacheMonitor` con `Root Container Type: Flex Container` y `props.direction: column`.
3. Crear las propiedades personalizadas en la vista (**custom**):

| Propiedad Custom | Tipo | Valor Inicial | Descripción |
| :--- | :--- | :--- | :--- |
| `custom.ttlSeconds` | `number` | `10` | Tiempo de vida del dato en memoria RAM |
| `custom.lastResult` | `object` | `{}` | Almacén reactivo de la última consulta de caché |
| `custom.statusMessage` | `string` | `"Listo para consultar"` | Mensaje informativo para el operador |

**Paso 7: Vinculación del Banner KPI de Caché**

1. Añadir un **Flex Container** horizontal (`banner_kpi`) con estilo: `display: flex`, `justifyContent: space-around`, `gap: 16px`, `margin: 12px 0`.
2. Añadir 3 tarjetas KPI dentro del contenedor:

   * **Tarjeta 1: Origen del Dato (`kpi_Origen`):**
     * Label principal con binding a `view.custom.lastResult.source`.
     * Texto descriptivo secundario: `"Origen del Dato (RAM vs SQL)"`.

   * **Tarjeta 2: Latencia de Respuesta (`kpi_Latencia`):**
     * Label con binding a `view.custom.lastResult.elapsed_ms` formateado (ej. `0.035 ms` vs `22.40 ms`).
     * Texto descriptivo secundario: `"Tiempo de Acceso al Dato"`.

   * **Tarjeta 3: Hits en Memoria RAM (`kpi_Hits`):**
     * Label con binding a `view.custom.lastResult.hits`.
     * Texto descriptivo secundario: `"Peticiones servidas sin tocar la BD"`.

**Paso 8: Configuración de la Barra de Controles y Tablas de Perspective**

1. Añadir una barra horizontal de herramientas (`toolbar_cache`):
   * **Numeric Entry Field** (`num_TTL`):
     * `props.value`: `10`
     * Label: `"TTL (Segundos)"`.
   * **Button** (`btn_Consultar`):
     * `props.text`: `"CONSULTAR CATÁLOGO (get_or_set)"`.
     * `style.backgroundColor`: `"#27ae60"`, `style.color`: `"#ffffff"`.
     * En `events -> onActionPerformed`, añadir la acción **Script (2 líneas exactas)**:

```python
# Evento onActionPerformed en btn_Consultar (2 lineas)
ttl = self.getSibling("num_TTL").props.value
self.view.custom.lastResult = util.cache.query_setpoints_cached_ui(ttl_seconds=ttl)
```

   * **Button** (`btn_Invalidar`):
     * `props.text`: `"INVALIDAR CACHÉ (clear_cache)"`.
     * `style.backgroundColor`: `"#c0392b"`, `style.color`: `"#ffffff"`.
     * En `events -> onActionPerformed`, añadir la acción **Script (2 líneas exactas)**:

```python
# Evento onActionPerformed en btn_Invalidar (2 lineas)
util.cache.clear_cache("cat_setpoints")
self.view.custom.statusMessage = "Caché invalidada manualmente en la memoria RAM del Gateway"
```

2. Añadir el componente **Table** (`table_Catalogo`) para visualizar las consignas:
   * En `props.data`, binding vinculado a `view.custom.lastResult.data`.
   * Añadir un **Script Transform de 1 línea**:

```python
# Script Transform en props.data de table_Catalogo (1 linea)
return util.cache.format_setpoints_table(value)
```

3. Añadir una sub-tabla para monitorizar el estado de la memoria (`table_Stats`):
   * En `props.data`, binding a `view.custom.lastResult` con un **Script Transform de 1 línea**:

```python
# Script Transform en props.data de table_Stats (1 linea)
return util.cache.format_cache_stats_table(util.cache.get_cache_stats())
```

**Paso 9: Verificación Interactiva en Perspective Workstation / Navegador**

1. Lanzar la vista en el navegador o activar el Modo Preview (`Ctrl+P`).
2. **Pulsación 1 (Cache Miss):**
   * Pulsar **CONSULTAR CATÁLOGO**.
   * Comprobar que la tarjeta de origen indica `BASE DE DATOS SQL (CACHE MISS)` y la latencia ronda los `20 ms`.
3. **Pulsaciones 2, 3 y 4 (Cache Hits sucesivos):**
   * Pulsar repetidamente **CONSULTAR CATÁLOGO** antes de que transcurran 10 segundos.
   * Observar cómo el indicador reporta instantáneamente `MEMORIA RAM (CACHE HIT)`, mientras la latencia cae a cifras de entre `0.02 ms` y `0.04 ms` (una aceleración de más de 500x) y el contador de hits incrementa en cada clic.
4. **Invalidación y expiración de TTL:**
   * Pulsar **INVALIDAR CACHÉ** o esperar a que transcurran más de 10 segundos.
   * Volver a pulsar **CONSULTAR CATÁLOGO** y verificar cómo el sistema detecta la ausencia del dato en RAM y realiza limpiamente una nueva consulta a PostgreSQL.

***

**Paso 10: Integración en Gateway Event Scripts (Timer Script & Tag Change Script)**

Para alcanzar una arquitectura de nivel de producción en plantas industriales, la gestión de la memoria no debe delegarse únicamente a las acciones de los usuarios en Perspective. El Gateway debe gobernar la caché de manera proactiva y reactiva:

#### 10.1 Patrón A: Gateway Timer Script (Pre-calentamiento y Refresco Periódico en Background)

* **Objetivo industrial (*Cache Warm-Up*):** Evitar que los operadores sufran la latencia de un *Cache Miss* (~20-50 ms) tras expirar el TTL. Un temporizador en el Gateway ejecuta la consulta en segundo plano, manteniendo los datos permanentemente calientes en la memoria RAM para que cualquier cliente web o sesión de Perspective experimente un *Cache Hit* instantáneo (< 0.05 ms).
* **Configuración en Ignition Designer:**
  1. En el **Project Browser**, navegar a **Server Scripting -> Gateway Event Scripts**.
  2. Seleccionar la categoría **Timer**.
  3. Pulsar en el icono de suma (**+**) para crear un nuevo Timer Script:
     * **Name:** `Timer_Warmup_Setpoints`
     * **Delay (ms):** `60000` (se ejecuta cada 60 segundos)
     * **Initial Delay (ms):** `5000` (espera 5 segundos tras el arranque del Gateway)
     * **Threading:** Seleccionar **Dedicated** (*Regla de Oro en Gateway:* Las tareas que realizan consultas a base de datos deben utilizar un hilo dedicado para nunca bloquear el pool compartido *Shared*).
  4. En el área de código, invocar la función de pre-calentamiento en 1 línea:
     ```python
     # Gateway Event Scripts -> Timer Script: Timer_Warmup_Setpoints
     # Delay: 60,000 ms (60 s) | Threading: Dedicated
     util.cache.warmup_catalog_cache(ttl_seconds=120)
     ```
  > [!TIP]
  > **Técnica del Solapamiento de TTL:** Obsérvese que el temporizador se ejecuta cada `60 s`, pero el TTL se fija en `120 s`. Esta holgura asegura que la clave en memoria nunca expire; la recarga ocurre silenciosamente en segundo plano mientras el dato anterior sigue vigente, garantizando una tasa de acierto del 100% (*Zero Cache Miss*) para todos los puestos de operación.

#### 10.2 Patrón B: Gateway Tag Change Script (Invalidación Reactiva Guiada por Eventos)

* **Objetivo industrial (*Event-Driven Invalidation*):** Si un autómata PLC, un lote de producción, un sistema MES o una pantalla de administración actualiza las consignas en base de datos, no es seguro esperar a que caduque el temporizador o el TTL. Un tag disparador de control (`[default]Config/ReloadCatalogs`) o un tag de versión de receta (`[default]Config/RecipeVersion`) notifica al Gateway para purgar la caché de inmediato.
* **Fase 1: Creación del Memory Tag Disparador:**
  1. En el **Tag Browser**, crear una carpeta `Config` dentro del Tag Provider `[default]`.
  2. Crear un nuevo Memory Tag:
     * **Tag Name:** `ReloadCatalogs`
     * **Data Type:** `Boolean`
     * **Value:** `false`
* **Fase 2: Configuración del Gateway Tag Change Script:**
  1. En el **Project Browser**, ir a **Server Scripting -> Gateway Event Scripts**.
  2. Seleccionar la categoría **Tag Change**.
  3. Pulsar en **+** para crear un nuevo script:
     * **Name:** `TagChange_Invalidate_Setpoints`
     * **Tag Paths:** Añadir la ruta `[default]Config/ReloadCatalogs`
  4. En el editor de script, programar la detección de flanco ascendente y delegar la invalidación:
     ```python
     # Gateway Event Scripts -> Tag Change Script: TagChange_Invalidate_Setpoints
     # Tag Path suscrito: [default]Config/ReloadCatalogs
     if not initialChange and currentValue.value:
         # Invalida la cache de consignas y resetea automaticamente el tag disparador a False
         util.cache.invalidate_on_tag_trigger(event.tagPath.toString(), reset_flag=True)
     ```

#### 10.3 Verificación de la Automatización en Tiempo Real

1. Abrir la vista `Lab8_2_CacheMonitor` en Perspective en modo Preview (`Ctrl+P`).
2. Pulsar **CONSULTAR CATÁLOGO**: comprobar que se encuentra en `CACHE HIT (MEMORIA RAM)` gracias al temporizador en background.
3. En el Designer, abrir el **Tag Browser** y modificar el valor de `[default]Config/ReloadCatalogs` a `true`:
   * Verificar que en milisegundos el tag regresa automáticamente a `false` (reseteo completado por `writeBlocking`).
4. Consultar los registros del Gateway (**Gateway Web -> Status -> Diagnostics -> Logs**):
   ```
   INFO  [SCADA.Cache.TagChange] Cache 'cat_setpoints' invalidada reactivamente desde tag: [default]Config/ReloadCatalogs
   ```
5. Regresar a la vista de Perspective y pulsar **CONSULTAR CATÁLOGO**:
   * La tarjeta conmuta inmediatamente a `BASE DE DATOS SQL (CACHE MISS)` con latencia de ~20 ms, demostrando que la caché fue purgada en el instante exacto del pulso del tag.
   * La pulsación siguiente vuelve a responder en < 0.05 ms como `CACHE HIT`, restableciendo la alta velocidad.

***

### 6. Test de Conceptos de la Sesión 8

#### Objetivos

* Validar la asimilación conceptual sobre SARGability y el impacto de funciones en cláusulas `WHERE`.
* Evaluar el entendimiento sobre el ciclo de vida de la memoria en la JVM y el coste de las conversiones repetidas de datasets.
* Comprobar el criterio de aplicación del patrón de caché con TTL en entornos industriales.

#### Contenidos

* Cuestionario técnico individual de opción múltiple y resolución de casos de optimización de código y bases de datos.
* [Cuestionario Sesión 8](https://docs.google.com/forms/d/e/1FAIpQLSeHDWwYuztymYsiqO5pwm-2v98rpIDJKeYyxA0LwKg8Eccltw/viewform?usp=header)
* [Feedback Sesión 8](https://imagina-formacion.typeform.com/to/LQzhO2AR)


#### Resultado esperado

* Comprobación del dominio de las técnicas de optimización de consultas SQL y de scripts Jython en el Gateway.

***

### 7. Feedback Individual y Cierre de la Sesión

#### Objetivos

* Revisar los resultados de los benchmarks de consultas SQL ejecutados por los alumnos en la Script Console.
* Comprobar la correcta implementación del módulo de caché y el manejo de tiempos de expiración.
* Presentar la planificación de la Sesión 9 (sesión intensiva de 5 horas).

#### Contenidos

* Rondas de revisión individual del código de optimización y de los módulos de caché.
* Entrega de la _Checklist de Rendimiento_ para auditoría de scripts previos al despliegue.
* Avance de la Sesión 9: Procesamiento asíncrono (`system.util.invokeAsynchronous`), seguridad avanzada y RBAC, integración con APIs REST (`system.net.httpClient`), microservicios FastAPI y metodología de refactorización de código heredado (_Legacy_).

#### Resultado esperado

* Cada alumno finaliza la sesión con sus módulos de caché operativos, benchmarks completados y comprensión de los criterios de rendimiento que se aplicarán en arquitecturas asíncronas y microservicios.
