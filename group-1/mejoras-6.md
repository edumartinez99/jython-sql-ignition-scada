---
description: Informe técnico de auditoría, transacciones atómicas ACID, Script Transforms puros en Perspective y desacoplamiento mediante Message Handlers (Sesión 6).
---

# Informe Técnico: Refactorización y Estandarización de Scripts SCADA (Fase 6: Transacciones ACID, Script Transforms Puros y Desacoplamiento por Message Handlers)

## 1. Resumen Ejecutivo

Este informe documenta la auditoría técnica y la propuesta de refactorización integral de los scripts del sistema SCADA en Ignition (`ejemplos-scripts-ignition.md`), fundamentada estrictamente en las directrices de integridad transaccional bajo estándar ACID, funciones puras en memoria para *Script Transforms* en Perspective y arquitectura orientada a eventos desacoplada mediante *Message Handlers* desarrolladas en los laboratorios de la **Sesión 6** (`gcp-con-eduardo/lab-06.md`).

### Principales Deficiencias Detectadas

1. **Operaciones Multi-Paso sin Control Transaccional ACID (*Auto-Commit* Fraccionado):** En componentes como el script de eventos de tag de enclavamientos (`valueChanged`, páginas 14 y 15 del documento técnico) y la subida de documentos (`uploadFile`), se ejecutan de forma secuencial e independiente múltiples llamadas a `runUpdateQuery` y `runNamedQuery`. Cada una opera con *auto-commit* individual. Si ocurre una caída de red, fallo del servidor o error en la segunda o tercera consulta, las modificaciones previas quedan grabadas permanentemente, corrompiendo la coherencia de los datos en la base de datos industrial (por ejemplo, enclavamientos creados sin auditoría o cabeceras de documentos registradas sin su contenido binario).
2. **Sobrecarga Bloqueante de I/O en Bindings Visuales de Perspective:** En los bindings de instancias para *Flex Container* (estados y consignas, páginas 2 a 8 del documento técnico), se incrustan decenas de líneas de código que ejecutan llamadas síncronas bloqueantes a la base de datos (`system.tag.queryTagHistory`, `system.db.runNamedQuery`), lecturas repetitivas de tags (`system.tag.readBlocking` dentro de bucles `for`) y recorridos celda por celda (`getValueAt`). Estas operaciones I/O en el hilo de la vista congelan la sesión del operador y degradan la tasa de refresco del cliente web. Además, los estilos se construyen vacíos (`"classes": ""`) sin badges normalizados de estado ni gestión defensiva ante desconexiones.
3. **Acoplamiento Rígido y Frágil por Navegación de Jerarquías (`self.parent.parent.parent`):** En el binding de consignas (página 8 del documento técnico, línea 488) se utiliza el anti-patrón de navegación relativa rígida: `self.parent.parent.parent.getChild("TabContainer").refreshBinding('props.tabs')`. Si un desarrollador añade un contenedor intermedio, reubica el componente o modifica la jerarquía visual de la vista, la instrucción falla inmediatamente con `AttributeError` en tiempo de ejecución.
4. **Fuga Crítica de Conexiones JDBC por Omisión de Cierre Incondicional:** En los scripts que interactúan con bases de datos relacionales no se utiliza un patrón formal de transacciones manuales (`beginTransaction`, `commitTransaction`, `rollbackTransaction`) protegido con `closeTransaction` dentro de un bloque `finally`. Ante errores o excepciones no controladas, las conexiones en el pool JDBC del Gateway quedan abiertas y bloqueadas, provocando saturación progresiva (*pool starvation*) y denegación de servicio en el Gateway de Ignition.
5. **Incrustación de Lógica de Negocio y Auditoría Dispersa en Componentes UI:** En eventos de interfaz (como el botón de envío de consignas `runAction` o el árbol de navegación), se acumulan más de 100 líneas de código ejecutando lectura de tags, formateo de nombres, escrituras directas y auditoría ad-hoc, imposibilitando la reutilización, el testeo unitario y la reactividad sincronizada entre componentes hermanos.

### Arquitectura Objetivo

* **Transacciones Atómicas ACID con Rollback Automático (`project.production.batch` y `project.enclavamientos.service`):** Agrupación de todas las operaciones DML dependientes bajo una única transacción JDBC explícita mediante `system.db.beginTransaction(database=db_conn, timeout=10000)`. Confirmación atómica mediante `commitTransaction(tx_id)` únicamente si todas las sentencias se completan con éxito y validación de filas (`rows_updated == 1`), reversión inmediata (`rollbackTransaction(tx_id)`) ante cualquier excepción, y liberación incondicional obligatoria de la conexión al pool físico mediante `closeTransaction(tx_id)` en el bloque `finally`.
* **Script Transforms Defensivos en Memoria con Cero I/O (`project.ui.transforms`):** Extracción de toda la lógica de presentación visual a funciones puras en la `Project Library`. Eliminación total de consultas SQL y lecturas de tags dentro del *transform*. Procesamiento ultrarrápido en memoria RAM (sub-milisegundo) que genera la estructura completa de visualización (texto descriptivo, color hexadecimal, icono, clase CSS de badge y animación de parpadeo), con tolerancia absoluta a valores nulos o estados de desconexión.
* **Arquitectura de Eventos Desacoplada mediante Message Handlers (`system.perspective.sendMessage`):** Erradicación total de referencias relativas frágiles (`self.parent.parent.parent`). Implementación del patrón Publicador-Suscriptor en Perspective, donde los emisores (botones, popups, servicios) procesan la lógica de negocio y publican eventos al bus con `system.perspective.sendMessage` (scopes `page` o `session`), y los receptores (tablas, contenedores de pestañas, tarjetas KPI) configuran *Message Handlers* dedicados para refrescar selectivamente sus propiedades (`self.refreshBinding`) de manera totalmente independiente de la jerarquía visual.
* **Separación de Capas y Contratos Homogéneos:** Delegación de la lógica de negocio y auditoría en módulos especializados de `Project Library`, manteniendo los eventos de componentes reducidos a llamadas limpias de 3 a 5 líneas y retorno canónico estructurado `(bool success, unicode message / dict payload)`.

---

## 2. Matriz de Hallazgos y Acciones Correctivas

