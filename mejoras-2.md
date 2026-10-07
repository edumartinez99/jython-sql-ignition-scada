---
description: Informe técnico de auditoría y refactorización de scripts en Ignition Perspective.
---

# Informe Técnico: Refactorización y Estandarización de Scripts SCADA

## 1. Resumen Ejecutivo

Este informe documenta la auditoría técnica y la propuesta de refactorización de los scripts que operan en las pantallas y eventos del sistema SCADA en Ignition.

### Principales Deficiencias Detectadas
1. **Acoplamiento excesivo en la capa visual:** Lógica crítica de negocio incrustada directamente en bindings de vistas y eventos de componentes (`runAction`), impidiendo la reutilización y el mantenimiento centralizado.
2. **Fragilidad ante datos nulos o esquemas incompletos:** Accesos indexados por corchetes (`rec['columna']`) que disparan excepciones fatales de tipo `KeyError`, e iteración celda a celda sobre Datasets mediante `.getValueAt()` sin control de valores nulos (`None`).
3. **Duplicación masiva de estructuras:** Construcción manual y repetitiva de diccionarios para propiedades de Perspective (`props.instances` en *Flex Repeater*), multiplicando el riesgo de errores tipográficos en clases CSS y propiedades.
4. **Ausencia de contratos de retorno y validaciones:** Funciones sin comprobación de tipos ni contratos estructurados `(success, message)`, lo que oculta fallos al operador o cuelga la interfaz.

### Arquitectura Objetivo
* **Centralización en `Project Library` (`project.*`):** Extracción de la lógica fuera de las vistas y tags hacia módulos reutilizables.
* **Transformador Universal Tabular:** Normalización estandarizada de Datasets a listas de diccionarios JSON con envoltura `system.dataset.toPyDataSet` y sustitución segura de valores nulos.
* **Programación Defensiva:** Acceso mediante `.get()` con valores por defecto, validación de tipos numéricos y control explícito contra división por cero.
* **Funciones Puras y Contratos Homogéneos:** Retorno sistemático de tuplas `(bool, resultado_o_error)` y documentación formal con docstrings.

---

## 2. Matriz de Hallazgos y Acciones Correctivas

| Componente Auditado | Deficiencia Detectada | Riesgo Operativo | Solución Técnica Aplicada |
| :--- | :--- | :--- | :--- |
| **Binding Flex Consignas** | Construcción duplicada de diccionarios e iteración con `getValueAt` | Degradación de rendimiento y fallo si una columna cambia | Módulo `dataset_to_dict_list` y generador unificado de instancias |
| **Binding Flex Estados** | 5 ramas condicionales replicando la estructura de instancia | Mantenimiento ineficiente ante cambios de estilo o campos | Clasificación centralizada por diccionarios y constructor único |
| **Botón Enviar Consignas** | Accesos directos `instance['tagPath']` y sustitución de nombres en UI | Fallo de ejecución en cliente (`KeyError`) y lógica dispersa | Función pura de resolución de tag y contrato `(bool, unicode)` |
| **Gestor Documental** | Acceso a `res[0]` en `downloadFile` sin comprobar filas | Excepción fatal `IndexError` si el archivo no existe | Comprobación defensiva de dataset vacío y validación de parámetros |
| **Árbol de Navegación** | Más de 200 líneas de lógica en `runAction` de componente | Imposibilidad de probar la lógica de historial en aislamiento | Funciones puras de migas de pan y pila de historial en librería |
| **Tag Enclavamientos** | Lectura directa de `getValueAt` sin control de límites de filas | Fallo silencioso en el hilo de tags si faltan filas | Función modular con validación de rango de bits y tipos |

---

## 3. Detalle de Refactorización por Componente

---

### 3.1. Binding de Instancias de Flex Container (Consignas)

#### Diagnóstico
El script original construye manualmente las instancias para el *Flex Repeater*, duplicando el mismo diccionario en cada rama condicional e iterando el resultado de base de datos (`SCADA_values`) celda por celda:

```python
# CÓDIGO ORIGINAL (Fragmento con duplicación e iteración frágil)
for row in range(SCADA_values.getRowCount()):
    if SCADA_values.getValueAt(row, 'tagpath') in str(result['fullPath']).lower():
        # Extraccion manual repetitiva...
        if intvalue is not None: value_new = intvalue
        ...
if '1' in doc:
    first.append({
        "instanceStyle": {"classes": ""},
        "instancePosition": {},
        "tagPath": str(result['fullPath']),
        'value_new': value_new,
        'view_SCADA_value': value.view,
        'selected': selected,
        'data': date
    })
elif '2' in doc:
    second.append({
        "instanceStyle": {"classes": ""},
        "instancePosition": {},
        "tagPath": str(result['fullPath']),
        'value_new': value_new,
        'view_SCADA_value': value.view,
        'selected': selected,
        'data': date
    })
# ... se repite exactamente igual en la rama else
```

#### Solución Propuesta

1. Implementar la función `dataset_to_dict_list` para normalizar datasets de forma segura.
2. Definir una función constructora `build_consigna_instance` que reciba los campos y cree la estructura de forma unificada.

```python
# Implementación en Project Library: project.ui.flex_builder

def dataset_to_dict_list(dataset):
    """
    Convierte un Dataset de Ignition en una lista de diccionarios de forma segura.
    Tolera datasets vacios o nulos y normaliza los valores None a valores por defecto.
    
    Args:
        dataset (Dataset): Objeto tabular inmutable de Ignition.
        
    Returns:
        list of dict: Lista de diccionarios con las cabeceras como claves.
    """
    if dataset is None or dataset.getRowCount() == 0:
        return []
        
    headers = list(dataset.getColumnNames())
    pyds = system.dataset.toPyDataSet(dataset)
    
    dict_list = []
    for row in pyds:
        row_dict = {}
        for col in headers:
            val = row[col]
            row_dict[col] = val if val is not None else ""
        dict_list.append(row_dict)
        
    return dict_list


def build_consigna_instance(tag_path, value_new, view_scada_value, selected, date_str, badge_class=""):
    """
    Genera de forma estandarizada un objeto de instancia para Flex Repeater.
    
    Args:
        tag_path (str): Ruta completa del tag.
        value_new (any): Valor de consigna asignado.
        view_scada_value (bool): Flag de visualizacion SCADA.
        selected (bool): Si la fila esta marcada para envio.
        date_str (str): Fecha en formato texto.
        badge_class (str): Clase CSS a inyectar en instanceStyle.
        
    Returns:
        dict: Estructura compatible con props.instances.
    """
    return {
        "instanceStyle": {
            "classes": badge_class
        },
        "instancePosition": {},
        "tagPath": str(tag_path) if tag_path is not None else "",
        "value_new": value_new if value_new is not None else 0,
        "view_SCADA_value": bool(view_scada_value),
        "selected": bool(selected),
        "data": str(date_str) if date_str is not None else ""
    }
```

#### Código Resultante en el Binding
```python
# Binding de props.instances (Limpio y sin duplicación de diccionarios)
instancia = project.ui.flex_builder.build_consigna_instance(
    tag_path=result['fullPath'],
    value_new=value_new,
    view_scada_value=value.view,
    selected=selected,
    date_str=date
)

if '1' in doc:
    first.append(instancia)
elif '2' in doc:
    second.append(instancia)
else:
    instances.append(instancia)

return first + second + instances
```

---

### 3.2. Binding de Instancias de Flex Container (Estados)

#### Diagnóstico
El script contiene 5 ramas condicionales que insertan o añaden exactamente el mismo diccionario, complicando cualquier cambio de estilo o metadatos de UI:

```python
# CÓDIGO ORIGINAL (Fragmento)
if '1' in system.tag.readBlocking(...):
    first.insert(0, {"instanceStyle": {"classes": ""}, "instancePosition": {1}, "tagPath": str(result['fullPath'])})
elif '2' in system.tag.readBlocking(...):
    second.insert(-1, {"instanceStyle": {"classes": ""}, "instancePosition": {}, "tagPath": str(result['fullPath'])})
# ... ramas identicas para '3', '4' e 'instances'
```

#### Solución Propuesta
Centralizar la construcción en `project.ui.states` utilizando un mapa de categorías para evitar la duplicación de bloques de código:

```python
# Implementación en Project Library: project.ui.states

def create_state_instance(tag_path, position=None, css_class=""):
    """
    Construye una instancia de estado para Flex Repeater con tipos seguros.
    
    Args:
        tag_path (str): Ruta completa del tag.
        position (dict|None): Propiedades de posicionamiento.
        css_class (str): Clases visuales asignadas.
        
    Returns:
        dict: Objeto de instancia.
    """
    return {
        "instanceStyle": {
            "classes": css_class
        },
        "instancePosition": position if isinstance(position, dict) else {},
        "tagPath": str(tag_path) if tag_path is not None else ""
    }


def process_state_categories(browse_results):
    """
    Agrupa los tags explorados en listas ordenadas por categoria sin duplicar codigo.
    
    Args:
        browse_results (list): Lista de resultados devueltos por system.tag.browse.
        
    Returns:
        list of dict: Lista consolidada para props.instances.
    """
    if not browse_results:
        return []
        
    buckets = {"1": [], "2": [], "3": [], "4": [], "default": []}
    
    for res in browse_results:
        full_path = str(res.get("fullPath", ""))
        reads = system.tag.readBlocking([full_path + ".Documentation", full_path + ".Enabled"])
        doc = reads[0].value if reads[0].value is not None else ""
        enabled = reads[1].value if reads[1].value is not None else False
        
        if "E" in doc and enabled:
            inst = create_state_instance(full_path)
            if "1" in doc:
                buckets["1"].insert(0, inst)
            elif "2" in doc:
                buckets["2"].append(inst)
            elif "3" in doc:
                buckets["3"].append(inst)
            elif "4" in doc:
                buckets["4"].append(inst)
            else:
                buckets["default"].append(inst)
                
    return buckets["1"] + buckets["2"] + buckets["3"] + buckets["4"] + buckets["default"]
```

---

### 3.3. Script de Evento para Enviar Consignas (`runAction` en Botón)

#### Diagnóstico
Toda la lógica de transformación de tags y escritura en base de datos está incrustada en el evento del botón con accesos directos por corchete vulnerables a `KeyError`:

```python
# CÓDIGO ORIGINAL (Fragmento con accesos inseguros)
for instance in instances:
    selected = instance['selected']      # Falla si la clave no existe
    if selected:
        tagPath = instance['tagPath']
        tag = tagPath.split('/')[-1]
        MName = 'M' + tag[1:len(tag)]
        if tag == 'SFOR':
            # Lectura de ESIM incrustada...
        value_new = instance['value_new'] # Falla si la clave no existe
        # Escrituras directas sin validacion ni retorno estructurado
```

#### Solución Propuesta
1. Extraer la lógica pura de cálculo del nombre del tag a una función aislada.
2. Extraer la lógica de despacho a `project.consignas.actions`, empleando `.get()` defensivo y un contrato de retorno estructurado `(bool, unicode)`.

