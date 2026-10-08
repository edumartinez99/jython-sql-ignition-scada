---
description: Informe técnico de auditoría, simulación con Mocks, pruebas unitarias y depuración sistemática de scripts SCADA en Ignition (Sesión 3).
---

# Informe Técnico: Refactorización y Estandarización de Scripts SCADA (Fase 3: Introspección, Mocking y Pruebas Unitarias)

## 1. Resumen Ejecutivo

Este informe documenta la auditoría técnica y la propuesta de refactorización integral de los scripts del sistema SCADA en Ignition (`ejemplos-scripts-ignition.md`), fundamentada estrictamente en los principios de ingeniería de software industrial, introspección de objetos en la JVM, simulación de datos (*Mocking*), depuración sistemática y pruebas unitarias desarrolladas en los laboratorios de la **Sesión 3** (`gcp-con-eduardo/lab-03.md`).

### Principales Deficiencias Detectadas

1. **Captura Indiscriminada de Excepciones (`try/except` ciegos):** En scripts de extracción histórica (`Historian/Consignas`), los fallos de consulta, problemas de red o discrepancias de tipos se enmascaran tras bloques `except:` vacíos. Esto oculta errores fatales, impide distinguir entre ausencia de datos y fallo de comunicación, y ejecuta mutaciones colaterales no deseadas (alteración forzada del parámetro `mes`).
2. **Acceso Frágil a Datasets y Ausencia de Coalescencia de Tipos:** Se itera celda a celda mediante `.getValueAt()` sin introspección previa de metadatos (`getColumnNames()`, `.getRowCount()`). La resolución de columnas heterogéneas (`intvalue`, `floatvalue`, `stringvalue`, `datevalue`) carece de un orden jerárquico estricto y seguro frente a valores nulos (`None`), provocando comportamientos erráticos o formateos inválidos.
3. **Manipulación Insegura de Cadenas y Conversiones sin Cláusulas de Guarda:** Partición directa de rutas mediante `.split('/')` y conversión forzada con `int()` en eventos de interfaz (árbol de navegación y envío de consignas: `tagPath.split('/')[-1]`, `int(itemPath.split('/')[0])`). Si la entrada llega vacía, con formato inesperado o con caracteres no numéricos, se disparan excepciones no controladas de tipo `IndexError` y `ValueError` que congelan la sesión del operador.
4. **Ausencia Total de Pruebas Unitarias y Entornos de Simulación (*Mocks*):** La lógica de negocio crítica (derivación de comandos `M*`, manipulación de bits `0..31` de enclavamientos, navegación jerárquica) carece de bancos de prueba desacoplados. Las validaciones se realizan "en caliente" en la planta activa, incrementando el riesgo de maniobras intempestivas sobre PLCs y paradas imprevistas.

### Arquitectura Objetivo

* **Simulación Aislada con Mocks (`system.dataset.toDataSet`):** Construcción de bancos de datos sintéticos en la Script Console para validar algoritmos de transformación sin requerir conexión física con el servidor de bases de datos (`sqlt_data_1_{mes}`) ni con autómatas.
* **Introspección y Recorrido Seguro con `PyDataSet`:** Auditoría previa de metadatos (`type`, `dir`, `.getColumnNames()`) y conversión a `system.dataset.toPyDataSet`, procesando registros con `enumerate()` y claves de columna para evitar indexaciones numéricas rígidas y llamadas repetitivas RPC a la JVM.
* **Coalescencia Jerárquica y Erradicación de `except:` Ciegos:** Tratamiento explícito de celdas nulas (`None`) priorizando tipos numéricos (Float > Int) seguidos de cadenas y marcas de tiempo, eliminando bloques `except:` genéricos y sustituyéndolos por validaciones lógicas tempranas.
* **Programación Defensiva mediante Cláusulas de Guarda (*Guard Clauses*):** Validación inmediata de tipos de datos (`isinstance`), cadenas vacías (`.strip()`) y longitudes de colecciones antes de acceder a índices o realizar conversiones de tipo con bloques `try/except ValueError` focalizados.
* **Mini-Runner de Pruebas Unitarias (`SCADATestRunner`):** Evaluación automatizada de funciones en la Script Console antes de su integración en pantallas o tags, emitiendo informes estructurados con recuento de pruebas superadas (`PASS`) y detalle de discrepancias (`FAIL`).
* **Contratos Estructurados de Retorno:** Retorno sistemático de tuplas `(bool success, unicode resultado_o_error)` para estandarizar la comunicación entre la capa de negocio y la interfaz gráfica.

---

## 2. Matriz de Hallazgos y Acciones Correctivas

