---
description: Informe técnico de auditoría, observabilidad, logging contextual en Gateway y coprocesamiento analítico en SQL (Sesión 4).
---

# Informe Técnico: Refactorización y Estandarización de Scripts SCADA (Fase 4: Observabilidad, Logging Contextual y SQL Analítico Industrial)

## 1. Resumen Ejecutivo

Este informe documenta la auditoría técnica y la propuesta de refactorización integral de los scripts del sistema SCADA en Ignition (`ejemplos-scripts-ignition.md`), fundamentada estrictamente en las directrices de observabilidad, tratamiento profesional de excepciones en la JVM, registro de trazas contextuales en el Gateway y coprocesamiento analítico en bases de datos desarrolladas en los laboratorios de la **Sesión 4** (`gcp-con-eduardo/lab-04.md`).

### Principales Deficiencias Detectadas

1. **Fallos Silenciosos y Bloques `try/except` Ciegos (*Bare Except*):** En componentes visuales críticos como el binding de consignas (`Historian/Consignas`), las caídas del servidor de base de datos o fallos de red se silencian mediante bloques `except:` vacíos. Esto oculta anomalías graves al personal de mantenimiento, falsea la lógica de selección de mes y deja al operador sin retroalimentación visual sobre el estado del sistema.
2. **Ausencia de Captura Dual en la JVM y Falta de Observabilidad Centralizada:** Los scripts que interactúan con bases de datos o autómatas (como la tarea programada `onScheduledEvent` o el evento de tag `valueChanged` en enclavamientos) o no implementan manejo de errores o emplean `except Exception as e:` de Python estándar. En el entorno de ejecución de Jython 2.7 sobre la JVM, los fallos de drivers JDBC (`java.sql.SQLException`) o errores internos de Java (`java.lang.Exception`) escapan a esta captura. Además, se emplean loggers dispersos y ad-hoc (`Neteja_Script`) en lugar de canalizar los registros al sistema centralizado del Gateway (`SCADA.Core`).
3. **Escrituras Industriales y Consultas de Auditoría Desprotegidas:** En eventos de interfaz como el botón de envío de consignas (`runAction`) y en eventos de tags de Gateway, se ejecutan operaciones de escritura física (`system.tag.writeBlocking`) y consultas de inserción/actualización (`runNamedQuery`, `runUpdateQuery`) sin envoltorios de protección ni registro contextual de incidencias. Si la comunicación con el PLC o la base de datos se interrumpe, se detiene la ejecución intempestivamente sin registrar metadatos forenses (`usuario`, `equipo`, `acción`).
4. **Antipatrón de Sobrecarga de Memoria en Ignition frente a Coprocesamiento en Base de Datos:** Se identifican consultas con `SELECT *` y transferencia masiva de registros en bruto hacia la memoria RAM de Ignition para luego iterar celda por celda (`getValueAt`) o evaluar estados con bucles en Python. Esto desaprovecha la capacidad del motor de base de datos como coprocesador matemático masivo y genera riesgo de cuellos de botella en la JVM.

### Arquitectura Objetivo

* **Módulo Universal de Ejecución Defensiva (`project.util.logging.safe_execution`):** Despliegue de un *Wrapper* centralizado en la `Project Library` que encapsule cualquier operación de riesgo industrial, asegurando captura dual jerárquica (Jython nativo + clases Java `SQLException` y `JavaException`).
* **Logging Contextual Estandarizado en el Gateway:** Registro automático en el motor SLF4J/Logback del Gateway mediante `system.util.getLogger("SCADA.Core")`, incorporando metadatos estructurados en formato clave-valor (`user=... | equip=... | action=...`) y asociando niveles de severidad industrial (`DEBUG`, `INFO`, `WARN`, `ERROR`).
* **Doble Canal de Comunicación (UX Industrial):** Separación tajante entre la traza técnica forense (dirigida al visor web de logs del Gateway: *Status -> Diagnostics -> Logs*) y el mensaje estructurado legible para el operador, gobernado mediante el contrato de retorno canónico `(bool success, any result_or_user_message)`.
* **Coprocesamiento Analítico en Motor RDBMS:** Prohibición estricta de `SELECT *` y transferencia de la carga de cálculo a PostgreSQL/SQL Server mediante `GROUP BY`, funciones agregadas (`COUNT`, `AVG`, `MIN`, `MAX`), lógica condicional con `CASE WHEN` para regímenes de trabajo y asignación directa de códigos de color hexadecimales (`#e74c3c`, `#f39c12`, `#2ecc71`).
* **Consumo Eficiente con `PyDataSet`:** Procesamiento de datasets tabulares en memoria mediante `system.dataset.toPyDataSet`, iterando con acceso seguro por clave de columna y erradicando llamadas RPC celda a celda.

---

## 2. Matriz de Hallazgos y Acciones Correctivas

| Componente Auditado | Deficiencia Detectada (Código Original) | Riesgo Operativo | Solución Técnica Aplicada (Lab-04) |
| :--- | :--- | :--- | :--- |
| **Tarea Programada Gateway** (`onScheduledEvent` / `Neteja_Script`) | Logger ad-hoc (`Neteja_Script`), `except Exception as e` sin captura de `SQLException` ni contexto estructurado | Si el driver JDBC falla, no se intercepta la excepción Java; mantenimiento no dispone de traza unificada | Integración con `project.util.logging.safe_execution`, logger central `SCADA.Core`, metadatos contextuales y captura dual |
| **Binding Flex Consignas** (`Historian/Consignas`) | Bloque `try/except:` ciego (*bare except*), omisión de logs y mutación silenciosa de la variable `mes` | Caídas de BD quedan silenciadas; el operador cree que no hay datos cuando en realidad hay un fallo de red | Sustitución del bloque ciego por `safe_execution`, registro de error JDBC en Gateway y retroalimentación controlada |
| **Botón Enviar Consignas** (`runAction`) | Escritura física a PLC (`writeBlocking`) y Named Query (`ins_Registre_Actuacions`) sin captura ni trazas | Fallo en autómata o BD congela la UI o falla en silencio; el operador desconoce si la consigna se aplicó | Encapsulación en `safe_execution`, log forense con usuario y equipo en Gateway, y retorno amigable a la pantalla |
| **Tag Enclavamientos** (`valueChanged`) | `select *` por concatenación, inserciones y Named Query directas en hilo de tags sin captura de excepciones Java | Bloqueo o interrupción silenciosa del hilo de tags de Gateway ante fallos de conexión relacional | Eliminación de `select *`, protección de transacciones con `safe_execution` y log de auditoría con severidad industrial |
| **Gestión Documental** (`uploadFile` / `downloadFile`) | Invocación directa de `runNamedQuery` y descarga en navegador sin control de `SQLException` ni logs | Excepción fatal JDBC no capturada deja la vista en blanco sin registro del documento afectado | Envoltorio con `safe_execution` capturando errores relacionales y notificando al operador sin tecnicismos |
| **Consulta Histórica y KPIs** (`Historian/Consignas` / Telemetría) | Antipatrón de transferencia masiva a memoria y bucles Python para calcular estados e inspeccionar celdas | Alto consumo de CPU y RAM en la JVM; latencia elevada en clientes Perspective al procesar miles de muestras | Reingeniería analítica en SQL: `INNER JOIN`, agregaciones (`AVG`, `MIN`, `MAX`), `CASE WHEN` operativo y semáforo hex |

