---
description: Procesamiento Asíncrono, Seguridad, Integración REST y Refactorización Legacy
---

# Sesión 9

### 1. Repaso Inicial y Resolución de Dudas de la Sesión 8

#### Objetivos

* Consolidar los conceptos de optimización de consultas SQL sargables e indexación B-Tree en tablas de telemetría industrial.
* Revisar la implementación del gestor de caché en memoria de Gateway con control de tiempo de vida (TTL).

#### Contenidos

* Repaso de las consultas sobre la reescritura de filtros temporales sin aplicar funciones sobre columnas indexadas.
* Revisión de los resultados de benchmarking y reducción de tiempos de CPU en la JVM.
* Comprobación del uso de la _Checklist de Rendimiento_ en los módulos de `Project Library`.

#### Resultado esperado

* Fijación de las directrices de alto rendimiento en base de datos y memoria, base técnica requerida para abordar tareas asíncronas e integraciones con servicios externos.

***

### 2. Tema 20: Procesamiento Asíncrono, Tareas Programadas e Idempotencia

#### Objetivos

* Comprender la diferencia entre la ejecución síncrona en hilos de interfaz de usuario y la ejecución en segundo plano (_Background Threads_).
* Dominar el uso de `system.util.invokeAsynchronous` para evitar el congelamiento de pantallas (_UI Freezing_) durante operaciones de larga duración.
* Diseñar tareas automáticas programadas en el Gateway bajo el principio de **idempotencia** y control de fallos desatendidos.

#### Contenidos

* Prevención del bloqueo de la interfaz de usuario:
  * El problema de las llamadas bloqueantes (consultas SQL pesadas, generación de informes, peticiones web) en el hilo principal de la UI.
  * Delegación de tareas a hilos secundarios con `system.util.invokeAsynchronous(funcion_objetivo)`.
* Notificación asíncrona hacia la interfaz:
  * Comunicación de fin de tarea hacia Perspective mediante Message Handlers (`system.util.sendMessage`) sin bloquear la interactividad del operador.
* Automatizaciones desatendidas en Gateway:
  * Programación temporal basada en el motor Quartz CRON para procesos de cambio de turno o mantenimiento de datos.
  * Registro de inicio, fin, duración y excepciones en tareas automáticas sin interacción humana.
* El principio de Idempotencia en procesos de planta:
  * Diseño de scripts capaces de ejecutarse múltiples veces consecutivas sin duplicar registros en base de datos, descontar stock por duplicado ni romper estados de máquina (uso de comprobaciones previas y sentencias `UPSERT` / `MERGE`).

```mermaid
flowchart TD
    subgraph Hilo_UI [Hilo Principal de Perspective UI]
        Click[Operador pulsa boton: Generar Reporte Pesado]
        AsyncCall[system.util.invokeAsynchronous: Despacha tarea]
        UI_Free[La interfaz permanece 100% fluida e interactiva]
        Click --> AsyncCall --> UI_Free
    end

    subgraph Hilo_Background [Pool de Hilos Secundarios JVM]
        TaskExec[Ejecucion en Segundo Plano: DB / API / Calculo]
        Notify[system.util.sendMessage: Notifica finalizacion]
        AsyncCall -.->|Crea Hilo Separado| TaskExec
        TaskExec --> Notify
    end

    Notify -.->|Actualiza Componente| UI_Free
```

#### Resultado esperado

* Capacidad para ejecutar procesos pesados en segundo plano sin congelar la pantalla del operador y diseñar tareas programadas de Gateway que se ejecuten de forma segura e idempotente.

***

### 3. Tema 21: Seguridad en Scripting, Validación RBAC en Servidor y Protección de Secretos

#### Objetivos

* Comprender el principio de defensa en profundidad aplicado al scripting SCADA, distinguiendo la seguridad en interfaz de la seguridad en servidor.
* Implementar control de acceso basado en roles (_Role-Based Access Control - RBAC_) directamente en la capa de lógica de negocio en Gateway.
* Aplicar directrices para la protección de credenciales y la sanitización de trazas de error hacia el usuario final.

#### Contenidos

* Seguridad en Backend (Servidor) vs. Seguridad en Frontend (UI):
  * La vulnerabilidad de depender únicamente de la propiedad `enabled` o `visible` en botones de pantalla.
  * Validación obligatoria de roles y permisos en el script de ejecución en Gateway (`session.props.auth.user.roles`).
