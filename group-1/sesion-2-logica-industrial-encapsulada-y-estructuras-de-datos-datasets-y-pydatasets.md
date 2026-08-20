# Sesión 2: Lógica Industrial Encapsulada y Estructuras de Datos (Datasets y PyDataSets)

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Encapsulación Funcional y Casos SCADA Avanzados (Tema 4 - Parte 2)

* **Conceptos clave a tratar:**
  * Modularización de cálculos repetitivos mediante funciones propias con firmas y retornos consistentes.
  * Tratamiento defensivo exhaustivo: gestión de `None`, strings vacíos, listas sin datos y respuestas nulas de tags o consultas.
  * Conversión segura de tipos entre señales OPC, propiedades de interfaz (Perspective/Vision), formularios y base de datos.
  * Documentación efectiva basada en intención de negocio y restricciones operativas en lugar de sintaxis elemental.
  * Aplicaciones industriales típicas: totalización de producción, validación de tolerancias, clasificación de estados de máquina, cálculo de disponibilidad y generación de mensajes dinámicos de operador.
* **Objetivos:**
  * Aprender a encapsular lógica de proceso en funciones puras y reutilizables.
  * Dominar el diseño de algoritmos de planta con tolerancia garantizada a fallos de señal y entradas nulas.
* **Resultado esperado:**
  * El alumno diseña funciones industriales modulares que validan rangos de ingeniería y retornan estados y métricas formateadas para operación.

***

#### 📘 Bloque Teórico 2: Estructuras de Datos, Datasets, PyDataSets e Inmutabilidad (Tema 5)

* **Conceptos clave a tratar:**
  * Diferenciación de tipos de colecciones: listas, tuplas, diccionarios, `BasicDataset` (Java) y `PyDataSet` (Jython).
  * Mecánica de lectura e iteración sobre `PyDataSet`: acceso por índice de columna y por nombre de columna.
  * **Principio de Inmutabilidad de Datasets:** comprensión de que funciones como `system.dataset.addRow` devuelven un nuevo dataset y no alteran el existente en memoria.
  * Transformación bidireccional entre estructuras: Datasets $\longleftrightarrow$ Listas de diccionarios para componentes de tabla y vistas.
  * Creación de estructuras intermedias para agrupación de datos por máquina, línea, turno, lote o estado.
  * Control defensivo en tablas: gestión de columnas ausentes, tipos inesperados y datasets vacíos (`getRowCount() == 0`).
  * Optimización y arquitectura: coste de conversiones repetidas en bindings de alta frecuencia y criterios técnicos para decidir si un cálculo se realiza en SQL o en Jython.
* **Objetivos:**
  * Comprender a fondo el funcionamiento interno y la inmutabilidad de los datasets nativos de Ignition.
  * Dominar la manipulación, agregación y conversión eficiente de datos tabulares en memoria.
  * Establecer criterios claros de rendimiento para delegar procesamiento entre el motor SQL y Jython.
* **Resultado esperado:**
  * El alumno manipula colecciones y datasets nativos sin cometer errores de inmutabilidad, sabe agrupar registros por criterios industriales y evita cuellos de botella en la JVM.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 2.1: Encapsulación de Lógica SCADA y Mensajería de Operador (Tema 4)

* **Objetivos:**
  * Crear funciones modulares que calculen disponibilidad de máquina y evalúen desviaciones de consigna con tolerancia a nulos.
  * Generar respuestas estructuradas que incluyan códigos de estado, colores identificativos y mensajes de aviso para el operario.
* **Resultado esperado:**
  * Un conjunto de funciones de cálculo industrial probadas en consola que gestionan entradas corruptas (`None`, strings en campos numéricos) y devuelven valores seguros de reserva (_fallbacks_).

***

#### 🧪 Laboratorio 2.2: Manipulación de Datasets vs PyDataSets e Inmutabilidad (Tema 5)

* **Objetivos:**
  * Experimentar la diferencia práctica de lectura entre `Dataset` nativo de Java y `PyDataSet`.
  * Comprobar el principio de inmutabilidad utilizando funciones del módulo `system.dataset.*` y reasignación de variables.
* **Resultado esperado:**
  * Un script funcional que valida si un dataset contiene filas, lo convierte a `PyDataSet` para lectura por nombre de columna y genera un nuevo dataset modificado mediante `system.dataset.addRow()`.

***

#### 🧪 Laboratorio 2.3: Conversión Bidireccional y Agrupación por Máquina/Turno (Tema 5)

* **Objetivos:**
  * Transformar un dataset crudo de producción en una lista de diccionarios para procesar en memoria.
  * Construir un algoritmo de acumulación para agrupar piezas buenas y rechazos por máquina/turno, reconstruyendo una tabla final con `system.dataset.toDataSet()`.
* **Resultado esperado:**
  * Un dataset final consolidado y estructurado con métricas de calidad y totales por equipo, listo para asociar a una tabla de interfaz gráfica.

***

#### 🧪 Laboratorio 2.4: Benchmark de Rendimiento y Decisión SQL vs Jython (Tema 5)

* **Objetivos:**
  * Medir el tiempo de ejecución en milisegundos al procesar grandes volúmenes de datos en la JVM de Jython mediante bucles tradicionales.
  * Establecer el límite volumétrico a partir del cual el cálculo debe resolverse directamente en la base de datos (SQL).
* **Resultado esperado:**
  * Un informe comparativo de tiempos de procesamiento sobre miles de filas que fundamenta la regla de decisión arquitectónica entre procesamiento en base de datos vs en memoria SCADA.
