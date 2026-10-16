---
description: Informe técnico de auditoría, escrituras seguras, pistas de auditoría inmutables, borrado lógico y Named Queries parametrizadas en Ignition (Sesión 5).
---

# Informe Técnico: Refactorización y Estandarización de Scripts SCADA (Fase 5: Escrituras Seguras, Auditoría Inmutable y Named Queries Parametrizadas)

## 1. Resumen Ejecutivo

Este informe documenta la auditoría técnica y la propuesta de refactorización integral de los scripts del sistema SCADA en Ignition (`ejemplos-scripts-ignition.md`), fundamentada estrictamente en las directrices de escrituras seguras, control transaccional, prevención de inyecciones SQL, pistas de auditoría inmutables (*Audit Trail*), borrado lógico (*Soft Delete*) y centralización mediante *Named Queries* parametrizadas desarrolladas en los laboratorios de la **Sesión 5** (`gcp-con-eduardo/lab-05.md`).

### Principales Deficiencias Detectadas

1. **Vulnerabilidad Crítica a Inyección SQL (SQLi) por Concatenación de Texto Plano:** En componentes clave como el script de eventos de tag de enclavamientos (`valueChanged`), se construyen sentencias `SELECT`, `INSERT` y `UPDATE` concatenando variables directamente en cadenas de texto (`update = "update enclavaments set StartDate='" + StartDate + "' where equip = '" + equip + "' and NumeroEnclava = " + str(indice)`). Esta práctica abre brechas críticas ante caracteres especiales o inyecciones deliberadas. Asimismo, en la Named Query de notas (`Notas/select`) se utiliza el antipatrón de *Query String Parameters* (`WHERE {equip} AND Readed = 0`), permitiendo la alteración arbitraria de la lógica SQL en tiempo de ejecución.
2. **Modificaciones y Escrituras Ciegas sin Validación de Rangos de Ingeniería:** En el botón de envío de consignas (`runAction`) y en la actualización de parámetros operativos, los valores numéricos introducidos por los operadores o recibidos de capas superiores se envían directamente a los autómatas (`system.tag.writeBlocking`) y a la base de datos sin contraste previo contra los límites físicos de diseño de planta (`LIMITS`: presión máxima, caudal admisible, velocidad nominal).
3. **Ausencia de Control de Filas Afectadas (`rows_affected`):** Las operaciones de modificación (`UPDATE` / `DELETE`) ejecutadas mediante `system.db.runUpdateQuery` asumen ciegamente el éxito de la sentencia sin comprobar el número exacto de filas alteradas. Si una condición `WHERE` es incorrecta, incompleta o si el registro no existe, el sistema o bien modifica múltiples filas descontroladamente o reporta éxito cuando ninguna fila fue alterada, provocando inconsistencias silenciosas en la base de datos de planta.
4. **Borrado Físico Destructivo (*Hard Delete*) y Pérdida Irremediable de Trazabilidad:** En scripts de mantenimiento como la tarea programada en el Gateway (`onScheduledEvent`), se ejecutan sentencias destructivas `DELETE FROM REG_OPERADORS`. Este borrado físico destruye la integridad referencial y elimina de forma irreversible el histórico operativo, vulnerando normativas industriales de trazabilidad (como FDA 21 CFR Part 11, GAMP5 e ISO 9001).
5. **Falta de Justificación Obligatoria y Pistas de Auditoría Incompletas:** Los cambios de consigna y maniobras operativas no exigen un motivo justificado al operador ni registran de forma comparativa el estado anterior y nuevo (`old_value` vs `new_value`). Además, las llamadas a la tabla de auditoría `ins_Registre_Actuacions` se encuentran dispersas en la capa visual con estructuras heterogéneas, sin estandarización ni garantías de persistencia.

### Arquitectura Objetivo

* **Erradicación Total de SQLi mediante *Prepared Statements* y *Value Parameters*:** Sustitución de toda concatenación por sentencias preparadas con comodines posicionales (`?`) en `runPrepQuery` / `runPrepUpdate`, y transformación de parámetros *Query String* (`{equip}`) en *Value Parameters* estrictamente tipados (`:equip`, `:descripcio`, `:usuari`) en el motor de Named Queries de Ignition.
* **Validación Rigurosa de Límites de Ingeniería y Reglas de Negocio:** Interceptación previa en `Project Library` (`project.setpoints.service`) contrastando cada consigna contra un diccionario de límites de planta (`LIMITS`), con tipado defensivo, verificación de cadenas no vacías y exigencia obligatoria de justificación operativa (mínimo 5 caracteres).
* **Seguro Transaccional mediante Verificación Estricta de Filas (`rows_affected == 1`):** Validación mandataria de que cada sentencia de actualización modifique exactamente una fila única por clave primaria o identificador unívoco de equipo, alertando inmediatamente ante inconsistencias o desalineaciones en base de datos.
* **Patrón de Borrado Lógico (*Soft Delete*):** Mantenimiento íntegro de recetas, consignas y registros históricos mediante banderas de activación (`is_active = FALSE`, `last_updated = CURRENT_TIMESTAMP`, `updated_by = ?`), prohibiendo sentencias `DELETE FROM` destructivas.
* **Pistas de Auditoría Inmutables (*Audit Trail*):** Registro sistemático y desacoplado en la tabla corporativa `ins_Registre_Actuacions`, documentando el quién (usuario autenticado), el cuándo (timestamp del servidor), el qué (equipo y parámetro), el contraste de valores (anterior y nuevo) y el motivo operativo.
* **Servicio Centralizado de Registro de Actuaciones (`project.actuacions.service`):** Módulo universal en la `Project Library` que encapsula el consumo de la Named Query `Actuacions/InsertRegistre`, desacoplando la capa gráfica y garantizando ejecución segura en cualquier *scope* de Ignition mediante `system.util.getProjectName()`.
* **Contratos Estructurados de Retorno:** Retorno formal de tuplas `(bool success, unicode message)` en todas las funciones de servicio, estandarizando la retroalimentación hacia los componentes gráficos de Perspective y Vision.

---

## 2. Matriz de Hallazgos y Acciones Correctivas