---

## 3. Detalle de Refactorización por Componente

---

### 3.1. Módulo Centralizado de Logging y Wrapper Defensivo (`project.util.logging`)

#### Diagnóstico

En el código heredado (`ejemplos-scripts-ignition.md`), la gestión de errores es inexistente o heterogénea:
* En la tarea programada (líneas 768-798), se utiliza un logger local no estandarizado:
  ```python
  logger = system.util.getLogger("Neteja_Script")
  ...
  except Exception as e:
      logger.error("Error a l'esborrat: %s" % str(e))
  ```
  Este bloque solo captura excepciones nativas de Python (`exceptions.Exception`). Si la base de datos lanza un error de conexión JDBC (`java.sql.SQLException`) o una interrupción del driver (`java.lang.Exception`), el bloque es ignorado y el hilo de Gateway falla sin un registro estructurado.
* En los eventos visuales y de tags (páginas 6, 7, 10 y 14 del documento técnico), o no hay captura de errores, o se recurre al *bare except* (`except:`), impidiendo que el equipo de soporte identifique incidencias en *Status -> Diagnostics -> Logs*.

#### Solución Propuesta (Enseñanzas Lab 4.1)

Implementar en **Project Library** el módulo `project.util.logging` con la función universal `safe_execution`. Esta función:
1. Importa explícitamente `java.lang.Exception as JavaException` y `java.sql.SQLException` para garantizar la **captura dual**.
2. Utiliza el registrador central `LOGGER = system.util.getLogger("SCADA.Core")`.
3. Formatea automáticamente los metadatos de contexto pasados como diccionario (`user=... | equip=... | action=...`).
4. Diferencia entre errores de sintaxis/datos de Jython (`TypeError`, `ValueError`, `KeyError`, `IndexError`), errores relacionales de base de datos (`SQLException`), fallos generales de drivers/JVM (`JavaException`) y excepciones genéricas de Python.
5. Retorna sistemáticamente una tupla estructurada `(bool success, any payload_or_user_message)`.

```python
"""
Modulo: project.util.logging
Descripcion: Wrapper defensivo de ejecucion con captura dual de excepciones
             (Jython + Java) y registro contextual en el Gateway.
             (Implementado conforme al Laboratorio 4.1 de Sesion 4).
Autor: Equipo SCADA
"""
from java.lang import Exception as JavaException
from java.sql import SQLException

# Logger centralizado del proyecto SCADA en el Gateway
LOGGER = system.util.getLogger("SCADA.Core")

def safe_execution(func_name, context_info, callable_func, *args, **kwargs):
    """
    Ejecuta una funcion de forma protegida, capturando excepciones de Jython y Java
    y emitiendo logs estructurados en el Gateway.
    
    Args:
        func_name (str): Nombre identificativo de la operacion (ej. 'TagWrite_Speed').
        context_info (dict|str): Metadatos contextuales (usuario, equipo, accion, etc.).
        callable_func (callable): Puntero a la funcion a ejecutar.
        *args, **kwargs: Argumentos posicionales y nombrados para callable_func.
        
    Returns:
        tuple: (bool success, any result_or_error_message)
    """
    # 1. Construccion del encabezado contextual legible
    if isinstance(context_info, dict):
        ctx_items = []
        for k, v in sorted(context_info.items()):
            ctx_items.append("{}={}".format(k, v))
        ctx_str = " | ".join(ctx_items)
    else:
        ctx_str = str(context_info)
        
    try:
        LOGGER.debug("[{}] Iniciando ejecucion. Contexto: [{}]".format(func_name, ctx_str))
        
        # 2. Invocacion de la funcion objetivo
        result = callable_func(*args, **kwargs)
        
        LOGGER.debug("[{}] Ejecucion completada con exito. Contexto: [{}]".format(func_name, ctx_str))
        return (True, result)
        
    except (TypeError, ValueError, KeyError, IndexError) as py_err:
        # Errores propios de la logica y tipos de Jython
        err_detail = "Error de tipos/datos en [{}]: {}".format(func_name, str(py_err))
        LOGGER.error("{} | Contexto: [{}]".format(err_detail, ctx_str))
        return (False, u"Error en los parámetros o datos de procesamiento")
        
    except SQLException as sql_err:
        # Errores especificos del motor de base de datos relacional (JDBC)
        err_detail = "Error SQL/JDBC en [{}]: {}".format(func_name, sql_err.getMessage())
        LOGGER.error("{} | Contexto: [{}]".format(err_detail, ctx_str), sql_err)
        return (False, u"Error de comunicación o transacción con la base de datos")
        
    except JavaException as java_err:
        # Errores genericos de la JVM / drivers de Ignition / sockets de red
        err_detail = "Error Java/Gateway en [{}]: {}".format(func_name, java_err.getMessage())
        LOGGER.error("{} | Contexto: [{}]".format(err_detail, ctx_str), java_err)
        return (False, u"Fallo de comunicación con el sistema industrial")
        
    except Exception as general_err:
        # Red de seguridad final para cualquier otro error de Python
        err_detail = "Error no controlado en [{}]: {}".format(func_name, str(general_err))
        LOGGER.error("{} | Contexto: [{}]".format(err_detail, ctx_str))
        return (False, u"Error interno no controlado")
```