| Componente Auditado | Deficiencia Detectada (Código Original) | Riesgo Operativo | Solución Técnica Aplicada (Lab-06) |
| :--- | :--- | :--- | :--- |
| **Tag Enclavamientos** (`valueChanged`, líneas 620-753) | Ejecución secuencial de `runUpdateQuery` e `ins_Registre_Actuacions` sin transacción unificada (*auto-commit* independiente). | Inconsistencia de datos: si falla la auditoría o la segunda actualización, la base de datos queda corrupta y desincronizada con el autómata. | Agrupación transaccional con `beginTransaction`, `runPrepUpdate` compartiendo `txId`, `commitTransaction`, `rollbackTransaction` y `closeTransaction` en `finally`. |
| **Gestión Documental** (`uploadFile`, líneas 35-66) | Inserción de metadatos en tabla `Documents` y posterior inserción de contenido binario en `Document Contents` sin transacción. | Creación de registros huérfanos: si la subida binaria falla por tamaño o red, el registro de archivo queda creado pero sin contenido accesible. | Transacción ACID integral que asegura que el registro del documento y sus bytes binarios se confirmen juntos o se reviertan totalmente ante fallos. |
| **Cierre de Lote y Stock** (Gestión de Planta / Producción) | Actualizaciones de estado operativo y descuento de existencias sin validación de stock concurrente ni reversión atómica. | Descuadres críticos de inventario, stock negativo y lotes cerrados sin registro de consumo de materias primas. | Implementación del módulo `project.production.batch.close_production_batch` con cláusula defensiva `WHERE stock_qty >= ?`, rollback ante rotura de stock y auditoría sincronizada. |
| **Binding Flex Estados** (Instancias Perspective, líneas 101-231) | 5 ramas condicionales que iteran con `readBlocking` en bucle, estilos vacíos (`"classes": ""`) y sin manejo defensivo ante tags desconectados. | Congelamiento visual de la vista Perspective, falta de contraste cromático y ausencia de retroalimentación de fallo de comunicación. | Delegación en función pura `project.ui.transforms.format_equipment_status_card`, ejecución en RAM en sub-milisegundos con mapeo exhaustivo de colores, iconos y badges. |
| **Binding Flex Consignas** (Instancias Perspective, líneas 232-493) | Consulta histórica masiva (`queryTagHistory`) y Named Query en el binding, rematado con `self.parent.parent.parent.getChild("TabContainer").refreshBinding(...)`. | Bloqueo síncrono del hilo de UI y fallo catastrófico (`AttributeError`) en cuanto se modifica la estructura del contenedor en Designer. | Extracción de I/O fuera del binding, desacoplamiento del refresco mediante mensaje `system.perspective.sendMessage("OnConsignasUpdated", scope="page")` y Script Transform puro. |
| **Botón Enviar Consignas / Alarmas** (`runAction`, líneas 494-579) | Escritura directa al PLC, parámetros de auditoría manuales y manipulación directa de hermanos en la vista. | Acoplamiento rígido con la UI; si se cambia el nombre o ubicación de los componentes, la acción deja de funcionar. | Desacoplamiento mediante `project.alarms.events.acknowledge_equipment_alarm` / `dispatch_consignas`, persistencia previa y emisión de evento al bus de Perspective. |
| **Árbol de Navegación** (`runAction`, líneas 844-1051) | Más de 100 líneas dentro del componente gestionando pila de historial, formateo de cadenas y emisión de mensajes. | Imposibilidad de probar la lógica de navegación en aislamiento; riesgo de corrupción de la pila de historial en sesión. | Modularización de la pila de historial y formateo en librería, limitando el evento a la coordinación del servicio y broadcast del mensaje. |

---

## 3. Detalle de Refactorización por Componente

---

### 3.1. Tag de Enclavamientos: Transacción Atómica ACID para Inserción, Actualización y Auditoría

#### Diagnóstico

En el script de cambio de valor del tag de enclavamientos (`ejemplos-scripts-ignition.md`, líneas 658-753), se ejecutan múltiples sentencias SQL de forma secuencial y totalmente desprotegida:

```python
# CÓDIGO ORIGINAL (Fragmento con auto-commit fraccionado - SIN CONTROL TRANSACCIONAL)
# Inserción de nuevo enclavamiento y auditoría secuencial
if select.getRowCount() == 0 and current[i] == 0:
    textoActive = descr.getValueAt(indice, "Descripcio")
    insert = "insert into enclavaments (StartDate, Equip, NumeroEnclava, TextEnclava) values ('" + StartDate + "','" + equip + "'," + str(indice) + ", '" + textoActive + "')"
    system.db.runUpdateQuery(insert, 'BD_LLIBRERIA')
    
    Enclavament = str(indice) + ": " + textoActive
    params = {"Equip": equip , "Descripcio": "Enclavament " + Enclavament, "Usuari": 'PLC'}
    system.db.runNamedQuery('AB_LLIBRERIA', "ins_Registre_Actuacions", params)

elif current[i] == 0:
    update = "update enclavaments set StartDate='" + StartDate + "' where equip = '" + equip + "' and NumeroEnclava = " + str(indice)
    system.db.runUpdateQuery(update, 'BD_LLIBRERIA')
    ...
    update_text = "update enclavaments set TextEnclava='" + textoActive + "' where equip = '" + equip + "' and NumeroEnclava = " + str(indice)
    system.db.runUpdateQuery(update_text, 'BD_LLIBRERIA')
    ...
    system.db.runNamedQuery('AB_LLIBRERIA', "ins_Registre_Actuacions", params)
```

**Riesgos técnicos y operativos identificados:**
1. **Falta de atomicidad:** Cada llamada a `system.db.runUpdateQuery` o `system.db.runNamedQuery` se ejecuta con *auto-commit* individual. Si el servidor de base de datos pierde la conexión o se reinicia entre la primera y la segunda consulta, la tabla `enclavaments` se modifica pero la tabla `ins_Registre_Actuacions` queda sin registrar, vulnerando los requerimientos de trazabilidad y auditoría.
2. **Actualización fragmentada:** En la rama `elif current[i] == 0`, se ejecutan dos sentencias `UPDATE` consecutivas (`update` de fecha y `update_text` de descripción). Si la segunda falla, el registro queda parcialmente actualizado.
3. **Fuga potencial de recursos:** No existe bloque `try/except/finally` que garantice la gestión ordenada de excepciones en el hilo de ejecución de tags del Gateway.

#### Solución Propuesta (Enseñanzas Lab 6.1)

1. Centralizar la operación en `project.enclavamientos.service.sync_interlock_transactional`.
2. Abrir una transacción explícita mediante `tx_id = system.db.beginTransaction(database=db_conn, timeout=10000)`.
3. Ejecutar todas las sentencias DML (actualización/inserción de enclavamiento + inserción de auditoría) vinculadas estrictamente al mismo `txId=tx_id`.
4. Validar que las sentencias afecten exactamente al número de filas esperado (`rows == 1`). En caso contrario, disparar excepción para provocar la reversión.
5. Confirmar mediante `system.db.commitTransaction(tx_id)` solo si todas las operaciones concluyen con éxito.
6. En caso de error, ejecutar `system.db.rollbackTransaction(tx_id)` y registrar la incidencia en el logger.
7. **Regla de oro obligatoria:** Liberar incondicionalmente la conexión JDBC al pool mediante `system.db.closeTransaction(tx_id)` en el bloque `finally`.