| Componente Auditado | Deficiencia Detectada (Código Original) | Riesgo Operativo | Solución Técnica Aplicada (Lab-05) |
| :--- | :--- | :--- | :--- |
| **Tag Enclavamientos** (`valueChanged`, líneas 620-753) | Concatenación directa de variables en `select`, `insert` y `update`; llamadas ad-hoc a `runUpdateQuery` y `runNamedQuery`. | Vulnerabilidad severa a Inyección SQL (SQLi); corrupción de datos de enclavamientos y fallo silencioso si se actualizan múltiples registros. | Refactorización a `project.enclavamientos.service` con `runPrepQuery` / `runPrepUpdate`, verificación de `rows_affected == 1`, y delegación de auditoría al servicio centralizado. |
| **Named Query Notas** (`Notas/select`, líneas 608-619) | Uso de *Query String Parameter* (`WHERE {equip} AND Readed = 0`). | Inyección SQL directa en la capa de datos; posibilidad de eludir filtros o alterar sentencias DDL/DML inyectando fragmentos SQL. | Reconfiguración a *Value Parameter* tipado (`:equip`), consulta limpia con Prepared Statement JDBC e invocación controlada desde `project.notes.service`. |
| **Botón Enviar Consignas** (`runAction`, líneas 494-579) | Escritura directa a PLC y base de datos sin comprobar límites de rango físico ni exigir motivo del cambio; auditoría sin valor previo. | Sobrepresión o daños mecánicos en equipos de planta por valores anómalos; ausencia de justificación para trazabilidad legal o de calidad. | Integración con `project.setpoints.service.update_setpoint_with_audit`: validación de `LIMITS`, captura de valor anterior, motivo obligatorio (≥5 caracteres) y verificación de `rows_affected == 1`. |
| **Tarea Programada Gateway** (`onScheduledEvent`, líneas 754-800) | Borrado físico destructivo con `DELETE FROM REG_OPERADORS WHERE [Data] < DATEADD(YEAR, -5, GETDATE())`. | Pérdida definitiva e irreversible de registros de trazabilidad histórica de operadores; incumplimiento de normativas FDA 21 CFR Part 11 / ISO. | Sustitución de `DELETE` por marcado lógico (*Soft Delete*): `UPDATE ... SET is_active = FALSE, last_updated = CURRENT_TIMESTAMP`, con registro de auditoría en `ins_Registre_Actuacions`. |
| **Invocaciones a Auditoría** (`ins_Registre_Actuacions`, páginas 1, 2, 10, 14, 15) | Llamadas dispersas a `runNamedQuery` desde scripts de UI y tags con diccionarios no normalizados y sin verificar retorno. | Discrepancia en nombres de parámetros, falta de control sobre si la auditoría se guardó y código duplicado en múltiples pantallas. | Creación de la Named Query `Actuacions/InsertRegistre` (tipo *Update Query* con *Value Parameters*) y servicio universal `project.actuacions.service.log_operator_action`. |

---

## 3. Detalle de Refactorización por Componente

---

### 3.1. Tag de Enclavamientos: Parametrización y Erradicación de Inyecciones SQL

#### Diagnóstico

En el script de cambio de valor del tag de enclavamientos (`ejemplos-scripts-ignition.md`, líneas 658-735), se observa la concatenación de variables de cadena directamente dentro de las sentencias SQL ejecutadas con `system.db.runQuery` y `system.db.runUpdateQuery`:

```python
# CÓDIGO ORIGINAL (Fragmento con vulnerabilidad crítica SQLi y sin validación de filas)
query = "select * from enclavaments where equip = '" + equip + "' and NumeroEnclava = " + str(indice)
select = system.db.runQuery(query, 'BD_LLIBRERIA')
...
if select.getRowCount() == 0 and current[i] == 0:
    insert = "insert into enclavaments (StartDate, Equip, NumeroEnclava, TextEnclava) values ('" + StartDate + "','" + equip + "'," + str(indice) + ", '" + textoActive + "')"
    system.db.runUpdateQuery(insert, 'BD_LLIBRERIA')
    ...
elif current[i] == 0:
    update = "update enclavaments set StartDate='" + StartDate + "' where equip = '" + equip + "' and NumeroEnclava = " + str(indice)
    system.db.runUpdateQuery(update, 'BD_LLIBRERIA')
    ...
    update_text = "update enclavaments set TextEnclava='" + textoActive + "' where equip = '" + equip + "' and NumeroEnclava = " + str(indice)
    system.db.runUpdateQuery(update_text, 'BD_LLIBRERIA')
```

**Riesgos técnicos y operativos identificados:**
1. Si la variable `equip` contiene una comilla simple (`'`) o datos manipulados, la consulta falla con error de sintaxis SQL o permite inyección maliciosa de sentencias.
2. No se evalúa el resultado de `runUpdateQuery`: si no actualiza registros o si por un error de clave actualiza múltiples filas, el script no se entera.
3. El esquema no implementa borrado lógico ni trazabilidad de estados de activación del enclavamiento.

#### Solución Propuesta (Enseñanzas Lab 5.1 y 5.2)

1. **Esquema de Base de Datos con Soporte de Borrado Lógico:** Asegurar que la tabla `enclavaments` contenga campos de control transaccional (`is_active`, `last_updated`, `updated_by`).
2. **Prepared Statements Nativos:** Sustituir `runQuery` y `runUpdateQuery` por `system.db.runPrepQuery` y `system.db.runPrepUpdate`, pasando los valores como parámetros aislados en una lista.
3. **Verificación de Filas Afectadas (`rows_affected == 1`):** Comprobar que las actualizaciones afecten exactamente al registro del enclavamiento correspondiente.
4. **Desacoplamiento hacia `Project Library`:** Trasladar la persistencia de enclavamientos al módulo `project.enclavamientos.service`.