* Prevención de fuga de credenciales e información sensible:
  * Prohibición absoluta de almacenar contraseñas, tokens de API o cadenas de conexión en texto plano dentro de scripts.
  * Uso de conexiones administradas de Gateway, variables de entorno y almacenes seguros.
* Sanitización de mensajes de error:
  * Separación estricta entre la traza técnica completa (registrada internamente con `system.util.getLogger`) y el mensaje mostrado al operador (mensaje funcional que no expone nombres de tablas, direcciones IP ni volcados de pila).

#### Resultado esperado

* Dominio del diseño de funciones protegidas mediante validación de roles en servidor y eliminación de credenciales expuestas en el código fuente.

***

### 4. Tema 22: Integración con APIs REST Externas y Microservicios

#### Objetivos

* Dominar la comunicación HTTP/REST desde Ignition utilizando la utilidad nativa `system.net.httpClient`.
* Comprender la arquitectura híbrida SCADA + Microservicios para desacoplar cálculos analíticos complejos (Python 3 / FastAPI) de la JVM de Ignition.
* Diseñar clientes REST industriales resilientes con timeouts estrictos, gestión de cabeceras de autenticación y mecanismos de respuesta de contingencia (_Fallback_).

#### Contenidos

* El cliente HTTP moderno de Ignition: `system.net.httpClient`:
  * Configuración de peticiones GET, POST, PUT, DELETE con parámetros, cabeceras (`Authorization: Bearer <token>`) y decodificación automática de JSON.
  * Configuración de timeouts cortos (1.000 a 3.000 ms) para impedir que caídas de servicios externos retengan hilos en el Gateway.
* Arquitectura Híbrida: Cuándo delegar a Microservicios externos:
  * Cuando un cálculo requiere librerías no soportadas por Jython 2.7 (NumPy, Pandas, modelos de IA/Machine Learning en Python 3, algoritmos de optimización pesados).
  * Creación de APIs internas en **FastAPI (Python 3)** consumidas por Ignition vía HTTP REST.
* Resiliencia y mecanismos de Fallback:
  * Manejo estructurado de fallos de red y códigos de estado HTTP anómalos (4xx, 5xx), entregando valores de contingencia seguros para que la planta continúe operando.

```mermaid
flowchart LR
    subgraph Ignition_SCADA [Ignition Gateway: Jython 2.7]
        Script[project.integrations: Cliente HTTP]
        Client[system.net.httpClient: Timeout = 2000 ms]
        Fallback[Mecanismo de Contingencia / Fallback Local]
        Script --> Client
        Client -->|Si ocurre Timeout o Error 500| Fallback
    end

    subgraph Microservicio_Externo [Microservicio: Python 3 / FastAPI]
        API[Endpoint REST: /v1/optimize-production]
        Engine[Calculo Matematico Avanzado / ML / Pandas]
        API --> Engine
    end

    Client <-->|JSON sobre HTTP / HTTPS| API
```

#### Resultado esperado

* Capacidad para integrar Ignition con servicios web y APIs REST empresariales de forma robusta, aplicando timeouts defensivos y arquitecturas híbridas con microservicios externos.

***

### 5. Laboratorio 9.1: Cliente REST Asíncrono con Timeout, Fallback Resiliente y Monitor en Perspective

#### Objetivos

* Construir un módulo en `Project Library` (`project.integrations.energy`) que consuma una API REST externa de tarificación eléctrica mediante `system.net.httpClient`.
* Parametrizar cabeceras seguras, timeouts estrictos (2.000 ms) y un mecanismo de respuesta de fallback que devuelva una tarifa predeterminada si la API externa no responde o arroja errores de conexión.
* Diseñar una vista operativa en Perspective (`Views/Laboratorios/Lab9_1_EnergyPricing`) con scripts ultra-ligeros de 1-2 líneas.

#### Resultado esperado

* Servicio probado en la Script Console ante respuestas nominales simuladas y ante escenarios de caída de red / timeout.
* Vista funcional en Perspective que visualiza el precio en EUR/MWh y distingue de forma inmediata si procede de la API en vivo o del fallback de contingencia sin congelar la interfaz.

#### Paso a Paso para la Realización