```python
# Implementación en Project Library: project.enclavamientos.service
"""
Modulo: project.enclavamientos.service
Descripcion: Gestion transaccional atomica (ACID) de enclavamientos y auditoria.
             Implementado conforme a las directrices del Laboratorio 6.1.
Autor: Equipo SCADA
"""

def sync_interlock_transactional(equip, bit_index, description, is_active, db_conn="BD_LLIBRERIA"):
    """
    Ejecuta la actualizacion de estado del enclavamiento y su registro de auditoria
    bajo una unica transaccion JDBC atomica con garantia de Rollback y cierre incondicional.
    
    Args:
        equip (str): Identificador del equipo (ej. 'BOMBA_01').
        bit_index (int): Numero del enclavamiento (0..31).
        description (str|unicode): Descripcion operativa del enclavamiento.
        is_active (bool): True si el enclavamiento se ha disparado (0 en logica de planta), False si se ha normalizado.
        db_conn (str): Nombre de la conexion de base de datos en Ignition.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    logger = system.util.getLogger("SCADA.Transactions.Interlocks")
    
    if not equip or str(equip).strip() == "":
        return (False, u"Identificador de equipo invalido")
        
    if bit_index < 0 or bit_index > 31:
        return (False, u"Indice de enclavamiento fuera de limites (0..31)")
        
    desc_str = unicode(description).strip() if description else u"Sin descripcion"
    now_ts = system.date.now()
    
    # 1. Iniciar transaccion JDBC con timeout de 10 segundos
    tx_id = system.db.beginTransaction(database=db_conn, timeout=10000)
    
    try:
        logger.info("Iniciando transaccion [{}] para enclavamiento {} bit {}".format(tx_id, equip, bit_index))
        
        # 2. Comprobar existencia previa dentro de la transaccion
        check_sql = "SELECT id, TextEnclava FROM enclavaments WHERE equip = ? AND NumeroEnclava = ?"
        existing = system.db.runPrepQuery(check_sql, [str(equip), int(bit_index)], database=db_conn, txId=tx_id)
        
        if len(existing) == 0:
            # Insercion de nuevo enclavamiento
            insert_sql = """
                INSERT INTO enclavaments (StartDate, Equip, NumeroEnclava, TextEnclava)
                VALUES (?, ?, ?, ?)
            """
            rows = system.db.runPrepUpdate(
                insert_sql, 
                [now_ts, str(equip), int(bit_index), desc_str], 
                database=db_conn, 
                txId=tx_id
            )
            if rows != 1:
                raise Exception("Fallo al insertar nuevo registro en enclavaments")
        else:
            # Actualizacion unificada de fecha y texto en una sola sentencia atomica
            update_sql = """
                UPDATE enclavaments 
                SET StartDate = ?, TextEnclava = ? 
                WHERE equip = ? AND NumeroEnclava = ?
            """
            rows = system.db.runPrepUpdate(
                update_sql, 
                [now_ts, desc_str, str(equip), int(bit_index)], 
                database=db_conn, 
                txId=tx_id
            )
            if rows != 1:
                raise Exception("Fallo al actualizar registro existente en enclavaments")
                
        # 3. Sentencia de Auditoria obligatoria bajo la misma transaccion
        action_suffix = u"actiu" if is_active else u"desenclavat"
        audit_desc = u"Enclavament {}: {} {}".format(bit_index, desc_str, action_suffix)
        
        audit_sql = """
            INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
            VALUES (?, ?, 'PLC', CURRENT_TIMESTAMP)
        """
        audit_rows = system.db.runPrepUpdate(
            audit_sql, 
            [str(equip), audit_desc], 
            database=db_conn, 
            txId=tx_id
        )
        if audit_rows != 1:
            raise Exception("No se pudo registrar la auditoria en ins_Registre_Actuacions")
            
        # 4. Confirmacion atomica (COMMIT)
        system.db.commitTransaction(tx_id)
        logger.info("Transaccion [{}] confirmada (COMMIT) con exito".format(tx_id))
        return (True, u"Enclavamiento y auditoria sincronizados correctamente")
        
    except Exception as err:
        # 5. Reversion total (ROLLBACK) ante cualquier fallo
        system.db.rollbackTransaction(tx_id)
        err_msg = str(err)
        logger.error("Transaccion [{}] REVERTIDA (ROLLBACK) por error: {}".format(tx_id, err_msg))
        return (False, u"Operacion abortada: {}".format(err_msg))
        
    finally:
        # 6. REGLA OBLIGATORIA: Retorno incondicional de la conexion fisica al pool JDBC
        system.db.closeTransaction(tx_id)
```

#### Código Resultante en el Evento del Tag (`valueChanged`)

```python
# Evento valueChanged en el tag de enclavamientos (Limpio y robusto)
current = currentValue.value
prev = previousValue.value
equip = tagPath.split('/')[-2]
descr_dataset = system.tag.readBlocking(["[.]WENC_DESC"])[0].value

for i in range(32):
    if current[i] != prev[i]:
        # En la logica de planta, 0 indica activo y 1 normalizado
        is_active = (current[i] == 0)
        desc_text = descr_dataset.getValueAt(i, "Descripcio") if descr_dataset and i < descr_dataset.getRowCount() else "Bit %d" % i
        
        ok, msg = project.enclavamientos.service.sync_interlock_transactional(
            equip=equip,
            bit_index=i,
            description=desc_text,
            is_active=is_active,
            db_conn="BD_LLIBRERIA"
        )
        if not ok:
            system.util.getLogger("SCADA.Tags.Interlocks").warn("Aviso en sincronizacion: " + msg)
```

---

### 3.2. Gestión Documental (`uploadFile`): Transacción JDBC Atómica para Metadatos y Contenido Binario

#### Diagnóstico

En el script de subida de archivos al gestor documental (`ejemplos-scripts-ignition.md`, líneas 35-66):

```python
# CÓDIGO ORIGINAL (Fragmento sin control transaccional)
id = system.db.runNamedQuery(path="Document Management/Documents/Exists", parameters={"folderId":folder.id, "name":name, 'equip':equip})

if id == None:
    id = system.db.runNamedQuery(path="Document Management/Documents/Add", parameters={"name":name, "filename":fileName, "extension":fileExtension, "size":fileSize, "modifiedBy":user, "folderId":folder.id, 'equip':equip }, getKey=True)
    system.db.runNamedQuery(path="Document Management/Documents/Add Document Contents", parameters={"documentId":fileName, "contents":fileBytes})
else:
    system.db.runNamedQuery(path="Document Management/Documents/Update Document File Size", parameters={"id":id, "size":fileSize, "modifiedBy":user})
    system.db.runNamedQuery(path="Document Management/Documents/Update Document Contents", parameters={"documentId":fileName, "contents":fileBytes})
```

**Riesgo:** Si la inserción o actualización de la cabecera en `Documents` tiene éxito pero falla la llamada que persiste el contenido binario en `Document Contents` (por ejemplo, por límite de memoria, bloqueo de tabla o desbordamiento de búfer), el documento queda registrado con metadatos pero sin datos descargables, o con tamaño desfasado respecto a su contenido real.

#### Solución Propuesta (Enseñanzas Lab 6.1)

Encapsular el alta o actualización en una transacción JDBC con sentencias preparadas (o Named Queries transaccionales) gestionadas bajo `tx_id`. Si la inserción binaria falla, la inserción de metadatos se revierte completamente.