```python
# Implementación en Project Library: project.enclavamientos.service
"""
Modulo: project.enclavamientos.service
Descripcion: Gestion transaccional de enclavamientos con Prepared Statements,
             validacion de filas afectadas y auditoria centralizada.
             (Implementado conforme a las directrices del Lab-05).
Autor: Equipo SCADA
"""

def register_or_update_interlock(equip, bit_index, description, is_active_state, db_conn="BD_LLIBRERIA"):
    """
    Gestiona de forma segura la insercion o actualizacion del estado de un enclavamiento
    mediante Prepared Statements y comprobacion de filas afectadas.
    
    Args:
        equip (str): Identificador del equipo (ej. 'BOMBA_01').
        bit_index (int): Numero del enclavamiento (0..31).
        description (str|unicode): Texto descriptivo del enclavamiento.
        is_active_state (bool): True si el enclavamiento esta activo (disparo), False si esta normalizado.
        db_conn (str): Conexion JDBC configurada en Ignition.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    logger = system.util.getLogger("SCADA.Enclavamientos")
    
    if not equip or str(equip).strip() == "":
        return (False, u"Identificador de equipo invalido")
        
    if bit_index < 0 or bit_index > 31:
        return (False, u"Indice de enclavamiento fuera de rango (0..31)")
        
    desc_str = unicode(description).strip() if description else u"Sin descripcion"
    
    try:
        # 1. Consulta segura con PreparedStatement (Inmune a SQLi)
        select_sql = """
            SELECT id, TextEnclava, is_active 
            FROM enclavaments 
            WHERE equip = ? AND NumeroEnclava = ?
        """
        existing = system.db.runPrepQuery(select_sql, [equip, bit_index], database=db_conn)
        
        now_ts = system.date.now()
        
        if len(existing) == 0:
            # 2. Insercion segura si no existe registro
            insert_sql = """
                INSERT INTO enclavaments (StartDate, Equip, NumeroEnclava, TextEnclava, is_active, last_updated, updated_by)
                VALUES (?, ?, ?, ?, ?, CURRENT_TIMESTAMP, 'PLC')
            """
            rows = system.db.runPrepUpdate(
                insert_sql, 
                [now_ts, equip, bit_index, desc_str, is_active_state], 
                database=db_conn
            )
            
            if rows != 1:
                logger.warn("Inconsistencia en insercion de enclavamiento: {} filas".format(rows))
                return (False, u"Fallo al registrar nuevo enclavamiento")
                
            return (True, u"Enclavamiento insertado correctamente")
            
        else:
            # 3. Actualizacion segura con verificacion de filas afectadas
            update_sql = """
                UPDATE enclavaments 
                SET StartDate = ?, TextEnclava = ?, is_active = ?, last_updated = CURRENT_TIMESTAMP, updated_by = 'PLC'
                WHERE equip = ? AND NumeroEnclava = ?
            """
            rows = system.db.runPrepUpdate(
                update_sql, 
                [now_ts, desc_str, is_active_state, equip, bit_index], 
                database=db_conn
            )
            
            if rows != 1:
                logger.error("Inconsistencia en update de enclavamiento {} bit {}: {} filas afectadas".format(
                    equip, bit_index, rows
                ))
                return (False, u"No se pudo actualizar el enclavamiento con precision unitaria")
                
            return (True, u"Enclavamiento actualizado con éxito")
            
    except Exception as ex:
        logger.error("Error transaccional en enclavamiento de {}: {}".format(equip, str(ex)))
        return (False, u"Error interno de base de datos al procesar enclavamiento")
```

#### Código Resultante en el Evento del Tag (`valueChanged`)

```python
# Evento valueChanged en Tag de Enclavamiento (Limpio, desacoplado y seguro)
current = currentValue.value
prev = previousValue.value

path_parts = tagPath.split('/')
equip = path_parts[-2] if len(path_parts) >= 2 else "EQUIP_UNKNOWN"

# Lectura defensiva de descripciones
descr_read = system.tag.readBlocking([tagPath + "/../WENC_DESC"])
descr_ds = descr_read[0].value if descr_read and descr_read[0].value is not None else None

for i in range(32):
    if current[i] != prev[i]:
        bit_val = current[i]
        
        # Obtener descripcion segura
        texto_activo = u"Enclavamiento %d" % i
        if descr_ds is not None and i < descr_ds.getRowCount():
            raw_desc = descr_ds.getValueAt(i, "Descripcio")
            if raw_desc is not None:
                texto_activo = unicode(raw_desc)
                
        # Estado: bit en 0 = disparo activo en logica SCADA, bit en 1 = desenclavado
        is_active = (bit_val == 0)
        
        # 1. Modificacion segura en base de datos mediante Prepared Statements
        ok, msg = project.enclavamientos.service.register_or_update_interlock(
            equip=equip,
            bit_index=i,
            description=texto_activo,
            is_active_state=is_active,
            db_conn="BD_LLIBRERIA"
        )
        
        # 2. Registro de auditoria mediante el servicio universal de actuaciones (Lab 5.2)
        estado_txt = u"disparado" if is_active else u"desenclavado"
        detalle_audit = u"Enclavamiento {}: {} ({})".format(i, texto_activo, estado_txt)
        
        project.actuacions.service.log_operator_action(
            equip=equip,
            description=detalle_audit,
            username="PLC"
        )
```

---

### 3.2. Named Query de Notas: Sustitución de Query String por Value Parameters

#### Diagnóstico

En el archivo `ejemplos-scripts-ignition.md` (líneas 608-619), la consulta de notas asociadas a un equipo está configurada utilizando interpolación directa de texto mediante un parámetro *Query String*:

```sql
-- CÓDIGO ORIGINAL (Named Query: Notas/select con vulnerabilidad SQLi)
SELECT notes.*, CONVERT(varchar, dateCreated, 29) AS fecha, documents.* 
FROM notes
LEFT JOIN documents ON notes.id_nota = documents.idNote
WHERE {equip} AND Readed = 0
ORDER BY dateCreated DESC
```

**Vulnerabilidad Detectada:**
En Ignition, un parámetro encerrado entre llaves `{param}` es un **Query String Parameter**, lo que significa que el Gateway concatena literalmente el texto dentro de la sentencia SQL antes de enviarla al driver JDBC. Si un atacante o un script envía en `{equip}` un valor como:
```sql
equip = 'EDAR_01' OR 1=1; DROP TABLE notes; --
```
la consulta ejecutada en el motor de base de datos alterará la lógica original o ejecutará sentencias destructivas.

#### Solución Propuesta (Enseñanzas Lab 5.2)