```python
# Implementación en Project Library: project.consignas.actions

def resolve_m_tag_name(tag_name, is_esim_active=False):
    """
    Calcula de forma pura el nombre del tag de comando M* correspondiente.
    
    Args:
        tag_name (str): Nombre del tag base (ej. 'SFOR', 'SP_NIVEL').
        is_esim_active (bool): Si el modo de simulacion ESIM esta activo.
        
    Returns:
        str: Nombre del tag destino.
    """
    if not tag_name or not isinstance(tag_name, (str, unicode)):
        return ""
        
    if tag_name == "SFOR" and is_esim_active:
        return "MFOS"
        
    return "M" + tag_name[1:]


def dispatch_consignas(instances, equipment_id, user_name):
    """
    Ejecuta el envio de consignas seleccionadas con programacion defensiva y auditoria.
    
    Args:
        instances (list of dict): Instancias procedentes de Perspective.
        equipment_id (str): Identificador del equipo destino.
        user_name (str): Usuario que realiza la accion.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    if not instances or not isinstance(instances, list):
        return (False, u"No se recibieron instancias validas")
        
    if not equipment_id:
        return (False, u"Identificador de equipo no especificado")
        
    dispatched = 0
    
    for inst in instances:
        if not isinstance(inst, dict):
            continue
            
        # Acceso defensivo con .get() y valores por defecto
        if not inst.get("selected", False):
            continue
            
        tag_path = inst.get("tagPath", "")
        value_new = inst.get("value_new", None)
        
        if not tag_path or value_new is None:
            continue
            
        tag = tag_path.split("/")[-1]
        
        # Comprobacion de simulacion si aplica
        esim_active = False
        if tag == "SFOR":
            esim_path = "/".join(tag_path.split("/")[:-1]) + "/ESIM"
            read_esim = system.tag.readBlocking([esim_path])
            if read_esim and read_esim[0].value:
                esim_active = True
                
        m_tag = resolve_m_tag_name(tag, esim_active)
        target_path = tag_path.replace(tag, m_tag)
        
        # Descripcion
        tip_read = system.tag.readBlocking([tag_path + ".Tooltip"])
        desc = tip_read[0].value if tip_read and tip_read[0].value else tag
        
        # Escritura fisica
        system.tag.writeBlocking([target_path], [value_new])
        
        # Auditoria en base de datos
        audit_params = {
            "Equip": equipment_id,
            "Descripcio": "Ordre Enviar Consigna %s: %s" % (tag, desc),
            "Usuari": user_name if user_name else "SISTEMA"
        }
        system.db.runNamedQuery("ins_Registre_Actuacions", audit_params)
        dispatched += 1
        
    if dispatched == 0:
        return (False, u"No se seleccionó ninguna consigna para enviar")
        
    return (True, u"Se enviaron %d consignas correctamente" % dispatched)
```

#### Código Resultante en el Evento del Botón
```python
# Evento runAction del componente (Totalmente desacoplado)
instances = self.getSibling("FlexRepeater").props.instances
equip = self.view.params.TAG
user = self.session.props.auth.user.userName

ok, mensaje = project.consignas.actions.dispatch_consignas(instances, equip, user)

if not ok:
    system.perspective.print("AVISO: " + mensaje)
```

---

### 3.4. Gestión Documental (`downloadFile`)

#### Diagnóstico
El script asume que la consulta siempre retorna filas y accede directamente al índice `0`:

```python
# CÓDIGO ORIGINAL
def downloadFile(id):
    res = system.dataset.toPyDataSet(system.db.runNamedQuery(path="Document Management/Documents/Get Document Contents", parameters={"id":id}))
    fileName = res[0]["filename"]   # Lanza IndexError si el ID no existe o esta vacio
    fileBytes = res[0]["contents"]
    system.perspective.download(fileName, fileBytes)
```

#### Solución Propuesta
Validar si el dataset resultante es nulo o tiene cero filas antes de indexar, devolviendo una tupla de contrato `(bool, unicode)`:

```python
# Implementación en Project Library: project.documents.manager

def download_document(document_id):
    """
    Descarga de forma segura un documento validando la existencia de datos.
    
    Args:
        document_id (int|str): Identificador unico del documento.
        
    Returns:
        tuple: (bool success, unicode message)
    """
    if document_id is None:
        return (False, u"El identificador del documento no puede ser nulo")
        
    try:
        raw_ds = system.db.runNamedQuery(
            path="Document Management/Documents/Get Document Contents",
            parameters={"id": document_id}
        )
    except Exception as err:
        return (False, u"Error al consultar el documento: " + unicode(err))
        
    # Validacion defensiva de dataset vacio
    if raw_ds is None or raw_ds.getRowCount() == 0:
        return (False, u"El documento solicitado no existe en el repositorio")
        
    pyds = system.dataset.toPyDataSet(raw_ds)
    first_row = pyds[0]
    
    file_name = first_row["filename"] if first_row["filename"] is not None else "archivo_sin_nombre"
    file_bytes = first_row["contents"]
    
    if file_bytes is None:
        return (False, u"El documento no contiene datos binarios descargables")
        
    system.perspective.download(file_name, file_bytes)
    return (True, u"Descarga completada")
```