```python
# Implementación en Project Library: project.documents.manager
"""
Modulo: project.documents.manager
Descripcion: Carga segura y transaccional de archivos documentales.
Autor: Equipo SCADA
"""

def upload_document_transactional(file_name, file_size, file_bytes, equip, folder_id, user_name, db_conn="BD_DOCUMENTS"):
    """
    Registra metadatos y contenido binario de forma atomica con garantia de Rollback.
    
    Args:
        file_name (str): Nombre completo del archivo.
        file_size (int|long): Tamaño en bytes.
        file_bytes (bytearray|str): Datos binarios.
        equip (str): Identificador del equipo asociado.
        folder_id (int): Identificador de la carpeta.
        user_name (str): Usuario responsable.
        db_conn (str): Conexion JDBC configurada.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    import os
    logger = system.util.getLogger("SCADA.Transactions.Documents")
    
    if not file_name or not file_bytes:
        return (False, u"Datos de archivo incompletos o vacios")
        
    name, ext = os.path.splitext(file_name)
    clean_ext = ext.replace(".", "")
    
    tx_id = system.db.beginTransaction(database=db_conn, timeout=15000)
    
    try:
        logger.info("Iniciando transaccion documental [{}] para archivo: {}".format(tx_id, file_name))
        
        # 1. Comprobar existencia previa
        check_sql = "SELECT id FROM documents WHERE folder_id = ? AND name = ? AND equip = ?"
        exists = system.db.runPrepQuery(check_sql, [int(folder_id), str(name), str(equip)], database=db_conn, txId=tx_id)
        
        if len(exists) == 0:
            # Insercion de cabecera
            ins_doc = """
                INSERT INTO documents (name, filename, extension, size, modified_by, folder_id, equip, date_created)
                VALUES (?, ?, ?, ?, ?, ?, ?, CURRENT_TIMESTAMP)
            """
            system.db.runPrepUpdate(ins_doc, [str(name), str(file_name), str(clean_ext), long(file_size), str(user_name), int(folder_id), str(equip)], database=db_conn, txId=tx_id)
            
            # Insercion de contenido binario
            ins_blob = "INSERT INTO document_contents (filename, contents) VALUES (?, ?)"
            system.db.runPrepUpdate(ins_blob, [str(file_name), file_bytes], database=db_conn, txId=tx_id)
        else:
            doc_id = exists[0]["id"]
            upd_doc = "UPDATE documents SET size = ?, modified_by = ?, date_modified = CURRENT_TIMESTAMP WHERE id = ?"
            system.db.runPrepUpdate(upd_doc, [long(file_size), str(user_name), int(doc_id)], database=db_conn, txId=tx_id)
            
            upd_blob = "UPDATE document_contents SET contents = ? WHERE filename = ?"
            system.db.runPrepUpdate(upd_blob, [file_bytes, str(file_name)], database=db_conn, txId=tx_id)
            
        # 2. Confirmacion atomica
        system.db.commitTransaction(tx_id)
        logger.info("Transaccion [{}] confirmada exitosamente".format(tx_id))
        return (True, u"Documento almacenado correctamente")
        
    except Exception as err:
        system.db.rollbackTransaction(tx_id)
        err_msg = str(err)
        logger.error("Transaccion [{}] REVERTIDA por fallo en carga documental: {}".format(tx_id, err_msg))
        return (False, u"Error al almacenar documento: {}".format(err_msg))
        
    finally:
        system.db.closeTransaction(tx_id)
```

---

### 3.3. Cierre de Lote de Producción y Control de Stock con Rollback Automático (`project.production.batch`)

#### Diagnóstico

En la operativa industrial de planta (como envasado, dosificación de reactivos o EDAR), el cierre de un lote de producción requiere registrar la cabecera del lote, descontar el inventario de múltiples reactivos y materias primas, y generar la pista de auditoría correspondiente.

Si estas acciones se ejecutan con llamadas individuales (`runUpdateQuery`), una rotura de stock sobrevenida a mitad del proceso o un fallo de conexión deja el lote registrado pero el stock a medio descontar, generando discrepancias contables e imposibilitando la trazabilidad.

#### Solución Propuesta (Enseñanzas Lab 6.1)

Implementar el servicio maestro `project.production.batch.close_production_batch` aplicando:
1. Cláusula defensiva concurrente `WHERE material_id = ? AND stock_qty >= ?`. Si el stock disponible es inferior a la cantidad consumida, `rows_updated` será `0`.
2. Detección inmediata: `if rows_updated != 1: raise Exception(...)`, provocando el salto al bloque `except` y revirtiendo la totalidad de los cambios (incluso la cabecera del lote ya insertada).
3. Auditoría en `ins_Registre_Actuacions` bajo el mismo identificador de transacción `tx_id`.
4. Cierre incondicional en el bloque `finally` con `system.db.closeTransaction(tx_id)`.

```python
# Implementación en Project Library: project.production.batch
"""
Modulo: project.production.batch
Descripcion: Gestion transaccional atomica (ACID) de cierre de lotes y regularizacion de stock.
             (Enseñanzas directas del Laboratorio 6.1).
Autor: Equipo SCADA
"""

def close_production_batch(batch_code, equip, units_produced, material_consumptions, username, db_conn="SANDBOX_DB"):
    """
    Ejecuta el cierre formal de un lote de produccion descontando stock dentro de una transaccion JDBC.
    
    Args:
        batch_code (str): Codigo identificador del lote (ej. 'LOT-2026-X1').
        equip (str): Identificador del equipo (ej. 'EDAR_BOMBA_01', 'LINEA_ENVASADO_01').
        units_produced (int): Cantidad producida.
        material_consumptions (list of dict): [{'material_id': 'REACTIVO_COAGULANTE', 'qty': 120.5}, ...]
        username (str): Operador que autoriza el cierre.
        db_conn (str): Conexion de base de datos en Ignition.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    logger = system.util.getLogger("SCADA.Transactions.Batch")
    
    # 1. Validaciones previas de entrada
    if not batch_code or not equip or not username:
        return (False, u"Parámetros de cabecera incompletos")
        
    if units_produced is None or int(units_produced) <= 0:
        return (False, u"La cantidad producida debe ser superior a cero")
        
    if not material_consumptions or not isinstance(material_consumptions, list):
        return (False, u"Se requiere una lista válida de consumos de materia prima")
        
    # 2. Iniciar transaccion JDBC con timeout de 10 segundos
    tx_id = system.db.beginTransaction(database=db_conn, timeout=10000)
    
    try:
        logger.info("Iniciando transacción [{}] para el cierre de lote: {}".format(tx_id, batch_code))
        
        # 3. Sentencia 1: Insercion de la cabecera del lote
        header_sql = """
            INSERT INTO production_batch_history (batch_code, equip, units_produced, closed_by, closed_at)
            VALUES (?, ?, ?, ?, CURRENT_TIMESTAMP)
        """
        h_rows = system.db.runPrepUpdate(
            header_sql, 
            [str(batch_code), str(equip), int(units_produced), str(username)], 
            database=db_conn, 
            txId=tx_id
        )
        
        if h_rows != 1:
            raise Exception("No se pudo insertar la cabecera del lote en el histórico")
            
        # 4. Sentencias 2..N: Descuento de stock de cada materia prima
        stock_sql = """
            UPDATE raw_materials_stock 
            SET stock_qty = stock_qty - ? 
            WHERE material_id = ? AND stock_qty >= ?
        """
        for item in material_consumptions:
            mat_id = item.get("material_id")
            qty = float(item.get("qty", 0.0))
            
            if qty <= 0.0:
                raise Exception("Cantidad de consumo no válida para el material: {}".format(mat_id))
                
            # La clausula AND stock_qty >= ? asegura que NO se descuente si no hay suficiente existencia
            rows_updated = system.db.runPrepUpdate(
                stock_sql, 
                [qty, mat_id, qty], 
                database=db_conn, 
                txId=tx_id
            )
            
            if rows_updated != 1:
                # Disparo de excepcion que forzara el Rollback completo
                raise Exception("Stock insuficiente o material inexistente para: {}".format(mat_id))
                
        # 5. Sentencia Final: Registro en auditoria de actuaciones (mismo txId)
        audit_sql = """
            INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
            VALUES (?, ?, ?, CURRENT_TIMESTAMP)
        """
        audit_desc = u"Cierre de Lote {} ({:.0f} unidades) y regularización de stock".format(
            unicode(batch_code), float(units_produced)
        )
        system.db.runPrepUpdate(
            audit_sql, 
            [str(equip), audit_desc, str(username)], 
            database=db_conn, 
            txId=tx_id
        )
        
        # 6. Confirmacion atomica de todos los cambios
        system.db.commitTransaction(tx_id)
        logger.info("Transacción [{}] confirmada (COMMIT) con éxito".format(tx_id))
        return (True, u"Lote {} cerrado y stock actualizado correctamente".format(batch_code))
        
    except Exception as err:
        # Reversion inmediata de todas las sentencias ejecutadas bajo este txId
        system.db.rollbackTransaction(tx_id)
        err_msg = str(err)
        logger.error("Transacción [{}] REVERTIDA (ROLLBACK) por error: {}".format(tx_id, err_msg))
        return (False, u"Operación abortada: {}".format(err_msg))
        
    finally:
        # REGLA OBLIGATORIA: Retorno incondicional de la conexion fisica al pool JDBC
        system.db.closeTransaction(tx_id)
```