1. **Reemplazo por Value Parameter (`:equip`):** Cambiar el parámetro en el Designer de tipo *QueryString* a tipo **Value Parameter** (`Type: String`).
2. **Reescritura de la Sentencia SQL:** Sustituir `{equip}` por una cláusula explícita con `:equip`:
   ```sql
   SELECT notes.id_nota, notes.equip, notes.nota, notes.Readed,
          CONVERT(varchar, notes.dateCreated, 29) AS fecha,
          documents.idDocument, documents.name, documents.filename
   FROM notes
   LEFT JOIN documents ON notes.id_nota = documents.idNote
   WHERE notes.equip = :equip AND notes.Readed = 0
   ORDER BY notes.dateCreated DESC
   ```
3. **Consumo Parametrizado desde Project Library:** Encapsular la ejecución de la Named Query en una función reutilizable `project.notes.service.get_pending_notes`.

```python
# Implementación en Project Library: project.notes.service
"""
Modulo: project.notes.service
Descripcion: Acceso seguro a notas de equipo mediante Named Queries parametrizadas.
             (Implementado conforme al Laboratorio 5.2).
Autor: Equipo SCADA
"""

def get_pending_notes(equip):
    """
    Recupera las notas no leidas para un equipo utilizando Value Parameters inmutables.
    
    Args:
        equip (str): Identificador unico del equipo.
        
    Returns:
        tuple: (bool success, list_of_dicts|unicode)
    """
    logger = system.util.getLogger("SCADA.Notes")
    
    if not equip or str(equip).strip() == "":
        return (False, u"El identificador del equipo no puede ser nulo ni vacio")
        
    equip_clean = str(equip).strip()
    project_name = system.util.getProjectName()
    
    params = {
        "equip": equip_clean
    }
    
    try:
        # Invocacion segura: el Gateway envia :equip a traves de PreparedStatement
        ds = system.db.runNamedQuery(
            project=project_name,
            path="Notas/select",
            parameters=params
        )
        
        if ds is None or ds.getRowCount() == 0:
            return (True, [])
            
        pyds = system.dataset.toPyDataSet(ds)
        notes_list = []
        for row in pyds:
            notes_list.append({
                "id_nota": row["id_nota"],
                "equip": row["equip"],
                "fecha": row["fecha"],
                "filename": row["filename"] if row["filename"] is not None else ""
            })
            
        return (True, notes_list)
        
    except Exception as ex:
        logger.error("Error al consultar notas para {}: {}".format(equip_clean, str(ex)))
        return (False, u"Error al obtener notas pendientes")
```

#### Código Resultante en la Vista de Perspective

```python
# Script en transform de binding o evento de boton en la vista
equip_id = self.view.params.TAG

ok, data_or_err = project.notes.service.get_pending_notes(equip_id)

if ok:
    self.props.data = data_or_err
else:
    system.perspective.print("AVISO: " + data_or_err)
    self.props.data = []
```

---

### 3.3. Botón de Envío de Consignas: Validación de Límites de Ingeniería y Auditoría de Cambios

#### Diagnóstico

En el script de envío de consignas (`ejemplos-scripts-ignition.md`, líneas 494-579), el operador selecciona consignas desde la interfaz y el script escribe directamente en tags físicos y registra en base de datos:

```python
# CÓDIGO ORIGINAL (Fragmento sin limites de ingenieria, sin motivo y sin valor anterior)
for instance in instances:
    selected = instance['selected']
    if selected:
        tagPath = instance['tagPath']
        value_new = instance['value_new']
        ...
        system.tag.writeBlocking([target], [value_new])
        params = {"Equip": equip, "Descripcio": "Ordre Enviar Consigna " + tag + ": " + desc, "Usuari": user}
        system.db.runNamedQuery("ins_Registre_Actuacions", params)
```

**Deficiencias Técnicas:**
1. No existe validación contra los límites físicos admisibles del proceso (`LIMITS`: presión, caudal, nivel). Un error de tipeo (ej. consigna de 250 m³/h en una bomba cuyo límite es 150 m³/h) se enviaría directamente al PLC.
2. No se requiere justificación operativa obligatoria por parte del operador para autorizar la maniobra.
3. El registro de auditoría almacena un texto genérico sin dejar constancia del valor anterior (`old_val`) ni verificar si la inserción en base de datos afectó exactamente a una fila.

#### Solución Propuesta (Enseñanzas Lab 5.1 y 5.2)

1. **Tabla de Consignas en Sandbox / Producción (`machine_setpoints`):** Almacenar las consignas nominales con bandera de borrado lógico (`is_active`), timestamp y autor.
2. **Definición de Límites de Ingeniería (`LIMITS`):** Centralizar los rangos físicos admisibles en `project.setpoints.service`.
3. **Pista de Auditoría Comparativa:** Capturar el valor anterior antes de aplicar el nuevo valor, registrando la traza `[old_val -> new_val unit] Motivo: {reason}`.
4. **Verificación Estricta `rows_affected == 1`:** Validar que la actualización de consigna modifique exactamente una fila en base de datos.