---

### 3.5. Script de Evento en Árbol de Navegación (`runAction`)

#### Diagnóstico
Más de 200 líneas de código acumuladas dentro del evento de componente para calcular títulos de migas de pan y recortar el array de historial de navegación en sesión.

#### Solución Propuesta
Extraer la lógica a funciones **100% puras** en `project.nav.history`, permitiendo probar el recorte del historial y el formateo de títulos sin necesidad de abrir la vista ni manipular la sesión web:

```python
# Implementación en Project Library: project.nav.history

def format_breadcrumb_title(labels):
    """
    Funcion pura que une una coleccion de etiquetas en un formato jerarquico.
    
    Args:
        labels (list of str): Lista ordenada de etiquetas de navegacion.
        
    Returns:
        unicode: Cadena con el formato 'Nivel 1 > Nivel 2 > Nivel 3'.
    """
    if not labels:
        return u""
    clean = [unicode(l).strip() for l in labels if l]
    return u" > ".join(clean)


def calculate_history_slice(current_history, current_index, new_item, max_items=5):
    """
    Funcion pura que gestiona la pila de historial de navegacion evitando desbordamientos.
    
    Args:
        current_history (list of dict): Lista actual de pantallas en el historial.
        current_index (int): Indice de la pantalla activa.
        new_item (dict): Nuevo objeto de navegacion {'title': ..., 'path': ...}.
        max_items (int): Limite maximo de elementos en la pila.
        
    Returns:
        tuple: (list updated_history, int new_index)
    """
    history = list(current_history) if isinstance(current_history, list) else []
    
    # Truncar historial si se habia navegado hacia atras
    if 0 <= current_index < len(history) - 1:
        history = history[:current_index + 1]
        
    # Evitar registrar dos veces seguidas la misma ruta
    if not history or history[-1].get("path") != new_item.get("path"):
        history.append(new_item)
        
    # Mantener el tope maximo
    if len(history) > max_items:
        history.pop(0)
        
    return (history, len(history) - 1)
```

---

### 3.6. Script en Tag de Enclavamientos (`valueChanged`)

#### Diagnóstico
El script indexa directamente `descr.getValueAt(indice, "Descripcio")` sobre el Dataset leído del tag `[.]WENC_DESC` sin validar si el índice de bit excede el número de filas disponibles.

#### Solución Propuesta
Encapsular la lectura de descripciones en una función con validación de límites:

```python
# Implementación en Project Library: project.enclavamientos.evaluator

def get_interlock_description(desc_dataset, bit_index):
    """
    Obtiene la descripcion asociada a un bit con validacion de limites.
    
    Args:
        desc_dataset (Dataset): Dataset con la columna 'Descripcio'.
        bit_index (int): Indice de bit a consultar (0 a 31).
        
    Returns:
        tuple: (bool success, unicode description)
    """
    if desc_dataset is None or desc_dataset.getRowCount() == 0:
        return (False, u"Dataset de descripciones vacio o nulo")
        
    if bit_index < 0 or bit_index >= desc_dataset.getRowCount():
        return (False, u"Bit fuera del rango de descripciones (%d)" % bit_index)
        
    try:
        val = desc_dataset.getValueAt(bit_index, "Descripcio")
        return (True, unicode(val) if val is not None else u"Sin descripcion")
    except Exception as e:
        return (False, u"Error al acceder al dataset: " + unicode(e))
```

---