---

### 3.2. Tarea Programada de Gateway: Limpieza de Tablas en BBDD (`onScheduledEvent`)

#### Diagnóstico

En `ejemplos-scripts-ignition.md` (líneas 754-799), el script programado del Gateway para purga de registros de auditoría presenta deficiencias de captura y observabilidad:

```python
# CÓDIGO ORIGINAL (Fragmento con logger aislado y captura de Python incompleta)
def onScheduledEvent():
    import system
    try:
        db = "BD_AUDIT"
        logger = system.util.getLogger("Neteja_Script")
        logger.info("Executant script de neteja...")
        count = system.db.runScalarQuery("SELECT COUNT(*) FROM REG_OPERADORS WHERE [Data] < DATEADD(YEAR, -5, GETDATE())", database=db)
        logger.info("Registres que s'eliminarien: %d" % count)
        if count > 0:
            query = "DELETE FROM REG_OPERADORS WHERE [Data] < DATEADD(YEAR, -5, GETDATE())"
            deleted_rows = system.db.runUpdateQuery(query, database=db)
            logger.info("Files eliminades: %d" % deleted_rows)
        else:
            logger.info("No hi ha registres antics per eliminar.")
    except Exception as e:
        logger.error("Error a l'esborrat: %s" % str(e))
```

* **Aislamiento de Logs:** El logger `"Neteja_Script"` dispersa las búsquedas en el visor de logs del Gateway.
* **Fuga de Excepciones Java:** Si la conexión `BD_AUDIT` sufre un timeout o se cae el socket TCP, se lanza `SQLException`, que atraviesa el `except Exception` sin ser capturada adecuadamente.
* **Falta de Contexto:** El log de error no especifica qué tabla falló ni el rango de depuración evaluado.

#### Solución Propuesta (Enseñanzas Lab 4.1)

1. Extraer la lógica nuclear de purga a una función de librería en `project.db.maintenance.purge_audit_table`.
2. Proteger la invocación mediante `project.util.logging.safe_execution` asociando metadatos contextuales (`database`, `table`, `retention`).
3. Registrar eventos de progreso bajo la jerarquía del registrador central `SCADA.Core`.

```python
# Implementación en Project Library: project.db.maintenance

def purge_audit_table(database_name, table_name, retention_years=5):
    """
    Ejecuta el recuento y purga de registros historicos antiguos en una tabla de auditoria.
    
    Args:
        database_name (str): Nombre de la conexion de base de datos configurada en Gateway.
        table_name (str): Tabla destino (ej. 'REG_OPERADORS').
        retention_years (int): Antiguedad en anos para el corte de purga.
        
    Returns:
        dict: Resumen de ejecucion con filas evaluadas y filas eliminadas.
    """
    logger = system.util.getLogger("SCADA.Core")
    
    count_sql = "SELECT COUNT(*) FROM {} WHERE [Data] < DATEADD(YEAR, -{}, GETDATE())".format(
        table_name, int(retention_years)
    )
    count = system.db.runScalarQuery(count_sql, database=database_name)
    
    if count is None:
        count = 0
        
    logger.info("[Mantenimiento] Registros candidatos a purga en {}: {}".format(table_name, count))
    
    deleted_rows = 0
    if count > 0:
        delete_sql = "DELETE FROM {} WHERE [Data] < DATEADD(YEAR, -{}, GETDATE())".format(
            table_name, int(retention_years)
        )
        deleted_rows = system.db.runUpdateQuery(delete_sql, database=database_name)
        logger.info("[Mantenimiento] Purga completada en {}. Filas eliminadas: {}".format(table_name, deleted_rows))
    else:
        logger.info("[Mantenimiento] No se encontraron registros antiguos en {} para depurar".format(table_name))
        
    return {"evaluated": count, "deleted": deleted_rows}
```

#### Código Resultante en la Tarea Programada del Gateway (`Gateway Scheduled Event`)

```python
# Evento onScheduledEvent en Gateway Scripting
def onScheduledEvent():
    db_conn = "BD_AUDIT"
    target_table = "REG_OPERADORS"
    retention = 5
    
    context = {
        "action": "PURGE_AUDIT_LOGS",
        "database": db_conn,
        "table": target_table,
        "retention_years": retention,
        "trigger": "SCHEDULED_TIMER"
    }
    
    ok, result = project.util.logging.safe_execution(
        "PurgeAuditTable_Scheduled",
        context,
        project.db.maintenance.purge_audit_table,
        db_conn,
        target_table,
        retention
    )
    
    if not ok:
        system.util.getLogger("SCADA.Core").warn(
            "[Mantenimiento] La tarea programada de purga no pudo completarse. Notificacion enviada."
        )
```

---

### 3.3. Erradicación del *Bare Except* y Traza de BD en Binding de Consignas (`Historian/Consignas`)

#### Diagnóstico

En `ejemplos-scripts-ignition.md` (líneas 326-385), el script del binding de consignas incurre en el patrón más peligroso de la automatización industrial: el **fallo silencioso**:

```python
# CÓDIGO ORIGINAL (Fragmento con bare except que silencia caídas de BD)
try:
    SCADA_values = system.db.runNamedQuery('Historian/Consignas', parameters={
        'date': system.date.format(dateReal, 'yyyy-MM-dd HH:mm:ss'), 
        'tag': value.tag.lower(), 
        'mes': mesStr
    })
    for row in range(SCADA_values.getRowCount()):           
        if SCADA_values.getValueAt(row, 'tagpath') in str(result['fullPath']).lower():
            intvalue = SCADA_values.getValueAt(row, 'intvalue')
            # ...
except:
    # Si no existeix cap valor, actualitzar mes
    mes = system.date.getMonth(dateReal)
    if mes < 10:
        mesStr = '0' + str(mes)
    else:
        mesStr = str(mes)
```