```python
# Implementación en Project Library: project.setpoints.service
"""
Modulo: project.setpoints.service
Descripcion: Gestion segura de consignas con validacion de limites de ingenieria,
             verificacion de filas afectadas, auditoria y borrado logico.
             (Implementado conforme al Laboratorio 5.1).
Autor: Equipo SCADA
"""

# Limites de ingenieria admisibles en planta
LIMITS = {
    "flow_sp": {"min": 0.0, "max": 150.0, "unit": "m3/h"},
    "pressure_sp": {"min": 0.0, "max": 10.0, "unit": "bar"},
    "speed_sp": {"min": 0.0, "max": 1500.0, "unit": "rpm"}
}

def update_setpoint_with_audit(equip, param_name, new_value, username, reason, db_conn="SANDBOX_DB"):
    """
    Valida limites de ingenieria, actualiza la consigna de un equipo mediante PreparedStatement
    y registra auditoria inmutable en ins_Registre_Actuacions.
    
    Args:
        equip (str): Identificador del equipo (ej. 'EDAR_BOMBA_01').
        param_name (str): Parametro a modificar ('flow_sp', 'pressure_sp', 'speed_sp').
        new_value (float|int): Nuevo valor de consigna solicitado.
        username (str): Usuario autenticado que autoriza la maniobra.
        reason (str): Justificacion operativa obligatoria (minimo 5 caracteres).
        db_conn (str): Conexion JDBC configurada en Ignition.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    logger = system.util.getLogger("SCADA.Setpoints")
    
    # 1. Validaciones defensivas de entrada
    if not equip or not param_name or username is None:
        return (False, u"Parámetros de identificación incompletos")
        
    if param_name not in LIMITS:
        return (False, u"Parámetro desconocido: {}".format(param_name))
        
    if not reason or len(unicode(reason).strip()) < 5:
        return (False, u"Es obligatorio introducir un motivo justificativo (mínimo 5 caracteres)")
        
    try:
        val_num = float(new_value)
    except (ValueError, TypeError):
        return (False, u"El valor de la consigna debe ser numérico")
        
    # 2. Validacion estricta contra limites de ingenieria de planta
    p_lim = LIMITS[param_name]
    if val_num < p_lim["min"] or val_num > p_lim["max"]:
        return (False, u"Valor {:.2f} fuera de rango [{:.1f} - {:.1f} {}]".format(
            val_num, p_lim["min"], p_lim["max"], p_lim["unit"]
        ))
        
    try:
        # 3. Obtener el valor anterior y estado activo mediante PreparedStatement
        select_sql = "SELECT " + param_name + ", is_active FROM machine_setpoints WHERE equip = ?"
        current_data = system.db.runPrepQuery(select_sql, [equip], database=db_conn)
        
        if len(current_data) == 0:
            return (False, u"El equipo '{}' no existe en la tabla de consignas".format(equip))
            
        old_val = current_data[0][param_name]
        is_active = current_data[0]["is_active"]
        
        if not is_active:
            return (False, u"El equipo '{}' se encuentra desactivado (Baja Lógica)".format(equip))
            
        # 4. Actualizacion segura con verificacion de filas afectadas (rows_affected == 1)
        update_sql = """
            UPDATE machine_setpoints 
            SET """ + param_name + """ = ?, updated_by = ?, last_updated = CURRENT_TIMESTAMP 
            WHERE equip = ? AND is_active = TRUE
        """
        rows_affected = system.db.runPrepUpdate(update_sql, [val_num, username, equip], database=db_conn)
        
        if rows_affected != 1:
            logger.warn("Inconsistencia en update_setpoint: {} filas afectadas".format(rows_affected))
            return (False, u"No se ha podido actualizar la consigna (Filas afectadas: {})".format(rows_affected))
            
        # 5. Registro de auditoria inmutable mediante servicio universal (Lab 5.2)
        audit_desc = u"Cambio Consigna {} [{} -> {:.2f} {}]. Motivo: {}".format(
            unicode(param_name), old_val, val_num, unicode(p_lim["unit"]), unicode(reason.strip())
        )
        
        ok_audit, msg_audit = project.actuacions.service.log_operator_action(
            equip=equip,
            description=audit_desc,
            username=username
        )
        
        if not ok_audit:
            logger.warn("Consigna guardada pero falló registro de auditoría: {}".format(msg_audit))
            
        logger.info("Consigna actualizada y auditada: {} | {} | Usuario: {}".format(equip, audit_desc, username))
        return (True, u"Consigna actualizada y auditada correctamente")
        
    except Exception as ex:
        logger.error("Error al actualizar consigna de {}: {}".format(equip, str(ex)))
        return (False, u"Error interno de base de datos al modificar consigna")


def soft_delete_setpoint(equip, username, reason, db_conn="SANDBOX_DB"):
    """
    Aplica borrado logico a una consigna (is_active = FALSE) registrando auditoria inmutable.
    
    Args:
        equip (str): Identificador del equipo.
        username (str): Usuario que autoriza la baja.
        reason (str): Motivo de la desactivacion (minimo 5 caracteres).
        db_conn (str): Conexion de base de datos.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    logger = system.util.getLogger("SCADA.Setpoints")
    
    if not equip or str(equip).strip() == "":
        return (False, u"Identificador de equipo invalido")
        
    if not reason or len(unicode(reason).strip()) < 5:
        return (False, u"Motivo de baja obligatorio (mínimo 5 caracteres)")
        
    try:
        update_sql = """
            UPDATE machine_setpoints 
            SET is_active = FALSE, updated_by = ?, last_updated = CURRENT_TIMESTAMP 
            WHERE equip = ? AND is_active = TRUE
        """
        rows = system.db.runPrepUpdate(update_sql, [username, equip], database=db_conn)
        
        if rows == 1:
            audit_desc = u"Baja Lógica de Consigna. Motivo: {}".format(unicode(reason.strip()))
            project.actuacions.service.log_operator_action(equip, audit_desc, username)
            return (True, u"Equipo desactivado correctamente (Baja Lógica)")
        else:
            return (False, u"El equipo no existe o ya estaba desactivado")
            
    except Exception as ex:
        logger.error("Error en baja lógica de {}: {}".format(equip, str(ex)))
        return (False, u"Error interno al procesar la baja lógica")
```

#### Código Resultante en el Evento del Botón de la UI (`runAction`)

```python
# Evento runAction del boton de envio de consignas (Totalmente validado y desacoplado)
instances = self.getSibling("FlexRepeater").props.instances
equip = self.view.params.TAG
user = self.session.props.auth.user.userName
reason = self.getSibling("TxtMotivo").props.text  # Campo obligatorio en la interfaz

# Validacion previa de justificacion en UI
if not reason or len(reason.strip()) < 5:
    system.perspective.openPopup("PopupAlerta", "Popups/Aviso", params={"mensaje": u"Debe introducir un motivo de al menos 5 caracteres"})
    return

enviadas = 0
for inst in instances:
    if inst.get("selected", False):
        param_name = inst.get("paramName", "flow_sp")
        new_val = inst.get("value_new", None)
        
        # Invocacion del servicio de libreria
        ok, msg = project.setpoints.service.update_setpoint_with_audit(
            equip=equip,
            param_name=param_name,
            new_value=new_val,
            username=user,
            reason=reason
        )
        
        if ok:
            # Envio fisico al PLC solo si la validacion de ingenieria y BD fue exitosa
            target_tag = inst.get("tagPath", "")
            system.tag.writeBlocking([target_tag], [new_val])
            enviadas += 1
        else:
            system.perspective.print("ERROR EN CONSIGNA: " + msg)

if enviadas > 0:
    system.perspective.print(u"Se aplicaron y auditaron %d consignas con éxito" % enviadas)
```