**Paso 1: Módulo en Project Library (`project.integrations.energy`)**
Crear el script `integrations.energy` con cliente HTTP, timeout estricto y funciones para UI:
```python
def get_current_energy_price(area_code="ES_PENINSULA", force_failure=False):
    logger = system.util.getLogger("SCADA.Integrations.Energy")
    FALLBACK_TARIFF = {
        "price_eur_mwh": 85.50,
        "is_fallback": True,
        "source": "DEFAULT_CONFIG_FALLBACK",
        "timestamp": system.date.format(system.date.now(), "dd/MM/yyyy HH:mm:ss")
    }
    client = system.net.httpClient(timeout=2000)
    headers = {"Accept": "application/json", "User-Agent": "Ignition-SCADA-Gateway/8.1", "X-Plant-ID": "EDAR_CENTRAL_01"}
    target_url = "https://api.energypricing.internal/v1/tariffs/current" if not force_failure else "https://10.255.255.1/timeout_test"
    try:
        response = client.get(target_url, params={"area": area_code}, headers=headers)
        if response.statusCode == 200:
            price_val = float(response.json.get("price", 85.50))
            return {"price_eur_mwh": round(price_val, 2), "is_fallback": False, "source": "LIVE_REST_API", "timestamp": system.date.format(system.date.now(), "dd/MM/yyyy HH:mm:ss")}
        logger.warn("API de energía devolvió HTTP {}. Aplicando fallback.".format(response.statusCode))
        return FALLBACK_TARIFF
    except Exception as ex:
        logger.error("Error de conexión API energía: {}. Aplicando fallback.".format(str(ex)))
        return FALLBACK_TARIFF

def fetch_energy_price_for_ui(area_code="ES_PENINSULA", force_failure=False):
    return get_current_energy_price(area_code=area_code, force_failure=force_failure)

def get_energy_status_badge(is_fallback):
    if is_fallback:
        return {"text": "TARIFA DE CONTINGENCIA (FALLBACK ACTIVO)", "backgroundColor": "#e67e22", "color": "#ffffff"}
    return {"text": "API REST EN VIVO (CONEXIÓN NOMINAL)", "backgroundColor": "#27ae60", "color": "#ffffff"}
```

**Paso 2: Validación en Script Console**
```python
res_fail = project.integrations.energy.fetch_energy_price_for_ui("ES_PENINSULA", force_failure=True)
print "Precio:", res_fail["price_eur_mwh"], "EUR/MWh | Fallback:", res_fail["is_fallback"]
badge = project.integrations.energy.get_energy_status_badge(res_fail["is_fallback"])
print "Badge UI:", badge["text"], "(Color:", badge["backgroundColor"], ")"
```

**Paso 3: Monitor en Perspective (`Views/Laboratorios/Lab9_1_EnergyPricing`)**
1. Crear la vista `Lab9_1_EnergyPricing` (`Flex Container`, `column`) con propiedad `custom`:
   * `custom.energyData`: `{}`
2. **Controles de Consulta:**
   * Dropdown (`drop_Zona`): Opciones `ES_PENINSULA`, `ES_BALEARES`, `ES_CANARIAS`.
   * Toggle Switch (`toggle_Timeout`): `props.label.text = "Simular Caída de Red / Timeout (2s)"`.
   * Botón Consultar (`btn_ConsultarTarifa -> onActionPerformed` - 2 líneas):
     ```python
     res = project.integrations.energy.fetch_energy_price_for_ui(self.getSibling("drop_Zona").props.value, self.getSibling("toggle_Timeout").props.selected)
     self.view.custom.energyData = res
     ```
3. **Tarjetas de Visualización:**
   * Precio (`kpi_Precio`): Binding a `view.custom.energyData.price_eur_mwh` con sufijo `" EUR / MWh"`.
   * Badge Estado (`kpi_Badge`): Binding a `view.custom.energyData.is_fallback`.
     * Transform en `props.text` (1 línea):
       ```python
       return project.integrations.energy.get_energy_status_badge(value)["text"]
       ```
     * Transform en `props.style.backgroundColor` (1 línea):
       ```python
       return project.integrations.energy.get_energy_status_badge(value)["backgroundColor"]
       ```
     * `props.style.color`: `"#ffffff"`.