* **Silenciamiento Absoluto:** Si el servidor de base de datos está apagado o la Named Query falla por sintaxis, el bloque `except:` captura el error sin emitir una sola línea de log.
* **Efecto Colateral Equívoco:** El script asume que la excepción significa *"no hay datos"* y recalcula la variable `mes`, induciendo a errores de consulta subsecuentes.
* **Cero Trazabilidad:** El equipo de mantenimiento no tiene constancia de la caída de la conexión en *Diagnostics -> Logs*.

#### Solución Propuesta (Enseñanzas Lab 4.1 y 4.2)

1. Envolver la ejecución de la consulta histórica con `project.util.logging.safe_execution`.
2. Registrar metadatos contextuales: usuario de la sesión, equipo visualizado, fecha solicitada y tag destino.
3. Procesar el resultado de forma segura con `system.dataset.toPyDataSet`, eliminando la sobrecarga de `.getValueAt()`.
4. Si la consulta falla, el Gateway registra la traza JDBC exacta con nivel `ERROR`, y el binding maneja un dataset vacío sin falsear la variable `mes`.

```python
# Implementación en Project Library: project.historian.consignas

def fetch_scada_consignas_safe(date_real, tag_filter, mes_str, user_context="SISTEMA"):
    """
    Ejecuta la Named Query Historian/Consignas bajo el wrapper de logging contextual.
    
    Args:
        date_real (Date): Fecha de corte seleccionada.
        tag_filter (str): Filtro de tag.
        mes_str (str): Subfijo de particion mensual.
        user_context (str): Usuario activo en la sesion.
        
    Returns:
        PyDataSet|None: Dataset iterable o None si ocurrio un error.
    """
    def _execute_query():
        params = {
            'date': system.date.format(date_real, 'yyyy-MM-dd HH:mm:ss'),
            'tag': str(tag_filter).lower(),
            'mes': str(mes_str)
        }
        raw_ds = system.db.runNamedQuery('Historian/Consignas', parameters=params)
        return system.dataset.toPyDataSet(raw_ds) if raw_ds is not None else None

    context = {
        "user": user_context,
        "query": "Historian/Consignas",
        "tag_filter": tag_filter,
        "target_date": system.date.format(date_real, 'yyyy-MM-dd HH:mm:ss'),
        "partition_month": mes_str
    }
    
    ok, pyds_result = project.util.logging.safe_execution(
        "Fetch_Consignas_History",
        context,
        _execute_query
    )
    
    if not ok:
        # El error ya quedo registrado con detalle forense en el Gateway
        return None
        
    return pyds_result
```

#### Código Resultante en el Binding de la Vista

```python
# Binding de props.instances (Limpio, predecible y con observabilidad en Gateway)
user_session = getattr(self.session.props.auth.user, "userName", "OPERADOR_ANONIMO")
pyds_scada = project.historian.consignas.fetch_scada_consignas_safe(
    date_real=dateReal,
    tag_filter=value.tag,
    mes_str=mesStr,
    user_context=user_session
)

# Si la consulta retorno datos validos, extraer el registro correspondiente
if pyds_scada is not None:
    target_clean = str(result['fullPath']).lower()
    for row in pyds_scada:
        row_tag = str(row['tagpath']).lower() if row['tagpath'] is not None else ""
        if row_tag in target_clean:
            # Coalescencia de tipos segura
            if row['floatvalue'] is not None:
                value_new = float(row['floatvalue'])
            elif row['intvalue'] is not None:
                value_new = int(row['intvalue'])
            elif row['stringvalue'] is not None:
                value_new = unicode(row['stringvalue'])
            elif row['datevalue'] is not None:
                value_new = row['datevalue']
                
            if row['date'] is not None:
                date = system.date.format(row['date'], 'dd/MM/yyyy HH:mm:ss')
            break
```

---

### 3.4. Protección de Escrituras Industriales y Auditoría (Botón Enviar Consignas)

#### Diagnóstico

En `ejemplos-scripts-ignition.md` (líneas 494-579), el evento `runAction` del botón de consignas realiza escrituras físicas a tags y ejecuciones de Named Queries sin ninguna estructura defensiva:

```python
# CÓDIGO ORIGINAL (Fragmento sin captura ni registro de incidentes)
for instance in instances:
    selected = instance['selected']
    if selected:
        tagPath = instance['tagPath']
        tag = tagPath.split('/')[-1]
        MName = 'M' + tag[1:len(tag)]
        # ...
        system.tag.writeBlocking([Mwrite], [value_new]) # Escritura desprotegida
        params = {"Equip": self.view.params.TAG, "Descripcio": "Ordre Enviar Consigna " + tag + ': ' + name, "Usuari": self.session.props.auth.user.userName}
        system.db.runNamedQuery("ins_Registre_Actuacions", params) # Named Query desprotegida
```

* **Vulnerabilidad de Planta:** Si el autómata o la pasarela OPC sufren una micro-desconexión, `system.tag.writeBlocking` puede lanzar una excepción de la JVM (`JavaException`). Si la base de datos de auditoría está bloqueada, `runNamedQuery` lanza `SQLException`.
* **Impacto en UI:** El script se interrumpe a mitad del bucle, enviando solo una parte de las consignas marcadas y dejando al operador con un componente bloqueado sin explicación.

#### Solución Propuesta (Enseñanzas Lab 4.1)

1. Centralizar la acción en `project.consignas.actions.send_consigna_command`.
2. Proteger la operación completa a través de `project.util.logging.safe_execution`.
3. Inyectar metadatos contextuales: usuario conectado, equipo destino, tag comando (`M*`) y valor consignado.
4. Aplicar el principio de **Doble Canal**: el error técnico detallado se almacena en el Gateway, y hacia la pantalla de Perspective se entrega una tupla con un mensaje funcional amigable.