| Componente Auditado | Deficiencia Detectada (Código Original) | Riesgo Operativo | Solución Técnica Aplicada (Lab-03) |
| :--- | :--- | :--- | :--- |
| **Binding Flex Consignas** (`Historian/Consignas`) | Bloque `try/except:` ciego, iteración con `.getValueAt()` y asignación de tipo desordenada | Enmascaramiento de caídas de BD, fallo al formatear valores nulos y alteración errónea de variable `mes` | Extracción modular a `Project Library`, uso de `toPyDataSet`, coalescencia jerárquica tipada y simulación con Mocks |
| **Botón Enviar Consignas** (`runAction`) | `tagPath.split('/')[-1]` y `MName = 'M' + tag[1:]` sin validación de longitud ni tipo | `IndexError` si la ruta es anómala; corrupción de rutas si el nombre coincide con carpetas intermedias | Función pura `resolve_command_tag_path` con cláusulas de guarda, verificación de longitud y sustitución delimitada |
| **Árbol de Navegación** (`runAction`) | `selectionData[0]` sin validar lista y `int(itemPath.split('/')[0])` sin control de tipos | `IndexError` si la selección está vacía; `ValueError` fatal si el nodo no es un índice numérico | Función pura `parse_navigation_breadcrumbs` con conversión protegida `try/except ValueError` y control de límites |
| **Tag Enclavamientos** (`valueChanged`) | `tagPath.split('/')[-2]` y `descr.getValueAt(indice, "Descripcio")` sin validar límites de filas | Fallo silencioso en el hilo de Gateway si `WENC_DESC` tiene menos de 32 filas o la ruta es corta | Función modular `get_interlock_description_safe` con validación estricta de límites de filas y tipos de datos |

---

## 3. Detalle de Refactorización por Componente

---

### 3.1. Extracción y Coalescencia de Datos Históricos SCADA (Binding Flex Consignas)

#### Diagnóstico

En el script del binding de consignas (`ejemplos-scripts-ignition.md`, líneas 326-386), la consulta histórica `Historian/Consignas` se envuelve en un bloque `try/except` sin tipo de excepción. Si la base de datos devuelve filas con valores nulos, o si ocurre un fallo de conexión, el script oculta el error y modifica la variable `mes`. Además, se accede celda a celda mediante múltiples llamadas `SCADA_values.getValueAt(row, col)`, penalizando el rendimiento:

```python
# CÓDIGO ORIGINAL (Fragmento con try/except ciego e iteración celda a celda)
try:
    SCADA_values = system.db.runNamedQuery('Historian/Consignas', parameters={'date': system.date.format(dateReal, 'yyyy-MM-dd HH:mm:ss'), 'tag': value.tag.lower(), 'mes': mesStr})
    for row in range(SCADA_values.getRowCount()):           
        if SCADA_values.getValueAt(row, 'tagpath') in str(result['fullPath']).lower():
            intvalue = SCADA_values.getValueAt(row, 'intvalue')
            floatvalue = SCADA_values.getValueAt(row, 'floatvalue')
            stringvalue = SCADA_values.getValueAt(row, 'stringvalue')
            datevalue = SCADA_values.getValueAt(row, 'datevalue')
            
            if intvalue is not None:                       
                value_new = intvalue
            elif floatvalue is not None:                       
                value_new = floatvalue                       
            elif stringvalue is not None:                       
                value_new = stringvalue
            elif datevalue is not None:                       
                value_new = datevalue
            
            date = system.date.format(SCADA_values.getValueAt(row, 'date'), 'dd/MM/YYYY HH:mm:ss')
except:
    # Si no existeix cap valor, actualitzar mes
    mes = system.date.getMonth(dateReal)
    if mes < 10:
        mesStr = '0' + str(mes)
    else:
        mesStr = str(mes)
```

#### Solución Propuesta (Enseñanzas Lab 3.1)

1. **Aislamiento en Módulo de Librería:** Extraer la extracción y consolidación de datos a `project.historian.consignas.extract_scada_history_record`.
2. **Conversión a `PyDataSet`:** Sustituir la iteración sobre el `Dataset` Java por `system.dataset.toPyDataSet`, eliminando la sobrecarga de `.getValueAt()`.
3. **Coalescencia Jerárquica Explícita:** Priorizar `floatvalue` sobre `intvalue`, seguido de `stringvalue` y `datevalue`, con control de tipos y formateo numérico seguro ante valores nulos, erradicando el bloque `except:` ciego.
4. **Capacidad de Mocking:** Permitir la inyección de datasets simulados (`system.dataset.toDataSet`) para validar la lógica en la Script Console sin requerir la base de datos de producción.