## 4. Banco de Pruebas Automatizado (Script Console)

Para certificar el correcto funcionamiento de las funciones puras y los transformadores sin intervenir en la planta en vivo, se debe ejecutar el siguiente bloque de validación unitaria en la **Script Console** de Ignition Designer (**Tools -> Script Console**):

```python
print "=" * 80
print "CERTIFICACION DE FUNCIONES REFACTORIZADAS"
print "=" * 80

# 1. Prueba de Transformador Universal con Casos Limite (None, Vacio, Nulos)
print "\n--- [TEST 1] dataset_to_dict_list ---"
headers = ["id", "tagpath", "valor"]
data = [
    [1, "EDAR/BOMBA_01", 1200.0],
    [2, "EDAR/BOMBA_02", None],      # Nulo de SQL
    [3, "EDAR/BOMBA_03", 0.0]
]
test_ds = system.dataset.toDataSet(headers, data)
resultado = project.ui.flex_builder.dataset_to_dict_list(test_ds)

assert len(resultado) == 3, "Error en conteo de registros"
assert resultado[1]["valor"] == "", "Fallo en normalizacion de valor None"
assert project.ui.flex_builder.dataset_to_dict_list(None) == [], "Fallo en dataset None"
print "[OK] dataset_to_dict_list supera todas las validaciones"


# 2. Prueba de Resolucion Pura de Tags M*
print "\n--- [TEST 2] resolve_m_tag_name ---"
casos = [
    ("SFOR", False, "MFOR"),
    ("SFOR", True,  "MFOS"),
    ("SP_TEMP", False, "MP_TEMP"),
    ("", False, ""),
    (None, False, "")
]

for tag, esim, esperado in casos:
    res = project.consignas.actions.resolve_m_tag_name(tag, esim)
    assert res == esperado, "Fallo en resolucion de tag: %s (Obtenido: %s, Esperado: %s)" % (tag, res, esperado)
print "[OK] resolve_m_tag_name supera todas las combinaciones nominales y limite"


# 3. Prueba de Pila Pura de Historial
print "\n--- [TEST 3] calculate_history_slice ---"
hist = []
idx = 0
items = [
    {"title": "P1", "path": "/p1"},
    {"title": "P2", "path": "/p2"},
    {"title": "P3", "path": "/p3"},
    {"title": "P4", "path": "/p4"},
    {"title": "P5", "path": "/p5"},
    {"title": "P6", "path": "/p6"} # Debe forzar pop(0) manteniendo max_items=5
]

for item in items:
    hist, idx = project.nav.history.calculate_history_slice(hist, idx, item, max_items=5)

assert len(hist) == 5, "El historial no respeto el limite maximo de 5"
assert hist[0]["path"] == "/p2", "Fallo en politica FIFO al eliminar el mas antiguo"
print "[OK] calculate_history_slice gestiona correctamente la pila y su capacidad"

print "\n" + "=" * 80
print "TODAS LAS PRUEBAS UNITARIAS HAN FINALIZADO CON EXITO"
print "=" * 80
```

---

## 5. Cuadro Comparativo de Beneficios

| Parámetro | Estado Anterior | Estado Refactorizado | Impacto Técnico |
| :--- | :--- | :--- | :--- |
| **Tolerancia a Nulos** | Excepción `TypeError` / `KeyError` | Normalización a tipos seguros con `.get()` y coalescencia | Cero paradas de renderizado en pantallas de operador |
| **Tiempo de Mantenimiento** | Edición en decenas de pantallas y bindings individuales | Modificación en un único script de `Project Library` | Reducción drástica del tiempo de despliegue y riesgo de error |
| **Capacidad de Prueba** | Obligatoriedad de abrir pantallas en runtime | Pruebas aisladas en Script Console con datos simulados | Verificación ágil antes del pase a producción |
| **Consistencia de UI** | Clases y estilos asignados ad-hoc en cada rama condicional | Inyección centralizada de `instanceStyle` | Estética homogénea y estandarizada en toda la planta |