---

### 3.4. Binding de Estados en Flex Container: Script Transform Defensivo en Memoria (Cero I/O)

#### Diagnóstico

En el script de binding para las instancias de estados (`ejemplos-scripts-ignition.md`, líneas 120-226):

```python
# CÓDIGO ORIGINAL (Fragmento con lecturas repetitivas síncronas en bucle y estilos vacíos)
for result in results:
    if 'E' in system.tag.readBlocking(str(result['fullPath']) + '.Documentation')[0].value and system.tag.readBlocking(str(result['fullPath']) + '.Enabled')[0].value:
        if '1' in system.tag.readBlocking(str(result['fullPath']) + '.Documentation')[0].value:
            first.insert(0, {
                "instanceStyle": {"classes": ""},
                "instancePosition": {1},
                "tagPath": str(result['fullPath'])
            })
        elif '2' in system.tag.readBlocking(str(result['fullPath']) + '.Documentation')[0].value:
            second.insert(-1, {
                "instanceStyle": {"classes": ""},
                "instancePosition": {},
                "tagPath": str(result['fullPath'])
            })
        # ... se repite para ramas 3, 4 y else con clases CSS vacías
```

**Problemas detectados:**
1. **Llamadas síncronas bloqueantes en bucle:** Múltiples lecturas individuales por tag mediante `system.tag.readBlocking` dentro de un bucle `for`, congelando la interfaz.
2. **Estilos vacíos:** La propiedad `instanceStyle.classes` se entrega vacía (`""`), obligando a definir estilos visuales ad-hoc en otros puntos o dejando componentes sin contraste cromático.
3. **Fragilidad ante nulos:** Si un tag devuelve `None` en `.Documentation` o pierde la comunicación, el script falla con `TypeError`.

#### Solución Propuesta (Enseñanzas Lab 6.2)

1. Centralizar la transformación visual en una función pura en `Project Library`: `project.ui.transforms.format_equipment_status_card`.
2. **Cero operaciones I/O:** Prohibición absoluta de llamadas a base de datos o lecturas de tags en el transform.
3. Tratamiento defensivo con valores por defecto ante estados nulos o arranque de pantalla.
4. Mapeo estructurado de estados operativos devolviendo texto, color de fondo, color de texto, icono, clase CSS de badge y animación.

```python
# Implementación en Project Library: project.ui.transforms
"""
Modulo: project.ui.transforms
Descripcion: Funciones puras para Script Transforms en Perspective.
             Cero I/O, ejecucion instantanea en memoria RAM.
             (Implementado conforme al Laboratorio 6.2).
Autor: Equipo SCADA
"""

def format_equipment_status_card(raw_state_data):
    """
    Transforma un diccionario o valor de estado crudo en metadatos visuales para UI.
    
    Args:
        raw_state_data (dict|any): Objeto que contiene {'status_code': int, 'flow_val': float, ...}
        
    Returns:
        dict: Propiedades listas para enlazar a estilos, texto e iconos de Perspective.
    """
    # 1. Tratamiento defensivo ante estados nulos o arranque de pantalla
    if raw_state_data is None or not isinstance(raw_state_data, dict):
        return {
            "status_text": "SIN COMUNICACION",
            "bg_color": "#7f8c8d",       # Gris oscuro
            "text_color": "#ffffff",
            "icon_path": "material/cloud_off",
            "blink_animation": True,
            "badge_class": "badge-offline"
        }
        
    state_code = raw_state_data.get("status_code", -1)
    flow_val = raw_state_data.get("flow_val", 0.0)
    
    # 2. Mapeo de estados y estilos condicionales
    if state_code == 1: # En Marcha / Produciendo
        flow_str = "{:.1f} m3/h".format(flow_val) if flow_val is not None else "0.0 m3/h"
        return {
            "status_text": "EN PRODUCCION ({})".format(flow_str),
            "bg_color": "#27ae60",       # Verde
            "text_color": "#ffffff",
            "icon_path": "material/play_circle_filled",
            "blink_animation": False,
            "badge_class": "badge-running"
        }
    elif state_code == 2: # Parada / Alarma
        return {
            "status_text": "PARADA DE EMERGENCIA / ALARMA",
            "bg_color": "#e74c3c",       # Rojo
            "text_color": "#ffffff",
            "icon_path": "material/error",
            "blink_animation": True,
            "badge_class": "badge-alarm"
        }
    elif state_code == 3: # En Mantenimiento
        return {
            "status_text": "MANTENIMIENTO PREVENTIVO",
            "bg_color": "#f39c12",       # Ambar
            "text_color": "#000000",
            "icon_path": "material/build",
            "blink_animation": False,
            "badge_class": "badge-maintenance"
        }
    elif state_code == 0: # En Espera / Standby
        return {
            "status_text": "STANDBY / EN ESPERA",
            "bg_color": "#95a5a6",       # Gris claro
            "text_color": "#000000",
            "icon_path": "material/pause_circle_outline",
            "blink_animation": False,
            "badge_class": "badge-standby"
        }
    else:
        return {
            "status_text": "ESTADO INDEFINIDO (Código: {})".format(state_code),
            "bg_color": "#34495e",       # Azul oscuro
            "text_color": "#ffffff",
            "icon_path": "material/help_outline",
            "blink_animation": False,
            "badge_class": "badge-unknown"
        }
```

#### Código Resultante en el Script Transform de Perspective

En el binding de la propiedad visual del componente (`custom.cardStyle` o `props.text`), el Script Transform se reduce a una sola línea:

```python
# Codigo limpio en el Script Transform de Perspective (Cero I/O, Sub-milisegundo)
return project.ui.transforms.format_equipment_status_card(value)
```

---

### 3.5. Binding de Consignas en Flex Container: Eliminación del Acoplamiento Rígido `self.parent.parent.parent` y Transform Puro

#### Diagnóstico

En las líneas 486-492 de `ejemplos-scripts-ignition.md`:

```python
# CÓDIGO ORIGINAL (Fragmento con acoplamiento rígido de navegación relativa)
# Actualitzar el binding del TabContainer per reflectir els canvis
self.parent.parent.parent.getChild("TabContainer").refreshBinding('props.tabs')

# Retornar la llista d'instàncies amb les consignes
return first + second + instances
```

**Riesgo crítico:** Esta instrucción asume una estructura jerárquica inmutable. Si se añade un contenedor intermedio (por ejemplo, para adaptar el diseño a pantallas móviles), la instrucción `self.parent.parent.parent` apunta al objeto incorrecto y arroja un error fatal `AttributeError: 'NoneType' object has no attribute 'getChild'` o no encuentra el `TabContainer`. Además, el binding mezcla lecturas históricas de tags con manipulación de la interfaz.

#### Solución Propuesta (Enseñanzas Lab 6.2 y 6.3)