```python
# Implementación en Project Library: project.historian.consignas

def extract_scada_history_record(scada_dataset, target_tagpath):
    """
    Extrae el registro historico mas reciente para un tag especifico aplicando
    coalescencia jerarquica de tipos e introspeccion defensiva.
    
    Args:
        scada_dataset (Dataset|PyDataSet|None): Dataset devuelto por la consulta historica.
        target_tagpath (str|unicode): Ruta completa del tag a localizar.
        
    Returns:
        tuple: (bool found, any effective_value, unicode formatted_date)
    """
    # 1. Cláusula de guarda e introspección básica
    if scada_dataset is None:
        return (False, None, u"")
        
    # Verificar si es un Dataset inmutable de Ignition y convertirlo
    if hasattr(scada_dataset, "getRowCount"):
        if scada_dataset.getRowCount() == 0:
            return (False, None, u"")
        pyds = system.dataset.toPyDataSet(scada_dataset)
    else:
        pyds = scada_dataset
        
    if len(pyds) == 0:
        return (False, None, u"")
        
    clean_target = str(target_tagpath).lower().strip()
    
    # 2. Recorrido seguro mediante claves de columna
    for row in pyds:
        row_tagpath = str(row["tagpath"]).lower() if row["tagpath"] is not None else ""
        
        if row_tagpath and (row_tagpath in clean_target or clean_target in row_tagpath):
            int_val = row["intvalue"]
            float_val = row["floatvalue"]
            str_val = row["stringvalue"]
            date_val = row["datevalue"]
            raw_date = row["date"] if "date" in pyds.underlyingDataset.getColumnNames() else None
            
            # 3. Coalescencia jerárquica de tipos (Float > Int > String > Date)
            if float_val is not None:
                effective_value = float(float_val)
            elif int_val is not None:
                effective_value = int(int_val)
            elif str_val is not None:
                effective_value = unicode(str_val)
            elif date_val is not None:
                effective_value = date_val
            else:
                effective_value = None
                
            # Formateo defensivo de fecha
            formatted_date = u""
            if raw_date is not None:
                try:
                    formatted_date = unicode(system.date.format(raw_date, "dd/MM/yyyy HH:mm:ss"))
                except Exception:
                    formatted_date = unicode(raw_date)
                    
            return (True, effective_value, formatted_date)
            
    return (False, None, u"")
```

#### Código Resultante en el Binding

```python
# Binding de props.instances (Limpio, predecible y sin try/except genérico)
try:
    scada_ds = system.db.runNamedQuery(
        'Historian/Consignas', 
        parameters={
            'date': system.date.format(dateReal, 'yyyy-MM-dd HH:mm:ss'), 
            'tag': value.tag.lower(), 
            'mes': mesStr
        }
    )
except Exception as db_err:
    system.util.getLogger("SCADA_Consignas").warn("Fallo al consultar Historian/Consignas: " + str(db_err))
    scada_ds = None

found, val_historico, date_scada = project.historian.consignas.extract_scada_history_record(
    scada_dataset=scada_ds,
    target_tagpath=result['fullPath']
)

if found and val_historico is not None:
    value_new = val_historico
    date = date_scada
```

---

### 3.2. Resolución Robusta de Tags de Comando M* (Botón Enviar Consignas)

#### Diagnóstico

En el evento del botón de envío de consignas (`ejemplos-scripts-ignition.md`, líneas 524-549), se extrae el nombre del tag haciendo particiones directas sobre cadenas y recortes posicionales (`tagPath.split('/')[-1]`, `'M' + tag[1:len(tag)]`). Asimismo, se realiza un `.replace(tag, MName)` que reemplaza todas las apariciones de la subcadena en la ruta completa, lo que corrompe la ruta si el nombre del equipo o de una carpeta intermedia coincide con el nombre de la variable:

```python
# CÓDIGO ORIGINAL (Fragmento con slicing desprotegido y sustitución peligrosa)
tag = tagPath.split('/')[-1]
MName = 'M' + tag[1:len(tag)]

if tag == 'SFOR':           
    ESIM = system.tag.readBlocking([self.view.params.PATH + self.view.params.TAG + '/ESIM'])[0].value
    if ESIM:
        MName = 'MFOS'

# Peligro: si tagPath es "[default]SFOR_DEP/SFOR", produce "[default]MFOS_DEP/MFOS"
Mwrite = tagPath.replace(tag, MName)
```

#### Solución Propuesta (Enseñanzas Lab 3.2 y 3.3)

1. **Aislamiento como Función Pura:** Encapsular la lógica en `project.consignas.tags.resolve_command_tag_path`.
2. **Cláusulas de Guarda (*Guard Clauses*):** Validar que `tag_path` no sea `None`, sea de tipo texto y contenga al menos un carácter.
3. **Validación de Longitud de Segmento:** Verificar que el tag tenga longitud suficiente para derivar el comando `M*`. Si el tag tiene menos de 2 caracteres, rechazar la operación con mensaje estructurado.
4. **Reemplazo Delimitado del Segmento Final:** Modificar únicamente el último nodo de la ruta (`parts[-1] = m_name`), preservando intacta la jerarquía de carpetas.

```python
# Implementación en Project Library: project.consignas.tags

def resolve_command_tag_path(tag_path, is_esim_active=False):
    """
    Resuelve de forma pura la ruta de comando M* correspondiente a un tag de consigna,
    aplicando clausulas de guarda y reemplazo estricto del segmento final.
    
    Args:
        tag_path (str|unicode): Ruta completa del tag base (ej. '[default]EDAR/BOMBA_01/SFOR').
        is_esim_active (bool): Flag que indica si la simulacion ESIM esta activa.
        
    Returns:
        tuple: (bool success, unicode target_path_or_error)
    """
    # 1. Validación de tipo y existencia
    if tag_path is None or not isinstance(tag_path, (str, unicode)):
        return (False, u"Ruta de tag nula o tipo de dato invalido")
        
    clean_path = tag_path.strip()
    if len(clean_path) == 0:
        return (False, u"La ruta de tag especificada esta vacia")
        
    # 2. Descomposición por niveles
    parts = clean_path.split("/")
    tag_name = parts[-1].strip()
    
    if len(tag_name) < 2:
        return (False, u"El nombre del tag '%s' es demasiado corto para derivar comando M*" % tag_name)
        
    # 3. Lógica determinista de comando
    if tag_name == "SFOR" and bool(is_esim_active):
        m_name = "MFOS"
    else:
        m_name = "M" + tag_name[1:]
        
    # 4. Sustitución segura únicamente en el último segmento de la ruta
    parts[-1] = m_name
    target_path = "/".join(parts)
    
    return (True, unicode(target_path))
```

