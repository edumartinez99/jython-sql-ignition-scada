# lab-11

## Guía Paso a Paso de Laboratorios - Sesión 11

***

### Laboratorio 11.1: Desarrollo e Integración Multicapa del Proyecto Final (Grado Industrial)

#### 1. Objetivos

* Implementar y ensamblar de forma ágil y realista las **6 capas arquitectónicas + Capa 0 de campo** del **Proyecto Final Integrador** en Ignition SCADA:
  * **Capa 0 (Transversal - Tags y Telemetría Viva):** Jerarquía de 8 tags esenciales por equipo (`system.tag.configure`), lectura/escritura masiva (`readBlocking` / `writeBlocking`) y simulador en tiempo real (`plant.simulator`).
  * **Capa 1 (Persistencia SQL DDL y Semilla Poblada):** Esquema relacional indexado (`plant_machines`, `production_batches`, `ins_Registre_Actuacions`) poblado con histórico multi-turno y lotes activos.
  * **Capa 2 (Named Queries Parametrizadas):** 2 consultas esenciales con *Value Parameters* (`Production/GetShiftSummary` y `Production/CloseBatchWithAudit`).
  * **Capa 3 (`Project Library` - `plant.service`):** Servicio construido heredando de la plantilla corporativa `plant.service_template` (`class PlantProductionService(BaseSCADAService)` del Lab 10.2), integrando funciones puras, lectura de 7 tags del PLC, transacciones ACID (`system.db.beginTransaction`) y handshake de cierre con el PLC.
  * **Seguridad RBAC (Transversal Capas 3 y 4 — Enfoque A y Enfoque B):** Implementación y prueba con **4 perfiles de usuario** (`visita_auditor`, `operador_noche`, `laura.vidal`, `marti.sanchez`) comparando **Enfoque A (Roles en Jython)** frente a **Enfoque B (Security Zones + Security Levels de Ignition)**.
  * **Capa 4 (Perspective UI Ágil + Message Handlers Reales):** Vista con solo **5 Indirect Tag Bindings esenciales** (con JSON base opcional para pegar el layout en un clic), cálculo reactivo de OEE mediante **Script Transform** y **2 Message Handlers receptores reales (`notify_toast` y `OnBatchStatusChanged`)**.
  * **Capa 5 (Gateway Scheduled Script):** Consolidación automática de fin de turno CRON 100% idempotente.
  * **Capa 6 (Certificación QA con `unittest`):** Suite automatizada y repetible usando la librería estándar **`unittest`** de Jython (`qa.test_plant_unittest`).

#### 2. Resultado esperado

* Un sistema SCADA industrial completo, limpio y manejable dentro de la sesión: los tags cambian en vivo, Perspective muestra el OEE por máquina con Indirect Tag Bindings, valida permisos en Enfoque A y Enfoque B con distintos usuarios, ejecuta transacciones ACID con handshake al PLC, refresca tablas mediante Message Handlers reales y supera al 100% los 5 tests de `unittest`.

#### 3. Mapa de Arquitectura y Topología Cloud

> [!IMPORTANT]
> **Topología del Entorno (Designer Local vs. Gateway en Servidor Cloud):**
> El **Ignition Gateway** y **PostgreSQL (`DB_Admin`)** corren en un **servidor en la nube** (`http://35.204.54.113:8088/`), separado del puesto donde ejecutas el **Designer** y el navegador:
> 1. **Scopes:** Perspective y el **Gateway Scheduled Script** se ejecutan en el servidor cloud; la **Script Console** corre en tu Designer local conectándose por red.
> 2. **Transacciones ACID Multi-Scope:** Usamos `system.db.beginTransaction(database=DB_CONN, timeout=5000)`, cuya firma funciona idéntica tanto desde **Perspective (Cloud)** como desde **Script Console (Designer)**.
> 3. **Security Zones en Cloud (`Zona_Planta_Control`):** El Gateway no ve `127.0.0.1`, sino la IP remota de tu puesto cliente. Verifícala en **Status -> Connections -> Perspective Sessions** al configurar `Zona_Planta_Control`.

```
 =========================================================================================
              MAPA DE INTEGRACIÓN MULTICAPA DEL PROYECTO FINAL (ÁGIL)
 =========================================================================================
  [ CAPA 0: TAGS & PLC SIM ]    -> [default]PLANTA_PRODUCCION/<EQUIPO>/ (8 tags esenciales)
                │ ▲                plant.simulator (Batch readBlocking / writeBlocking)
                ▼ │
  [ CAPA 1: SQL DDL + ÍNDICES ] -> plant_machines, production_batches, ins_Registre_Actuacions
                │ ▲
                ▼ │ (:param / ACID)
  [ CAPA 2: NAMED QUERIES ]     -> Production/GetShiftSummary y Production/CloseBatchWithAudit
                │ ▲
                ▼ │
  [ CAPA 3: PROJECT LIBRARY ]   -> PlantProductionService (hereda de plant.service_template)
                                -> Funciones Puras (OEE + Enclavamientos) + Lectura 7 Tags
                                -> RBAC Enfoque A (Roles Jython) y Enfoque B (Zones + Levels)
         │               │               │
         ▼               ▼               ▼
  [ CAPA 4: UI ]  [ CAPA 5: CRON ]  [ CAPA 6: QA TESTS ]
  Perspective     Gateway Scheduled  Librería estándar Jython `unittest`
  5 Indirect Tags Cierre de Turno    5 tests reproducibles (4 cuadrantes +
  ScriptTransform Idempotente 24/7   Mocks OPC + Multi-Usuario RBAC A/B)
  2 Msg Handlers
 =========================================================================================
```

***

#### 4. Paso a Paso Detallado para la Realización

---

### Paso 0: Capa Transversal de Campo — Creación de Tags Esenciales y Simulador en Vivo

