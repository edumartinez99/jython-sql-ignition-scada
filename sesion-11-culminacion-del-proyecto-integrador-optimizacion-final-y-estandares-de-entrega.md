# Sesión 11: Culminación del Proyecto Integrador, Optimización Final y Estándares de Entrega

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Orquestación Integral de Capas y Script Transforms Ligeros (Tema 26 - Culminación)

* **Conceptos clave a tratar:**
  * Consolidación de la arquitectura global en 4 capas desacopladas: $$\text{Planta (Tags OPC)} \longrightarrow \text{Persistencia (Named Queries / SQL)} \longrightarrow \text{Lógica Pura (Project Library)} \longrightarrow \text{Presentación (Perspective / UI)}$$
  * Implementación de **Script Transforms ligeros**: transformación de propiedades de interfaz delegando el 100% de la lógica pesada en la Project Library.
  * Prevención estricta de bloqueos en la interfaz: eliminación definitiva de consultas SQL o llamadas de E/S dentro de los transforms de enlace (_bindings_).
  * Manejo de estados de degradación en interfaz: visualización elegante (_fallbacks_, estilos neutros) ante desconexiones de base de datos o tags en calidad `Bad`.
  * Comunicación reactiva entre vistas desacopladas mediante **Message Handlers** (`system.perspective.sendMessage`) para actualizar tablas y gráficos sin recargar la pantalla completa.
* **Objetivos:**
  * Aprender a orquestar el flujo completo de datos desde la base de datos hasta la interfaz gráfica de forma eficiente y no bloqueante.
  * Dominar la construcción de bindings y transforms ultra-ligeros conectados a librerías centralizadas.
* **Resultado esperado:**
  * El alumno conecta las pantallas de supervisión con la base de datos y tags mediante transforms concisos que mantienen la interfaz fluida y desacoplada.

***

#### 📘 Bloque Teórico 2: Acciones Controladas, Automatización en Servidor y Auditoría Final (Tema 26 - Cierre)

* **Conceptos clave a tratar:**
  * Diseño de scripts de eventos de componente (_Button Events_) con **triple barrera de protección**:
    1. _Validación de roles y permisos (RBAC)._
    2. _Validación de precondiciones de proceso y rangos de ingeniería._
    3. _Escritura transaccional protegida (`system.db.beginTransaction` / `commitTransaction`) con registro automático en auditoría._
  * Sincronización bidireccional segura: actualización de la base de datos relacional y escritura simultánea a los tags del PLC (`system.tag.writeBlocking`).
  * Automatización desatendida en el Gateway: activación de **Scheduled Scripts** para consolidación horaria de KPIs con garantía de idempotencia.
  * **Auditoría final de rendimiento:** verificación de latencias de Named Queries (<50 ms), eliminación del problema N+1 y medición de tiempos de procesamiento en la JVM.
  * Elaboración del documento de entrega: catálogo de Named Queries, mapa de módulos en Project Library y guía de mantenimiento para el equipo técnico.
* **Objetivos:**
  * Integrar acciones de usuario transaccionales y procesos automáticos desatendidos en el servidor.
  * Aplicar la checklist integral de calidad previa al pase a producción y consolidar el estándar corporativo.
* **Resultado esperado:**
  * El alumno entrega un proyecto SCADA completo, seguro, transaccional y de alto rendimiento, documentado bajo los estándares corporativos aprendidos en el curso.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 11.1: Integración de Vistas y Script Transforms Ligeros en Perspective (Tema 26)

* **Objetivos:**
  * Vincular la tabla principal del dashboard a la Named Query de resumen de línea y aplicar un _Script Transform_ ligero que consuma la función `proyecto.produccion.procesar_dataset_resumen`.
  * Configurar estilos visuales condicionales (colores de estado, badges e iconos) gestionando respuestas vacías sin lanzar excepciones en pantalla.
* **Resultado esperado:**
  * Un dashboard operativo en tiempo real que renderiza los estados de línea y métricas de producción de forma fluida y tolerante a fallos.

***

#### 🧪 Laboratorio 11.2: Event Scripts de Acción Controlada con RBAC y Transaccionalidad (Tema 26)

* **Objetivos:**
  * Programar el evento de confirmación de "Cierre de Orden y Declaración de Lote" desde la interfaz con validación de roles de supervisor.
  * Ejecutar la transacción atómica que actualiza el estado de la orden en SQL, descuenta materias primas, registra la pista de auditoría inmutable y actualiza los tags del PLC.
* **Resultado esperado:**
  * Un flujo de acción de usuario protegido que garantiza consistencia total en base de datos y emite un mensaje desacoplado (_Message Handler_) para refrescar el resto de vistas del SCADA.

***

#### 🧪 Laboratorio 11.3: Automatización de Consolidación Desatendida en Gateway (Tema 26)

* **Objetivos:**
  * Configurar un _Gateway Scheduled Script_ que ejecute la consolidación automática de KPIs al cierre de cada turno de forma desatendida.
  * Validar la idempotencia del proceso frente a reintentos y verificar el registro de métricas de ejecución (duración en ms, estado y registros afectados) en la tabla de control.
* **Resultado esperado:**
  * Una tarea de fondo operativa en el Gateway que consolida el histórico de producción sin intervención de operarios y sin riesgo de duplicar datos.

***

#### 🧪 Laboratorio 11.4: Auditoría de Rendimiento, Checklist de Calidad y Entrega del Estándar (Tema 26)

* **Objetivos:**
  * Medir cuantitativamente los tiempos de respuesta de todas las consultas y transformaciones del proyecto final.
  * Auditar la solución completa contra la _Checklist Corporativa de Calidad_ (cero SQL concatenado, cero I/O en transforms, manejo de nulos, docstrings y RBAC).
* **Resultado esperado:**
  * Un proyecto final validado con métricas de alto rendimiento y una propuesta formal de estándar interno lista para ser aplicada en futuros desarrollos de scripting en Ignition.