#### Código Resultante en el Evento del Botón

```python
# Evento runAction del botón de consignas
instances = self.getSibling("FlexRepeater").props.instances

for instance in instances:
    if not instance.get("selected", False):
        continue
        
    tag_path = instance.get("tagPath", "")
    val_new = instance.get("value_new", None)
    
    if not tag_path or val_new is None:
        continue
        
    # Comprobar simulación ESIM si aplica
    esim_active = False
    if tag_path.endswith("/SFOR"):
        esim_tag = "/".join(tag_path.split("/")[:-1]) + "/ESIM"
        read_esim = system.tag.readBlocking([esim_tag])
        if read_esim and read_esim[0].value:
            esim_active = True
            
    ok, target_path = project.consignas.tags.resolve_command_tag_path(tag_path, esim_active)
    
    if not ok:
        system.perspective.print("ERROR al resolver tag: " + target_path)
        continue
        
    # Escritura física y registro
    system.tag.writeBlocking([target_path], [val_new])
```

---

### 3.3. Parsing y Descomposición Segura de Niveles en Árbol de Navegación

#### Diagnóstico

En el evento `runAction` del árbol de navegación (`ejemplos-scripts-ignition.md`, líneas 868-910), se asume que `selectionData` siempre contiene al menos un elemento y que cada segmento de `itemPath` puede convertirse ciegamente a `int`:

```python
# CÓDIGO ORIGINAL (Fragmento vulnerable a IndexError y ValueError)
itemPath = self.props.selectionData[0].itemPath
leng = len(itemPath.split('/'))

if leng > 1:
    nivel1 = int(itemPath.split('/')[0]) # Lanza ValueError si el segmento es texto
    PreTitol1 = self.props.items[nivel1].label # Lanza IndexError si nivel1 >= len(items)
if leng > 2:
    nivel2 = int(itemPath.split('/')[1])
    PreTitol2 = ' > ' + self.props.items[nivel1].items[nivel2].label
if leng > 3:
    nivel3 = int(itemPath.split('/')[2])
    PreTitol3 = ' > ' + self.props.items[nivel1].items[nivel2].items[nivel3].label
```

#### Solución Propuesta (Enseñanzas Lab 3.3)

1. **Aplicación del *Debug Loop*:** Aislar los puntos de rotura (`selectionData` vacío, cadenas alfanuméricas no convertibles a entero, desbordamiento de índices en el árbol de componentes).
2. **Función Pura con Cláusulas de Guarda:** Implementar `project.nav.tree.parse_navigation_breadcrumbs` validando que `item_path` sea válido y exista estructura arbórea.
3. **Conversión Numérica Protegida:** Acotar el bloque `try/except ValueError` estrictamente a la conversión de enteros por segmento, descartando segmentos no numéricos sin detener la ejecución.
4. **Validación de Límites en Listas:** Comprobar sistemáticamente que `0 <= idx < len(current_level)` antes de indexar nodos hijos.

