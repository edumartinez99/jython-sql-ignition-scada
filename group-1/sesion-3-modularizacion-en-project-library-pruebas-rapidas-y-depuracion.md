# Sesión 3: Modularización en Project Library, Pruebas Rápidas y Depuración

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Arquitectura Modular y Project Library (Tema 6)

* **Conceptos clave a tratar:**
  * Centralización de lógica de negocio en la **Project Library** para eliminar el código duplicado en botones, bindings y eventos.
  * Organización jerárquica de scripts por paquetes temáticos: `datos`, `validaciones`, `calculos`, `formato`, `queries`, `tags`, `alarmas`, `utilidades` y `logs`.
  * Definición estricta de firmas de función con parámetros explícitos, valores de retorno predecibles y control de excepciones.
  * Desacoplamiento de dependencias: evitar el uso de variables globales ocultas o referencias directas a componentes visuales específicos.
  * Estandarización de documentación técnica con _docstrings_ (propósito, tipado de parámetros, retornos y advertencias).
  * Separación de responsabilidades: diferenciación entre **funciones puras** (cálculo y transformación sin efectos secundarios) y **funciones con efectos de E/S** (lectura/escritura de tags, base de datos o red).
  * Gestión de impacto y control de cambios en librerías compartidas consumidas transversalmente por múltiples pantallas.
* **Objetivos:**
  * Aprender a diseñar y organizar un repositorio centralizado de scripts reutilizables en Ignition.
  * Dominar el desacoplamiento entre lógica pura de cálculo y operaciones de entrada/salida (_I/O_).
* **Resultado esperado:**
  * El alumno estructura paquetes limpios en la Project Library, documentados con docstrings y desacoplados de la interfaz visual.

***

#### 📘 Bloque Teórico 2: Script Console, Simulación y Metodología de Depuración (Tema 7)

* **Conceptos clave a tratar:**
  * Uso de la **Script Console** como banco de pruebas y prototipado rápido antes de integrar código en producción.
  * Inspección profunda de tipos, estructuras y contenidos reales en tiempo de ejecución (`type()`, `len()`, `dir()`).
  * Simulación de datos (_Mocking_): generación de datasets y diccionarios sintéticos para probar transformaciones complejas sin depender de PLCs o base de datos en vivo.
  * Diagnóstico y clasificación de fallos: sintaxis, excepciones en tiempo de ejecución (`TypeError`, `KeyError`, `IndexError`) y problemas de entorno/scope (permisos, desconexión).
  * Higiene en la depuración: pautas para utilizar mensajes temporales sin contaminar los logs del servidor ni dejarlos en producción.
  * Protocolo metódico de depuración en 6 pasos: **Reproducir $\rightarrow$ Aislar $\rightarrow$ Inspeccionar $\rightarrow$ Corregir $\rightarrow$ Verificar $\rightarrow$ Limpiar**.
  * Construcción de pequeños arneses de prueba (_Test Harnesses_) para verificar funciones críticas antes de su despliegue.
* **Objetivos:**
  * Utilizar la Script Console de forma profesional para testear y validar lógica con datos simulados.
  * Adoptar una metodología sistemática de depuración y diagnóstico de fallos en Ignition.
* **Resultado esperado:**
  * El alumno depura scripts de forma metódica, diagnostica la causa raíz de las excepciones en la JVM y valida funciones críticas mediante pruebas manuales controladas.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 3.1: Construcción de una Project Library Base Modular (Tema 6)

* **Objetivos:**
  * Crear una jerarquía de paquetes estandarizada en la Project Library del Designer (`app.formato`, `app.calculos`, `app.validaciones`).
  * Implementar y documentar con docstrings funciones de conversión de tiempo, estilos de estado y validación de rangos de proceso.
* **Resultado esperado:**
  * Un paquete de librerías modular y documentado en el Designer, listo para ser consumido de forma global por cualquier vista o script del proyecto.

***

#### 🧪 Laboratorio 3.2: Separación de Funciones Puras vs Funciones con Efectos de E/S (Tema 6)

* **Objetivos:**
  * Analizar un script monolítico acoplado a la interfaz y refactorizarlo separando la lógica de cálculo puro de la capa de comunicación y persistencia.
  * Aislar el cálculo de totales y ratios de turno en una función pura independiente de componentes gráficos o tags.
* **Resultado esperado:**
  * Una función pura testeable en consola y un manejador de evento de botón limpio de solo 3 líneas que orquesta la lectura, cálculo y escritura.

***

#### 🧪 Laboratorio 3.3: Mocking de Datos y Diagnóstico de Errores en Script Console (Tema 7)

* **Objetivo:**
  * Generar un dataset sintético con registros anómalos (nulos, tipos mixtos y valores extremos) para probar transformaciones en la Script Console.
  * Aplicar el protocolo metódico de depuración para localizar y corregir excepciones de tipo `KeyError` y `TypeError` provocadas en tiempo de ejecución.
* **Resultado esperado:**
  * Un procedimiento de prueba interactivo completado con éxito en consola, verificando la estabilidad del script ante anomalías de datos sin tocar planta real.

***

#### 🧪 Laboratorio 3.4: Test Harness Manual y Depuración Sistemática (Tema 7)

* **Objetivos:**
  * Diseñar un script de verificación automatizado (_Test Harness_) en la consola para evaluar funciones críticas de la Project Library con matrices de prueba (casos nominales, límites y entradas corruptas).
  * Validar la no regresión emitiendo un reporte estructurado de validación `[PASS]` / `[FAIL]`.
* **Resultado esperado:**
  * Un arnés de pruebas funcional que evalúa automáticamente múltiples casos de uso y certifica que la función es apta para pasar a producción.