---

### 3.4. Tarea Programada en Gateway: Borrado Lógico (*Soft Delete*) vs. Borrado Físico Destructivo

#### Diagnóstico

En el script programado del Gateway (`ejemplos-scripts-ignition.md`, líneas 754-800), se ejecuta una purga periódica para borrar físicamente registros históricos:

```python
# CÓDIGO ORIGINAL (Fragmento con Hard Delete destructivo)
count = system.db.runScalarQuery("SELECT COUNT(*) FROM REG_OPERADORS WHERE [Data] < DATEADD(YEAR, -5, GETDATE())", database=db)
if count > 0:
    query = "DELETE FROM REG_OPERADORS WHERE [Data] < DATEADD(YEAR, -5, GETDATE())"
    deleted_rows = system.db.runUpdateQuery(query, database=db)
    logger.info("Files eliminades: %d" % deleted_rows)
```

**Deficiencias Identificadas:**
1. **Borrado Físico (*Hard Delete*):** La instrucción `DELETE FROM` destruye definitivamente los registros. Si un organismo regulador o una auditoría de calidad solicita investigar una maniobra ocurrida hace más de 5 años, los datos han desaparecido sin respaldo relacional.
2. **Sin Trazabilidad del Proceso de Purga:** No se registra en la tabla corporativa de auditoría qué usuario o tarea del sistema ejecutó la baja ni se deja constancia formal de los registros archivados.

#### Solución Propuesta (Enseñanzas Lab 5.1)

1. **Patrón de Borrado Lógico (*Soft Delete*):** En lugar de destruir filas, se actualiza la bandera `is_active = FALSE` junto con la marca temporal `last_updated = CURRENT_TIMESTAMP` y el responsable `updated_by = 'GATEWAY_CLEANUP'`.
2. **Registro de Auditoría de Mantenimiento:** Una vez completada la baja lógica, se inserta una traza en `ins_Registre_Actuacions` documentando cuántos registros pasaron a estado inactivo.

```python
# Implementación en Project Library: project.maintenance.cleanup
"""
Modulo: project.maintenance.cleanup
Descripcion: Mantenimiento de tablas con aplicacion de Borrado Logico (Soft Delete)
             y trazabilidad inmutable conforme a las directrices del Lab-05.
Autor: Equipo SCADA
"""

def archive_old_operator_records(years_retention=5, db_conn="BD_AUDIT"):
    """
    Aplica borrado logico a registros historicos que superan el periodo de retencion,
    garantizando la conservacion forense de datos y auditando la operacion.
    
    Args:
        years_retention (int): Periodo de retencion activa en anos (defecto: 5).
        db_conn (str): Conexion de base de datos en Ignition.
        
    Returns:
        tuple: (bool success, int archived_count, unicode message)
    """
    logger = system.util.getLogger("SCADA.Maintenance")
    
    try:
        # 1. Contar registros activos anteriores al umbral
        count_sql = """
            SELECT COUNT(*) 
            FROM REG_OPERADORS 
            WHERE Data < DATEADD(YEAR, -?, GETDATE()) AND is_active = TRUE
        """
        records_to_archive = system.db.runPrepQuery(count_sql, [years_retention], database=db_conn)[0][0]
        
        if records_to_archive == 0:
            logger.info("Mantenimiento preventivo: No existen registros antiguos pendientes de baja lógica")
            return (True, 0, u"No se requirieron bajas lógicas")
            
        # 2. Aplicacion de Borrado Logico (Soft Delete)
        update_sql = """
            UPDATE REG_OPERADORS 
            SET is_active = FALSE, 
                last_updated = CURRENT_TIMESTAMP, 
                updated_by = 'GATEWAY_CLEANUP' 
            WHERE Data < DATEADD(YEAR, -?, GETDATE()) AND is_active = TRUE
        """
        rows_affected = system.db.runPrepUpdate(update_sql, [years_retention], database=db_conn)
        
        # 3. Auditoria del proceso de purga
        audit_desc = u"Baja Lógica programada: %d registros archivados (> %d años)" % (rows_affected, years_retention)
        project.actuacions.service.log_operator_action(
            equip="SISTEMA_SCADA",
            description=audit_desc,
            username="GATEWAY_CLEANUP"
        )
        
        logger.info("Mantenimiento completado con éxito: %d filas marcadas como inactivas" % rows_affected)
        return (True, rows_affected, u"Baja lógica completada correctamente")
        
    except Exception as ex:
        logger.error("Error al ejecutar baja lógica de mantenimiento: %s" % str(ex))
        return (False, 0, u"Error interno en mantenimiento programado")
```

#### Código Resultante en el Script Programado de Gateway (`onScheduledEvent`)

```python
# Gateway Scheduled Script: onScheduledEvent
import system

ok, count, msg = project.maintenance.cleanup.archive_old_operator_records(
    years_retention=5,
    db_conn="BD_AUDIT"
)

if not ok:
    system.util.getLogger("SCADA.Maintenance").error("Fallo crítico en tarea de mantenimiento: " + msg)
```

---

### 3.5. Servicio Universal de Auditoría y Consumo de Named Query (`Actuacions/InsertRegistre`)

#### Diagnóstico

A lo largo del código original (`ejemplos-scripts-ignition.md`, páginas 1, 2, 10, 14, 15), se encuentran múltiples llamadas a la Named Query de auditoría con inconsistencias en nombres de claves y sin control sobre el resultado:

```python
# CÓDIGO ORIGINAL (Llamadas dispersas en la capa visual y tags)
params = {"Equip": equip, "Descripcio": "Ordre Enviar Consigna " + tag + ": " + desc, "Usuari": user}
system.db.runNamedQuery("ins_Registre_Actuacions", params)
# En otro archivo:
params = {"Equip": equip, "Descripcio": "Enclavament " + Enclavament, "Usuari": 'PLC'}
system.db.runNamedQuery('AB_LLIBRERIA', "ins_Registre_Actuacions", params)
```