```python
# Implementación en Project Library: project.nav.tree

def parse_navigation_breadcrumbs(item_path, items_tree, current_label=u""):
    """
    Parsea de forma segura una ruta jerarquica de navegacion ('0/1/2') y resuelve
    los titulos de migas de pan contrastandolos con la estructura de items.
    
    Args:
        item_path (str|unicode): Cadena jerarquica de indices ('0/1/2').
        items_tree (list): Lista jerarquica de nodos (self.props.items).
        current_label (str|unicode): Etiqueta del nodo activo seleccionado.
        
    Returns:
        tuple: (bool success, dict breadcrumbs, unicode full_title)
    """
    empty_result = {
        "pretittle1": u"",
        "pretittle2": u"",
        "pretittle3": u"",
        "tittle": unicode(current_label) if current_label else u""
    }
    
    # 1. Cláusulas de guarda
    if item_path is None or not isinstance(item_path, (str, unicode)):
        return (False, empty_result, empty_result["tittle"])
        
    clean_path = item_path.strip()
    if len(clean_path) == 0:
        return (False, empty_result, empty_result["tittle"])
        
    raw_segments = clean_path.split("/")
    numeric_indices = []
    
    # 2. Conversión segura de índices numéricos
    for seg in raw_segments:
        s = seg.strip()
        if not s:
            continue
        try:
            numeric_indices.append(int(s))
        except ValueError:
            # Segmento no numérico (ej. clave alfanumérica): abortar recorrido jerárquico
            return (False, empty_result, empty_result["tittle"])
            
    if not numeric_indices or not items_tree or not isinstance(items_tree, (list, tuple)):
        return (True, empty_result, empty_result["tittle"])
        
    # 3. Navegación defensiva con validación de límites de lista
    labels_found = []
    current_level = items_tree
    
    for idx in numeric_indices:
        if 0 <= idx < len(current_level):
            node = current_level[idx]
            label = getattr(node, "label", node.get("label", u"") if isinstance(node, dict) else u"")
            labels_found.append(unicode(label))
            
            # Descender al siguiente nivel si existe
            child_items = getattr(node, "items", node.get("items", []) if isinstance(node, dict) else [])
            if isinstance(child_items, (list, tuple)):
                current_level = child_items
            else:
                break
        else:
            # Índice fuera de límites: truncar navegación
            break
            
    # 4. Construcción de títulos
    pretittle1 = labels_found[0] if len(labels_found) > 0 else u""
    pretittle2 = (u" > " + labels_found[1]) if len(labels_found) > 1 else u""
    pretittle3 = (u" > " + labels_found[2]) if len(labels_found) > 2 else u""
    
    active_title = (u" > " + unicode(current_label)) if len(numeric_indices) > 1 else unicode(current_label)
    
    breadcrumbs_payload = {
        "pretittle1": pretittle1,
        "pretittle2": pretittle2,
        "pretittle3": pretittle3,
        "tittle": active_title
    }
    
    full_title = pretittle1 + pretittle2 + pretittle3 + active_title
    return (True, breadcrumbs_payload, full_title)
```

#### Código Resultante en el Evento del Componente

```python
# Evento runAction del árbol de navegación
selection = self.props.selectionData

if not selection or len(selection) == 0:
    return

item_path = selection[0].itemPath
items_tree = self.props.items
active_label = event.label

ok, payload, full_title = project.nav.tree.parse_navigation_breadcrumbs(
    item_path=item_path,
    items_tree=items_tree,
    current_label=active_label
)

payload["path"] = event.data.get("path", "")

system.perspective.sendMessage("change-pretittle", payload=payload)
```

---

### 3.4. Evaluación de Bits de Enclavamiento y Límites de Dataset (Tag `valueChanged`)

#### Diagnóstico

En el script de tag de enclavamientos (`ejemplos-scripts-ignition.md`, líneas 621-753), se realiza una extracción posicional `tagPath.split('/')[-2]` y se itera sobre un rango rígido de 32 bits. Cuando detecta un cambio, consulta directamente `descr.getValueAt(indice, "Descripcio")` sobre el tag de descripciones `[.]WENC_DESC` sin validar si el índice de bit excede las filas del dataset o si este viene nulo:

```python
# CÓDIGO ORIGINAL (Fragmento con indexación rígida de bits y dataset)
equip = tagPath.split('/')[-2] # Falla si tagPath no tiene suficientes niveles
for i in range(32):
    if current[i] != prev[i]: # Falla si current/prev no son listas/arrays indexables
        indice = i
        ...
        descr = system.tag.read("[.]WENC_DESC").value
        textoActive = descr.getValueAt(indice, "Descripcio") # Lanza ArrayIndexOutOfBoundsException
```

#### Solución Propuesta (Enseñanzas Lab 3.1 y 3.2)

1. **Extracción Segura del Identificador de Equipo:** Utilizar una función con cláusula de guarda que verifique la estructura mínima del path del tag.
2. **Detección Defensiva de Transición de Bits:** Validar que los valores anterior y actual sean colecciones iterables o enteros comparables por máscaras antes de acceder por corchetes.
3. **Auditoría de Límites sobre el Dataset de Descripciones:** Implementar `project.enclavamientos.evaluator.get_interlock_description_safe`, validando que `descr` no sea nulo, contenga la columna requerida y que `0 <= bit_index < getRowCount()`.

```python
# Implementación en Project Library: project.enclavamientos.evaluator

def extract_equipment_name(tag_path):
    """
    Extrae de forma segura el nombre del equipo a partir del tagPath.
    """
    if not tag_path or not isinstance(tag_path, (str, unicode)):
        return u"EQUIPO_DESCONOCIDO"
        
    parts = tag_path.strip().split("/")
    if len(parts) >= 2:
        return unicode(parts[-2])
    return unicode(parts[0])


def get_interlock_description_safe(desc_dataset, bit_index):
    """
    Obtiene la descripcion de un bit de enclavamiento validando limites de filas
    y evitando excepciones de runtime en hilos de tags de Gateway.
    
    Args:
        desc_dataset (Dataset|None): Dataset del tag [.]WENC_DESC.
        bit_index (int): Indice de bit evaluado (0 a 31).
        
    Returns:
        tuple: (bool success, unicode description)
    """
    if desc_dataset is None:
        return (False, u"Dataset de descripciones no disponible (None)")
        
    row_count = getattr(desc_dataset, "getRowCount", lambda: 0)()
    if row_count == 0:
        return (False, u"Dataset de descripciones vacio")
        
    if bit_index < 0 or bit_index >= row_count:
        return (False, u"Indice de bit (%d) fuera del rango del dataset (filas: %d)" % (bit_index, row_count))
        
    try:
        val = desc_dataset.getValueAt(bit_index, "Descripcio")
        clean_desc = unicode(val) if val is not None else u"Sin descripcion configurada"
        return (True, clean_desc)
    except Exception as err:
        return (False, u"Error al leer columna 'Descripcio': " + unicode(err))
```