**Paso 4: Verificación Interactiva**
Lanzar en Preview (`Ctrl+P`). Activar el conmutador de simulación y pulsar el botón: tras 2 segundos de timeout estricto, la pantalla se desbloquea mostrando el precio de contingencia (85.50 EUR/MWh) y el badge en naranja sin haber bloqueado el hilo de ejecución web.

***

### 6. Laboratorio 9.2: Control de Acceso Basado en Roles (RBAC), Auditoría en Servidor y Panel en Perspective

#### Objetivos

* Crear un servicio centralizado en `Project Library` (`project.security.core`) que valide si los roles del usuario autenticado cumplen los privilegios necesarios antes de ejecutar cualquier acción operativa o cambio de consigna.
* Registrar en el log del Gateway y persistir en base de datos (`ins_Registre_Actuacions`) cualquier intento no autorizado y las operaciones exitosas.
* Construir un panel interactivo en Perspective (`Views/Laboratorios/Lab9_2_RBACMonitor`) con scripts de 1-2 líneas que evidencie la autorización para supervisores y la denegación para operadores.

#### Resultado esperado

* Función modular verificada en consola simulando llamadas con usuarios operadores (rol básico, denegado con advertencia de seguridad) y supervisores (rol autorizado, ejecución completada y auditada).
* Vista funcional en Perspective con feedback cromático inmediato (rojo para denegado, verde para autorizado) y refresco automático de la tabla de auditoría.

#### Paso a Paso para la Realización

**Paso 1: Módulo en Project Library (`project.security.core`)**
Crear el script `security.core` con validación de roles en servidor, auditoría y soporte UI:
```python
def execute_authorized_action(user_roles, required_roles, action_name, equip, username, action_func, *args, **kwargs):
    logger = system.util.getLogger("SCADA.Security.RBAC")
    user_str = str(username) if username else "ANONYMOUS"
    roles_user_set = set(user_roles) if user_roles else set()
    roles_req_set = set(required_roles) if required_roles else set()
    
    authorized = len(roles_user_set.intersection(roles_req_set)) > 0
    if not authorized:
        logger.warn("INTENTO NO AUTORIZADO: Usuario '{}' (Roles: {}) intentó ejecutar [{}] en '{}'".format(user_str, list(roles_user_set), action_name, equip))
        try:
            sql = "INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari) VALUES (?, ?, ?)"
            system.db.runPrepUpdate(sql, [str(equip), u"SEGURIDAD: Intento no autorizado de ejecutar [{}]".format(unicode(action_name)), user_str], database="SANDBOX_DB")
        except Exception as db_ex:
            logger.error("Error al registrar auditoría de seguridad: {}".format(str(db_ex)))
        return (False, u"Acceso Denegado: El usuario no dispone de los roles requeridos ({})".format(", ".join(required_roles)))
        
    try:
        result = action_func(*args, **kwargs)
        sql = "INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari) VALUES (?, ?, ?)"
        system.db.runPrepUpdate(sql, [str(equip), u"OPERACION: Maniobra autorizada [{}] ejecutada con exito".format(unicode(action_name)), user_str], database="SANDBOX_DB")
        return (True, result)
    except Exception as ex:
        logger.error("Error ejecutando acción autorizada [{}]: {}".format(action_name, str(ex)))
        return (False, u"Error interno durante la ejecución de la maniobra")

def _maniobra_ajuste_velocidad(equip, new_speed):
    return u"Velocidad de {} ajustada a {} m3/h con éxito".format(equip, new_speed)

def execute_critical_setpoint_ui(user_roles, username, equip, new_speed):
    ok, msg = execute_authorized_action(user_roles, ["Supervisor", "Administrator"], "AjusteCaudalCritico", equip, username, _maniobra_ajuste_velocidad, equip, new_speed)
    return {"success": ok, "message": msg, "color": "#27ae60" if ok else "#c0392b"}

def get_security_audit_table(equip="EDAR_BOMBA_01", db_conn="SANDBOX_DB"):
    sql = "SELECT id, Equip, Descripcio, Usuari, DataHora FROM ins_Registre_Actuacions WHERE Equip = ? ORDER BY DataHora DESC LIMIT 10"
    pyds = system.dataset.toPyDataSet(system.db.runPrepQuery(sql, [equip], database=db_conn))
    rows = []
    for r in pyds:
        desc = r["Descripcio"] or ""
        is_sec = "SEGURIDAD" in desc
        rows.append({
            "id": r["id"], "Equipo": r["Equip"], "Descripcion": desc, "Usuario": r["Usuari"],
            "Fecha": system.date.format(r["DataHora"], "dd/MM/yyyy HH:mm:ss") if r["DataHora"] else "---",
            "_style": {"backgroundColor": "#fdedec" if is_sec else "#ffffff", "color": "#c0392b" if is_sec else "#2c3e50", "fontWeight": "bold" if is_sec else "normal"}
        })
    return rows
```