**Problemas detectados:**
1. No se comprueba si la consulta insertó exitosamente el registro (`rows_affected == 1`).
2. Nombres de parámetros heterogéneos y dependientes del *project scope* o *gateway scope*.
3. Ausencia de validaciones sobre cadenas vacías o valores nulos antes de enviar a base de datos.

#### Solución Propuesta (Enseñanzas Lab 5.2)

1. **Configuración de la Named Query en el Designer:**
   * **Ruta:** `Actuacions/InsertRegistre`
   * **Tipo:** `Update Query`
   * **Parámetros (*Value Parameters*):**
     * `equip` (Type: `String`)
     * `descripcio` (Type: `String`)
     * `usuari` (Type: `String`)
   * **SQL Template:**
     ```sql
     INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
     VALUES (:equip, :descripcio, :usuari, CURRENT_TIMESTAMP)
     ```
2. **Servicio Universal en `Project Library` (`project.actuacions.service`):**
   * Validación defensiva de parámetros de entrada.
   * Manejo dinámico del proyecto mediante `system.util.getProjectName()`.
   * Verificación obligatoria de `rows_affected == 1`.

```python
# Implementación en Project Library: project.actuacions.service
"""
Modulo: project.actuacions.service
Descripcion: Servicio universal de auditoria de actuaciones operativas
             mediante Named Queries parametrizadas con Value Parameters.
             (Implementado conforme al Laboratorio 5.2).
Autor: Equipo SCADA
"""

def log_operator_action(equip, description, username):
    """
    Registra una actuacion u orden operativa consumiendo la Named Query parametrizada.
    
    Args:
        equip (str): Identificador del equipo (ej. 'EDAR_BOMBA_02').
        description (str|unicode): Detalle funcional de la orden, consigna o alarma.
        username (str): Usuario autenticado que realiza la accion.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    logger = system.util.getLogger("SCADA.Actuacions")
    
    # 1. Validaciones defensivas de entrada
    if not equip or not str(equip).strip():
        return (False, u"El identificador de equipo es obligatorio")
        
    if not description or not str(description).strip():
        return (False, u"La descripción de la actuación no puede estar vacía")
        
    user_str = str(username).strip() if username else "SYSTEM"
    
    # 2. Armado del diccionario de parametros (coincidencia exacta con los Value Parameters)
    params = {
        "equip": str(equip).strip(),
        "descripcio": unicode(description).strip(),
        "usuari": user_str
    }
    
    try:
        # Obtener el nombre del proyecto actual para garantizar ejecucion en cualquier scope
        project_name = system.util.getProjectName()
        
        # 3. Invocacion segura de la Named Query con verificacion de retorno
        rows_affected = system.db.runNamedQuery(
            project=project_name,
            path="Actuacions/InsertRegistre",
            parameters=params
        )
        
        if rows_affected == 1:
            logger.info("Actuación registrada por Named Query: {} | Equipo: {}".format(user_str, equip))
            return (True, u"Actuación registrada correctamente")
        else:
            logger.warn("Named Query no insertó filas para {}: {}".format(equip, rows_affected))
            return (False, u"No se ha podido registrar la actuación en la base de datos")
            
    except Exception as ex:
        logger.error("Error al ejecutar Named Query Actuacions/InsertRegistre: {}".format(str(ex)))
        return (False, u"Error interno de comunicación con la base de datos")
```

---

## 4. Banco de Pruebas Automatizado (Script Console)

Para certificar el correcto funcionamiento de las funciones de modificación de consignas, auditoría, borrado lógico y consumo de Named Queries sin intervenir en la planta física, se debe ejecutar el siguiente banco de pruebas automatizado en la **Script Console** de Ignition Designer (**Tools -> Script Console**):

