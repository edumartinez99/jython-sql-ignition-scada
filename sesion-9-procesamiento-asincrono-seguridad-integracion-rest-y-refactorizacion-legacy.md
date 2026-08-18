# Sesión 9: Procesamiento Asíncrono, Seguridad, Integración REST y Refactorización Legacy

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Procesamiento Asíncrono y Automatización Desatendida en Gateway (Tema 20)

* **Conceptos clave a tratar:**
  * Mecanismo de hilos de la JVM y prevención de bloqueos de interfaz (_UI Freezing_) provocados por scripts lentos.
  * Criterios de delegación: ejecución asíncrona en cliente (`system.util.invokeAsynchronous`) vs ejecución centralizada en Gateway.
  * Configuración de tareas desatendidas: **Gateway Scheduled Scripts** (expresiones cron) y **Gateway Timer Scripts** para consolidaciones periódicas y mantenimiento de datos.
  * **Principio de Idempotencia:** diseño de scripts que puedan ejecutarse múltiples veces de forma accidental sin duplicar registros ni corromper estados de proceso.
  * Control de concurrencia y mecanismos de exclusión mutua para evitar que múltiples instancias compitan por las mismas tablas o registros.
  * Supervisión de tareas sin operador (_Headless Execution_): captura obligatoria de metadatos (hora de inicio, fin, duración en milisegundos, filas procesadas y estado final).
  * Coordinación de tareas automáticas con el calendario de planta (cambios de turno, paradas técnicas y cierres de lote).
* **Objetivos:**
  * Aprender a programar procesos en segundo plano de forma no bloqueante.
  * Dominar la automatización desatendida en el servidor bajo criterios estrictos de idempotencia y trazabilidad.
* **Resultado esperado:**
  * El alumno diseña tareas programadas en el Gateway que procesan datos de forma segura e independiente de los clientes abiertos.

***

#### 📘 Bloque Teórico 2: Seguridad, Control de Acceso (RBAC) y Prevención de Fugas (Tema 21)

* **Conceptos clave a tratar:**
  * Mitigación de inyección de código mediante el uso exclusivo de Named Queries y Prepared Statements.
  * **Control de Acceso Basado en Roles (**_**RBAC**_**):** validación programática de permisos antes de ejecutar escrituras críticas, cambios de consigna, transiciones de estado o anulaciones.
  * Segregación multidimensional de seguridad: permisos de interfaz (visibilidad), permisos de script (ejecución), permisos de base de datos y permisos de control de planta.
  * Higiene estricta de credenciales: prohibición de almacenar claves, contraseñas o tokens en texto plano dentro de scripts; prevención de fugas de datos sensibles en logs.
  * **Sanitización de mensajes de error:** técnicas para registrar el detalle técnico completo en los logs internos del Gateway mientras se presenta al operador un mensaje funcional seguro sin información de infraestructura.
  * Revisión y neutralización de scripts heredados con operaciones peligrosas sin validar.
  * Elaboración del decálogo de normas internas de scripting seguro en Ignition.
* **Objetivos:**
  * Implementar controles de seguridad y validación de roles en la capa de scripting.
  * Proteger el sistema contra fugas de información y accesos no autorizados a operaciones críticas.
* **Resultado esperado:**
  * El alumno blindará funciones de persistencia con validación de roles, sanitización de trazas y protección contra inyecciones.

***

#### 📘 Bloque Teórico 3: Integración REST, Microservicios Externos y JSON (Tema 22)

* **Conceptos clave a tratar:**
  * Comunicaciones HTTP avanzadas mediante el cliente nativo `system.net.httpClient` (métodos `GET`, `POST`, `PUT`, `DELETE`).
  * Configuración de cabeceras (_Headers_), autenticación (Bearer Tokens, Basic Auth) y **definición estricta de timeouts** para evitar llamadas colgadas.
  * **Fronteras arquitectónicas:** cuándo resolver una lógica dentro de Ignition y cuándo delegarla a microservicios externos en Python moderno (**FastAPI**, Flask) para tareas de analítica avanzada o machine learning.
  * Límites del SCADA: evitar transformar el Gateway en un backend web de propósito general.
  * Serialización y deserialización de estructuras JSON mediante `system.util.jsonDecode` y `system.util.jsonEncode`.
  * Patrones de resiliencia: valores por defecto (_fallback_), reintentos controlados y estados de degradación en pantalla ante caídas de la API externa.