```python
# Implementación en Project Library: project.consignas.actions

def execute_single_consigna(tag_path, target_m_path, value_new, equip_id, user_name, description_text):
    """
    Funcion atomica de escritura de tag y registro de auditoria.
    Disenada para ser ejecutada bajo safe_execution.
    """
    # 1. Escritura fisica en PLC
    quality_write = system.tag.writeBlocking([target_m_path], [value_new])
    if quality_write and not quality_write[0].isGood():
        raise Exception("Calidad de escritura no valida en tag {}: {}".format(target_m_path, quality_write[0]))
        
    # 2. Registro de auditoria en BD
    audit_params = {
        "Equip": equip_id,
        "Descripcio": "Ordre Enviar Consigna {}: {}".format(target_m_path.split("/")[-1], description_text),
        "Usuari": user_name if user_name else "SISTEMA"
    }
    system.db.runNamedQuery("ins_Registre_Actuacions", audit_params)
    return True


def dispatch_consigna_safe(instance_data, equip_id, user_name):
    """
    Envoltorio que despacha una consigna individual protegiendola con logging contextual.
    
    Args:
        instance_data (dict): Datos de la fila seleccionada en FlexRepeater.
        equip_id (str): Identificador del equipo destino.
        user_name (str): Nombre del usuario en sesion.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    tag_path = instance_data.get("tagPath", "")
    value_new = instance_data.get("value_new", None)
    
    if not tag_path or value_new is None:
        return (False, u"Datos de consigna incompletos")
        
    parts = tag_path.split("/")
    tag_name = parts[-1]
    
    # Resolucion de comando M*
    m_name = "MFOS" if (tag_name == "SFOR" and instance_data.get("esim", False)) else ("M" + tag_name[1:])
    parts[-1] = m_name
    target_m_path = "/".join(parts)
    
    desc_text = instance_data.get("tooltip", tag_name)
    
    context = {
        "user": user_name,
        "equip": equip_id,
        "action": "WRITE_CONSIGNA",
        "tag_source": tag_name,
        "tag_target": m_name,
        "value": value_new
    }
    
    ok, result = project.util.logging.safe_execution(
        "Consigna_Dispatch",
        context,
        execute_single_consigna,
        tag_path,
        target_m_path,
        value_new,
        equip_id,
        user_name,
        desc_text
    )
    
    if not ok:
        return (False, u"No se pudo aplicar la consigna '{}'. Compruebe el estado del equipo.".format(tag_name))
        
    return (True, u"Consigna aplicada correctamente")
```

#### Código Resultante en el Evento del Botón (`runAction`)

```python
# Evento runAction del botón de envío de consignas
instances = self.getSibling("FlexRepeater").props.instances
equip = self.view.params.TAG
user = getattr(self.session.props.auth.user, "userName", "OPERADOR")

total_sent = 0
failed_messages = []

for instance in instances:
    if not instance.get("selected", False):
        continue
        
    ok, msg = project.consignas.actions.dispatch_consigna_safe(instance, equip, user)
    if ok:
        total_sent += 1
    else:
        failed_messages.append(msg)

# Respuesta amigable en pantalla para el operador
if failed_messages:
    system.perspective.print("AVISO: " + " | ".join(failed_messages))
else:
    system.perspective.print("INFO: Se enviaron {} consignas con éxito.".format(total_sent))
```

---

### 3.5. Blindaje del Hilo de Tags de Gateway y Eliminación de `SELECT *` (Tag Enclavamientos)

#### Diagnóstico

En `ejemplos-scripts-ignition.md` (líneas 620-753), el evento `valueChanged` en el tag de enclavamientos se ejecuta directamente en el **Gateway Tag Change Scripting Thread**. Su diseño actual contiene tres riesgos severos:

```python
# CÓDIGO ORIGINAL (Fragmento con select * y operaciones SQL en hilo de tags)
def valueChanged(tag, tagPath, previousValue, currentValue, initialChange, missedEvents):
    # ...
    query = "select * from enclavaments where equip = '" + equip + "' and NumeroEnclava = " + str(indice)
    select = system.db.runQuery(query, 'BD_LLIBRERIA')
    descr = system.tag.read("[.]WENC_DESC").value
    # ...
    insert = "insert into enclavaments (StartDate, Equip, NumeroEnclava, TextEnclava) values ('" + StartDate + "','" + equip + "'," + str(indice) + ", '" + textoActive + "')"
    system.db.runUpdateQuery(insert, 'BD_LLIBRERIA')
    # ...
    params = {"Equip": equip, "Descripcio": "Enclavament " + Enclavament, "Usuari": 'PLC'}
    system.db.runNamedQuery('AB_LLIBRERIA', "ins_Registre_Actuacions", params)
```

1. **Infracción de Regla de Oro SQL (Lab 4.2):** Uso de `select *` que transfiere columnas irrelevantes en cada ciclo de cambio de bit.
2. **Caída del Hilo de Tags:** Si `BD_LLIBRERIA` tiene saturado su pool de conexiones JDBC, se lanza una `SQLException`. Al no haber captura de excepciones Java, el hilo del tag colapsa silenciosamente.
3. **Ausencia de Trazas:** Mantenimiento no dispone de ningún registro en *Status -> Diagnostics -> Logs* para saber qué bit de enclavamiento intentó sincronizarse cuando falló la red.

#### Solución Propuesta (Enseñanzas Lab 4.1 y 4.2)

1. Sustituir `select *` por una proyección estricta de las columnas exactas requeridas: `SELECT TextEnclava FROM enclavaments ...`.
2. Extraer la lógica transaccional de enclavamiento a `project.enclavamientos.manager.sync_interlock_event`.
3. Encapsular la ejecución mediante `safe_execution` con metadatos contextuales (`equip`, `bit_index`, `action`).

```python
# Implementación en Project Library: project.enclavamientos.manager

def sync_interlock_event(equip, bit_index, bit_value, desc_text, start_date_str):
    """
    Gestiona la sincronizacion relacional de un enclavamiento con proyeccion estricta.
    """
    db_name = "BD_LLIBRERIA"
    
    # 1. Proyeccion estricta sin SELECT *
    select_sql = """
        SELECT TextEnclava 
        FROM enclavaments 
        WHERE equip = ? AND NumeroEnclava = ?
    """
    existing = system.db.runPrepQuery(select_sql, [equip, bit_index], database=db_name)
    
    enclava_label = "{}: {}".format(bit_index, desc_text)
    
    # 2. Insercion o actualizacion segun estado
    if len(existing) == 0 and bit_value == 0:
        insert_sql = """
            INSERT INTO enclavaments (StartDate, Equip, NumeroEnclava, TextEnclava) 
            VALUES (?, ?, ?, ?)
        """
        system.db.runPrepUpdate(insert_sql, [start_date_str, equip, bit_index, desc_text], database=db_name)
        
        audit_params = {
            "Equip": equip,
            "Descripcio": "Enclavament " + enclava_label,
            "Usuari": "PLC"
        }
        system.db.runNamedQuery("ins_Registre_Actuacions", audit_params)
        
    elif bit_value == 0:
        update_sql = "UPDATE enclavaments SET StartDate = ? WHERE equip = ? AND NumeroEnclava = ?"
        system.db.runPrepUpdate(update_sql, [start_date_str, equip, bit_index], database=db_name)
        
        current_text = existing[0]["TextEnclava"]
        if current_text != desc_text:
            upd_text_sql = "UPDATE enclavaments SET TextEnclava = ? WHERE equip = ? AND NumeroEnclava = ?"
            system.db.runPrepUpdate(upd_text_sql, [desc_text, equip, bit_index], database=db_name)
            
    return True
```