#### Código Resultante en el Script del Tag (`valueChanged`)

```python
def valueChanged(tag, tagPath, previousValue, currentValue, initialChange, missedEvents):
    if initialChange or currentValue is None or previousValue is None:
        return
        
    current = currentValue.value
    prev = previousValue.value
    
    # Validar que ambos valores sean indexables
    if not hasattr(current, "__getitem__") or not hasattr(prev, "__getitem__"):
        return
        
    equip = project.enclavamientos.evaluator.extract_equipment_name(tagPath)
    desc_tag = system.tag.readBlocking(["[.]WENC_DESC"])[0].value
    
    max_bits = min(32, len(current), len(prev))
    
    for i in range(max_bits):
        if current[i] != prev[i]:
            bit_val = current[i]
            ok_desc, texto_desc = project.enclavamientos.evaluator.get_interlock_description_safe(desc_tag, i)
            
            if not ok_desc:
                texto_desc = u"Alarma Bit %d" % i
                
            # Procesar transiciones con datos validados...
```

---

## 4. Banco de Pruebas Automatizado (Script Console)

Siguiendo la metodología del **Laboratorio 3.2** (`SCADATestRunner`) y las técnicas de simulación con Mocks del **Laboratorio 3.1** (`system.dataset.toDataSet`), se consolida el siguiente script ejecutable directamente en la **Script Console** de Ignition (**Tools -> Script Console**). Este banco de pruebas evalúa exhaustivamente todas las funciones refactorizadas sometiéndolas a casos nominales, entradas nulas y valores límite:

```python
# ==============================================================================
# BANCO DE PRUEBAS UNITARIAS: SESION 3 (SCRIPT CONSOLE RUNNER)
# ==============================================================================

class SCADATestRunner(object):
    """
    Runner ligero de pruebas unitarias para validar funciones en Ignition Designer.
    (Implementado conforme al Laboratorio 3.2 de sesion-03).
    """
    def __init__(self, suite_name):
        self.suite_name = suite_name
        self.total_tests = 0
        self.passed_tests = 0
        self.failures = []

    def assert_equal(self, test_name, actual, expected):
        """Comprueba igualdad estricta entre resultado obtenido y esperado."""
        self.total_tests += 1
        if actual == expected:
            self.passed_tests += 1
            print "  [ PASS ] {}".format(test_name)
        else:
            self.failures.append({
                "name": test_name,
                "expected": expected,
                "actual": actual
            })
            print "  [ FAIL ] {}".format(test_name)

    def print_report(self):
        """Imprime informe consolidado de la suite de pruebas."""
        print "\n" + "=" * 75
        print "INFORME DE TESTING: {}".format(self.suite_name)
        print "Total Ejecutados: {} | Superados: {} | Fallidos: {}".format(
            self.total_tests, self.passed_tests, len(self.failures)
        )
        print "=" * 75
        if self.failures:
            print "DETALLE DE FALLOS ENCONTRADOS:"
            for f in self.failures:
                print " - Test:     '{}'".format(f["name"])
                print "   Esperado: {}".format(f["expected"])
                print "   Obtenido: {}".format(f["actual"])
        else:
            print ">>> TODOS LOS CASOS DE PRUEBA FUERON SUPERADOS SATISFACTORIAMENTE <<<"
        print "=" * 75


# Instanciar Suite
runner = SCADATestRunner("Certificacion de Scripts Refactorizados (Sesion 3)")

print "=== INICIANDO BATERIA DE ASIONES EN SCRIPT CONSOLE ==="

# ------------------------------------------------------------------------------
# 1. PRUEBAS DE MOCKING Y COALESCENCIA SQL (Lab 3.1)
# ------------------------------------------------------------------------------
# Mock idéntico a sqlt_data_1_{mes} con casos nominales y nulos
mock_headers = ["tagpath", "intvalue", "floatvalue", "stringvalue", "datevalue", "date"]
mock_data = [
    ["[default]EDAR/BOMBA_01/SCFL", None,  45.8, "ESTADO_OK", None, "2026-10-08 14:00:00"],
    ["[default]EDAR/BOMBA_01/SCMX",  100,  None, "ESTADO_OK", None, "2026-10-08 14:05:00"],
    ["[default]EDAR/COMPRESOR/SCFL", None,  None, "ALERTA",    None, "2026-10-08 14:10:00"],
    ["[default]EDAR/COMPRESOR/VACIO",None,  None, None,        None, "2026-10-08 14:15:00"]
]
mock_ds = system.dataset.toDataSet(mock_headers, mock_data)

# Test 1.1: Prioridad de Float sobre otras columnas
found, val, dt = project.historian.consignas.extract_scada_history_record(mock_ds, "[default]EDAR/BOMBA_01/SCFL")
runner.assert_equal("Mock SQL: Coalescencia Float correcta (45.8)", (found, val), (True, 45.8))

# Test 1.2: Asignación de Int cuando Float es nulo
found, val, dt = project.historian.consignas.extract_scada_history_record(mock_ds, "[default]EDAR/BOMBA_01/SCMX")
runner.assert_equal("Mock SQL: Coalescencia Int correcta (100)", (found, val), (True, 100))

# Test 1.3: Asignación de String cuando numéricos son nulos
found, val, dt = project.historian.consignas.extract_scada_history_record(mock_ds, "[default]EDAR/COMPRESOR/SCFL")
runner.assert_equal("Mock SQL: Coalescencia String correcta ('ALERTA')", (found, val), (True, u"ALERTA"))

# Test 1.4: Tolerancia a Dataset nulo (None)
found, val, dt = project.historian.consignas.extract_scada_history_record(None, "[default]EDAR/BOMBA_01/SCFL")
runner.assert_equal("Mock SQL: Dataset nulo manejado sin excepcion", (found, val), (False, None))

# ------------------------------------------------------------------------------
# 2. PRUEBAS DE RESOLUCIÓN DE COMANDOS M* (Lab 3.2 y 3.3)
# ------------------------------------------------------------------------------
# Test 2.1: Caso nominal estándar (S* -> M*)
runner.assert_equal(
    "Comando M: Caso nominal '[default]PLANTA/BOMBA/SCFL' -> 'MCFL'",
    project.consignas.tags.resolve_command_tag_path("[default]PLANTA/BOMBA/SCFL", False),
    (True, u"[default]PLANTA/BOMBA/MCFL")
)

# Test 2.2: Caso especial SFOR sin simulación
runner.assert_equal(
    "Comando M: 'SFOR' con ESIM=False -> 'MFOR'",
    project.consignas.tags.resolve_command_tag_path("[default]PLANTA/BOMBA/SFOR", False),
    (True, u"[default]PLANTA/BOMBA/MFOR")
)

# Test 2.3: Caso especial SFOR con simulación activa
runner.assert_equal(
    "Comando M: 'SFOR' con ESIM=True -> 'MFOS'",
    project.consignas.tags.resolve_command_tag_path("[default]PLANTA/BOMBA/SFOR", True),
    (True, u"[default]PLANTA/BOMBA/MFOS")
)

# Test 2.4: Protección contra rutas nulas o vacías
runner.assert_equal(
    "Comando M: Rechazo seguro ante tagPath nulo (None)",
    project.consignas.tags.resolve_command_tag_path(None, False)[0],
    False
)

# Test 2.5: Preservación de nombres de carpeta que contengan el tag
runner.assert_equal(
    "Comando M: No sustituir carpetas intermedias coincidentes ('.../SFOR_AREA/SFOR')",
    project.consignas.tags.resolve_command_tag_path("[default]EDAR/SFOR_AREA/SFOR", False),
    (True, u"[default]EDAR/SFOR_AREA/MFOR")
)

# ------------------------------------------------------------------------------
# 3. PRUEBAS DE DESCOMPOSICIÓN DE ÁRBOL DE NAVEGACIÓN (Lab 3.3)
# ------------------------------------------------------------------------------
mock_tree = [
    {
        "label": "Depuracion",
        "items": [
            {"label": "Bombeo Primario", "items": [{"label": "Bomba 1"}]},
            {"label": "Reactores"}
        ]
    },
    {"label": "Filtracion"}
]

# Test 3.1: Navegación nominal a 3 niveles ('0/0/0')
ok, payload, full_title = project.nav.tree.parse_navigation_breadcrumbs("0/0/0", mock_tree, "Detalle")
runner.assert_equal(
    "Arbol: Jerarquia completa nominal a 3 niveles",
    (ok, payload["pretittle1"], payload["pretittle2"], payload["pretittle3"]),
    (True, u"Depuracion", u" > Bombeo Primario", u" > Bomba 1")
)

# Test 3.2: Protección contra segmentos corruptos no numéricos ('0/CORRUPTO/1')
ok, payload, full_title = project.nav.tree.parse_navigation_breadcrumbs("0/CORRUPTO/1", mock_tree, "Detalle")
runner.assert_equal(
    "Arbol: Rechazo ante clave no numerica sin lanzar ValueError",
    ok,
    False
)

# Test 3.3: Protección ante índices fuera de rango ('0/99')
ok, payload, full_title = project.nav.tree.parse_navigation_breadcrumbs("0/99", mock_tree, "Detalle")
runner.assert_equal(
    "Arbol: Truncado seguro ante desbordamiento de indice sin lanzar IndexError",
    (ok, payload["pretittle1"], payload["pretittle2"]),
    (True, u"Depuracion", u"")
)

# ------------------------------------------------------------------------------
# 4. PRUEBAS DE ENCLAVAMIENTOS Y LÍMITES DE DATASET (Lab 3.1 y 3.2)
# ------------------------------------------------------------------------------
desc_headers = ["Descripcio"]
desc_data = [
    ["Termico Bomba 01 Disparado"],
    ["Nivel Muy Alto Pozo"],
    ["Parada de Emergencia Pulsada"]
]
mock_desc_ds = system.dataset.toDataSet(desc_headers, desc_data)

# Test 4.1: Lectura nominal dentro del rango
runner.assert_equal(
    "Enclavamiento: Indice 0 dentro de limites",
    project.enclavamientos.evaluator.get_interlock_description_safe(mock_desc_ds, 0),
    (True, u"Termico Bomba 01 Disparado")
)

# Test 4.2: Protección contra desbordamiento de índice (bit 10 en dataset de 3 filas)
runner.assert_equal(
    "Enclavamiento: Rechazo seguro de bit fuera de rango (10 >= 3)",
    project.enclavamientos.evaluator.get_interlock_description_safe(mock_desc_ds, 10)[0],
    False
)

# Test 4.3: Protección ante dataset de descripciones nulo
runner.assert_equal(
    "Enclavamiento: Manejo defensivo de dataset nulo (None)",
    project.enclavamientos.evaluator.get_interlock_description_safe(None, 0)[0],
    False
)

# Emitir informe consolidado
runner.print_report()
```