* **Objetivos:**
  * Conectar Ignition con servicios web REST y microservicios externos de forma robusta y asíncrona.
  * Dominar el intercambio y transformación de datos en formato JSON.
* **Resultado esperado:**
  * El alumno consume servicios REST externos utilizando `system.net.httpClient`, gestionando códigos de estado HTTP, timeouts y respuestas JSON sin congelar la interfaz.

***

#### 📘 Bloque Teórico 4: Metodología de Refactorización de Scripts Heredados (Tema 23)

* **Conceptos clave a tratar:**
  * Auditoría e inspección técnica de código existente: identificación de propósito, dependencias ocultas, variables crípticas y riesgos de operación.
  * Detección de "malos olores" (_Code Smells_): scripts duplicados en botones, consultas SQL concatenadas, bucles ineficientes y capturas silenciosas (`except: pass`).
  * **Estrategia de desacoplamiento en 4 capas:** Validación $\rightarrow$ Acceso a Datos $\rightarrow$ Lógica/Cálculo $\rightarrow$ Presentación.
  * Extracción de lógica monolítica hacia módulos limpios en la **Project Library**.
  * Sustitución de sentencias inseguras por Named Queries y adición de logging contextual.
  * Protocolo de no regresión: diseño de pruebas de verificación antes y después de refactorizar para asegurar que el resultado funcional se mantiene idéntico.
  * Planificación de mejora continua progresiva evitando reescrituras globales descontroladas (_Big Bang rewrites_).
* **Objetivos:**
  * Aprender a auditar, limpiar y modularizar código heredado (_Legacy_) en proyectos SCADA en producción.
  * Aplicar técnicas de refactorización segura garantizando la no regresión operativa.
* **Resultado esperado:**
  * El alumno toma scripts antiguos, desordenados e inseguros y los transforma en código modular, testeable y alineado con los estándares de la Project Library.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 9.1: Automatización Desatendida Idempotente en Gateway (Tema 20)

* **Objetivos:**
  * Diseñar un _Gateway Scheduled Script_ que consolide los registros de producción del turno recién cerrado de forma automática.
  * Implementar una validación de idempotencia en base de datos para asegurar que múltiples ejecuciones no dupliquen datos, registrando las métricas de ejecución en la tabla de tareas de fondo.
* **Resultado esperado:**
  * Un proceso en servidor que se ejecuta de forma desatendida, genera el consolidado histórico sin duplicados y deja una pista de auditoría con la duración exacta en milisegundos.

***

#### 🧪 Laboratorio 9.2: Blindaje de Seguridad, Validación de Roles y Sanitización de Errores (Tema 21)

* **Objetivos:**
  * Implementar una función de modificación de parámetros críticos que verifique programáticamente los privilegios y roles del usuario en sesión (`system.security.getUserRoles`).
  * Configurar una captura de excepciones que registre la traza técnica detallada en los logs del Gateway y retorne al operador un mensaje amigable y seguro con un código de referencia.
* **Resultado esperado:**
  * Un script de confirmación probado en consola que deniega accesos no autorizados y gestiona fallos de persistencia sin filtrar detalles internos de la base de datos a la interfaz.

***

#### 🧪 Laboratorio 9.3: Integración con Microservicio REST (FastAPI) mediante `httpClient` (Tema 22)

* **Objetivos:**
  * Instanciar `system.net.httpClient` para enviar una petición `POST` con un payload JSON conteniendo variables de proceso hacia una API externa.
  * Decodificar la respuesta JSON retornada (`system.util.jsonDecode`), aplicar un timeout estricto de 3 segundos y definir un valor de reserva (_fallback_) en caso de indisponibilidad del servicio.
* **Resultado esperado:**
  * Un cliente HTTP funcional en Jython que procesa datos externos de forma segura, resistente a caídas de red y preparado para vincularse a la interfaz.

***

#### 🧪 Laboratorio 9.4: Refactorización Integral de un Script Legacy Monolítico (Tema 23)

* **Objetivo:**
  * Analizar un script antiguo de 40 líneas incrustado en un botón con SQL concatenado, variables crípticas y fallos silenciosos.
  * Desacoplar el script en capas: extraer el SQL a una Named Query parametrizada, trasladar el cálculo a una función pura en `Project Library` y dejar el botón con una llamada limpia de 3 líneas.
* **Resultado esperado:**
  * Un módulo refactorizado en la Project Library que produce exactamente los mismos resultados numéricos que el script original pero con código modular, tipado, documentado y libre de riesgos de inyección.