> [!NOTE]
> **Recursos Base Entregados al Alumno:** Tanto el script de creación de tags (`0.1`) como el simulador `plant.simulator` (`0.2`) se entregan hechos en [sesion-11.md](file:///mnt/c/repos/jython-sql-ignition-scada/sesion-11.md) (*Recursos Entregados 1 y 2*).

#### 0.1. Creación Automatizada de los 8 Tags Esenciales por Equipo (`system.tag.configure`)

1. En **Ignition Designer**, abrir **Tools -> Script Console**.
2. Pegar y ejecutar el script que crea `[default]PLANTA_PRODUCCION` con 4 equipos y sus **8 tags esenciales** (7 de lectura + 1 de handshake de cierre):

```python
# ==============================================================================
# PASO 0.1: APROVISIONAMIENTO AUTOMÁTICO DE TAGS DE PLANTA (8 TAGS ESENCIALES)
# Ejecutar en: Tools -> Script Console
# ==============================================================================
tag_provider = "[default]"
root_folder = "PLANTA_PRODUCCION"

equipos_config = [
    {"id": "EDAR_BOMBA_01",     "lote": "LOT-2026-FINAL-01", "target": 5000,  "good": 4850,  "scrap": 65,  "op_min": 420.0},
    {"id": "EDAR_BOMBA_02",     "lote": "LOT-2026-FINAL-04", "target": 4500,  "good": 4380,  "scrap": 95,  "op_min": 410.0},
    {"id": "EDAR_COMPRESOR_01", "lote": "LOT-2026-FINAL-03", "target": 1200,  "good": 980,   "scrap": 15,  "op_min": 310.0},
    {"id": "LINEA_ENVASADO_01", "lote": "LOT-2026-FINAL-02", "target": 15000, "good": 11200, "scrap": 180, "op_min": 280.0}
]

equipment_folders = []
for eq in equipos_config:
    equipment_folders.append({
        "name": eq["id"],
        "tagType": "Folder",
        "tags": [
            # NOTA IGNITION 8.1: Usar "valueSource": "memory" y "dataType"
            {"name": "ModoRemoto",         "tagType": "AtomicTag", "valueSource": "memory", "dataType": "Boolean", "value": True},
            {"name": "SetaEmergencia",     "tagType": "AtomicTag", "valueSource": "memory", "dataType": "Boolean", "value": False},
            {"name": "OrdenCierrePLC",     "tagType": "AtomicTag", "valueSource": "memory", "dataType": "Boolean", "value": False},
            {"name": "LoteActivoID",       "tagType": "AtomicTag", "valueSource": "memory", "dataType": "String",  "value": eq["lote"]},
            {"name": "UnidadesObjetivo",   "tagType": "AtomicTag", "valueSource": "memory", "dataType": "Int4",    "value": eq["target"]},
            {"name": "UnidadesBuenas",     "tagType": "AtomicTag", "valueSource": "memory", "dataType": "Int4",    "value": eq["good"]},
            {"name": "UnidadesScrap",      "tagType": "AtomicTag", "valueSource": "memory", "dataType": "Int4",    "value": eq["scrap"]},
            {"name": "TiempoOperativoMin", "tagType": "AtomicTag", "valueSource": "memory", "dataType": "Float8",  "value": eq["op_min"]}
        ]
    })

full_root_path = "{}{}".format(tag_provider, root_folder)
if system.tag.exists(full_root_path):
    system.tag.deleteTags([full_root_path])

results = system.tag.configure(tag_provider, [{"name": root_folder, "tagType": "Folder", "tags": equipment_folders}], "o")
print "Tags creados en [default]PLANTA_PRODUCCION/ ->", [r.toString() for r in results]
```

#### 0.2. Simulador de Telemetría Viva (`plant.simulator`)

1. En **Project Browser -> Server Scripting -> Project Library**, crear el paquete `plant` y dentro el script `simulator` (`plant.simulator`):

```python
"""
Modulo: plant.simulator
Descripcion: Simulador ligero de contadores PLC mediante lectura/escritura en bloque.
"""
import random

TAG_ROOT = "[default]PLANTA_PRODUCCION"
MACHINES = ["EDAR_BOMBA_01", "EDAR_BOMBA_02", "EDAR_COMPRESOR_01", "LINEA_ENVASADO_01"]


def step_simulation_tick(increment_counters=True):
    """Incrementa en bloque UnidadesBuenas, UnidadesScrap y TiempoOperativoMin si no hay SetaEmergencia."""
    read_paths = []
    for m in MACHINES:
        base = "{}/{}".format(TAG_ROOT, m)
        read_paths.extend([
            base + "/ModoRemoto",
            base + "/SetaEmergencia",
            base + "/UnidadesBuenas",
            base + "/UnidadesScrap",
            base + "/TiempoOperativoMin"
        ])

    qvs = system.tag.readBlocking(read_paths, 3000)
    write_paths, write_values, summary = [], [], {}

    for idx, m in enumerate(MACHINES):
        off = idx * 5
        is_remote = bool(qvs[off].value) if qvs[off].quality.isGood() else False
        is_estop = bool(qvs[off + 1].value) if qvs[off + 1].quality.isGood() else False
        good = int(qvs[off + 2].value or 0)
        scrap = int(qvs[off + 3].value or 0)
        op_min = float(qvs[off + 4].value or 0.0)

        if is_remote and not is_estop and increment_counters:
            good += random.randint(5, 15)
            scrap += (1 if random.random() < 0.25 else 0)
            op_min = round(op_min + 0.2, 2)

        base = "{}/{}".format(TAG_ROOT, m)
        write_paths.extend([base + "/UnidadesBuenas", base + "/UnidadesScrap", base + "/TiempoOperativoMin"])
        write_values.extend([good, scrap, op_min])
        summary[m] = {"good": good, "scrap": scrap, "op_min": op_min}

    system.tag.writeBlocking(write_paths, write_values, 3000)
    return summary


def reset_seed_counters_for_qa():
    """Restaura los valores semilla deterministas de las 4 maquinas para la suite unittest y Perspective."""
    seeds = [
        ("EDAR_BOMBA_01",     "LOT-2026-FINAL-01", 5000,  4850,  65,  420.0),
        ("EDAR_BOMBA_02",     "LOT-2026-FINAL-04", 4500,  4380,  95,  410.0),
        ("EDAR_COMPRESOR_01", "LOT-2026-FINAL-03", 1200,   980,  15,  310.0),
        ("LINEA_ENVASADO_01", "LOT-2026-FINAL-02", 15000, 11200, 180, 280.0)
    ]
    paths, vals = [], []
    for eq, lote, target, good, scrap, op_min in seeds:
        base = "{}/{}".format(TAG_ROOT, eq)
        paths.extend([
            base + "/ModoRemoto", base + "/SetaEmergencia", base + "/OrdenCierrePLC",
            base + "/LoteActivoID", base + "/UnidadesObjetivo", base + "/UnidadesBuenas",
            base + "/UnidadesScrap", base + "/TiempoOperativoMin"
        ])
        vals.extend([True, False, False, lote, target, good, scrap, op_min])
    system.tag.writeBlocking(paths, vals, 3000)
    return True
```

2. Guardar el proyecto (`Ctrl+S`). *(Opcional: Si quieres avance automático continuo sin pulsar botón, añade una línea `plant.simulator.step_simulation_tick()` en **Gateway Event Scripts -> Timer** cada `5000 ms`)*.

---

### Paso 1: Capa 1 — Persistencia Relacional SQL DDL y Semilla Poblada

> [!NOTE]
> En [sesion-11.md](file:///mnt/c/repos/jython-sql-ignition-scada/sesion-11.md) el alumno recibe únicamente las sentencias `INSERT INTO` (*Recurso Entregado 3*) y debe redactar por sí mismo los `CREATE TABLE` y `CREATE INDEX` del apartado 1.1.

#### 1.1. Creación del Esquema Relacional e Índices (`CREATE TABLE` + `CREATE INDEX`)

Abrir **Tools -> Database Query Browser** (conexión `DB_Admin`) y ejecutar:

```sql
-- =============================================================================
-- PASO 1.1: SOLUCIÓN DDL (ESCRITO POR EL ALUMNO EN EL RETO 1)
-- =============================================================================
CREATE TABLE IF NOT EXISTS plant_machines (
    machine_id VARCHAR(50) PRIMARY KEY,
    line_name VARCHAR(60) NOT NULL,
    ideal_cycle_time_sec DOUBLE PRECISION NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE IF NOT EXISTS production_batches (
    batch_id VARCHAR(50) PRIMARY KEY,
    machine_id VARCHAR(50) NOT NULL REFERENCES plant_machines(machine_id),
    shift_id INT NOT NULL CHECK (shift_id IN (1, 2, 3)),
    target_units INT NOT NULL,
    good_units INT NOT NULL DEFAULT 0,
    scrap_units INT NOT NULL DEFAULT 0,
    operating_time_min DOUBLE PRECISION NOT NULL DEFAULT 0.0,
    status VARCHAR(20) NOT NULL DEFAULT 'IN_PROGRESS',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    closed_at TIMESTAMP NULL,
    closed_by VARCHAR(100) NULL
);

CREATE TABLE IF NOT EXISTS ins_Registre_Actuacions (
    id SERIAL PRIMARY KEY,
    Equip VARCHAR(60) NOT NULL,
    Descripcio TEXT NOT NULL,
    Usuari VARCHAR(80) NOT NULL,
    DataHora TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_batches_shift_created ON production_batches (shift_id, created_at);
CREATE INDEX IF NOT EXISTS idx_batches_status_created ON production_batches (status, created_at);
CREATE INDEX IF NOT EXISTS idx_audit_equip_datahora ON ins_Registre_Actuacions (Equip, DataHora DESC);
```

#### 1.2. Ejecución de los `INSERT INTO` de Semilla (Entregados al Alumno)

Ejecutar en **Database Query Browser** (`DB_Admin`):

```sql
-- =============================================================================
-- PASO 1.2: POBLADO COMPARTIBLE DE DATOS (RECURSO ENTREGADO 3)
-- =============================================================================
INSERT INTO plant_machines (machine_id, line_name, ideal_cycle_time_sec, is_active)
VALUES
    ('EDAR_BOMBA_01',     'LINEA_TRACTAMENT_AIGUA', 2.5,  TRUE),
    ('EDAR_BOMBA_02',     'LINEA_TRACTAMENT_AIGUA', 2.5,  TRUE),
    ('EDAR_COMPRESOR_01', 'LINEA_GENERACIO_AIRE',  12.0,  TRUE),
    ('LINEA_ENVASADO_01', 'LINEA_ENVASAT_PET',      1.2,  TRUE)
ON CONFLICT (machine_id) DO UPDATE SET
    line_name = EXCLUDED.line_name,
    ideal_cycle_time_sec = EXCLUDED.ideal_cycle_time_sec,
    is_active = EXCLUDED.is_active;

INSERT INTO production_batches (batch_id, machine_id, shift_id, target_units, good_units, scrap_units, operating_time_min, status, created_at, closed_at, closed_by)
VALUES
    -- Histórico cerrado en Turnos 1 y 2
    ('LOT-2026-HIST-01',  'EDAR_BOMBA_01',     1, 5000,  4910,  45, 435.0, 'CLOSED',      CURRENT_TIMESTAMP - INTERVAL '6 hours',  CURRENT_TIMESTAMP - INTERVAL '2 hours', 'laura.vidal'),
    ('LOT-2026-HIST-02',  'EDAR_BOMBA_02',     1, 4500,  4380,  95, 410.0, 'CLOSED',      CURRENT_TIMESTAMP - INTERVAL '5 hours',  CURRENT_TIMESTAMP - INTERVAL '1 hour',  'laura.vidal'),
    ('LOT-2026-HIST-03',  'LINEA_ENVASADO_01', 1, 15000, 14650, 210, 395.0, 'CLOSED',     CURRENT_TIMESTAMP - INTERVAL '5 hours',  CURRENT_TIMESTAMP - INTERVAL '1 hour',  'marti.sanchez'),
    ('LOT-2026-HIST-04',  'EDAR_BOMBA_01',     2, 5000,  4720, 130, 405.0, 'CLOSED',      CURRENT_TIMESTAMP - INTERVAL '14 hours', CURRENT_TIMESTAMP - INTERVAL '8 hours', 'GATEWAY_CRON_AUTO'),
    ('LOT-2026-HIST-05',  'EDAR_COMPRESOR_01', 2, 1200,  1150,  18, 380.0, 'CLOSED',      CURRENT_TIMESTAMP - INTERVAL '13 hours', CURRENT_TIMESTAMP - INTERVAL '8 hours', 'GATEWAY_CRON_AUTO'),

    -- 4 Lotes ACTIVOS ('IN_PROGRESS', 1 por maquina) para pruebas en Perspective, RBAC, QA y CRON
    ('LOT-2026-FINAL-01', 'EDAR_BOMBA_01',     1, 5000,  4850,  65, 420.0, 'IN_PROGRESS', CURRENT_TIMESTAMP - INTERVAL '30 minutes', NULL, NULL),
    ('LOT-2026-FINAL-02', 'LINEA_ENVASADO_01', 1, 15000, 11200, 180, 280.0, 'IN_PROGRESS', CURRENT_TIMESTAMP - INTERVAL '25 minutes', NULL, NULL),
    ('LOT-2026-FINAL-03', 'EDAR_COMPRESOR_01', 1, 1200,   980,  15, 310.0, 'IN_PROGRESS', CURRENT_TIMESTAMP - INTERVAL '20 minutes', NULL, NULL),
    ('LOT-2026-FINAL-04', 'EDAR_BOMBA_02',     1, 4500,  4380,  95, 410.0, 'IN_PROGRESS', CURRENT_TIMESTAMP - INTERVAL '15 minutes', NULL, NULL)
ON CONFLICT (batch_id) DO UPDATE SET
    good_units = EXCLUDED.good_units,
    scrap_units = EXCLUDED.scrap_units,
    operating_time_min = EXCLUDED.operating_time_min,
    status = EXCLUDED.status,
    created_at = EXCLUDED.created_at,
    closed_at = EXCLUDED.closed_at,
    closed_by = EXCLUDED.closed_by;

INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
SELECT 'EDAR_BOMBA_01', 'OPERACION: Inicio de lote LOT-2026-FINAL-01 en modo REMOTO', 'laura.vidal', CURRENT_TIMESTAMP - INTERVAL '30 minutes'
WHERE NOT EXISTS (SELECT 1 FROM ins_Registre_Actuacions WHERE Descripcio LIKE '%LOT-2026-FINAL-01%');

INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
SELECT 'EDAR_BOMBA_01', 'SEGURIDAD: Intento no autorizado de cierre de lote sin rol Supervisor', 'operador_noche', CURRENT_TIMESTAMP - INTERVAL '15 minutes'
WHERE NOT EXISTS (SELECT 1 FROM ins_Registre_Actuacions WHERE Usuari = 'operador_noche' AND Equip = 'EDAR_BOMBA_01');
```

---

### Paso 2: Capa 2 — Las 2 Named Queries Parametrizadas (`Production/*`)

En **Project Browser -> Named Queries**, crear la carpeta `Production` y configurar estas **2 únicas consultas** con **Value Parameters** (`:param`):

#### 2.1. Named Query 1: `Production/GetShiftSummary` (Consulta Analítica de Turno)
* **Database Connection:** `DB_Admin` | **Query Type:** `Query`
* **Value Parameters:** `shiftId` (`Integer`), `startDate` (`DateTime`), `endDate` (`DateTime`)
* **SQL:**

```sql
SELECT
    b.machine_id,
    m.line_name,
    COUNT(b.batch_id) AS total_batches,
    SUM(CASE WHEN b.status = 'IN_PROGRESS' THEN 1 ELSE 0 END) AS active_batches,
    SUM(b.good_units) AS sum_good,
    SUM(b.scrap_units) AS sum_scrap,
    ROUND(
        (SUM(b.scrap_units) * 100.0) / NULLIF(SUM(b.good_units + b.scrap_units), 0)::numeric,
        2
    ) AS scrap_rate_pct
FROM production_batches b
INNER JOIN plant_machines m ON b.machine_id = m.machine_id
WHERE
    b.shift_id = :shiftId
    AND b.created_at >= :startDate
    AND b.created_at <= :endDate
GROUP BY b.machine_id, m.line_name
ORDER BY m.line_name ASC, b.machine_id ASC;
```

#### 2.2. Named Query 2: `Production/CloseBatchWithAudit` (Escritura Idempotente)
* **Database Connection:** `DB_Admin` | **Query Type:** `Update Query`
* **Value Parameters:** `batchId` (`String`), `goodUnits` (`Integer`), `scrapUnits` (`Integer`), `user` (`String`)
* **SQL:**

```sql
UPDATE production_batches
SET
    good_units = :goodUnits,
    scrap_units = :scrapUnits,
    status = 'CLOSED',
    closed_at = CURRENT_TIMESTAMP,
    closed_by = :user
WHERE batch_id = :batchId
  AND status = 'IN_PROGRESS';
```

---

### Paso 3: Seguridad Industrial RBAC (Enfoque A y Enfoque B en Gateway Cloud)

#### 3.1. Creación de los 4 Usuarios de Prueba en el Gateway Cloud

1. Entrar en la web del Gateway en la nube (`http://35.204.54.113:8088/` -> **Config -> Security -> Users, Roles** -> User Source `default`).
2. Verificar los 4 roles (`PlantViewer`, `Operator`, `Supervisor`, `Administrator`) y los **4 usuarios de prueba**:

| Usuario (`Username`) | Contraseña | Roles Asignados | Comportamiento Esperado al Cerrar Lote |
| :--- | :--- | :--- | :--- |
| **`visita_auditor`** | `Ignition123!` | `PlantViewer` | **RECHAZADO** (Auditado como alerta RBAC). |
| **`operador_noche`** | `Ignition123!` | `Operator` | **RECHAZADO** (Operador sin privilegio de cierre). |
| **`laura.vidal`** | `Ignition123!` | `Operator`, `Supervisor` | **AUTORIZADA** (Supervisora de turno). |
| **`marti.sanchez`** | `Ignition123!` | `Administrator`, `Supervisor` | **AUTORIZADO** (Administrador SCADA). |

#### 3.2. Configuración de Security Zone y Security Levels (Para el Enfoque B)

1. En `http://35.204.54.113:8088/`, abrir **Status -> Connections -> Designer Sessions** (o **Perspective Sessions**) y anotar la **IP Address** con la que el servidor cloud ve a tu equipo cliente.
2. Ir a **Config -> Security -> Security Zones -> Create new Security Zone...**:
   * **Name:** `Zona_Planta_Control`
   * **IP Addresses:** `<TU_IP_CLIENTE>, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.1`
3. En **Config -> Security -> Security Levels**, verificar que existen `Authenticated/Roles/Supervisor`, `Authenticated/Roles/Administrator` y `SecurityZones/Zona_Planta_Control`.

---

### Paso 4: Capa 3 — Lógica de Negocio en `Project Library` (`plant.service`)

#### 4.1. Plantilla Canónica Entregada (`plant.service_template` del Lab 10.2)

En **Server Scripting -> Project Library**, dentro del paquete `plant`, crear `service_template` (`plant.service_template` — *Recurso Entregado 4* en [sesion-11.md](file:///mnt/c/repos/jython-sql-ignition-scada/sesion-11.md)):

```python
"""
Modulo: plant.service_template
Descripcion: Plantilla canonica corporativa (Sesion 10 - Lab 10.2) con soporte para
             RBAC Enfoque A (Roles), RBAC Enfoque B (Security Levels + Zones),
             auditoria en ins_Registre_Actuacions y transacciones ACID multi-scope.
"""
from java.lang import Exception as JavaException
from java.sql import SQLException

DEFAULT_DB_CONNECTION = "DB_Admin"


class BaseSCADAService(object):
    """Clase base corporativa para herencia POO en servicios de planta."""

    def __init__(self, module_name, db_connection=DEFAULT_DB_CONNECTION):
        self.module_name = module_name
        self.db_connection = db_connection
        self.logger = system.util.getLogger("SCADA.{}".format(module_name))

    def check_rbac_roles(self, user_roles, authorized_roles):
        """RBAC Enfoque A: Valida interseccion de lista de roles en Jython."""
        user_set = set(user_roles) if user_roles else set()
        auth_set = set(authorized_roles) if authorized_roles else set()
        return bool(user_set.intersection(auth_set))

    def check_rbac_security_levels(self, security_levels, required_levels, security_zones=None, allowed_zones=None):
        """RBAC Enfoque B: Valida Security Levels jerarquicos y Security Zones de Ignition."""
        lvl_set = set(security_levels) if security_levels else set()
        req_set = set(required_levels) if required_levels else set()
        if not bool(lvl_set.intersection(req_set)):
            return False
        if allowed_zones:
            zone_set = set(security_zones) if security_zones else set()
            return bool(zone_set.intersection(set(allowed_zones)))
        return True

    def record_audit(self, equipment_id, description, user_str, tx=None):
        """Inserta traza en ins_Registre_Actuacions (dentro de la transaccion ACID si tx != None)."""
        sql = """
            INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
            VALUES (?, ?, ?, CURRENT_TIMESTAMP)
        """
        params = [str(equipment_id), unicode(description), str(user_str)]
        if tx is not None:
            system.db.runPrepUpdate(sql, params, tx=tx)
        else:
            system.db.runPrepUpdate(sql, params, database=self.db_connection)

    def dispatch_acid_transaction(self, action_name, equipment_id, user_str, tx_operation_fn):
        """Ejecuta tx_operation_fn(tx_id) bajo transaccion ACID con commit/rollback/close garantizados."""
        tx_id = None
        try:
            tx_id = system.db.beginTransaction(database=self.db_connection, timeout=5000)
            payload = tx_operation_fn(tx_id)
            system.db.commitTransaction(tx_id)
            self.logger.info("[ACID:{}] Commit exitoso en '{}' por '{}'".format(action_name, equipment_id, user_str))
            return (True, payload)

        except SQLException as sql_ex:
            if tx_id is not None:
                system.db.rollbackTransaction(tx_id)
            self.logger.error("[ACID:{}] Rollback SQL en '{}': {}".format(action_name, equipment_id, sql_ex.getMessage()), sql_ex)
            return (False, {"error_code": "DB_SQL_ROLLBACK", "user_message": u"Error SQL: Transacción revertida"})

        except JavaException as java_ex:
            if tx_id is not None:
                system.db.rollbackTransaction(tx_id)
            self.logger.error("[ACID:{}] Rollback Java en '{}': {}".format(action_name, equipment_id, java_ex.getMessage()), java_ex)
            return (False, {"error_code": "GATEWAY_RUNTIME_ROLLBACK", "user_message": u"Error Gateway: Transacción revertida"})

        except Exception as py_ex:
            if tx_id is not None:
                system.db.rollbackTransaction(tx_id)
            self.logger.error("[ACID:{}] Rollback en '{}': {}".format(action_name, equipment_id, str(py_ex)))
            return (False, {"error_code": "INTERNAL_LOGIC_ROLLBACK", "user_message": unicode(str(py_ex))})

        finally:
            if tx_id is not None:
                try:
                    system.db.closeTransaction(tx_id)
                except Exception:
                    pass
```

---

#### 4.2. Implementación Concisa de `plant.service` (Heredando de `plant.service_template`)

En **Server Scripting -> Project Library**, dentro de `plant`, crear el script `service` (`plant.service`).
Observa cómo queda reducido a ~175 líneas limpias con las 2 funciones puras, la lectura en bloque de los **7 tags esenciales**, el handshake de cierre al PLC y la clase `PlantProductionService(BaseSCADAService)`:

```python
"""
Modulo: plant.service
Descripcion: Servicio de planta del Proyecto Final (Sesion 11) heredado de plant.service_template.
"""
from plant.service_template import BaseSCADAService

DB_CONN = "DB_Admin"
TAG_ROOT = "[default]PLANTA_PRODUCCION"

ALLOWED_ROLES = ["Supervisor", "Administrator"]
ALLOWED_SECURITY_LEVELS = [
    "Authenticated/Roles/Supervisor",
    "Authenticated/Roles/Administrator"
]
ALLOWED_SECURITY_ZONES = ["Zona_Planta_Control"]


# -----------------------------------------------------------------------------
# 1. FUNCIONES PURAS (0% I/O, 100% Testeables en Memoria)
# -----------------------------------------------------------------------------
def calculate_batch_kpis(target_units, good_units, scrap_units, operating_min, ideal_cycle_sec=2.5):
    """Calcula Disponibilidad, Rendimiento, Calidad y OEE sin acceso a BD ni Tags."""
    if None in (target_units, good_units, scrap_units, operating_min, ideal_cycle_sec):
        return {"availability": 0.0, "performance": 0.0, "quality": 0.0, "oee": 0.0, "scrap_rate": 0.0}
    try:
        good = float(good_units)
        scrap = float(scrap_units)
        op_min = float(operating_min)
        cycle_sec = float(ideal_cycle_sec)
    except (ValueError, TypeError):
        return {"availability": 0.0, "performance": 0.0, "quality": 0.0, "oee": 0.0, "scrap_rate": 0.0}

    if good < 0.0 or scrap < 0.0 or op_min <= 0.0 or cycle_sec <= 0.0:
        return {"availability": 0.0, "performance": 0.0, "quality": 0.0, "oee": 0.0, "scrap_rate": 0.0}

    total = good + scrap
    quality = (good / total * 100.0) if total > 0.0 else 100.0
    scrap_rate = (scrap / total * 100.0) if total > 0.0 else 0.0

    theoretical_max = (op_min * 60.0) / cycle_sec
    performance = min(max((total / theoretical_max * 100.0) if theoretical_max > 0.0 else 0.0, 0.0), 100.0)
    availability = 100.0
    oee = round((availability * performance * quality) / 10000.0, 2)

    return {
        "availability": round(availability, 2),
        "performance": round(performance, 2),
        "quality": round(quality, 2),
        "oee": oee,
        "scrap_rate": round(scrap_rate, 2)
    }


def evaluate_plc_interlocks_pure(is_remote, is_estop, all_qualities_good=True):
    """Evalua enclavamientos fisicos del cuadro PLC (Calidad OPC, SetaEmergencia y ModoRemoto)."""
    if not all_qualities_good:
        return (False, "ERR_BAD_TAG_QUALITY", u"Calidad OPC-UA degradada (Bad Quality). Maniobra bloqueada.")
    if bool(is_estop):
        return (False, "INTERLOCK_ESTOP", u"Seta de emergencia activa en máquina. Maniobra bloqueada.")
    if not bool(is_remote):
        return (False, "INTERLOCK_LOCAL_MODE", u"Equipo en modo LOCAL. Pase a REMOTO para operar desde SCADA.")
    return (True, "PERMISSIVE_OK", u"Permisivos OK.")


# -----------------------------------------------------------------------------
# 2. INTEGRACIÓN CON LOS 7 TAGS ESENCIALES DE CAMPO (Batch I/O sin Bucles)
# -----------------------------------------------------------------------------
def read_machine_live_telemetry(equip="EDAR_BOMBA_01", tag_reader_fn=None):
    """Lee en 1 sola llamada readBlocking los 7 tags esenciales del equipo."""
    reader = tag_reader_fn if tag_reader_fn is not None else system.tag.readBlocking
    base = "{}/{}".format(TAG_ROOT, equip)
    paths = [
        base + "/ModoRemoto",
        base + "/SetaEmergencia",
        base + "/LoteActivoID",
        base + "/UnidadesObjetivo",
        base + "/UnidadesBuenas",
        base + "/UnidadesScrap",
        base + "/TiempoOperativoMin"
    ]
    qvs = reader(paths)
    all_good = bool(qvs and len(qvs) == len(paths) and all(q.quality.isGood() for q in qvs))
    if not all_good:
        return {"equip": equip, "quality_ok": False, "can_operate": False, "error": u"Calidad Bad en tags PLC"}

    is_remote, is_estop = bool(qvs[0].value), bool(qvs[1].value)
    batch_id = str(qvs[2].value or "")
    target, good, scrap = int(qvs[3].value or 0), int(qvs[4].value or 0), int(qvs[5].value or 0)
    op_min = float(qvs[6].value or 0.0)

    can_op, code, msg = evaluate_plc_interlocks_pure(is_remote, is_estop, all_qualities_good=True)
    return {
        "equip": equip,
        "quality_ok": True,
        "can_operate": can_op,
        "interlock_code": code,
        "interlock_msg": msg,
        "batch_id": batch_id,
        "target": target,
        "good": good,
        "scrap": scrap,
        "operating_min": round(op_min, 2),
        "kpis": calculate_batch_kpis(target, good, scrap, op_min)
    }


def _execute_plc_closure_handshake(equip, closed_batch_id):
    """Abre el siguiente lote en BD y actualiza los tags del PLC en 1 writeBlocking."""
    base = "{}/{}".format(TAG_ROOT, equip)
    target_qv = system.tag.readBlocking([base + "/UnidadesObjetivo"], 2000)
    target_units = int(target_qv[0].value or 5000) if target_qv and target_qv[0].quality.isGood() else 5000

    if str(closed_batch_id).endswith("-NEXT"):
        base_id = str(closed_batch_id)[:-5]
        next_batch_id = "{}-{}".format(base_id, system.date.format(system.date.now(), "HHmmss"))
    else:
        next_batch_id = "{}-NEXT".format(closed_batch_id)

    # Registramos el nuevo lote como IN_PROGRESS en PostgreSQL para permitir cierres sucesivos
    system.db.runPrepUpdate(
        """INSERT INTO production_batches
               (batch_id, machine_id, shift_id, target_units, good_units, scrap_units, operating_time_min, status, created_at)
           VALUES (?, ?, 1, ?, 0, 0, 0.0, 'IN_PROGRESS', CURRENT_TIMESTAMP)
           ON CONFLICT (batch_id) DO UPDATE SET
               machine_id = EXCLUDED.machine_id,
               good_units = 0,
               scrap_units = 0,
               operating_time_min = 0.0,
               status = 'IN_PROGRESS',
               closed_at = NULL,
               closed_by = NULL""",
        [next_batch_id, str(equip), target_units],
        database=DB_CONN
    )

    paths = [
        base + "/OrdenCierrePLC",
        base + "/LoteActivoID",
        base + "/UnidadesBuenas",
        base + "/UnidadesScrap",
        base + "/TiempoOperativoMin"
    ]
    vals = [True, next_batch_id, 0, 0, 0.0]
    qcs = system.tag.writeBlocking(paths, vals, 2000)
    return all(qc.isGood() for qc in qcs)


# -----------------------------------------------------------------------------
# 3. SERVICIO DE PLANTA BASADO EN PLANTILLA (PlantProductionService)
# -----------------------------------------------------------------------------
class PlantProductionService(BaseSCADAService):
    """Servicio transaccional heredado de plant.service_template.BaseSCADAService."""

    def __init__(self, db_connection=DB_CONN):
        super(PlantProductionService, self).__init__("Plant.ProductionService", db_connection)

    def close_batch(self, batch_id, equip, good_units, scrap_units, username,
                    user_roles=None, rbac_mode="A", security_levels=None, security_zones=None,
                    check_plc_interlocks=True):
        clean_user = str(username).strip() if username else "UNKNOWN"
        clean_batch = str(batch_id).strip() if batch_id else ""

        # 1. Seguridad RBAC (Enfoque A: Roles en Jython | Enfoque B: Security Levels + Zones)
        if str(rbac_mode).upper() == "B":
            eff_levels = security_levels if security_levels is not None else [
                "Authenticated/Roles/{}".format(r) for r in (user_roles or [])
            ]
            eff_zones = security_zones if security_zones is not None else ["Zona_Planta_Control"]
            authorized = self.check_rbac_security_levels(
                eff_levels, ALLOWED_SECURITY_LEVELS, eff_zones, ALLOWED_SECURITY_ZONES
            )
            sec_tag = "RBAC-B/Zones"
        else:
            authorized = self.check_rbac_roles(user_roles, ALLOWED_ROLES)
            sec_tag = "RBAC-A/Roles"

        if not authorized:
            self.logger.warn("[{}] Denegado cierre de '{}' a '{}'".format(sec_tag, clean_batch, clean_user))
            try:
                self.record_audit(
                    equip,
                    u"SEGURIDAD [{}]: Intento denegado de cerrar lote {}".format(sec_tag, unicode(clean_batch)),
                    clean_user
                )
            except Exception:
                pass
            return (False, u"Acceso Denegado ({}): '{}' no autorizado.".format(sec_tag, unicode(clean_user)))

        # 2. Validacion de parametros y enclavamientos PLC
        if not clean_batch or int(good_units) < 0 or int(scrap_units) < 0:
            return (False, u"Error de validación: datos de lote inválidos.")

        if check_plc_interlocks:
            snap = read_machine_live_telemetry(equip)
            if snap.get("quality_ok") and not snap.get("can_operate"):
                return (False, snap["interlock_msg"])

        # 3. Transaccion ACID (Named Query CloseBatchWithAudit + Auditoria en la misma tx_id)
        def _acid_unit_of_work(tx_id):
            proj = system.util.getProjectName()
            rows = system.db.runNamedQuery(
                project=proj,
                path="Production/CloseBatchWithAudit",
                parameters={
                    "batchId": clean_batch,
                    "goodUnits": int(good_units),
                    "scrapUnits": int(scrap_units),
                    "user": clean_user
                },
                tx=tx_id
            )
            if not rows or int(rows) == 0:
                return {"closed": False, "msg": u"Aviso: El lote '{}' ya estaba cerrado o no existe.".format(unicode(clean_batch))}

            self.record_audit(
                equip,
                u"CIERRE LOTE [{}]: {} (Buenas: {}, Scrap: {})".format(sec_tag, unicode(clean_batch), int(good_units), int(scrap_units)),
                clean_user,
                tx=tx_id
            )
            return {"closed": True, "msg": u"Lote '{}' cerrado y auditado correctamente.".format(unicode(clean_batch))}

        ok_tx, res = self.dispatch_acid_transaction("CLOSE_BATCH", equip, clean_user, _acid_unit_of_work)
        if not ok_tx:
            return (False, res.get("user_message", u"Fallo transaccional."))
        if not res.get("closed"):
            return (False, res.get("msg"))

        # 4. Handshake con el PLC y apertura del siguiente lote tras confirmar el commit en BD
        try:
            _execute_plc_closure_handshake(equip, clean_batch)
        except Exception as e:
            self.logger.warn("Error en handshake PLC post-cierre para '{}': {}".format(equip, str(e)))
        return (True, res.get("msg"))


_plant_service = PlantProductionService()


def finalize_batch_service(batch_id, equip, good_units, scrap_units, username, user_roles,
                           rbac_mode="A", security_levels=None, security_zones=None,
                           check_plc_interlocks=True):
    """Punto de entrada publico para Perspective y la suite QA unittest."""
    return _plant_service.close_batch(
        batch_id=batch_id, equip=equip, good_units=good_units, scrap_units=scrap_units,
        username=username, user_roles=user_roles, rbac_mode=rbac_mode,
        security_levels=security_levels, security_zones=security_zones,
        check_plc_interlocks=check_plc_interlocks
    )


def get_recent_audit_logs(equip=None, limit=10):
    """Consulta las ultimas trazas de ins_Registre_Actuacions para la tabla de Perspective."""
    lim = int(limit) if limit else 10
    if equip and equip != "TODOS":
        sql = "SELECT id, Equip, Descripcio, Usuari, DataHora FROM ins_Registre_Actuacions WHERE Equip = ? ORDER BY DataHora DESC LIMIT {:d}".format(lim)
        pyds = system.db.runPrepQuery(sql, [str(equip)], database=DB_CONN)
    else:
        sql = "SELECT id, Equip, Descripcio, Usuari, DataHora FROM ins_Registre_Actuacions ORDER BY DataHora DESC LIMIT {:d}".format(lim)
        pyds = system.db.runPrepQuery(sql, [], database=DB_CONN)

    rows = []
    for r in pyds:
        desc = unicode(r[2] or "")
        rows.append({
            "id": r[0],
            "Equip": r[1],
            "Tipo": "ALERTA RBAC" if "SEGURIDAD" in desc else "OPERACION",
            "Descripcio": desc,
            "Usuari": r[3],
            "DataHora": system.date.format(r[4], "dd/MM/yyyy HH:mm:ss") if r[4] else "---"
        })
    return rows
```

Guardar el proyecto en el Designer (`Ctrl+S`).

---

### Paso 5: Capa 4 — Interfaz Ágil en Perspective (5 Tags Esenciales + JSON Opcional + Message Handlers)

Para que la construcción visual sea rápida y se centre al 100% en la ingeniería de scripting SCADA, hemos reducido las propiedades de telemetría a **solo 5 tags esenciales** y dispones del **JSON base de la vista** para crear todos los componentes y propiedades de un solo clic.

#### 5.1. Creación Rápida de la Vista (`Paste JSON` en 10 segundos o Manual con 5 Propiedades)

1. En **Perspective -> Views**, crear una nueva vista `Laboratorios/Lab11_OEE_Dashboard` (`Flex Container`).
2. **Opción Rápida Recomendada (`Paste JSON`):**
   * Copia el bloque JSON inferior al portapapeles.
   * En el **Project Browser**, mantén pulsada la tecla **`Shift` + Clic Derecho** sobre la vista `Lab11_OEE_Dashboard` y selecciona **`Paste JSON`**.
   * Esto deja creadas automáticamente las 4 propiedades de `view.custom`, los **5 Indirect Tag Bindings** de `root.custom` y los componentes visuales (`lbl_BannerNotificacion`, `barra_Controles`, `lbl_OEE`, `tbl_ResumenTurno`, `tbl_Auditoria`), listo para pasar directamente al **apartado 5.2**.

<details>
<summary>Haz clic aquí para desplegar y copiar el JSON base de la vista <code>Lab11_OEE_Dashboard</code></summary>

```json
{
  "custom": {
    "lastNotification": "Sistema SCADA listo y conectado a PLC",
    "rbacMode": "A",
    "selectedEquip": "EDAR_BOMBA_01",
    "simulatedUser": "laura.vidal"
  },
  "params": {},
  "propConfig": {
    "custom.lastNotification": { "persistent": true },
    "custom.rbacMode": { "persistent": true },
    "custom.selectedEquip": { "persistent": true },
    "custom.simulatedUser": { "persistent": true }
  },
  "props": {
    "defaultSize": { "height": 740, "width": 1080 }
  },
  "root": {
    "children": [
      {
        "meta": { "name": "lbl_BannerNotificacion" },
        "propConfig": {
          "props.text": {
            "binding": {
              "config": { "path": "view.custom.lastNotification" },
              "type": "property"
            }
          }
        },
        "props": {
          "style": {
            "backgroundColor": "#2980b9",
            "borderRadius": "6px",
            "color": "#ffffff",
            "fontWeight": "bold",
            "padding": "10px 14px"
          }
        },
        "type": "ia.display.label",
        "version": 0
      },
      {
        "children": [
          {
            "meta": { "name": "drop_Equipo" },
            "propConfig": {
              "props.value": {
                "binding": {
                  "config": { "bidirectional": true, "path": "view.custom.selectedEquip" },
                  "type": "property"
                }
              }
            },
            "props": {
              "options": [
                { "label": "EDAR_BOMBA_01", "value": "EDAR_BOMBA_01" },
                { "label": "EDAR_BOMBA_02", "value": "EDAR_BOMBA_02" },
                { "label": "EDAR_COMPRESOR_01", "value": "EDAR_COMPRESOR_01" },
                { "label": "LINEA_ENVASADO_01", "value": "LINEA_ENVASADO_01" }
              ],
              "style": { "flexBasis": "200px" }
            },
            "type": "ia.input.dropdown",
            "version": 0
          },
          {
            "meta": { "name": "drop_UsuarioPrueba" },
            "propConfig": {
              "props.value": {
                "binding": {
                  "config": { "bidirectional": true, "path": "view.custom.simulatedUser" },
                  "type": "property"
                }
              }
            },
            "props": {
              "options": [
                { "label": "visita_auditor (Viewer)", "value": "visita_auditor" },
                { "label": "operador_noche (Operator)", "value": "operador_noche" },
                { "label": "laura.vidal (Supervisor)", "value": "laura.vidal" },
                { "label": "marti.sanchez (Admin)", "value": "marti.sanchez" }
              ],
              "style": { "flexBasis": "220px" }
            },
            "type": "ia.input.dropdown",
            "version": 0
          },
          {
            "meta": { "name": "drop_ModoRBAC" },
            "propConfig": {
              "props.value": {
                "binding": {
                  "config": { "bidirectional": true, "path": "view.custom.rbacMode" },
                  "type": "property"
                }
              }
            },
            "props": {
              "options": [
                { "label": "RBAC A (Roles Jython)", "value": "A" },
                { "label": "RBAC B (Zones + Levels)", "value": "B" }
              ],
              "style": { "flexBasis": "190px" }
            },
            "type": "ia.input.dropdown",
            "version": 0
          },
          {
            "meta": { "name": "btn_SimularPulsoPLC" },
            "props": {
              "style": { "backgroundColor": "#34495e", "color": "#ffffff", "fontWeight": "bold" },
              "text": "SIMULAR PULSO PLC"
            },
            "type": "ia.input.button",
            "version": 0
          },
          {
            "meta": { "name": "btn_CerrarLote" },
            "props": {
              "style": { "backgroundColor": "#27ae60", "color": "#ffffff", "fontWeight": "bold" },
              "text": "CERRAR Y AUDITAR LOTE"
            },
            "type": "ia.input.button",
            "version": 0
          }
        ],
        "meta": { "name": "barra_Controles" },
        "props": {
          "direction": "row",
          "style": { "alignItems": "center", "gap": "10px" },
          "wrap": "wrap"
        },
        "type": "ia.container.flex",
        "version": 0
      },
      {
        "meta": { "name": "lbl_OEE" },
        "props": {
          "style": {
            "backgroundColor": "#ecf0f1",
            "borderRadius": "6px",
            "fontSize": "18px",
            "fontWeight": "bold",
            "padding": "12px"
          },
          "text": "Configurar Binding OEE en Paso 5.2"
        },
        "type": "ia.display.label",
        "version": 0
      },
      {
        "meta": { "name": "tbl_ResumenTurno" },
        "props": { "style": { "flexBasis": "210px" } },
        "type": "ia.display.table",
        "version": 0
      },
      {
        "meta": { "name": "tbl_Auditoria" },
        "props": { "style": { "flexBasis": "240px" } },
        "type": "ia.display.table",
        "version": 0
      }
    ],
    "custom": {
      "batchId": "LOT-2026-FINAL-01",
      "good": 4850,
      "operating_min": 420.0,
      "scrap": 65,
      "target": 5000
    },
    "meta": { "name": "root" },
    "propConfig": {
      "custom.batchId": {
        "binding": {
          "config": {
            "bidirectional": false,
            "fallbackDelay": 2.5,
            "mode": "indirect",
            "references": { "1": "{view.custom.selectedEquip}" },
            "tagPath": "[default]PLANTA_PRODUCCION/{1}/LoteActivoID"
          },
          "type": "tag"
        }
      },
      "custom.good": {
        "binding": {
          "config": {
            "bidirectional": false,
            "fallbackDelay": 2.5,
            "mode": "indirect",
            "references": { "1": "{view.custom.selectedEquip}" },
            "tagPath": "[default]PLANTA_PRODUCCION/{1}/UnidadesBuenas"
          },
          "type": "tag"
        }
      },
      "custom.operating_min": {
        "binding": {
          "config": {
            "bidirectional": false,
            "fallbackDelay": 2.5,
            "mode": "indirect",
            "references": { "1": "{view.custom.selectedEquip}" },
            "tagPath": "[default]PLANTA_PRODUCCION/{1}/TiempoOperativoMin"
          },
          "type": "tag"
        }
      },
      "custom.scrap": {
        "binding": {
          "config": {
            "bidirectional": false,
            "fallbackDelay": 2.5,
            "mode": "indirect",
            "references": { "1": "{view.custom.selectedEquip}" },
            "tagPath": "[default]PLANTA_PRODUCCION/{1}/UnidadesScrap"
          },
          "type": "tag"
        }
      },
      "custom.target": {
        "binding": {
          "config": {
            "bidirectional": false,
            "fallbackDelay": 2.5,
            "mode": "indirect",
            "references": { "1": "{view.custom.selectedEquip}" },
            "tagPath": "[default]PLANTA_PRODUCCION/{1}/UnidadesObjetivo"
          },
          "type": "tag"
        }
      }
    },
    "props": {
      "direction": "column",
      "style": { "gap": "12px", "padding": "16px" }
    },
    "type": "ia.container.flex",
    "version": 0
  }
}
```

</details>

3. **Referencia si prefieres crear las propiedades a mano:**
   * En `view.custom`: `selectedEquip` (`"EDAR_BOMBA_01"`), `rbacMode` (`"A"`), `simulatedUser` (`"laura.vidal"`), `lastNotification` (`"Sistema SCADA listo"`).
   * En `root.custom` (**solo 5 Indirect Tag Bindings** con `{1}` = `view.custom.selectedEquip`):

| Propiedad en `root.custom` | Ruta de Indirect Tag Binding |
| :--- | :--- |
| `root.custom.batchId` | `[default]PLANTA_PRODUCCION/{1}/LoteActivoID` |
| `root.custom.target` | `[default]PLANTA_PRODUCCION/{1}/UnidadesObjetivo` |
| `root.custom.good` | `[default]PLANTA_PRODUCCION/{1}/UnidadesBuenas` |
| `root.custom.scrap` | `[default]PLANTA_PRODUCCION/{1}/UnidadesScrap` |
| `root.custom.operating_min` | `[default]PLANTA_PRODUCCION/{1}/TiempoOperativoMin` |

---

#### 5.2. Cálculo Reactivo de KPIs OEE en `lbl_OEE` (Script Transform Ligero)

Seleccionar el componente `lbl_OEE` y configurar un **Binding** sobre `props.text`:
1. Tipo de Binding: **Expression Structure**, vinculando desde el selector de propiedades las 5 propiedades de `root.custom`:
   * `batchId` $\rightarrow$ `root.custom.batchId` (`{parent.custom.batchId}`)
   * `target`: `root.custom.target`
   * `good`: `root.custom.good`
   * `scrap`: `root.custom.scrap`
   * `operating_min`: `root.custom.operating_min`
2. Añadir un **Script Transform** de 3 líneas que delega en la función pura de `plant.service`:

```python
kpis = plant.service.calculate_batch_kpis(
    value["target"], value["good"], value["scrap"], value["operating_min"]
)
return u"Lote: {} | Buenas: {} | Scrap: {} | Calidad: {:.1f}% | Rendimiento: {:.1f}% | OEE: {:.1f}%".format(
    value["batchId"], value["good"], value["scrap"], kpis["quality"], kpis["performance"], kpis["oee"]
)
```

---

#### 5.3. Scripts de los 2 Botones y Bindings de las 2 Tablas

1. **Botón `btn_SimularPulsoPLC` (`Events -> onActionPerformed -> Script`):**
   ```python
   snap = plant.simulator.step_simulation_tick(increment_counters=True)
   eq = self.view.custom.selectedEquip
   msg = u"Pulso PLC en {}: Buenas={} | Scrap={} | Min={}".format(
       eq, snap[eq]["good"], snap[eq]["scrap"], snap[eq]["op_min"]
   )
   system.perspective.sendMessage("notify_toast", payload={"msg": msg, "type": "info", "user": "PLC_SIM"}, scope="page")
   ```

2. **Botón `btn_CerrarLote` (`Events -> onActionPerformed -> Script`):**
   ```python
   root_c = self.view.getChild("root").custom
   equip = self.view.custom.selectedEquip
   batch_id = root_c.batchId
   good_qty = root_c.good
   scrap_qty = root_c.scrap
   rbac_mode = self.view.custom.rbacMode

   perfiles = {
       "visita_auditor": {"roles": ["PlantViewer"],               "levels": ["Authenticated/Roles/PlantViewer"]},
       "operador_noche": {"roles": ["Operator"],                  "levels": ["Authenticated/Roles/Operator"]},
       "laura.vidal":    {"roles": ["Operator", "Supervisor"],    "levels": ["Authenticated/Roles/Supervisor"]},
       "marti.sanchez":  {"roles": ["Administrator", "Supervisor"],"levels": ["Authenticated/Roles/Administrator", "Authenticated/Roles/Supervisor"]}
   }

   if self.session.props.auth.authenticated and self.session.props.auth.user.userName:
       user = self.session.props.auth.user.userName
       roles = list(self.session.props.auth.user.roles)
       levels = ["Authenticated/Roles/{}".format(r) for r in roles]
       zones = list(self.session.props.securityZones) if getattr(self.session.props, "securityZones", None) else ["Zona_Planta_Control"]
   else:
       user = self.view.custom.simulatedUser
       p = perfiles.get(user, perfiles["operador_noche"])
       roles, levels, zones = p["roles"], p["levels"], ["Zona_Planta_Control"]

   ok, msg = plant.service.finalize_batch_service(
       batch_id, equip, good_qty, scrap_qty, user, roles,
       rbac_mode=rbac_mode, security_levels=levels, security_zones=zones
   )

   system.perspective.sendMessage(
       "notify_toast",
       payload={"msg": msg, "type": "success" if ok else "error", "user": user},
       scope="page"
   )
   system.perspective.sendMessage(
       "OnBatchStatusChanged",
       payload={"batch_id": batch_id, "equip": equip, "success": ok},
       scope="page"
   )
   ```

3. **Bindings de las 2 Tablas Inferiores:**
   * **`tbl_ResumenTurno.props.data` (Query Binding -> Named Query `Production/GetShiftSummary`):**
     * `shiftId`: `1`
     * `startDate`: `addHours(now(0), -24)`
     * `endDate`: `addHours(now(0), 1)`
   * **`tbl_Auditoria.props.data` (Property Binding a `view.custom.selectedEquip` + Script Transform de 1 línea):**
     ```python
     return plant.service.get_recent_audit_logs(equip=value, limit=10)
     ```

---

#### 5.4. Configuración de los 2 Message Handlers Receptores en `root`

1. Hacer clic derecho sobre el contenedor **`root` -> Configure Scripts...**.
2. Bajo **Message Handlers**, añadir el **Receptor 1**:
   * **Message Type:** `notify_toast` | **Listen Scopes:** Marcar **`Page`** y **`Session`**.

```python
# MESSAGE HANDLER 1 en root: 'notify_toast'
now_str = system.date.format(system.date.now(), "HH:mm:ss")
msg_text = payload.get("msg", u"Notificación")
msg_type = payload.get("type", "info")
usr = payload.get("user", "SYSTEM")

self.view.custom.lastNotification = u"[{}] ({}) {}".format(now_str, usr, unicode(msg_text))

banner = self.getChild("lbl_BannerNotificacion")
if banner is not None:
    bg = {"success": "#27ae60", "error": "#c0392b", "info": "#2980b9"}.get(msg_type, "#2980b9")
    banner.props.style.backgroundColor = bg
    banner.props.style.color = "#ffffff"
```

3. Añadir el **Receptor 2** en el mismo **`root`**:
   * **Message Type:** `OnBatchStatusChanged` | **Listen Scopes:** Marcar **`Page`** y **`Session`**.

```python
# MESSAGE HANDLER 2 en root: 'OnBatchStatusChanged'
tbl_resumen = self.getChild("tbl_ResumenTurno")
if tbl_resumen is not None:
    tbl_resumen.refreshBinding("props.data")

tbl_audit = self.getChild("tbl_Auditoria")
if tbl_audit is not None:
    tbl_audit.refreshBinding("props.data")
```

4. Pulsar **OK** y guardar (`Ctrl+S`).

---

#### 5.5. Prueba Interactiva Rápida en Perspective (`F5`)

1. **Pulso PLC:** Pulsar **`SIMULAR PULSO PLC`** y comprobar que suben las unidades en `lbl_OEE` y el banner cambia en azul.
2. **Rechazo RBAC (Enfoque A y B):** Elegir `operador_noche`, probar con `RBAC A` y `RBAC B` pulsando **`CERRAR Y AUDITAR LOTE`** $\rightarrow$ Banner **ROJO** y nueva alerta `SEGURIDAD [RBAC-...]` en `tbl_Auditoria`.
3. **Enclavamiento PLC:** Poner `EDAR_BOMBA_01/SetaEmergencia = True` en el Tag Browser e intentar cerrar con `laura.vidal` $\rightarrow$ Bloqueo inmediato por seta activa. Devolver `SetaEmergencia` a `False`.
4. **Cierre Autorizado:** Seleccionar `EDAR_COMPRESOR_01` con `laura.vidal` y pulsar **`CERRAR Y AUDITAR LOTE`** $\rightarrow$ Banner **VERDE**, `OrdenCierrePLC = True`, contadores del PLC a `0` y refresco automático de ambas tablas.

---

### Paso 6: Capa 5 — Automatización Desatendida en Gateway (CRON Idempotente)

1. En **Server Scripting -> Gateway Event Scripts -> Scheduled**, crear la tarea `ConsolidacionFinTurno` con expresión CRON `0 0 6,14,22 * * ?`:

```python
"""
Gateway Scheduled Script: ConsolidacionFinTurno (CRON: 0 0 6,14,22 * * ?)
"""
logger = system.util.getLogger("SCADA.Gateway.ShiftClosure")

def execute_scheduled_shift_closure():
    db_conn = "DB_Admin"
    now = system.date.now()
    try:
        pending = system.db.runScalarPrepQuery(
            "SELECT COUNT(*) FROM production_batches WHERE status = 'IN_PROGRESS' AND created_at <= ?",
            [now], database=db_conn
        )
        if not pending or int(pending) == 0:
            logger.info("Idempotencia OK: 0 lotes abiertos. Sin escrituras.")
            return 0

        rows = system.db.runPrepUpdate(
            """UPDATE production_batches
               SET status = 'CLOSED', closed_at = CURRENT_TIMESTAMP, closed_by = 'GATEWAY_CRON_AUTO'
               WHERE status = 'IN_PROGRESS' AND created_at <= ?""",
            [now], database=db_conn
        )
        system.db.runPrepUpdate(
            """INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
               VALUES ('PLANTA_GLOBAL', ?, 'GATEWAY_CRON_AUTO', CURRENT_TIMESTAMP)""",
            [u"CONSOLIDACION FIN DE TURNO: {} lotes cerrados por CRON".format(rows)],
            database=db_conn
        )
        logger.info("Consolidacion completada: {} lotes cerrados.".format(rows))
        return rows
    except Exception as ex:
        logger.error("Fallo en cierre automatico de turno: {}".format(str(ex)))
        return -1

execute_scheduled_shift_closure()
```

---

### Paso 7: Capa 6 — Suite de Certificación QA con `unittest` (`qa.test_plant_unittest`)

#### 7.1. Creación del Módulo `qa.test_plant_unittest` en `Project Library`

En **Server Scripting -> Project Library**, crear el paquete `qa` y dentro el script `test_plant_unittest` (`qa.test_plant_unittest`):

```python
"""
Modulo: qa.test_plant_unittest
Descripcion: Suite de certificacion QA con la libreria estandar `unittest` de Jython.
"""
import unittest
import StringIO
import plant.service
import plant.simulator

reload(plant.service)
reload(plant.simulator)


def reset_test_batches_in_db():
    """Restaura los 4 lotes semilla y los tags de las 4 maquinas para pruebas 100% repetibles."""
    plant.simulator.reset_seed_counters_for_qa()
    batches_seed = [
        ("LOT-2026-FINAL-01", "EDAR_BOMBA_01",     5000,  4850,  65,  420.0),
        ("LOT-2026-FINAL-02", "LINEA_ENVASADO_01", 15000, 11200, 180, 280.0),
        ("LOT-2026-FINAL-03", "EDAR_COMPRESOR_01", 1200,   980,  15,  310.0),
        ("LOT-2026-FINAL-04", "EDAR_BOMBA_02",     4500,  4380,  95,  410.0)
    ]
    for b_id, m_id, target, good, scrap, op_min in batches_seed:
        system.db.runPrepUpdate(
            """INSERT INTO production_batches
                   (batch_id, machine_id, shift_id, target_units, good_units, scrap_units, operating_time_min, status, created_at)
               VALUES (?, ?, 1, ?, ?, ?, ?, 'IN_PROGRESS', CURRENT_TIMESTAMP)
               ON CONFLICT (batch_id) DO UPDATE SET
                   good_units = EXCLUDED.good_units,
                   scrap_units = EXCLUDED.scrap_units,
                   operating_time_min = EXCLUDED.operating_time_min,
                   status = 'IN_PROGRESS',
                   closed_at = NULL,
                   closed_by = NULL""",
            [b_id, m_id, target, good, scrap, op_min],
            database="DB_Admin"
        )


class MockQuality(object):
    def __init__(self, is_good=True):
        self._good = is_good
    def isGood(self):
        return self._good


class MockQualifiedValue(object):
    def __init__(self, value, is_good=True):
        self.value = value
        self.quality = MockQuality(is_good)


class TestProyectoFinalSCADA(unittest.TestCase):
    """5 pruebas integrales: OEE, Limites/Nulos, Enclavamientos/Mocks OPC y RBAC A/B + ACID."""

    @classmethod
    def setUpClass(cls):
        reset_test_batches_in_db()

    def test_01_calculo_oee_nominal(self):
        res = plant.service.calculate_batch_kpis(5000, 4850, 65, 420.0, 2.5)
        self.assertAlmostEqual(res["quality"], 98.68, places=2)
        self.assertAlmostEqual(res["performance"], 48.76, places=2)
        self.assertAlmostEqual(res["oee"], 48.12, places=2)

    def test_02_calculo_oee_casos_limite_y_nulos(self):
        self.assertEqual(plant.service.calculate_batch_kpis(5000, 0, 0, 0.0, 0.0)["oee"], 0.0)
        self.assertEqual(plant.service.calculate_batch_kpis(None, 4850, None, 420.0, 2.5)["oee"], 0.0)

    def test_03_enclavamientos_plc_y_mock_bad_quality(self):
        ok_nom, code_nom, _ = plant.service.evaluate_plc_interlocks_pure(True, False, True)
        self.assertTrue(ok_nom)
        self.assertEqual(code_nom, "PERMISSIVE_OK")

        ok_estop, code_estop, _ = plant.service.evaluate_plc_interlocks_pure(True, True, True)
        self.assertFalse(ok_estop)
        self.assertEqual(code_estop, "INTERLOCK_ESTOP")

        def mock_bad_reader(paths):
            return [MockQualifiedValue(None, is_good=False) for _ in paths]

        snap_bad = plant.service.read_machine_live_telemetry("EDAR_BOMBA_01", tag_reader_fn=mock_bad_reader)
        self.assertFalse(snap_bad["quality_ok"])

    def test_04_rbac_rechazo_usuarios_no_autorizados_a_y_b(self):
        ok_visita, _ = plant.service.finalize_batch_service(
            "LOT-2026-FINAL-01", "EDAR_BOMBA_01", 4850, 65,
            username="visita_auditor", user_roles=["PlantViewer"], rbac_mode="A"
        )
        self.assertFalse(ok_visita)

        ok_oper, _ = plant.service.finalize_batch_service(
            "LOT-2026-FINAL-02", "LINEA_ENVASADO_01", 11200, 180,
            username="operador_noche", user_roles=["Operator"], rbac_mode="B",
            security_levels=["Authenticated/Roles/Operator"],
            security_zones=["Zona_Planta_Control"]
        )
        self.assertFalse(ok_oper)

        ok_bad_zone, _ = plant.service.finalize_batch_service(
            "LOT-2026-FINAL-02", "LINEA_ENVASADO_01", 11200, 180,
            username="laura.vidal", user_roles=["Supervisor"], rbac_mode="B",
            security_levels=["Authenticated/Roles/Supervisor"],
            security_zones=["Zona_WiFi_Externa"]
        )
        self.assertFalse(ok_bad_zone)

    def test_05_cierre_acido_idempotencia_y_handshake_plc_a_y_b(self):
        # 1. Cierre autorizado RBAC Modo A + verificacion de Handshake PLC e Idempotencia
        ok_a, msg_a = plant.service.finalize_batch_service(
            "LOT-2026-FINAL-01", "EDAR_BOMBA_01", 4850, 65,
            username="marti.sanchez", user_roles=["Supervisor", "Administrator"], rbac_mode="A"
        )
        self.assertTrue(ok_a, msg_a)

        qvs = system.tag.readBlocking([
            "[default]PLANTA_PRODUCCION/EDAR_BOMBA_01/OrdenCierrePLC",
            "[default]PLANTA_PRODUCCION/EDAR_BOMBA_01/UnidadesBuenas",
            "[default]PLANTA_PRODUCCION/EDAR_BOMBA_01/LoteActivoID"
        ])
        self.assertTrue(bool(qvs[0].value))
        self.assertEqual(int(qvs[1].value), 0)
        self.assertEqual(str(qvs[2].value), "LOT-2026-FINAL-01-NEXT")

        ok_idem_a, _ = plant.service.finalize_batch_service(
            "LOT-2026-FINAL-01", "EDAR_BOMBA_01", 4850, 65,
            username="marti.sanchez", user_roles=["Supervisor"], rbac_mode="A"
        )
        self.assertFalse(ok_idem_a)

        # 2. Cierre autorizado RBAC Modo B + verificacion de Idempotencia
        ok_b, msg_b = plant.service.finalize_batch_service(
            "LOT-2026-FINAL-02", "LINEA_ENVASADO_01", 11200, 180,
            username="laura.vidal", user_roles=["Operator", "Supervisor"], rbac_mode="B",
            security_levels=["Authenticated/Roles/Supervisor"],
            security_zones=["Zona_Planta_Control"]
        )
        self.assertTrue(ok_b, msg_b)

        ok_idem_b, _ = plant.service.finalize_batch_service(
            "LOT-2026-FINAL-02", "LINEA_ENVASADO_01", 11200, 180,
            username="laura.vidal", user_roles=["Supervisor"], rbac_mode="B"
        )
        self.assertFalse(ok_idem_b)


def run_unittest_suite():
    suite = unittest.TestLoader().loadTestsFromTestCase(TestProyectoFinalSCADA)
    buf = StringIO.StringIO()
    result = unittest.TextTestRunner(stream=buf, verbosity=2).run(suite)
    reset_test_batches_in_db()
    return result.wasSuccessful(), buf.getvalue(), result.testsRun, len(result.failures) + len(result.errors)
```

#### 7.2. Ejecución desde la `Script Console`

Abrir **Tools -> Script Console** y lanzar en 5 líneas la certificación completa:

```python
import qa.test_plant_unittest
reload(qa.test_plant_unittest)

ok, report, total, errs = qa.test_plant_unittest.run_unittest_suite()
print report
print "Resumen QA unittest -> Tests: {} | Errores: {} | Estado: {}".format(
    total, errs, "CERTIFICADO 100% OK" if ok else "REVISAR FALLOS"
)
```

#### 7.3. Salida Esperada en la Consola

```
test_01_calculo_oee_nominal (qa.test_plant_unittest.TestProyectoFinalSCADA) ... ok
test_02_calculo_oee_casos_limite_y_nulos (qa.test_plant_unittest.TestProyectoFinalSCADA) ... ok
test_03_enclavamientos_plc_y_mock_bad_quality (qa.test_plant_unittest.TestProyectoFinalSCADA) ... ok
test_04_rbac_rechazo_usuarios_no_autorizados_a_y_b (qa.test_plant_unittest.TestProyectoFinalSCADA) ... ok
test_05_cierre_acido_idempotencia_y_handshake_plc_a_y_b (qa.test_plant_unittest.TestProyectoFinalSCADA) ... ok

----------------------------------------------------------------------
Ran 5 tests in 0.045s

OK

Resumen QA unittest -> Tests: 5 | Errores: 0 | Estado: CERTIFICADO 100% OK
```

***

### Laboratorio 11.2: Presentación Técnica de Proyectos y Code Review en Vivo

#### 1. Objetivos

* Exponer la arquitectura implementada por cada alumno ante el grupo, justificando la comparativa entre **RBAC Enfoque A (Roles Jython)** y **RBAC Enfoque B (Security Zones + Security Levels)** sobre `PlantProductionService`.
* Realizar una revisión de código por pares (*Peer Code Review*) en vivo.

#### 2. Procedimiento de la Sesión de Code Review

1. **Demostración Funcional en Vivo (4 min):**
   * Simulación de pulso PLC en Perspective y recálculo instantáneo de OEE al cambiar de máquina.
   * Prueba de rechazo con `operador_noche` / `visita_auditor` (toast rojo en `notify_toast` + fila de alerta en `tbl_Auditoria` vía `OnBatchStatusChanged`) y cierre exitoso con `laura.vidal` / `marti.sanchez`.
   * Ejecución en 1 clic de `qa.test_plant_unittest.run_unittest_suite()` con los 5 tests en `OK`.
2. **Rúbrica de Auditoría de Código (5 min):**
   * **Seguridad y RBAC (25 pts):** 0 SQL concatenado (`:param` y `?`) y validación A/B auditada en `ins_Registre_Actuacions`.
   * **Rendimiento y Tags (25 pts):** Lectura/escritura por lote (`readBlocking`/`writeBlocking` sin bucles) e idempotencia en el CRON del Gateway.
   * **Robustez y UI (25 pts):** Transacción ACID en `BaseSCADAService` y Message Handlers receptores reales (`notify_toast`, `OnBatchStatusChanged`).
   * **Arquitectura y QA (25 pts):** Separación de funciones puras respecto a `PlantProductionService` (`plant.service_template`) y suite `unittest` reproducible.
