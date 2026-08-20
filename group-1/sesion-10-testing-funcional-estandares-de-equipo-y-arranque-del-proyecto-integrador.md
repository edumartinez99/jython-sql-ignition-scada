# Sesión 10: Testing Funcional, Estándares de Equipo y Arranque del Proyecto Integrador

### 1. Bloques Teóricos

#### 📘 Bloque Teórico 1: Testing Funcional y Validación de Scopes en SCADA (Tema 24)

* **Conceptos clave a tratar:**
  * Metodología de pruebas funcionales adaptada a entornos de control y automatización industrial sin frameworks CI/CD tradicionales.
  * Diseño sistemático de matrices de prueba:
    * _Camino feliz (Happy Path):_ Entradas nominales de operación.
    * _Valores límite (Edge Cases):_ Cero, desbordamientos, timestamps en cambio de año/turno y cadenas de longitud máxima.
    * _Casos de fallo y robustez:_ Entradas `None`, strings vacíos, listas sin datos y respuestas nulas de tags o base de datos.
  * Técnicas de simulación de datos (_Mocking_): creación de datasets y diccionarios sintéticos reproducibles para testear transformaciones sin depender de hardware de planta.
  * Verificación de contratos en Named Queries: validación de tipado de parámetros, columnas devueltas y comportamiento ante consultas vacías.
  * **La trampa del Scope:** por qué un script probado en la Script Console del Designer puede fallar al ejecutarse en una sesión de Perspective o en un Gateway Timer Script debido a variables de contexto, permisos o dependencias de sesión.
  * Creación y aplicación de la **Checklist Pre-Despliegue** (seguridad, rendimiento, manejo de errores, logging contextual y permisos validados).
* **Objetivos:**
  * Aprender a diseñar y ejecutar baterías de pruebas funcionales para scripts y consultas en Ignition.
  * Validar el comportamiento de las funciones en su scope real de ejecución.
* **Resultado esperado:**
  * El alumno valida scripts de forma metódica mediante pruebas unitarias manuales y datasets simulados antes de integrarlos en componentes de planta.

***

#### 📘 Bloque Teórico 2: Estándares Corporativos, Docstrings y Revisión por Pares (Tema 25)

* **Conceptos clave a tratar:**
  * Homogeneización del estilo de desarrollo en equipos de ingeniería: convenciones de nomenclatura claras para variables, funciones, Named Queries, módulos y parámetros.
  * Principio de Responsabilidad Única (_SRP_): diseño de funciones breves, legibles y con responsabilidades desacopladas.
  * Redacción de comentarios orientados a la intención de negocio y restricciones técnicas de proceso, descartando comentarios obvios de sintaxis.
  * **Estandarización corporativa de&#x20;**_**Docstrings**_**:** estructura uniforme obligatoria (propósito, tipado de parámetros, valor de retorno, excepciones y ejemplos de uso).
  * Gobernanza de la **Project Library**: estructuración por **dominio funcional de planta** (`produccion`, `calidad`, `mantenimiento`) y prohibición estricta de organizar paquetes por nombres de programadores.
  * Catálogo de buenas prácticas (_Do's & Don'ts_): pautas claras de acceso a base de datos, logging estructurado, escrituras protegidas y transforms limpios.
  * Protocolo de revisión de código por pares (_Peer Code Review_) y uso de plantillas (_snippets/templates_) para uniformar nuevos desarrollos.
* **Objetivos:**
  * Asimilar las pautas de estilo corporativo, documentación y gobernanza de código en Ignition.
  * Aprender a auditar código entre compañeros mediante revisiones estructuradas.
* **Resultado esperado:**
  * El alumno produce código limpio, legible y documentado bajo estándares de equipo, capaz de superar una revisión de código formal.

***

#### 📘 Bloque Teórico 3: Proyecto Integrador — Fase 1: Arquitectura de Datos y Librería Base (Tema 26 - Parte 1)

* **Conceptos clave a tratar:**
  * Especificación técnica del caso de estudio industrial: supervisión integral de una línea de fabricación con registro de eventos, control de estados, cálculo de KPIs y acciones operativas seguras.
  * Modelo entidad-relación de la base de datos de pruebas: tablas de líneas, órdenes de fabricación, eventos de parada, acumulados de turno y pistas de auditoría.
  * Diseño y parametrización del árbol de Named Queries centralizadas (lecturas con `JOIN`, inserciones transaccionales y actualizaciones protegidas).
  * Construcción del esqueleto de módulos en la Project Library aplicando los estándares de documentación y control defensivo de errores establecidos.
* **Objetivos:**
  * Diseñar la arquitectura completa de datos y lógica del proyecto final integrador.
  * Implementar las capas base de base de datos, consultas y librerías compartidas.
* **Resultado esperado:**
  * El alumno despliega el modelo relacional de datos y configura la estructura base de Named Queries y funciones de librería para el proyecto final.

***

### 2. Bloque de Laboratorios Prácticos

#### 🧪 Laboratorio 10.1: Suite de Pruebas Funcionales con Datasets Simulados (Mocking) (Tema 24)

* **Objetivos:**
  * Diseñar un arnés de pruebas automatizado en la Script Console que evalúe funciones críticas de cálculo y validación frente a una matriz de casos de prueba (nominales, límites y datos corruptos).
  * Simular un dataset de entrada heterogéneo (_mock dataset_) y verificar mediante aserciones que las funciones retornan los valores esperados o _fallbacks_ controlados.
* **Resultado esperado:**
  * Un script de testing funcional ejecutado en consola que valida automáticamente la suite completa de pruebas emitiendo un reporte de conformidad `[PASS]` / `[FAIL]`.

***

#### 🧪 Laboratorio 10.2: Auditoría Cruzada por Pares (Peer Review) y Plantillas de Código (Tema 25)

* **Objetivos:**
  * Intercambiar un script desarrollado en sesiones previas con un compañero y realizar una auditoría técnica aplicando la rúbrica de estándares corporativos.
  * Crear una plantilla estandarizada (_template_) de función para la Project Library con cabecera de docstrings, logging contextual y estructura defensiva `try / except / finally`.
* **Resultado esperado:**
  * Un informe de revisión de código documentando puntos de mejora técnica y un archivo de plantillas corporativas listo para reutilizarse en futuros desarrollos.

***

#### 🧪 Laboratorio 10.3: Proyecto Integrador — Modelo DDL y Capa de Named Queries (Tema 26 - Fase 1)

* **Objetivos:**
  * Ejecutar el script DDL para crear las tablas de producción, eventos y auditoría en la base de datos de pruebas de Ignition.
  * Configurar y testear en el Designer el conjunto de Named Queries parametrizadas para lectura de estado de línea, inserción de eventos de paro y actualización de órdenes.
* **Resultado esperado:**
  * Una base de datos poblada con datos iniciales de prueba y un árbol de Named Queries organizado por dominios, tipado y verificado desde el panel de pruebas del Designer.

***

#### 🧪 Laboratorio 10.4: Proyecto Integrador — Módulos Base en Project Library (Tema 26 - Fase 2)

* **Objetivos:**
  * Implementar los paquetes `proyecto.produccion` y `proyecto.eventos` en la Project Library con docstrings estandarizados y logging estructurado.
  * Desarrollar las funciones de validación de roles, verificación de precondiciones de proceso y transformación de datos tabulares para la interfaz.
* **Resultado esperado:**
  * La librería base del proyecto integrador completamente programada y verificada mediante la Script Console, lista para ser conectada a vistas y tareas en la sesión final.