**Paso 2: Validación en Script Console**
```python
res1 = project.security.core.execute_critical_setpoint_ui(["Operator"], "operador_noche", "EDAR_BOMBA_01", 80.0)
print "Operador Basico:", res1["success"], "| Mensaje:", res1["message"]
res2 = project.security.core.execute_critical_setpoint_ui(["Operator", "Supervisor"], "laura.vidal", "EDAR_BOMBA_01", 80.0)
print "Supervisor:     ", res2["success"], "| Mensaje:", res2["message"]
```

**Paso 3: Panel en Perspective (`Views/Laboratorios/Lab9_2_RBACMonitor`)**
1. Crear la vista `Lab9_2_RBACMonitor` (`Flex Container`, `column`) con propiedades `custom`:
   * `custom.simulatedRoles`: `["Operator"]`
   * `custom.securityFeedback`: `{}`
2. **Controles y Botón de Maniobra:**
   * Dropdown (`drop_Roles`): Opciones `Operator` y `Supervisor`.
   * Campo numérico (`num_Velocidad`): Valor inicial `85.0`.
   * Botón Ejecutar (`btn_EjecutarManiobra -> onActionPerformed` - 2 líneas):
     ```python
     roles = [self.getSibling("drop_Roles").props.value]
     self.view.custom.securityFeedback = project.security.core.execute_critical_setpoint_ui(roles, self.session.props.auth.user.userName, "EDAR_BOMBA_01", self.getSibling("num_Velocidad").props.value)
     ```
3. **Feedback y Tabla de Auditoría:**
   * Label Feedback (`lbl_Feedback`): Binding a `view.custom.securityFeedback.message` y color a `view.custom.securityFeedback.color`.
   * Tabla Auditoría (`table_Auditoria.props.data` Transform - 1 línea):
     ```python
     return project.security.core.get_security_audit_table("EDAR_BOMBA_01")
     ```

**Paso 4: Verificación Interactiva**
Lanzar en Preview (`Ctrl+P`). Al probar con perfil `Operador Básico`, el sistema rechaza la orden con feedback en rojo y registra el intento no autorizado en la tabla. Al cambiar a `Supervisor de Planta`, la orden se aprueba en verde y queda auditada.

***

### 7. Tema 23: Metodología de Refactorización de Scripts Heredados (Legacy)

#### Objetivos

* Identificar síntomas de código degradado (_Code Smells_) habituales en proyectos SCADA antiguos.
* Aplicar una metodología secuencial de refactorización para transformar scripts monolíticos e inseguros en código modular y mantenible.
* Diseñar pruebas de no-regresión para garantizar que los cambios estructurales conservan exactamente el comportamiento funcional requerido por la planta.

#### Contenidos

* Diagnóstico de "Code Smells" en aplicaciones SCADA:
  * Consultas SQL dinámicas concatenadas en texto plano.
  * Scripts de decenas de líneas incrustados en botones y componentes gráficos.
  * Bucles `for` que ejecutan llamadas unitarias a `system.tag.readBlocking`.
  * Ausencia de validación de tipos, nulos o captura de excepciones.
* Flujo de refactorización en 5 pasos:
  1. Caracterización: Mapear entradas, salidas y casos límite del script original.
  2. Banco de Pruebas: Crear casos de prueba de referencia en la Script Console.
  3. Extracción: Mover la lógica matemática a funciones puras en `Project Library`.
  4. Sustitución de I/O: Reemplazar SQL inseguro por Named Queries y lecturas de tags por llamadas masivas en lote.
  5. Verificación de No-Regresión: Ejecutar la batería de pruebas para comprobar que los resultados son idénticos a los del sistema original.

#### Resultado esperado

* Dominio de la metodología de modernización de código heredado, transformando scripts frágiles en componentes modulares, testeables y seguros.