1. Eliminar la instrucción `self.parent.parent.parent.getChild("TabContainer").refreshBinding('props.tabs')` del interior del binding.
2. Trasladar la notificación al bus de mensajes de Perspective mediante `system.perspective.sendMessage("OnConsignasUpdated", scope="page")` desde el evento que genera la mutación de datos.
3. Configurar en el `TabContainer` un *Message Handler* llamado `OnConsignasUpdated` que refresque su propio binding (`self.refreshBinding('props.tabs')`).
4. Reemplazar la construcción de diccionarios de instancias por una función constructora en `project.ui.transforms`.

```python
# Implementación en Project Library: project.ui.transforms (Generador de Instancias de Consigna)

def build_consigna_ui_instance(tag_path, value_new, is_scada_view, selected, date_str):
    """
    Construye la estructura de instancia para el Flex Repeater de consignas con badge condicional.
    
    Args:
        tag_path (str): Ruta completa del tag.
        value_new (any): Valor de consigna actual o propuesto.
        is_scada_view (bool): Si la vista muestra valor SCADA o actual.
        selected (bool): Si la fila esta seleccionada para envio.
        date_str (str): Fecha formateada.
        
    Returns:
        dict: Objeto de instancia compatible con props.instances.
    """
    badge_cls = "badge-modified" if selected else "badge-normal"
    
    return {
        "instanceStyle": {
            "classes": badge_cls
        },
        "instancePosition": {},
        "tagPath": str(tag_path) if tag_path else "",
        "value_new": value_new if value_new is not None else 0,
        "view_SCADA_value": bool(is_scada_view),
        "selected": bool(selected),
        "data": str(date_str) if date_str else ""
    }
```

---

### 3.6. Botón de Maniobra / Reconocimiento de Alarmas: Publicación de Eventos al Bus de Mensajes y Message Handlers Receptores

#### Diagnóstico

En el botón de envío de consignas (`ejemplos-scripts-ignition.md`, líneas 494-579):

```python
# CÓDIGO ORIGINAL (Fragmento con acoplamiento directo y lógica dispersa)
instances = self.getSibling("FlexRepeater").props.instances

for instance in instances:
    selected = instance['selected']
    if selected:
        tagPath = instance['tagPath']
        ...
        system.tag.writeBlocking([Mwrite], [value_new])
        params = {"Equip": self.view.params.TAG, "Descripcio": "Ordre Enviar Consigna " + tag + ': ' + name, "Usuari": self.session.props.auth.user.userName}
        system.db.runNamedQuery("ins_Registre_Actuacions", params)
```

**Problemas detectados:**
1. Dependencia de componentes hermanos mediante `self.getSibling("FlexRepeater")`. Si el componente se renombra o se encapsula en otro contenedor, el botón deja de funcionar.
2. Inexistencia de broadcast: el resto de la interfaz (tablas de alarmas, badges de estado, cabecera de la vista) no se entera de que una consigna o alarma fue maniobrada, requiriendo recargas forzadas de pantalla.

#### Solución Propuesta (Enseñanzas Lab 6.3)

Implementar una arquitectura desacoplada basada en el patrón Publicador-Suscriptor:
1. **Lógica de negocio y persistencia en `Project Library` (`project.alarms.events`):** La función procesa la validación, registra la auditoría en base de datos mediante sentencias preparadas y retorna un payload estructurado para difusión.
2. **Componente Emisor (Botón):** Ejecuta la función de librería y publica el mensaje en el bus mediante `system.perspective.sendMessage(messageHandler="OnAlarmAcknowledged", payload=result, scope="page")`.
3. **Componentes Receptores (Tabla Principal, Badges, TabContainer):** Se configuran con un *Message Handler* que escucha en el scope seleccionado y ejecuta `self.refreshBinding()`.

```mermaid
flowchart LR
    subgraph Emisor [Componente Emisor: Botón de Maniobra / Modal]
        Click[onActionPerformed] --> Service[project.alarms.events.acknowledge_equipment_alarm]
        Service --> SendMsg[system.perspective.sendMessage: OnAlarmAcknowledged]
    end

    subgraph Bus [Bus de Mensajería de Perspective: Scope Page / Session]
        SendMsg --> Router[Enrutador de Mensajes]
    end

    subgraph Receptores [Componentes Receptores: Message Handlers]
        Router --> Table[Tabla Principal: refreshBinding('props.data')]
        Router --> TabCont[TabContainer: refreshBinding('props.tabs')]
        Router --> Badge[Indicador KPI: refreshBinding('props.text')]
    end
```

```python
# Implementación en Project Library: project.alarms.events
"""
Modulo: project.alarms.events
Descripcion: Logica de reconocimiento y broadcast desacoplado de eventos SCADA.
             (Implementado conforme al Laboratorio 6.3).
Autor: Equipo SCADA
"""

def acknowledge_equipment_alarm(alarm_id, equip, username, reason="Reconocimiento Operativo", db_conn="SANDBOX_DB"):
    """
    Reconoce una alarma, registra la actuacion en base de datos y prepara el payload de difusion.
    
    Args:
        alarm_id (int|str): Identificador unico de la alarma o enclavamiento.
        equip (str): Identificador del equipo (ej. 'EDAR_BOMBA_01').
        username (str): Usuario que reconoce la alarma.
        reason (str): Justificacion o motivo operativo.
        db_conn (str): Conexion de base de datos para auditoria.
        
    Returns:
        tuple: (bool success, dict payload_or_error)
    """
    logger = system.util.getLogger("SCADA.Alarms.Events")
    
    if not alarm_id or not equip:
        return (False, {"error": u"Identificador de alarma o equipo no válido"})
        
    user_str = str(username) if username else "SYSTEM"
    
    try:
        # 1. Registrar la actuacion en la tabla ins_Registre_Actuacions
        audit_desc = u"Alarma [{}] reconocida para {}. Motivo: {}".format(
            unicode(alarm_id), unicode(equip), unicode(reason)
        )
        sql = """
            INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
            VALUES (?, ?, ?, CURRENT_TIMESTAMP)
        """
        system.db.runPrepUpdate(sql, [str(equip), audit_desc, user_str], database=db_conn)
        
        logger.info("Alarma {} reconocida por {} en el equipo {}".format(alarm_id, user_str, equip))
        
        # 2. Retornar payload limpio para el Message Handler
        broadcast_payload = {
            "alarm_id": alarm_id,
            "equip": str(equip),
            "ack_by": user_str,
            "ack_timestamp": system.date.format(system.date.now(), "dd/MM/yyyy HH:mm:ss")
        }
        return (True, broadcast_payload)
        
    except Exception as ex:
        logger.error("Error al reconocer alarma {}: {}".format(alarm_id, str(ex)))
        return (False, {"error": u"Error interno de base de datos"})
```

#### Código Resultante en el Evento del Botón Emisor (`onActionPerformed`)

```python
# Evento onActionPerformed del Boton (Emisor Desacoplado)
equip = self.view.params.TAG if hasattr(self.view.params, "TAG") else "EDAR_BOMBA_01"
alarm_id = self.view.params.alarmId if hasattr(self.view.params, "alarmId") else 104
user = self.session.props.auth.user.userName if hasattr(self.session.props.auth, "user") else "operador"

# 1. Ejecutar logica de negocio y persistencia
ok, result = project.alarms.events.acknowledge_equipment_alarm(
    alarm_id=alarm_id,
    equip=equip,
    username=user,
    reason="Maniobra confirmada por operador en pantalla"
)

if ok:
    # 2. Emitir mensaje desacoplado al bus (Scope: Page)
    system.perspective.sendMessage(
        messageHandler="OnAlarmAcknowledged",
        payload=result,
        scope="page"
    )
    
    # 3. Notificacion emergente amigable
    system.perspective.openPopup(
        id="ack_popup",
        view="Popups/ConfirmNotification",
        params={"msg": "Acción procesada correctamente"},
        showCloseIcon=True
    )
else:
    system.perspective.print("ERROR: " + result.get("error", "Fallo desconocido"))
```