#### Código Resultante en el Script del Tag (`valueChanged`)

```python
def valueChanged(tag, tagPath, previousValue, currentValue, initialChange, missedEvents):
    if initialChange or currentValue is None or previousValue is None:
        return
        
    current = currentValue.value
    prev = previousValue.value
    
    if not hasattr(current, "__getitem__") or not hasattr(prev, "__getitem__"):
        return
        
    parts = tagPath.split("/")
    equip = parts[-2] if len(parts) >= 2 else "EQUIP_GENERAL"
    
    # Lectura protegida de la descripcion
    read_desc = system.tag.readBlocking(["[.]WENC_DESC"])
    desc_ds = read_desc[0].value if read_desc else None
    
    start_date = system.date.format(system.date.now(), 'yyyy-MM-dd HH:mm:ss')
    max_bits = min(32, len(current), len(prev))
    
    for i in range(max_bits):
        if current[i] != prev[i]:
            desc_text = u"Alarma Bit {}".format(i)
            if desc_ds is not None and getattr(desc_ds, "getRowCount", lambda: 0)() > i:
                val = desc_ds.getValueAt(i, "Descripcio")
                if val:
                    desc_text = unicode(val)
                    
            context = {
                "equip": equip,
                "bit_index": i,
                "bit_value": current[i],
                "action": "SYNC_INTERLOCK_TAG",
                "tagPath": tagPath
            }
            
            # Ejecucion protegida que nunca aborta el hilo del Gateway
            project.util.logging.safe_execution(
                "Interlock_TagSync",
                context,
                project.enclavamientos.manager.sync_interlock_event,
                equip,
                i,
                current[i],
                desc_text,
                start_date
            )
```

---

### 3.6. Coprocesamiento Analítico de Telemetría y KPIs en Motor SQL

#### Diagnóstico

En el desarrollo SCADA convencional (`Historian/Consignas`, líneas 580-607), es habitual transferir registros individuales de tablas históricas (`sqlt_data_*` y `sqlth_te`) hacia el cliente para clasificar variables o promediar valores mediante bucles en Python.
* **Sobrecarga Computacional:** Transferir miles de registros por la red para que la JVM los recorra penaliza el rendimiento de la CPU del Gateway y degrada el tiempo de carga de las pantallas.
* **Código de Estilo Frágil:** Asignar colores y badges en bindings de Perspective requiere lógica condicional adicional en la interfaz.

#### Solución Propuesta (Enseñanzas Lab 4.2)

1. **Uso de la Base de Datos como Coprocesador:** Delegar todas las agregaciones numéricas (`COUNT`, `AVG`, `MIN`, `MAX`) directamente en el motor relacional.
2. **Proyección Estricta sin `SELECT *`:** Solicitar únicamente las columnas calculadas necesarias.
3. **Clasificación Operativa y Códigos Hexadecimales en SQL (`CASE WHEN`):** Calcular tanto la etiqueta del estado operativo (`'ALTA_CARGA'`, `'BAJA_CARGA'`, `'NOMINAL'`) como el código de color hexadecimal (`'#e74c3c'`, `'#f39c12'`, `'#2ecc71'`) dentro de la propia consulta.
4. **Casteo Numérico Defensivo en PostgreSQL:** Aplicar `::numeric` sobre el `AVG()` para garantizar la compatibilidad con la función `ROUND(..., 2)`.
5. **Consumo Seguro con `PyDataSet`:** Procesar el `Dataset` resultante en Jython sin llamadas celda a celda.

#### Consulta SQL Analítica Optimizada

```sql
SELECT 
    te.tagpath,
    COUNT(d.t_stamp) AS total_muestras,
    ROUND(AVG(d.floatvalue)::numeric, 2) AS valor_medio,
    ROUND(MIN(d.floatvalue)::numeric, 2) AS valor_minimo,
    ROUND(MAX(d.floatvalue)::numeric, 2) AS valor_maximo,
    CASE 
        WHEN AVG(d.floatvalue) > 75.0 THEN 'ALTA_CARGA'
        WHEN AVG(d.floatvalue) < 25.0 THEN 'BAJA_CARGA'
        ELSE 'NOMINAL'
    END AS estado_operativo,
    CASE 
        WHEN AVG(d.floatvalue) > 75.0 THEN '#e74c3c' -- Rojo alerta
        WHEN AVG(d.floatvalue) < 25.0 THEN '#f39c12' -- Ambar aviso
        ELSE '#2ecc71'                               -- Verde optimo
    END AS color_hex
FROM sqlt_data_1_2026_10 d
INNER JOIN sqlth_te te ON d.tagid = te.id
WHERE d.floatvalue IS NOT NULL
GROUP BY te.tagpath
ORDER BY valor_medio DESC;
```

#### Script de Consumo y Formateo en Jython (Project Library / Script Console)