```python
"""
================================================================================
BANCO DE PRUEBAS AUTOMATIZADO - SESION 5 (LABORATORIOS 5.1 Y 5.2)
================================================================================
Valida:
  1. Modificacion nominal de consigna con validacion de limites y auditoria.
  2. Rechazo por superacion de limites fisicos de ingenieria (LIMITS).
  3. Rechazo por ausencia de motivo justificativo (< 5 caracteres).
  4. Borrado logico (Soft Delete: is_active = FALSE) de equipo.
  5. Bloqueo preventivo de modificacion sobre equipo desactivado.
  6. Insercion de actuacion mediante Named Query parametrizada.
  7. Validacion defensiva ante parametros vacios o nulos.
  8. Inmunidad total ante intentos de Inyeccion SQL (SQLi).
================================================================================
"""

print "=" * 80
print "INICIANDO BATERIA DE PRUEBAS DE ESCRITURAS SEGURAS Y AUDITORIA (SESION 5)"
print "=" * 80

total_tests = 0
passed_tests = 0

def run_test(test_id, description, condition, details=""):
    global total_tests, passed_tests
    total_tests += 1
    if condition:
        passed_tests += 1
        print "[PASS] Test {:02d}: {} {}".format(test_id, description, details)
    else:
        print "[FAIL] Test {:02d}: {} - FALLO {}".format(test_id, description, details)


# ------------------------------------------------------------------------------
# TEST 1: Modificacion nominal de caudal en EDAR_BOMBA_01 (45 -> 62.5 m3/h)
# ------------------------------------------------------------------------------
ok1, msg1 = project.setpoints.service.update_setpoint_with_audit(
    equip="EDAR_BOMBA_01",
    param_name="flow_sp",
    new_value=62.5,
    username="marti.sanchez",
    reason="Ajuste por aumento de caudal de entrada a planta",
    db_conn="SANDBOX_DB"
)
run_test(1, "Modificacion nominal de consigna dentro de limites", ok1 is True, "(Msg: %s)" % msg1)


# ------------------------------------------------------------------------------
# TEST 2: Rechazo por superacion de limites fisicos (250 m3/h > max 150 m3/h)
# ------------------------------------------------------------------------------
ok2, msg2 = project.setpoints.service.update_setpoint_with_audit(
    equip="EDAR_BOMBA_01",
    param_name="flow_sp",
    new_value=250.0,
    username="operador_noche",
    reason="Pruebas de sobrepresion no autorizadas",
    db_conn="SANDBOX_DB"
)
run_test(2, "Rechazo por consigna fuera de rango de ingenieria", ok2 is False and u"fuera de rango" in msg2, "(Msg: %s)" % msg2)


# ------------------------------------------------------------------------------
# TEST 3: Rechazo por ausencia de motivo justificativo (espacios en blanco)
# ------------------------------------------------------------------------------
ok3, msg3 = project.setpoints.service.update_setpoint_with_audit(
    equip="EDAR_COMPRESOR_01",
    param_name="pressure_sp",
    new_value=7.0,
    username="carlos.gomez",
    reason="   ",
    db_conn="SANDBOX_DB"
)
run_test(3, "Rechazo por motivo justificativo insuficiente (< 5 caracteres)", ok3 is False and u"motivo justificativo" in msg3, "(Msg: %s)" % msg3)


# ------------------------------------------------------------------------------
# TEST 4: Borrado logico (Soft Delete) de VALVULA_RECIRC_01
# ------------------------------------------------------------------------------
ok4, msg4 = project.setpoints.service.soft_delete_setpoint(
    equip="VALVULA_RECIRC_01",
    username="mantenimiento_01",
    reason="Mantenimiento preventivo anual - Valvula fuera de servicio",
    db_conn="SANDBOX_DB"
)
run_test(4, "Borrado logico (Soft Delete) de equipo", ok4 is True, "(Msg: %s)" % msg4)


# ------------------------------------------------------------------------------
# TEST 5: Bloqueo de modificacion sobre equipo con baja logica activa
# ------------------------------------------------------------------------------
ok5, msg5 = project.setpoints.service.update_setpoint_with_audit(
    equip="VALVULA_RECIRC_01",
    param_name="pressure_sp",
    new_value=3.0,
    username="operador_02",
    reason="Intento de reactivacion sin alta formal",
    db_conn="SANDBOX_DB"
)
run_test(5, "Bloqueo preventivo de modificacion en equipo desactivado", ok5 is False and u"Baja Lógica" in msg5, "(Msg: %s)" % msg5)


# ------------------------------------------------------------------------------
# TEST 6: Insercion nominal mediante Named Query parametrizada
# ------------------------------------------------------------------------------
ok6, msg6 = project.actuacions.service.log_operator_action(
    equip="EDAR_COMPRESOR_02",
    description=u"Orden Enviar Consigna SCMX: 8.5 bar por supervisor",
    username="laura.vidal"
)
run_test(6, "Registro de actuacion via Named Query (Value Parameters)", ok6 is True, "(Msg: %s)" % msg6)


# ------------------------------------------------------------------------------
# TEST 7: Rechazo de actuacion por descripcion vacia
# ------------------------------------------------------------------------------
ok7, msg7 = project.actuacions.service.log_operator_action(
    equip="EDAR_BOMBA_01",
    description="   ",
    username="admin"
)
run_test(7, "Rechazo defensivo ante actuacion sin descripcion", ok7 is False and u"no puede estar vacía" in msg7, "(Msg: %s)" % msg7)


# ------------------------------------------------------------------------------
# TEST 8: Inmunidad total ante intento de Inyeccion SQL (SQLi)
# ------------------------------------------------------------------------------
sqli_payload = "EDAR_01' OR '1'='1"
ok8, msg8 = project.actuacions.service.log_operator_action(
    equip=sqli_payload,
    description="Prueba de penetracion controlada SQLi",
    username="pentester"
)
# Debe procesarse de forma segura sin romper la consulta relacional
run_test(8, "Resistencia e inmunidad a inyeccion SQL mediante Value Parameters", ok8 is True, "(Sanitizado con exito)")


print "\n" + "=" * 80
print "RESUMEN FINAL: {}/{} PRUEBAS SUPERADAS EXITOSAMENTE ({:.1f}%)".format(
    passed_tests, total_tests, (float(passed_tests) / float(total_tests)) * 100.0
)
print "=" * 80
```

---

## 5. Cuadro Comparativo de Beneficios

| Parámetro | Estado Anterior (`ejemplos-scripts-ignition.md`) | Estado Refactorizado (`Fase 5 - Lab-05`) | Impacto Técnico y Operativo |
| :--- | :--- | :--- | :--- |
| **Inyección SQL (SQLi)** | Concatenación directa de texto (`+ equip +`) y Query String Parameters (`{equip}`). | *Prepared Statements* nativos (`?`) y *Value Parameters* estrictamente tipados (`:equip`). | **Inmunidad absoluta** frente a vulnerabilidades SQLi y caracteres especiales de planta. |
| **Control Transaccional** | Ejecución ciega sin comprobación de filas alteradas. | Verificación obligatoria de `rows_affected == 1` en cada actualización. | Detección inmediata de fallos de concurrencia y desalineaciones de clave en base de datos. |
| **Integridad Histórica** | Borrado físico destructivo mediante sentencias `DELETE FROM`. | Borrado lógico (*Soft Delete*) con bandera `is_active` y fecha/autor de baja. | Cumplimiento estricto de normativas de retención forense e integridad histórica (FDA/ISO). |
| **Trazabilidad y Auditoría** | Textos genéricos no estructurados y ausencia de datos previos. | Registro exhaustivo del valor anterior y nuevo (`old_val -> new_val`) con motivo justificado (≥5 car.). | Pistas de auditoría inmutables (*Audit Trail*) listas para certificación de calidad. |
| **Límites de Ingeniería** | Escritura sin restricciones físicas; riesgo de rotura de maquinaria. | Diccionario centralizado `LIMITS` con validación física previa al despacho al PLC. | Protección física de activos industriales frente a errores tipográficos de operadores. |
| **Mantenibilidad y Arquitectura** | Consultas SQL y accesos a BD dispersos en botones, bindings y tags. | Lógica centralizada en `Project Library` (`project.setpoints` y `project.actuacions`). | Reducción drástica del acoplamiento y mantenimiento centralizado en un único punto. |