***

### 8. Laboratorio 9.3: Taller de Refactorización Integral de Código Heredado con Implementación en Perspective

#### Objetivos

* Analizar un script industrial monolítico real del cliente (páginas 9-10) que ejecutaba lecturas y escrituras de tags individuales en bucle dentro de un botón de Perspective.
* Refactorizar el código separando la lógica en una función pura en `Project Library` (`project.consignas.service`) y orquestando el lote con operaciones atómicas seguras.
* Rediseñar el script del botón en Perspective (`Views/Laboratorios/Lab9_3_ConsignasRefactored`), reduciendo las 20 líneas de anti-patrones originales a **2 líneas exactas** y seguras.

#### Resultado esperado

* Script heredado completamente transformado en un módulo limpio en `Project Library` (`project.consignas.service`), validado mediante la Script Console ante un vector de pruebas que certifica la equivalencia funcional.
* Vista en Perspective con Flex Repeater donde el botón de envío masivo se ejecuta de forma atómica en una sola trama de red con registro unificado de auditoría.

#### Paso a Paso para la Realización

**Paso 1: Módulo en Project Library (`project.consignas.service`)**
Crear el script `consignas.service` con función pura de mapeo, orquestador por lotes y generador de datos:
```python
def resolve_target_tag_name(source_tag_name, is_esim_active=False):
    if not source_tag_name:
        return ""
    tag_clean = source_tag_name.strip()
    if tag_clean == "SFOR" and is_esim_active:
        return "MFOS"
    if len(tag_clean) > 1 and tag_clean.startswith("S"):
        return "M" + tag_clean[1:]
    return "M_" + tag_clean

def process_and_send_consignas_batch(instances_data, base_path, equip_name, username):
    logger = system.util.getLogger("SCADA.Consignas.BatchService")
    if not instances_data or not isinstance(instances_data, list):
        return (False, {"msg": u"No se han recibido instancias para procesar", "count": 0})
    selected_items = [inst for inst in instances_data if inst.get("selected", False) is True]
    if len(selected_items) == 0:
        return (True, {"msg": u"No hay ninguna consigna seleccionada para enviar", "count": 0})
    try:
        # 1. Lectura por lote de prerrequisitos (ESIM y Tooltips en 1 llamada)
        root_path = (base_path or "").rstrip("/")
        all_read_paths = ["{}/ESIM".format(root_path)] + ["{}.Tooltip".format(item.get("tagPath", "")) for item in selected_items]
        read_qvs = system.tag.readBlocking(all_read_paths, 3000)
        esim_val = bool(read_qvs[0].value) if read_qvs[0].quality.isGood() else False
        
        # 2. Preparacion de listas masivas
        write_paths, write_values, audit_records = [], [], []
        for item, tt_qv in zip(selected_items, read_qvs[1:]):
            src_tagpath, val_new = item.get("tagPath", ""), item.get("value_new")
            if not src_tagpath or val_new is None:
                continue
            tag_name = src_tagpath.split("/")[-1]
            target_write_path = src_tagpath.rsplit("/", 1)[0] + "/" + resolve_target_tag_name(tag_name, esim_val)
            tooltip_desc = unicode(tt_qv.value) if tt_qv.quality.isGood() and tt_qv.value else tag_name
            write_paths.append(target_write_path)
            write_values.append(val_new)
            audit_records.append(u"Orden Enviar Consigna {}: {} -> Valor: {}".format(tag_name, tooltip_desc, val_new))
            
        # 3. Escritura atomica en lote y auditoria
        write_qcs = system.tag.writeBlocking(write_paths, write_values, 3000)
        project_name, user_str = system.util.getProjectName(), str(username) if username else "SYSTEM"
        for desc in audit_records:
            system.db.runNamedQuery(project=project_name, path="Actuacions/InsertRegistre", parameters={"equip": str(equip_name), "descripcio": desc, "usuari": user_str})
        return (True, {"msg": u"Consignas enviadas y auditadas con éxito ({} actualizadas)".format(len(write_paths)), "count": len(write_paths)})
    except Exception as ex:
        logger.error("Error en batch de consignas: {}".format(str(ex)))
        return (False, {"msg": u"Fallo operativo: {}".format(str(ex)), "count": 0})

def get_sample_consignas_instances(base_path="[default]EDAR/EDAR_BOMBA_01"):
    root = (base_path or "").rstrip("/")
    return [
        {"selected": True,  "tagPath": "{}/SCFL".format(root), "value_new": 65.5, "label": "Caudal Consigna"},
        {"selected": True,  "tagPath": "{}/SCMX".format(root), "value_new": 105.0, "label": "Caudal Máximo"},
        {"selected": False, "tagPath": "{}/SFOR".format(root), "value_new": 0.0, "label": "Modo Forzado"}
    ]
```