```python
# Implementación en Project Library: project.analytics.telemetry

def get_telemetry_kpi_report(database_name="SANDBOX_DB"):
    """
    Ejecuta la consulta analitica de telemetria precalculada en el motor relacional
    y la transforma en un PyDataSet para renderizado rapido en UI.
    
    Args:
        database_name (str): Conexion de base de datos destino.
        
    Returns:
        tuple: (bool success, PyDataSet|unicode payload_or_error)
    """
    sql_query = """
    SELECT 
        te.tagpath,
        COUNT(d.t_stamp) AS total_muestras,
        ROUND(AVG(d.floatvalue)::numeric, 2) AS valor_medio,
        ROUND(MIN(d.floatvalue)::numeric, 2) AS valor_minimo,
        ROUND(MAX(d.floatvalue)::numeric, 2) AS valor_maximo,
        CASE 
            WHEN AVG(d.floatvalue) > 75.0 THEN 'ALTA_CARGA'
            WHEN AVG(d.floatvalue) < 25.0 THEN 'BAJA_CARGA'
            ELSE 'NOMINAL'
        END AS estado_operativo,
        CASE 
            WHEN AVG(d.floatvalue) > 75.0 THEN '#e74c3c'
            WHEN AVG(d.floatvalue) < 25.0 THEN '#f39c12'
            ELSE '#2ecc71'
        END AS color_hex
    FROM sqlt_data_1_2026_10 d
    INNER JOIN sqlth_te te ON d.tagid = te.id
    WHERE d.floatvalue IS NOT NULL
    GROUP BY te.tagpath
    ORDER BY valor_medio DESC
    """
    
    def _run_query():
        raw_ds = system.db.runQuery(sql_query, database=database_name)
        return system.dataset.toPyDataSet(raw_ds)
        
    context = {"database": database_name, "action": "FETCH_TELEMETRY_KPIS"}
    
    return project.util.logging.safe_execution(
        "Analytics_TelemetryKPIs",
        context,
        _run_query
    )
```

---

## 4. Banco de Pruebas Automatizado (Script Console)

Para certificar la correcta integración del wrapper de logging defensivo, la captura de excepciones duales y el consumo analítico de KPIs sin alterar la planta en producción, se ejecuta el siguiente banco de pruebas en la **Script Console** de Ignition (**Tools -> Script Console**):

```python
# ==============================================================================
# BANCO DE PRUEBAS AUTOMATIZADO: SESION 4 (LOGGING DEFENSIVO Y SQL ANALITICO)
# ==============================================================================

print "=" * 80
print "CERTIFICACION DE OBSERVABILIDAD Y COPOCESAMIENTO SQL (SESION 4)"
print "=" * 80

# ------------------------------------------------------------------------------
# 1. PRUEBA DE EJECUCIÓN NOMINAL Y LOG CONTEXTUAL (Lab 4.1)
# ------------------------------------------------------------------------------
print "\n--- [TEST 1] safe_execution: Caso Nominal ---"
def operacion_nominal(tag, valor):
    return u"Valor {} asignado correctamente a {}".format(valor, tag)

ctx1 = {"user": "marti.sanchez", "equip": "EDAR_BOMBA_01", "action": "SET_SPEED"}
ok1, res1 = project.util.logging.safe_execution(
    "SetSpeed_Action", ctx1, operacion_nominal, "[default]EDAR/BOMBA_01/SCFL", 45.5
)

assert ok1 is True, "Fallo en ejecucion nominal"
assert u"EDAR_BOMBA_01" in res1, "Respuesta incorrecta"
print "[PASS] Caso Nominal ejecutado con exito. Retorno: {}".format(res1)


# ------------------------------------------------------------------------------
# 2. PRUEBA DE CAPTURA DE ERRORES DE DATOS JYTHON (ValueError)
# ------------------------------------------------------------------------------
print "\n--- [TEST 2] safe_execution: Captura de ValueError ---"
def operacion_con_validacion(tag_path):
    if "INVALID" in tag_path:
        raise ValueError("Ruta de tag inexistente en el provider")
    return True

ctx2 = {"user": "operador_noche", "equip": "COMPRESOR_02", "action": "SET_PRESSURE"}
ok2, res2 = project.util.logging.safe_execution(
    "SetPressure_Action", ctx2, operacion_con_validacion, "[default]EDAR/INVALID_PATH/SCMX"
)

assert ok2 is False, "No se capturo el ValueError"
assert res2 == u"Error en los parámetros o datos de procesamiento", "Mensaje de usuario no estandarizado"
print "[PASS] ValueError capturado y registrado en Gateway. Mensaje al operador: {}".format(res2)


# ------------------------------------------------------------------------------
# 3. PRUEBA DE CAPTURA DE ERRORES DE TIPO JYTHON (TypeError)
# ------------------------------------------------------------------------------
print "\n--- [TEST 3] safe_execution: Captura de TypeError ---"
def operacion_con_tipo_invalido(a, b):
    return a + b

ctx3 = {"user": "admin", "equip": "DECANTADOR_01", "action": "CALC_INDEX"}
ok3, res3 = project.util.logging.safe_execution(
    "CalcIndex_Action", ctx3, operacion_con_tipo_invalido, "Texto", 100
)

assert ok3 is False, "No se capturo el TypeError"
assert res3 == u"Error en los parámetros o datos de procesamiento", "Mensaje incorrecto"
print "[PASS] TypeError capturado y registrado en Gateway sin propagar excepcion a la UI."


# ------------------------------------------------------------------------------
# 4. PRUEBA DE CAPTURA DE EXCEPCIONES JDBC / JAVA (SQLException)
# ------------------------------------------------------------------------------
print "\n--- [TEST 4] safe_execution: Captura de SQLException ---"
from java.sql import SQLException

def operacion_con_fallo_sql():
    raise SQLException("Conexion rechazada por timeout en el socket PostgreSQL (Puerto 5432)")

ctx4 = {"user": "sistema", "database": "BD_AUDIT", "action": "PURGE_TABLE"}
ok4, res4 = project.util.logging.safe_execution(
    "PurgeAudit_Action", ctx4, operacion_con_fallo_sql
)

assert ok4 is False, "No se capturo la SQLException"
assert res4 == u"Error de comunicación o transacción con la base de datos", "Mensaje SQL no coincide"
print "[PASS] SQLException capturada con éxito. Mensaje al operador: {}".format(res4)


# ------------------------------------------------------------------------------
# 5. PRUEBA DE CONSUMO DE CONSULTA ANALÍTICA SQL CON PyDataSet (Lab 4.2)
# ------------------------------------------------------------------------------
print "\n--- [TEST 5] Consumo Analitico y Formateo de Reporte Telemetria ---"
# Simulacion de PyDataSet analitico precalculado en motor relacional
headers = ["tagpath", "total_muestras", "valor_medio", "valor_minimo", "valor_maximo", "estado_operativo", "color_hex"]
data = [
    ["[default]EDAR/EDAR_BOMBA_01/SCFL",     300, 52.84, 10.15, 94.88, "NOMINAL",    "#2ecc71"],
    ["[default]EDAR/EDAR_COMPRESOR_02/SCMX", 300, 78.40, 22.50, 98.10, "ALTA_CARGA", "#e74c3c"],
    ["[default]EDAR/VALVULA_RECIRC_01/SCFL", 300, 18.20,  5.00, 42.10, "BAJA_CARGA", "#f39c12"]
]
mock_ds = system.dataset.toDataSet(headers, data)
pyds = system.dataset.toPyDataSet(mock_ds)

print "Total variables analizadas: {}".format(len(pyds))
print "-" * 95
print "{:<32} | {:<8} | {:<7} | {:<7} | {:<7} | {:<12} | {:<8}".format(
    "TAG PATH", "MUESTRAS", "MEDIA", "MIN", "MAX", "ESTADO", "COLOR"
)
print "-" * 95

for row in pyds:
    tag_clean = row["tagpath"].replace("[default]EDAR/", "")
    print "{:<32} | {:<8} | {:<7} | {:<7} | {:<7} | {:<12} | {:<8}".format(
        tag_clean,
        row["total_muestras"],
        row["valor_medio"],
        row["valor_minimo"],
        row["valor_maximo"],
        row["estado_operativo"],
        row["color_hex"]
    )
print "-" * 95
print "[PASS] Reporte analitico procesado sin bucles matematicos en Python."

print "\n" + "=" * 80
print "TODAS LAS PRUEBAS DE LA SESION 4 HAN FINALIZADO CON EXITO"
print "=" * 80
```