#### Salida Estructurada Esperada en Script Console

```
=== INICIANDO BATERIA DE ASIONES EN SCRIPT CONSOLE ===
  [ PASS ] Mock SQL: Coalescencia Float correcta (45.8)
  [ PASS ] Mock SQL: Coalescencia Int correcta (100)
  [ PASS ] Mock SQL: Coalescencia String correcta ('ALERTA')
  [ PASS ] Mock SQL: Dataset nulo manejado sin excepcion
  [ PASS ] Comando M: Caso nominal '[default]PLANTA/BOMBA/SCFL' -> 'MCFL'
  [ PASS ] Comando M: 'SFOR' con ESIM=False -> 'MFOR'
  [ PASS ] Comando M: 'SFOR' con ESIM=True -> 'MFOS'
  [ PASS ] Comando M: Rechazo seguro ante tagPath nulo (None)
  [ PASS ] Comando M: No sustituir carpetas intermedias coincidentes ('.../SFOR_AREA/SFOR')
  [ PASS ] Arbol: Jerarquia completa nominal a 3 niveles
  [ PASS ] Arbol: Rechazo ante clave no numerica sin lanzar ValueError
  [ PASS ] Arbol: Truncado seguro ante desbordamiento de indice sin lanzar IndexError
  [ PASS ] Enclavamiento: Indice 0 dentro de limites
  [ PASS ] Enclavamiento: Rechazo seguro de bit fuera de rango (10 >= 3)
  [ PASS ] Enclavamiento: Manejo defensivo de dataset nulo (None)

===========================================================================
INFORME DE TESTING: Certificacion de Scripts Refactorizados (Sesion 3)
Total Ejecutados: 15 | Superados: 15 | Fallidos: 0
===========================================================================
>>> TODOS LOS CASOS DE PRUEBA FUERON SUPERADOS SATISFACTORIAMENTE <<<
===========================================================================
```

---

## 5. Cuadro Comparativo de Beneficios

| Parámetro | Estado Anterior (Código Heredado) | Estado Refactorizado (Sesión 3 / Lab-03) | Impacto Técnico en Operación |
| :--- | :--- | :--- | :--- |
| **Tolerancia a Excepciones** | Bloques `try/except:` ciegos que ocultan caídas de BD y fallos de tipo | Validaciones de esquema, control explícito de nulos y `except` tipados | Diagnóstico inmediato de caídas de red y retención de integridad de datos |
| **Lectura de Datasets** | Bucle for con múltiples llamadas RPC `.getValueAt(row, col)` | Transformación a `PyDataSet` e iteración nativa en memoria | Reducción del tiempo de ejecución en bindings y menor uso de CPU en Gateway |
| **Coalescencia de Tipos** | Comprobaciones dispersas de `intvalue`/`floatvalue` en UI | Función modular pura con orden jerárquico estricto (Float > Int > Str > Date) | Datos de proceso precisos sin valores truncados ni excepciones `TypeError` |
| **Manipulación de Rutas** | Partición con `.split()` y conversiones `int()` a ciegas | Cláusulas de guarda, validación de longitud y casteo protegido con `ValueError` | Cero congelamientos de interfaz en árbol de navegación y botón de consignas |
| **Resolución de Comandos M\*** | `.replace(tag, MName)` global sobre la ruta completa | Reemplazo acotado exclusivamente en el nodo hoja (`parts[-1]`) | Eliminación de escrituras erróneas en carpetas intermedias del SCADA |
| **Testing y Calidad** | Pruebas manuales directamente sobre la planta en producción | Banco de pruebas unitarias (`SCADATestRunner`) con datos sintéticos (*Mocks*) | Validación al 100% en Designer Scope antes de desplegar en producción |
| **Contratos de Salida** | Variables sin formato predefinido o fallos no capturados | Retorno estructurado de tuplas `(bool success, unicode resultado_o_error)` | Interfaz informada con mensajes claros al operador ante cualquier fallo |
