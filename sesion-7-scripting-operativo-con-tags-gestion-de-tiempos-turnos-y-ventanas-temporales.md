# Sesión 7: Scripting Operativo con Tags, Gestión de Tiempos, Turnos y Ventanas Temporales

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Scripting Operativo con Tags y Control de Calidad (Tema 16)

* **Conceptos clave a tratar:**
  * Tipología de tags en Ignition: tags de proceso (OPC-UA), tags de memoria, tags de configuración, tags calculados/expresión y propiedades de interfaz.
  * Interacción eficiente mediante el API `system.tag.*`: uso correcto de `system.tag.readBlocking` y `system.tag.writeBlocking`.
  * **Optimización por lectura agrupada (**_**Batch Reads**_**):** eliminación del antipatrón de leer tags en bucle individual y sustitución por peticiones en bloque mediante listas de rutas.
  * Anatomía del objeto `QualifiedValue`: inspección obligatoria de valor (`.value`), código de calidad (`.quality.isGood()`) y marca de tiempo de origen (`.timestamp`).
  * Responsabilidad en la escritura de planta: control de consignas y diferenciación estricta entre la lógica de interfaz SCADA y la lógica crítica determinista del PLC.
  * Sincronización bidireccional Tag $\longleftrightarrow$ Base de Datos: volcado de instantáneas de proceso y descarga de parámetros de receta a controladores.
  * Auditoría y documentación técnica de scripts que interactúan con tags para soporte y mantenimiento de planta.
* **Objetivos:**
  * Aprender a interactuar programáticamente con el motor de tags de forma eficiente y no bloqueante.
  * Dominar el tratamiento defensivo de la calidad de señal y la sincronización segura con bases de datos.
* **Resultado esperado:**
  * El alumno diseña lecturas y escrituras de tags agrupadas por lotes, verificando la calidad del dato antes de ejecutar cálculos u operaciones de control.

***

#### 📘 Bloque Teórico 2: Gestión Temporal, Calendario Industrial y Turnos Nocturnos (Tema 17)

* **Conceptos clave a tratar:**
  * Representación y convivencia de tipos temporales: objetos `java.util.Date`, cadenas de texto, marcas SQL (`TIMESTAMP`) y API `system.date.*`.
  * Riesgos de la comparación alfabética de fechas como texto frente a la comparación cronológica real.
  * Manipulación temporal nativa con `system.date.*`: suma/resta de intervalos, cálculo de diferencias (`diff`), parseo y formateo localizado.
  * Modelado del calendario industrial y **gestión de turnos nocturnos:** algoritmos para tratar turnos que cruzan la medianoche y días operativos desfasados del calendario natural.
  * Tratamiento de husos horarios (_Timezones_) y cambios estacionales de horario de verano/invierno (DST).
  * Algoritmos de cálculo de duraciones: tiempos de paro acumulados, tiempos de ciclo entre eventos y tiempos medios entre fallos (_MTBF_ simplificado).
  * Optimización de consultas SQL por rango de fechas: filtrado sobre columnas indexadas mediante parámetros tipados sin romper la SARGability.
* **Objetivos:**
  * Dominar la aritmética de fechas y el cálculo de ventanas temporales industriales en Jython.
  * Aprender a estructurar consultas SQL históricas eficientes sobre columnas temporales indexadas.
* **Resultado esperado:**
  * El alumno implementa algoritmos precisos para determinar turnos operativos (incluyendo nocturnos) y calcula métricas de duración sin errores de zona horaria ni formato.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 7.1: Lectura Agrupada en Bloque (Batch) y Validación de Calidad (Tema 16)

* **Objetivos:**
  * Implementar una función modular en Project Library que lea un conjunto de 10 tags en una única llamada `system.tag.readBlocking`.
  * Validar programáticamente la calidad de cada señal recibida, reemplazando lecturas en estado `Bad` o `Uncertain` por valores seguros de reserva (_fallback_).
* **Resultado esperado:**
  * Un script probado en consola que devuelve un diccionario tipado de variables de línea, listo para alimentar interfaces sin riesgo de fallos por tags desconectados.

***

#### 🧪 Laboratorio 7.2: Sincronización Bidireccional Tag ↔ SQL con Auditoría (Tema 16)

* **Objetivos:**
  * Construir un procedimiento que descargue parámetros de receta desde una Named Query hacia tags de proceso del PLC mediante `system.tag.writeBlocking`.
  * Registrar automáticamente una traza en la base de datos confirmando el éxito o fallo de la escritura y el usuario responsable.
* **Resultado esperado:**
  * Un flujo de carga de recetas bidireccional y trazable que verifica la respuesta del PLC (`QualityCode`) antes de confirmar la operación al operador.

***

#### 🧪 Laboratorio 7.3: Motor de Cálculo de Turnos y Día Operativo Nocturno (Tema 17)

* **Objetivos:**
  * Diseñar una función en Project Library que reciba una marca de tiempo y determine el identificador de turno y los objetos `Date` de inicio y fin exactos.
  * Gestionar correctamente el turno de noche que cruza la medianoche (22:00h a 06:00h) asociándolo a la fecha operativa correspondiente.
* **Resultado esperado:**
  * Una función testeada en consola que asigna sin errores el turno y la ventana de consulta para cualquier hora del día o de la noche.

***

#### 🧪 Laboratorio 7.4: Consultas Temporales Indexadas y Cálculo de Tiempos de Paro (Tema 17)

* **Objetivos:**
  * Ejecutar una consulta SQL parametrizada por rango de fechas (`system.db.runPrepQuery`) para extraer el historial de paradas de una máquina en un turno.
  * Calcular la duración individual y acumulada de cada parada en minutos utilizando `system.date.minutesBetween()`, formateando los resultados para su presentación en tabla.
* **Resultado esperado:**
  * Un reporte estructurado en consola con el tiempo total de indisponibilidad y la lista de eventos temporales listos para vincular a un componente visual.