#### Código Resultante en el Message Handler Receptor (Tabla o Contenedor)

En la tabla o contenedor receptor de la vista, se añade un handler con:
* **Message Type:** `OnAlarmAcknowledged`
* **Listen Scopes:** `Session` y `Page`

```python
# Script dentro del Message Handler 'OnAlarmAcknowledged'
# Parametros disponibles: self, payload
logger = system.util.getLogger("SCADA.UI.MessageReceiver")

alarm_id = payload.get("alarm_id")
equip = payload.get("equip")
ack_by = payload.get("ack_by")

logger.info("Mensaje recibido: Alarma {} reconocida por {}".format(alarm_id, ack_by))

# Refrescar de forma desacoplada los bindings requeridos sin depender de self.parent...
if hasattr(self, "refreshBinding"):
    if "data" in self.props:
        self.refreshBinding("props.data")
    if "instances" in self.props:
        self.refreshBinding("props.instances")
    if "tabs" in self.props:
        self.refreshBinding("props.tabs")
```

---

### 3.7. Árbol de Navegación: Desacoplamiento del Bus de Perspective y Modularización de la Pila de Historial

#### Diagnóstico

En el script de evento del árbol de navegación (`ejemplos-scripts-ignition.md`, líneas 844-1051), se observan más de 100 líneas de código acumuladas dentro del evento de componente (`runAction`). El script mezcla:
1. Navegación física de la vista (`system.perspective.navigate`).
2. Cálculo de profundidad del árbol e indexación sobre `self.props.items`.
3. Gestión directa de la pila de navegación en `self.session.custom.navHistorique`.
4. Emisión directa de mensajes (`system.perspective.sendMessage('change-pretittle', ...)`).

Esta acumulación impide reutilizar la lógica de migas de pan en otras vistas, dificulta el testeo unitario y hace frágil la manipulación del array en sesión.

#### Solución Propuesta (Enseñanzas Lab 6.2 y 6.3)

1. Extraer el formateo de títulos jerárquicos a una función pura en `project.nav.history.format_breadcrumb_title`.
2. Extraer la gestión de la pila de historial (control de tope máximo FIFO de 5 elementos y retroceso de índice) a `project.nav.history.calculate_history_slice`.
3. En el evento del componente, delegar en la librería y emitir el mensaje al bus de Perspective con un payload estructurado.

```python
# Implementación en Project Library: project.nav.history
"""
Modulo: project.nav.history
Descripcion: Funciones puras para formateo de migas de pan y gestion de la pila de historial.
Autor: Equipo SCADA
"""

def format_breadcrumb_title(labels):
    """
    Funcion pura que une una lista de etiquetas en un formato jerarquico estandarizado.
    
    Args:
        labels (list of str): Lista ordenada de etiquetas de navegacion.
        
    Returns:
        unicode: Cadena con formato 'Nivel 1 > Nivel 2 > Nivel 3'.
    """
    if not labels:
        return u""
    clean = [unicode(l).strip() for l in labels if l and unicode(l).strip() != ""]
    return u" > ".join(clean)


def calculate_history_slice(current_history, current_index, new_item, max_items=5):
    """
    Funcion pura que actualiza la pila de historial de navegacion evitando desbordamientos.
    
    Args:
        current_history (list of dict): Lista actual de pantallas en el historial.
        current_index (int): Indice de la pantalla activa.
        new_item (dict): Objeto {'title': str, 'path': str}.
        max_items (int): Limite maximo de elementos en la pila.
        
    Returns:
        tuple: (list updated_history, int new_index)
    """
    history = list(current_history) if isinstance(current_history, list) else []
    
    # Truncar historial si se habia navegado hacia atras
    if 0 <= current_index < len(history) - 1:
        history = history[:current_index + 1]
        
    # Evitar registrar dos veces consecutivas la misma ruta
    if not history or history[-1].get("path") != new_item.get("path"):
        history.append(new_item)
        
    # Mantener el limite maximo eliminando el mas antiguo (FIFO)
    while len(history) > max_items:
        history.pop(0)
        
    return (history, len(history) - 1)
```

#### Código Resultante en el Evento del Árbol (`runAction`)

```python
# Evento runAction del componente Arbol (Conciso y desacoplado)
item_path = event.itemPath
target_path = event.data.get("path", "")

if target_path:
    # 1. Navegacion Perspective
    system.perspective.navigate(
        view='ECO/10_ECO_BreakpointView',
        params={'view': target_path}
    )
    
    # 2. Extraccion de etiquetas de migas de pan
    levels = item_path.split("/")
    labels = []
    current_items = self.props.items
    for lvl in levels:
        idx = int(lvl)
        if idx < len(current_items):
            labels.append(current_items[idx].label)
            current_items = current_items[idx].get("items", [])
    
    title_str = project.nav.history.format_breadcrumb_title(labels)
    
    # 3. Emitir mensaje desacoplado al bus (Scope: Page)
    system.perspective.sendMessage(
        messageHandler='change-pretittle',
        payload={
            'title': title_str,
            'path': target_path
        },
        scope='page'
    )
    
    # 4. Actualizar historial de sesion mediante funcion pura
    hist = self.session.custom.navHistorique
    idx = self.session.custom.navActual
    new_entry = {'title': title_str, 'path': target_path}
    
    new_hist, new_idx = project.nav.history.calculate_history_slice(hist, idx, new_entry, max_items=5)
    self.session.custom.navHistorique = new_hist
    self.session.custom.navActual = new_idx
```

---

## 4. Banco de Pruebas Automatizado (Script Console)

Para verificar y certificar de manera integral el correcto funcionamiento de las transacciones atómicas ACID, la reversión por rotura de stock, la liberación incondicional de conexiones JDBC y el comportamiento de las funciones puras en memoria, se debe ejecutar el siguiente banco de pruebas en la **Script Console** de Ignition Designer (**Tools -> Script Console**):