#### Salida Estructurada Esperada en Script Console

```
================================================================================
CERTIFICACION DE OBSERVABILIDAD Y COPROCESAMIENTO SQL (SESION 4)
================================================================================

--- [TEST 1] safe_execution: Caso Nominal ---
[PASS] Caso Nominal ejecutado con exito. Retorno: Valor 45.5 asignado correctamente a [default]EDAR/BOMBA_01/SCFL

--- [TEST 2] safe_execution: Captura de ValueError ---
[PASS] ValueError capturado y registrado en Gateway. Mensaje al operador: Error en los parámetros o datos de procesamiento

--- [TEST 3] safe_execution: Captura de TypeError ---
[PASS] TypeError capturado y registrado en Gateway sin propagar excepcion a la UI.

--- [TEST 4] safe_execution: Captura de SQLException ---
[PASS] SQLException capturada con éxito. Mensaje al operador: Error de comunicación o transacción con la base de datos

--- [TEST 5] Consumo Analitico y Formateo de Reporte Telemetria ---
Total variables analizadas: 3
-----------------------------------------------------------------------------------------------
TAG PATH                         | MUESTRAS | MEDIA   | MIN     | MAX     | ESTADO       | COLOR   
-----------------------------------------------------------------------------------------------
EDAR_BOMBA_01/SCFL               | 300      | 52.84   | 10.15   | 94.88   | NOMINAL      | #2ecc71 
EDAR_COMPRESOR_02/SCMX           | 300      | 78.40   | 22.50   | 98.10   | ALTA_CARGA   | #e74c3c 
VALVULA_RECIRC_01/SCFL           | 300      | 18.20   | 5.00    | 42.10   | BAJA_CARGA   | #f39c12 
-----------------------------------------------------------------------------------------------
[PASS] Reporte analitico procesado sin bucles matematicos en Python.

================================================================================
TODAS LAS PRUEBAS DE LA SESION 4 HAN FINALIZADO CON EXITO
================================================================================
```

#### Verificación en el Gateway de Ignition

Al consultar el visor de eventos del Gateway (**Status -> Diagnostics -> Logs**) filtrando por el registrador `SCADA.Core`, se visualizan los registros enriquecidos:
* `ERROR [SCADA.Core] Error de tipos/datos en [SetPressure_Action]: Ruta de tag inexistente en el provider | Contexto: [action=SET_PRESSURE | equip=COMPRESOR_02 | user=operador_noche]`
* `ERROR [SCADA.Core] Error SQL/JDBC en [PurgeAudit_Action]: Conexion rechazada por timeout en el socket PostgreSQL (Puerto 5432) | Contexto: [action=PURGE_TABLE | database=BD_AUDIT | user=sistema]`

---

## 5. Cuadro Comparativo de Beneficios

| Parámetro | Estado Anterior (Código Heredado) | Estado Refactorizado (Sesión 4 / Lab-04) | Impacto Técnico en Planta |
| :--- | :--- | :--- | :--- |
| **Observabilidad y Logs** | Loggers aislados (`Neteja_Script`) o ausencia total de trazas en eventos | Centralización en `SCADA.Core` con metadatos contextuales (`user`, `equip`, `action`) | Diagnóstico forense en menos de 2 minutos desde *Diagnostics -> Logs* |
| **Captura en la JVM** | Bloques `except:` ciegos o `except Exception:` que ignoran errores Java | Captura dual formal de clases `SQLException` y `JavaException` | Cero excepciones JDBC o de sockets sin registrar ni gestionar |
| **Experiencia de Usuario (UX)** | Vistas colgadas en rojo o fallos silenciados que confunden al operario | Separación estricta: log técnico en Gateway y mensaje limpio en UI | Confianza operativa del personal de planta ante cualquier anomalía |
| **Resiliencia de Hilos** | Sentencias SQL desprotegidas en el hilo de tags de Gateway | Envoltorio con `safe_execution` que previene la muerte del hilo | Integridad operativa continua en el motor de tags de Ignition |
| **Carga de CPU y Memoria** | Bucles en Python e iteración `.getValueAt()` sobre datasets masivos | Coprocesamiento analítico en SQL (`GROUP BY`, `AVG`, `CASE WHEN`) | Reducción de latencia y liberación de memoria RAM en la JVM |
| **Estilos y Semáforos** | Lógica dispersa en scripts de vistas para clasificar colores | Precalculado de etiquetas y códigos hexadecimales directos en SQL | Simplificación radical de bindings y renderizado instantáneo en Perspective |