**Paso 2: Validación en Script Console**
```python
assert project.consignas.service.resolve_target_tag_name("SCFL", False) == "MCFL"
assert project.consignas.service.resolve_target_tag_name("SFOR", True) == "MFOS"
print "1. Mapeo de nombres:", "[ PASS ]"
instances = project.consignas.service.get_sample_consignas_instances()
ok, summary = project.consignas.service.process_and_send_consignas_batch(instances, "[default]EDAR/EDAR_BOMBA_01", "EDAR_BOMBA_01", "marti.sanchez")
print "2. Servicio Batch:", "Exito =", ok, "|", summary["msg"]
```

**Paso 3: Vista Refactorizada en Perspective (`Views/Laboratorios/Lab9_3_ConsignasRefactored`)**
1. Crear la vista `Lab9_3_ConsignasRefactored` (`Flex Container`, `column`) con propiedades `custom`:
   * `custom.basePath`: `"[default]EDAR/EDAR_BOMBA_01"`
   * `custom.equipName`: `"EDAR_BOMBA_01"`
   * `custom.feedbackMessage`: `"Listo para enviar"`
2. **Flex Repeater de Consignas (`rep_Consignas.props.instances` Transform - 1 línea):**
   ```python
   return project.consignas.service.get_sample_consignas_instances(self.view.custom.basePath)
   ```
3. **Botón Enviar Consignas (`btn_EnviarConsignas -> onActionPerformed` - 2 líneas):**
   ```python
   ok, summary = project.consignas.service.process_and_send_consignas_batch(self.getSibling("rep_Consignas").props.instances, self.view.custom.basePath, self.view.custom.equipName, self.session.props.auth.user.userName)
   self.view.custom.feedbackMessage = summary["msg"]
   ```
4. **Label de Estado (`lbl_Resultado`):**
   * Property Binding en `props.text` hacia `view.custom.feedbackMessage`.

**Paso 4: Verificación Interactiva**
Lanzar en Preview (`Ctrl+P`), marcar las consignas deseadas y pulsar el botón. Verificar la actualización inmediata en menos de 50 ms en un único bloque de ejecución seguro y auditado, habiendo reemplazado 20 líneas de código monolítico por 2 líneas limpias.

***

### 9. Test de Conceptos de la Sesión 9

#### Objetivos

* Evaluar la comprensión sobre el procesamiento asíncrono y el principio de idempotencia en tareas de Gateway.
* Validar el entendimiento de la seguridad RBAC en backend frente a frontend.
* Comprobar el dominio sobre el consumo de APIs REST con `system.net.httpClient` y la metodología de refactorización de código heredado.

#### Contenidos

* Cuestionario técnico individual de opción múltiple y resolución de casos de arquitectura asíncrona, seguridad y refactorización.

#### Resultado esperado

* Fijación de los principios de diseño asíncrono, ciberseguridad industrial y modernización de código en Ignition SCADA.

***

### 10. Feedback Individual y Cierre de la Sesión

#### Objetivos

* Revisar las soluciones de refactorización implementadas por los alumnos en el Laboratorio 9.3.
* Verificar el correcto funcionamiento de los clientes REST y el control de roles RBAC.
* Presentar la planificación de la Sesión 10.

#### Contenidos

* Rondas de revisión individual del código refactorizado y validación de estándares de diseño.
* Resolución de dudas sobre integración con microservicios externos y gestión de timeouts.
* Avance de la Sesión 10: Testing funcional formal de scripts y consultas, estándares de estilo y documentación de equipo, y presentación del enunciado del Proyecto Final Integrador (Tema 26).

#### Resultado esperado

* Cada participante finaliza la sesión intensiva con sus laboratorios validados, su ejercicio de refactorización completado y la preparación técnica requerida para abordar las fases de testing formal y desarrollo del proyecto final.