```python
print "=" * 80
print "CERTIFICACION DE REFACTORIZACION SCADA - FASE 6 (LAB-06)"
print "=" * 80

# -----------------------------------------------------------------------------
# TEST 1: Transacción Atómica ACID con Rollback y Cierre JDBC (Lab 6.1)
# -----------------------------------------------------------------------------
print "\n--- [TEST 1] Transaccion Atomica de Lote con Control de Stock y Rollback ---"

# Caso 1.1: Cierre Nominal con Stock Suficiente (Exito -> COMMIT)
consumos_nominales = [
    {"material_id": "REACTIVO_COAGULANTE", "qty": 100.0},
    {"material_id": "REACTIVO_FLOCULANTE",  "qty": 20.0}
]

ok1, msg1 = project.production.batch.close_production_batch(
    batch_code="LOT-TEST-NOMINAL-01",
    equip="EDAR_BOMBA_01",
    units_produced=2500,
    material_consumptions=consumos_nominales,
    username="operador.certificacion",
    db_conn="SANDBOX_DB"
)
print "Caso 1.1 (Nominal): Exito = {} | Mensaje = {}".format(ok1, msg1)
assert ok1 == True, "Fallo en cierre nominal de lote"

# Caso 1.2: Cierre con Rotura de Stock Forzada (Fallo -> ROLLBACK AUTOMATICO)
# Se solicitan 999.999 unidades, superando las existencias fisicas
consumos_excesivos = [
    {"material_id": "REACTIVO_COAGULANTE", "qty": 50.0},
    {"material_id": "REACTIVO_FLOCULANTE",  "qty": 999999.0} # Dispara Rollback
]

ok2, msg2 = project.production.batch.close_production_batch(
    batch_code="LOT-TEST-FAIL-02",
    equip="EDAR_BOMBA_02",
    units_produced=1000,
    material_consumptions=consumos_excesivos,
    username="operador.certificacion",
    db_conn="SANDBOX_DB"
)
print "Caso 1.2 (Rollback): Exito = {} | Mensaje = {}".format(ok2, msg2)
assert ok2 == False, "El rollback no se activo ante rotura de stock"
assert u"Stock insuficiente" in msg2 or u"Operación abortada" in msg2, "Mensaje de error inesperado"
print "[OK] Transacciones ACID: Commit nominal y Rollback forzado verificados."


# -----------------------------------------------------------------------------
# TEST 2: Script Transforms Puros Defensivos en Memoria RAM (Lab 6.2)
# -----------------------------------------------------------------------------
print "\n--- [TEST 2] Script Transforms Puros en Memoria (Cero I/O) ---"

casos_transform = [
    {"name": "Estado En Produccion", "data": {"status_code": 1, "flow_val": 45.5}, "esperado_bg": "#27ae60", "esperado_blink": False},
    {"name": "Estado En Alarma",      "data": {"status_code": 2, "flow_val": 0.0},  "esperado_bg": "#e74c3c", "esperado_blink": True},
    {"name": "Estado Mantenimiento",  "data": {"status_code": 3, "flow_val": 0.0},  "esperado_bg": "#f39c12", "esperado_blink": False},
    {"name": "Estado Standby",        "data": {"status_code": 0, "flow_val": 0.0},  "esperado_bg": "#95a5a6", "esperado_blink": False},
    {"name": "Dato Nulo / Desconexion", "data": None,                                "esperado_bg": "#7f8c8d", "esperado_blink": True}
]

for ct in casos_transform:
    res = project.ui.transforms.format_equipment_status_card(ct["data"])
    assert res["bg_color"] == ct["esperado_bg"], "Fallo en color de fondo para: " + ct["name"]
    assert res["blink_animation"] == ct["esperado_blink"], "Fallo en flag de parpadeo para: " + ct["name"]
    assert "status_text" in res and "icon_path" in res and "badge_class" in res, "Estructura incompleta"
    print " - Caso {:<26} -> Texto: {:<28} | Color: {} | Blink: {}".format(
        ct["name"], res["status_text"], res["bg_color"], res["blink_animation"]
    )
print "[OK] format_equipment_status_card supera todos los casos nominales y de desconexion."


# -----------------------------------------------------------------------------
# TEST 3: Desacoplamiento de Eventos y Preparación de Payload (Lab 6.3)
# -----------------------------------------------------------------------------
print "\n--- [TEST 3] Desacoplamiento de Eventos y Broadcast ---"

ok3, payload3 = project.alarms.events.acknowledge_equipment_alarm(
    alarm_id="ALM-2026-99",
    equip="EDAR_COMPRESOR_01",
    username="operador.certificacion",
    reason="Prueba automatizada de broadcast",
    db_conn="SANDBOX_DB"
)

assert ok3 == True, "Fallo en servicio de reconocimiento de alarma"
assert payload3.get("alarm_id") == "ALM-2026-99", "Payload con alarm_id incorrecto"
assert payload3.get("equip") == "EDAR_COMPRESOR_01", "Payload con equipo incorrecto"
assert "ack_timestamp" in payload3, "Payload no incluye marca temporal"
print "[OK] acknowledge_equipment_alarm genera correctamente el payload para el bus de mensajes:"
for k, v in payload3.items():
    print "   * {}: {}".format(k, v)


# -----------------------------------------------------------------------------
# TEST 4: Pila Pura de Historial de Navegación
# -----------------------------------------------------------------------------
print "\n--- [TEST 4] Pila Pura de Historial (Limite FIFO max_items=5) ---"

pila = []
idx = 0
paginas = [
    {"title": "P1", "path": "/p1"},
    {"title": "P2", "path": "/p2"},
    {"title": "P3", "path": "/p3"},
    {"title": "P4", "path": "/p4"},
    {"title": "P5", "path": "/p5"},
    {"title": "P6", "path": "/p6"} # Debe desplazar a P1
]

for p in paginas:
    pila, idx = project.nav.history.calculate_history_slice(pila, idx, p, max_items=5)

assert len(pila) == 5, "La pila de historial debe contener exactamente 5 elementos"
assert pila[0]["path"] == "/p2", "Politica FIFO no respeto la eliminacion del mas antiguo"
assert pila[-1]["path"] == "/p6", "El elemento mas reciente no es P6"
print "[OK] calculate_history_slice gestiona correctamente la capacidad y orden de la pila."

print "\n" + "=" * 80
print "TODAS LAS PRUEBAS UNITARIAS DE LA FASE 6 HAN FINALIZADO CON EXITO"
print "=" * 80
```

---

## 5. Cuadro Comparativo de Beneficios

| Parámetro | Estado Anterior (Código Original) | Estado Refactorizado (Lab-06) | Impacto Técnico y Operativo |
| :--- | :--- | :--- | :--- |
| **Control Transaccional** | *Auto-commit* individual en cada consulta; sin transacciones compuestas. | Transacciones explícitas con `beginTransaction`, `commit` y `rollback`. | Eliminación total de inconsistencias de datos y registros huérfanos entre tablas. |
| **Gestión de Conexiones JDBC** | Sin control de liberación manual; consultas dispersas en la JVM. | Liberación incondicional garantizada con `closeTransaction(tx_id)` en `finally`. | Cero fugas de conexión en el pool JDBC del Gateway (*Zero pool starvation*). |
| **Rendimiento de UI (Perspective)** | Bindings con consultas SQL y lecturas masivas de tags en bucles bloqueantes. | *Script Transforms* puros en RAM (`project.ui.transforms`) con cero I/O. | Evaluación instantánea (sub-milisegundo), eliminando congelamientos visuales. |
| **Robustez ante Estados Nulos** | Clases CSS vacías (`""`) y excepciones `TypeError` al perder comunicación. | Mapeo cromático exhaustivo y manejo defensivo con estado `SIN COMUNICACION`. | Alta disponibilidad visual; el operador identifica inmediatamente desconexiones. |
| **Acoplamiento de Componentes** | Navegación jerárquica frágil `self.parent.parent.parent.getChild(...)`. | Arquitectura orientada a eventos con `system.perspective.sendMessage`. | Mantenimiento ágil; mover o rediseñar pantallas nunca rompe los scripts de refresco. |
| **Trazabilidad y Auditoría** | Llamadas ad-hoc desincronizadas de la lógica de negocio. | Persistencia de auditoría atómica vinculada a la misma transacción JDBC. | Cumplimiento normativo estricto; imposible operar sin registrar auditoría. |
| **Capacidad de Prueba** | Lógica de negocio incrustada en eventos visuales sin aislamiento. | Funciones puras y servicios desacoplados en `Project Library`. | Validación unitaria automatizada en Script Console antes de pasar a producción. |
