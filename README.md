---
description: Setup y Validación del Entorno de Trabajo LOC
---

# Setup inicial alumnos - LOCAL

## Guía del Alumno — Sesión 0: Setup y Validación del Entorno de Trabajo

* **Curso:** SQL y Python/Jython para Scripting en Ignition (SCADA)
* **Objetivo de la sesión:** Configurar, replicar y validar el entorno de laboratorio (tanto en local mediante contenedores como en el servidor Cloud de prácticas) con la base de datos PostgreSQL poblada con los esquemas reales del proyecto.

***

### 1. Herramientas Necesarias

Antes de comenzar, asegúrate de tener instalado en tu equipo:

1. **Docker Desktop** (con Docker Compose v2) habilitado y en ejecución (en caso de trabajar en local). [https://docs.docker.com/desktop/setup/install/windows-install/](https://docs.docker.com/desktop/setup/install/windows-install/)\
   ![](.gitbook/assets/image.png)<a href="https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe?utm_source=docker&#x26;utm_medium=webreferral&#x26;utm_campaign=docs-driven-download-win-amd64&#x26;_gl=1*o16p7h*_gcl_au*MTM0MDM3MDU3My4xNzg3MjM2Mjcx*_ga*Nzg5MTk5MzU3LjE3ODcyMzYyNzE.*_ga_XJWPQMJYHQ*czE3ODcyMzYyNzAkbzEkZzEkdDE3ODcyMzYyOTUkajM1JGwwJGgw" class="button secondary">Docker Desktop for Windows - x86_64</a>\
   ejecutamos el exe\
   ![](<.gitbook/assets/image (1).png>)\
   per user\
   esperamos a que instale
2. **Cliente de Base de Datos** (opcional pero recomendado: DBeaver, pgAdmin o extensión de VS Code) para inspeccionar PostgreSQL.\
   ![](<.gitbook/assets/image (3).png>)\
   descargar la versión zip\
   extraer todo\
   ![](<.gitbook/assets/image (4).png>)\
   el dbeaver.exe ya nos ejectua el cliente\
   seguimos la configuración inicial a nuestro gusto\
   ![](<.gitbook/assets/image (5).png>)

***

### 2. Opción A: Despliegue del Entorno Completo en Local (Docker)

Esta opción te permite tener una réplica exacta del servidor de formación en tu propia máquina para desarrollar y practicar fuera de las sesiones online.

```
                      ARQUITECTURA DEL ENTORNO LOCAL
                      
 ╔═══════════════════════════════════════════════════════════════════════════╗
 ║                         TU ORDENADOR (HOST LOCAL)                         ║
 ║                                                                           ║
 ║  [ Ignition Designer Launcher ]                                           ║
 ║             │                                                             ║
 ║             ▼ (http://localhost:8088)                                     ║
 ║  ╔═════════════════════════════════════════════════════════════════════╗  ║
 ║  ║                   DOCKER ENGINE / DOCKER COMPOSE                    ║  ║
 ║  ║                                                                     ║  ║
 ║  ║  ┌──────────────────────────────┐  ┌─────────────────────────────┐  ║  ║
 ║  ║  │      ignition-gateway        │  │         postgres-db         │  ║  ║
 ║  ║  │  (Ignition 8.1.33 / Jython)  │  │  (PostgreSQL 15 - Alpine)   │  ║  ║
 ║  ║  │  Puerto: 8088 / 8043         │  │  Puerto: 5432               │  ║  ║
 ║  ║  └──────────────┬───────────────┘  └──────────────┬──────────────┘  ║  ║
 ║  ║                 │                                 │                 ║  ║
 ║  ║                 └──── JDBC: SandboxDB ────────────┘                 ║  ║
 ║  ╚═════════════════════════════════════════════════════════════════════╝  ║
 ╚═══════════════════════════════════════════════════════════════════════════╝
```

***

#### Paso A.1: Crear el archivo `docker-compose.yml`

Crea una carpeta en tu ordenador llamada `ignition-local-lab` y dentro de ella un archivo denominado `docker-compose.yml` con el siguiente contenido:\
\
![](<.gitbook/assets/image (18).png>)

\
\
<img src=".gitbook/assets/image (19).png" alt="" data-size="original">\
\
\
![](<.gitbook/assets/image (20).png>)\
\
\
\
\
\
\
![](<.gitbook/assets/image (21).png>)

instalar drives

test connection

<figure><img src=".gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>





```yaml
version: '3.8'

services:
  ignition:
    image: inductiveautomation/ignition:8.1.33
    container_name: ignition-maker-gateway
    ports:
      - "8088:8088"
      - "8043:8043"
    environment:
      - ACCEPT_IGNITION_EULA=Y
      - IGNITION_EDITION=maker
      - GATEWAY_ADMIN_USERNAME=admin
      - GATEWAY_ADMIN_PASSWORD=AdminMasterPass123!
      - GATEWAY_HTTP_PORT=8088
      - IGNITION_LICENSE_KEY=2K22-F3NR
      - IGNITION_ACTIVATION_TOKEN=eyJrdHkiOiJSU0EiLCJraWQiOiI4NWIwYmI0Yy1hYTNlLTQzNzEtODI3OC0wMDNiOTRkOTNhNzgiLCJ1c2UiOiJzaWciLCJhbGciOiJSUzI1NiIsIm4iOiJ0S3hXdzU3Y1p6cGVBdWFnTFFJRnBFaHJwd1RfLXJVSmF5b3NHbm5ZOW1uZTRzV1pZWFVaOEx2cGhLc3FXTEJUeTktYlVyaUtXdzBweGtuZzIwYTV1cXZMbDFEVEtkRlpzWVZnRlp6U1dZdjZ1Z0NidWV3ZzBHZXBRYlNsdlh0LXE3QjdtYkNwSVdOODhmdllQc3haRHlQcTMwTFpEU1NPdkxibi1hbDI4SHdCZjRGTjltam0xSVY5OV9pYlppS05sZTBkQ05HSFBXTXZJcEltQVVNbXo5eTRYdEdFZDRsdHpiQmtzOWhHLWo1YnloX21NOWdabE5oS2p1cXBQSGEyZFdWWEFBMmV6WFk0bmFyaVgwb0N6cHNKdmFPLVhhNUZieUZSeUw0X3NnNHdxSllYR2trNDlPWTJobUMwVDMwb2JNcVhPRGo0U0dJalRZc29lZ2NTWnciLCJlIjoiQVFBQiIsImQiOiJRVXN4RmdSQTZldDlrelhPdUVWTlRhaEFnMzA2bDF1NnVpaGxCUEo1RnVCX0JOeGRpSXQ5NHZLLU8weHlndGZDaU9EWUxNWjFkZDlsY193NkQzVFpaVjExdDhzbGJTeno2Y1JtUzVrX3FjMzlRQ0E5SnkxdlZISEtDTzFrb254UmRISFUwVi0zVjlJeEV2Q3RuMUJSZFRmSlI0Tk1sc1BaODloME1GVnVud3pPT0d2Zi1abkpNN0lZdjhsUzlkcWdUclpmZDQzR29ETVYtUDN6NXFYcWZUeUJYcWJBeFhzSTlSUXk2Q2JKMHNRSzFRaXl3b2NiNzFsdlRqUG0xaTRFU0pHN2lkcFE1ZEllSkhKbUhGclQ4elBuM2djVUlWcXJicDZETFdCMnV0Y1k0OXhJTlczNy1ISDgzMjZLQk5aVVhtSDA1aGVBdTJTbGcweVVFbzZsZ1EiLCJwIjoiMktHUF81OF80MkdFemtvZktMYTdyMWVIbWdYWk11ZFBYdTZfQTFCcy1mT2hiTkxlV0RHcW5SakFVU1JiTHItam1wNDlkVFY5al83d2VkcTVOVmVCYlc0VzktZGk1TGxYNTg3c0d4SS02cGNubzVpM0lXOHI2T2JsMzlsOWI1bEVRbDBMNXMyLTNHS2xkaV9uYS1paDBiREJqWHhZM1BtRTRBUkktcENHZnVjIiwicSI6IjFZSGppQWhRR3dCdl9BOTVLcFA1UXB3MWpTYW9iZDNsMzh4eld0aTVFTlpCSXRiV1NDOGI5Z2laN2tiLUFXdl9INlppbi1sMDB5SmJIYkZ6YnpFUjBMeXBJd2Y2MFNva0hzLWRhVWljeHZINm95MDFzenNhTERtR0FDc1AyUnBHNEtrSW41UUliTnJfRHNMM3RUM3MtSmZsNnRZbkRxaFVaME9CM0VzeDRJRSIsImRwIjoiRXpBMFhoTVFDS2NCcVhnZFRIRHJMUHZXMGdqRWxXS3h3Qm5ycDNKX1JLQ1U0dHZHd0E4ZUtxNGZrdEJpbDBCNFVHREYxdFQzRzBNY3I5NTAyMG0xLUNoeE5tSXplMGtEaFVfcHpfZ014S0RBN1JmQTJPQk5CbU0xWjEtUFljdzBwS0F6UnExZ3c0cWxWMU9rN3dUN0dHVE1zQ2ljZ201RG04Z2xZclJjaFc4IiwiZHEiOiJtR0pmT2Z6czU0aTFaSXJLcVJmNTVJX0hMTm8xaGt6RXY2bVZmM2FGQjc1VHVRRHE2WlF0LWJrRDNHdnc2S1RpN3Z6N0VUVTN1MldlOEo5eFN5QVRuZzY1RFJhcDdsV01lQzBvSlRlOUpjVVpaUk5rYTJxNGNHNFI5TmJITmVXcVJyaC1QaDhTc0ZiUmlnQ2ZlVTBjY0FWQ0JRMFp6VDFaR0dhM0xickJlNEUiLCJxaSI6ImlKcHVzVm5OX2toS0lub2VGNmZhSkxlSkJxUGlQSlRHdUhJOEdHa0hEelF3NzBLNWNyeVFkbl9kQV9uUW9Ud3pyVmhfUERLZU1RVVFuTXp2VnJTWjJlSWEtc21pWTJ2VnZDVEdDbGNWTmdhaEp0RjFhYzRyMTdUWjRmck93elRmZDJUWk9kZEo4elhSYzAtTWJjV2l4aXlSNmJkdWYxdGd3Z3p3R1VCb1VRZyJ9
    volumes:
      - ignition_maker_data:/usr/local/bin/ignition/data
    restart: unless-stopped
    depends_on:
      - postgres

  postgres:
    image: postgres:15-alpine
    container_name: postgres-maker-db
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=sandbox_db
      - POSTGRES_USER=ignition_user
      - POSTGRES_PASSWORD=DBPassword123!
    volumes:
      - postgres_maker_data:/var/lib/postgresql/data
      - ./init-db:/docker-entrypoint-initdb.d
    restart: unless-stopped

volumes:
  ignition_maker_data:
  postgres_maker_data:
  
  
  
```

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

inductiveautomation.com\
![](<.gitbook/assets/image (11).png>)\
\
![](<.gitbook/assets/image (13).png>)\
\
![](<.gitbook/assets/image (14).png>)\
\
![](<.gitbook/assets/image (15).png>)\
\
![](<.gitbook/assets/image (16).png>)\
\
![](<.gitbook/assets/image (24).png>)\
\
\
\
<br>

***

#### Paso A.2: Script de Inicialización de Base de Datos (`init.sql`)

![](<.gitbook/assets/image (23).png>)\
Dentro de la carpeta `ignition-local-lab`, crea una subcarpeta llamada `init-db` y dentro de ella un archivo llamado `init.sql`.

Este script genera las tablas requeridas para simular exactamente los casos de producción del proyecto (**Gestor Documental, Enclavamientos, Registro de Actuaciones, Histórico de Consignas, Notas y Limpieza de Operadores**):

```sql
-- =============================================================================
-- ESQUEMA Y POBLADO PROGRAMÁTICO MASIVO: SANDBOX_DB (POSTGRESQL 15)
-- Modelado para simulación de scripts del cliente:
--  1. Gestor Documental y Descargas Binarias
--  2. Histórico de Telemetría y Consignas (sqlt_data_1_{mes} / sqlth_te)
--  3. Matriz de Enclavamientos de 32 bits y Registro de Actuaciones
--  4. Sistema de Notas y Documentos Adjuntos
--  5. Depuración / Limpieza de Tablas Masivas (> 5 años de antigüedad)
-- =============================================================================

-- Extensiones útiles
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- -----------------------------------------------------------------------------
-- 1. ESTRUCTURA DE TABLAS
-- -----------------------------------------------------------------------------

-- 1.1 Gestor Documental (PDF Págs. 1 y 2)
CREATE TABLE IF NOT EXISTS documents_metadata (
    id SERIAL PRIMARY KEY,
    folder_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    filename VARCHAR(255) NOT NULL,
    extension VARCHAR(50) NOT NULL,
    size BIGINT NOT NULL,
    equip VARCHAR(100) NOT NULL,
    modified_by VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS documents_contents (
    document_id VARCHAR(255) PRIMARY KEY,
    contents BYTEA NOT NULL,
    uploaded_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 1.2 Enclavamientos y Registro de Actuaciones (PDF Págs. 10, 13, 14, 15)
CREATE TABLE IF NOT EXISTS enclavaments (
    id SERIAL PRIMARY KEY,
    StartDate TIMESTAMP NOT NULL,
    Equip VARCHAR(100) NOT NULL,
    NumeroEnclava INT NOT NULL,
    TextEnclava TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS ins_Registre_Actuacions (
    id SERIAL PRIMARY KEY,
    Equip VARCHAR(100) NOT NULL,
    Descripcio TEXT NOT NULL,
    Usuari VARCHAR(100) NOT NULL,
    DataHora TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 1.3 Gestión de Notas y Documentos Vinculados (PDF Pág. 12)
CREATE TABLE IF NOT EXISTS notes (
    id_nota SERIAL PRIMARY KEY,
    equip VARCHAR(100) NOT NULL,
    dateCreated TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    Readed INT NOT NULL DEFAULT 0,
    nota_texto TEXT
);

CREATE TABLE IF NOT EXISTS documents (
    id_documento SERIAL PRIMARY KEY,
    idNote INT REFERENCES notes(id_nota) ON DELETE CASCADE,
    nombre_adjunto VARCHAR(255) NOT NULL,
    tamanio_bytes INT NOT NULL,
    fecha_adjunto TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 1.4 Histórico de Auditoría y Operadores (PDF Págs. 15 y 16)
CREATE TABLE IF NOT EXISTS REG_OPERADORS (
    id SERIAL PRIMARY KEY,
    Data TIMESTAMP NOT NULL,
    Operador VARCHAR(100) NOT NULL,
    Accio VARCHAR(255) NOT NULL,
    Detall TEXT
);

CREATE INDEX IF NOT EXISTS idx_reg_operadors_data ON REG_OPERADORS(Data);

-- 1.5 Diccionario de Tags e Histórico SCADA (PDF Págs. 6, 7 y 11)
CREATE TABLE IF NOT EXISTS sqlth_te (
    id SERIAL PRIMARY KEY,
    tagpath VARCHAR(255) UNIQUE NOT NULL
);

-- Partición del mes actual (Octubre 2026)
CREATE TABLE IF NOT EXISTS sqlt_data_1_2026_10 (
    tagid INT REFERENCES sqlth_te(id),
    intvalue BIGINT,
    floatvalue DOUBLE PRECISION,
    stringvalue TEXT,
    datevalue TIMESTAMP,
    t_stamp BIGINT NOT NULL
);

-- Partición del mes anterior (Septiembre 2026)
CREATE TABLE IF NOT EXISTS sqlt_data_1_2026_09 (
    tagid INT REFERENCES sqlth_te(id),
    intvalue BIGINT,
    floatvalue DOUBLE PRECISION,
    stringvalue TEXT,
    datevalue TIMESTAMP,
    t_stamp BIGINT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_sqlt_2026_10_tag_stamp ON sqlt_data_1_2026_10(tagid, t_stamp);
CREATE INDEX IF NOT EXISTS idx_sqlt_2026_09_tag_stamp ON sqlt_data_1_2026_09(tagid, t_stamp);

-- -----------------------------------------------------------------------------
-- 2. GENERACIÓN PROGRAMÁTICA MASIVA DE DATOS (SEED DATA)
-- -----------------------------------------------------------------------------

DO $$
DECLARE
    -- Equipos de planta
    equips TEXT[] := ARRAY['EDAR_BOMBA_01', 'EDAR_BOMBA_02', 'EDAR_COMPRESOR_01', 'EDAR_COMPRESOR_02', 'DECANTADOR_01', 'VALVULA_RECIRC_01', 'LINEA_ENVASADO_01', 'LINEA_ENVASADO_02'];
    operadors TEXT[] := ARRAY['marti.sanchez', 'laura.vidal', 'carlos.gomez', 'admin', 'operador_noche', 'mantenimiento_01'];
    defects TEXT[] := ARRAY['Nivel aspiración bajo', 'Presión aceite insuficiente', 'Sobretemperatura devanado', 'Fallo confirmación marcha', 'Parada seta emergencia', 'Sobrecarga térmica relé', 'Fallo flujo caudalímetro', 'Desalineación eje'];
    
    i INT;
    eq TEXT;
    doc_name TEXT;
    doc_filename TEXT;
    note_id INT;
BEGIN
    RAISE NOTICE '=== INICIANDO GENERACIÓN PROGRAMÁTICA DE DATOS ===';

    -- -------------------------------------------------------------------------
    -- A. GENERAR HISTÓRICO DE AUDITORÍA REG_OPERADORS (6.000 Registros)
    -- Simula datos desde 2018 hasta 2026 para probar scripts de limpieza (> 5 años)
    -- -------------------------------------------------------------------------
    RAISE NOTICE 'Poblando tabla REG_OPERADORS (6.000 registros históricos)...';
    
    -- 1. Registros antiguos (> 5 años: 2018 - 2020) -> ~3.500 filas (deben ser purgadas por el script)
    INSERT INTO REG_OPERADORS (Data, Operador, Accio, Detall)
    SELECT 
        TIMESTAMP '2018-01-01 00:00:00' + (random() * (TIMESTAMP '2020-12-31 23:59:59' - TIMESTAMP '2018-01-01 00:00:00')),
        operadors[1 + floor(random() * array_length(operadors, 1))::int],
        (ARRAY['LOGIN', 'LOGOUT', 'MODIFICACIO_CONSIGNA', 'RECONEIXEMENT_ALARMA', 'ARRANQUE_EQUIP', 'PARADA_MANUAL'])[1 + floor(random() * 6)::int],
        'Registre d actuacio automatica de prova historica antiga ID: ' || g
    FROM generate_series(1, 3500) AS g;

    -- 2. Registros intermedios (2024 - 2025, incluyendo el día específico de prueba 2025-03-11) -> ~1.500 filas
    INSERT INTO REG_OPERADORS (Data, Operador, Accio, Detall)
    SELECT 
        TIMESTAMP '2024-01-01 00:00:00' + (random() * (TIMESTAMP '2025-12-31 23:59:59' - TIMESTAMP '2024-01-01 00:00:00')),
        operadors[1 + floor(random() * array_length(operadors, 1))::int],
        (ARRAY['CANVI_PANTALLA', 'EXPORTACIO_PDF', 'MODIFICACIO_CONSIGNA', 'VERIFICACIO_ENCLAVAMENT'])[1 + floor(random() * 4)::int],
        'Operacio intermitja registrada en sistema SCADA ID: ' || g
    FROM generate_series(3501, 4800) AS g;

    -- Registros exactos para el intervalo de prueba del PDF (2025-03-11)
    INSERT INTO REG_OPERADORS (Data, Operador, Accio, Detall)
    SELECT 
        TIMESTAMP '2025-03-11 00:00:00' + (random() * (INTERVAL '23 hours 59 minutes')),
        'operador_turno_2025',
        'PROVA_INTERVAL_DADES',
        'Registre especific per al test de neteja per rang de dates concret (PDF Pag 16)'
    FROM generate_series(1, 200);

    -- 3. Registros recientes (2026 en curso) -> ~1.000 filas
    INSERT INTO REG_OPERADORS (Data, Operador, Accio, Detall)
    SELECT 
        TIMESTAMP '2026-01-01 00:00:00' + (random() * (TIMESTAMP '2026-10-20 12:00:00' - TIMESTAMP '2026-01-01 00:00:00')),
        operadors[1 + floor(random() * array_length(operadors, 1))::int],
        (ARRAY['ENVIAR_CONSIGNA', 'ORDRE_MARXA', 'CANVI_ESTAT_LINEA', 'LOGIN_PERSPECTIVE'])[1 + floor(random() * 4)::int],
        'Actuacio operativa recent en planta'
    FROM generate_series(1, 1000);

    -- -------------------------------------------------------------------------
    -- B. GENERAR GESTOR DOCUMENTAL (80 Documentos con Payload Binario)
    -- -------------------------------------------------------------------------
    RAISE NOTICE 'Poblando Gestor Documental (80 documentos con contenido BYTEA)...';

    FOR i IN 1..80 LOOP
        eq := equips[1 + (i % array_length(equips, 1))];
        doc_name := 'Doc_Tecnico_' || eq || '_Rev' || (i % 5 + 1);
        doc_filename := doc_name || (CASE WHEN i % 2 = 0 THEN '.pdf' ELSE '.docx' END);

        -- Metadatos
        INSERT INTO documents_metadata (folder_id, name, filename, extension, size, equip, modified_by, created_at)
        VALUES (
            (i % 10) + 1,
            doc_name,
            doc_filename,
            CASE WHEN i % 2 = 0 THEN 'pdf' ELSE 'docx' END,
            floor(100000 + random() * 4000000)::bigint,
            eq,
            operadors[1 + (i % array_length(operadors, 1))],
            NOW() - (random() * INTERVAL '90 days')
        );

        -- Contenido binario simulado (Cabecera PDF '%PDF-1.4...' en formato BYTEA)
        INSERT INTO documents_contents (document_id, contents, uploaded_at)
        VALUES (
            doc_filename,
            decode('255044462d312e340a25d0d4c5d80a342030206f626a0a3c3c2f547970652f436174616c6f672f50616765732032203020523e3e0a656e646f626a', 'hex') || 
            decode(repeat(to_hex((i * 13) % 255), 128), 'hex'),
            NOW() - (random() * INTERVAL '90 days')
        );
    END LOOP;

    -- -------------------------------------------------------------------------
    -- C. GENERAR NOTAS Y DOCUMENTOS VINCULADOS (150 Notas)
    -- Simula notas leídas y pendientes para probar la consulta de Notas/select
    -- -------------------------------------------------------------------------
    RAISE NOTICE 'Poblando Sistema de Notas y Adjuntos (150 notas)...';

    FOR i IN 1..150 LOOP
        eq := equips[1 + (i % array_length(equips, 1))];
        
        INSERT INTO notes (equip, dateCreated, Readed, nota_texto)
        VALUES (
            eq,
            NOW() - (random() * INTERVAL '45 days'),
            CASE WHEN i % 3 = 0 THEN 0 ELSE 1 END, -- 33% de notas no leídas (Readed = 0)
            'Observacio de manteniment per a ' || eq || '. Verificacio de vibracions i consum electric en cicle ' || i
        ) RETURNING id_nota INTO note_id;

        -- Adjuntar documento en el 50% de las notas
        IF i % 2 = 0 THEN
            INSERT INTO documents (idNote, nombre_adjunto, tamanio_bytes, fecha_adjunto)
            VALUES (
                note_id,
                'adjunt_nota_' || note_id || '_' || eq || '.pdf',
                floor(50000 + random() * 850000)::int,
                NOW() - (random() * INTERVAL '45 days')
            );
        END IF;
    END LOOP;

    -- -------------------------------------------------------------------------
    -- D. GENERAR MATRIZ DE ENCLAVAMIENTOS Y ACTUACIONES PLC (300 Registros)
    -- Simula los 32 bits de enclavamientos del UDT (PDF Págs. 13-15)
    -- -------------------------------------------------------------------------
    RAISE NOTICE 'Poblando Enclavamientos y Registro de Actuaciones...';

    -- Estados activos de enclavamientos
    FOREACH eq IN ARRAY equips LOOP
        FOR i IN 0..7 LOOP
            INSERT INTO enclavaments (StartDate, Equip, NumeroEnclava, TextEnclava)
            VALUES (
                NOW() - (random() * INTERVAL '10 days'),
                eq,
                i,
                defects[1 + i] || ' a ' || eq
            );
        END LOOP;
    END LOOP;

    -- Registro de actuaciones del operador y PLC
    INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
    SELECT 
        equips[1 + floor(random() * array_length(equips, 1))::int],
        'Ordre Enviar Consigna SFOR: Consigna Cabal ' || (floor(10 + random() * 90)) || ' m3/h',
        operadors[1 + floor(random() * array_length(operadors, 1))::int],
        NOW() - (random() * INTERVAL '30 days')
    FROM generate_series(1, 400);

    INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari, DataHora)
    SELECT 
        equips[1 + floor(random() * array_length(equips, 1))::int],
        'Enclavament ' || (floor(random() * 32)) || ': ' || defects[1 + floor(random() * array_length(defects, 1))::int] || 
        CASE WHEN random() > 0.5 THEN ' desenclavat' ELSE ' actiu' END,
        'PLC',
        NOW() - (random() * INTERVAL '15 days')
    FROM generate_series(1, 200);

    -- -------------------------------------------------------------------------
    -- E. GENERAR CATÁLOGO DE TAGS Y TELEMETRÍA HISTÓRICA (5.000 Muestras)
    -- Simula tablas `sqlth_te` y `sqlt_data_1_{mes}` para Named Queries
    -- -------------------------------------------------------------------------
    RAISE NOTICE 'Poblando Diccionario de Tags y Telemetría Histórica (5.000 muestras)...';

    -- Catálogo de Tags
    INSERT INTO sqlth_te (id, tagpath) VALUES
    (1,  '[default]EDAR/EDAR_BOMBA_01/SCFL'),
    (2,  '[default]EDAR/EDAR_BOMBA_01/SCMX'),
    (3,  '[default]EDAR/EDAR_BOMBA_01/ESIM'),
    (4,  '[default]EDAR/EDAR_BOMBA_01/SFOR'),
    (5,  '[default]EDAR/EDAR_BOMBA_02/SCFL'),
    (6,  '[default]EDAR/EDAR_BOMBA_02/SCMX'),
    (7,  '[default]EDAR/EDAR_COMPRESOR_01/SCFL'),
    (8,  '[default]EDAR/EDAR_COMPRESOR_01/SCMX'),
    (9,  '[default]EDAR/DECANTADOR_01/SCFL'),
    (10, '[default]EDAR/VALVULA_RECIRC_01/SCFL')
    ON CONFLICT (id) DO NOTHING;

    -- Telemetría de Octubre 2026 (sqlt_data_1_2026_10) -> 3.000 muestras
    INSERT INTO sqlt_data_1_2026_10 (tagid, intvalue, floatvalue, stringvalue, datevalue, t_stamp)
    SELECT 
        1 + (g % 10),
        floor(random() * 100)::bigint,
        round((10.0 + (random() * 85.0))::numeric, 2)::double precision,
        'ESTAT_OK',
        TIMESTAMP '2026-10-01 00:00:00' + (g * INTERVAL '10 minutes'),
        -- Timestamp en milisegundos Epoch
        (EXTRACT(EPOCH FROM (TIMESTAMP '2026-10-01 00:00:00' + (g * INTERVAL '10 minutes'))) * 1000)::bigint
    FROM generate_series(1, 3000) AS g;

    -- Telemetría de Septiembre 2026 (sqlt_data_1_2026_09) -> 2.000 muestras
    INSERT INTO sqlt_data_1_2026_09 (tagid, intvalue, floatvalue, stringvalue, datevalue, t_stamp)
    SELECT 
        1 + (g % 10),
        floor(random() * 100)::bigint,
        round((10.0 + (random() * 85.0))::numeric, 2)::double precision,
        'ESTAT_OK',
        TIMESTAMP '2026-09-01 00:00:00' + (g * INTERVAL '15 minutes'),
        (EXTRACT(EPOCH FROM (TIMESTAMP '2026-09-01 00:00:00' + (g * INTERVAL '15 minutes'))) * 1000)::bigint
    FROM generate_series(1, 2000) AS g;

    RAISE NOTICE '=== GENERACIÓN DE DATOS COMPLETADA CON ÉXITO ===';
END $$;

-- -----------------------------------------------------------------------------
-- 3. RESUMEN Y VALIDACIÓN DE VOLÚMENES GENERADOS
-- -----------------------------------------------------------------------------
SELECT 'REG_OPERADORS' AS tabla, COUNT(*) AS total_registros FROM REG_OPERADORS
UNION ALL
SELECT 'documents_metadata', COUNT(*) FROM documents_metadata
UNION ALL
SELECT 'documents_contents', COUNT(*) FROM documents_contents
UNION ALL
SELECT 'notes', COUNT(*) FROM notes
UNION ALL
SELECT 'documents (adjuntos)', COUNT(*) FROM documents
UNION ALL
SELECT 'enclavaments', COUNT(*) FROM enclavaments
UNION ALL
SELECT 'ins_Registre_Actuacions', COUNT(*) FROM ins_Registre_Actuacions
UNION ALL
SELECT 'sqlth_te (catalogo tags)', COUNT(*) FROM sqlth_te
UNION ALL
SELECT 'sqlt_data_1_2026_10 (telemetria)', COUNT(*) FROM sqlt_data_1_2026_10
UNION ALL
SELECT 'sqlt_data_1_2026_09 (telemetria)', COUNT(*) FROM sqlt_data_1_2026_09;
```

***

#### Paso A.3: Levantar el Entorno Local

Abre una terminal en la carpeta `ignition-local-lab` y ejecuta:

```bash
docker compose up -d
```

Espera aproximadamente 60 segundos. Verifica que ambos contenedores estén en estado `healthy` o `Up`:

```bash
docker compose ps
```

***

#### Paso A.4: Configurar la Conexión de Base de Datos en el Gateway Local

1. Abre en tu navegador: `http://localhost:8088`
2. Inicia sesión con el usuario `admin` y contraseña `AdminMasterPass123!`.
3. Dirígete a **Config $\rightarrow$ Databases $\rightarrow$ Connections**.
4. Pulsa en **Create new Database Connection...**
5. Selecciona **PostgreSQL JDBC Driver** y pulsa **Next**.
6. Introduce los siguientes parámetros:
   * **Name:** `SandboxDB` _(respetar mayúsculas/minúsculas)_
   * **Connect URL:** `jdbc:postgresql://postgres:5432/sandbox_db`
   * **Username:** `ignition_user`
   * **Password:** `DBPassword123!`
7. Pulsa **Create New Database Connection**.
8. Confirma que el estado en la columna _Status_ figure como **`Valid`**.\
   ![](<.gitbook/assets/image (25).png>)

<figure><img src=".gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

get designer\
![](<.gitbook/assets/image (27).png>)\
\
![](<.gitbook/assets/image (28).png>)

<figure><img src=".gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

login

<figure><img src=".gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

\
<br>

***

### 3. Opción B: Conexión al Servidor de Formación Cloud (GCP)

Si vas a realizar las prácticas conectado directamente a la máquina virtual del curso:

* **IP Pública del Servidor:** `34.7.33.184`
* **URL del Gateway Web:** `http://34.7.33.184:8088`
* **Conexión de Base de Datos preconfigurada:** `SandboxDB`
* **Tag Provider por defecto:** `default`
* **Tus credenciales individuales de diseñador:**
  * Usuario: `alumnoXX` (ej. `alumno01`, `alumno02`, ..., `alumno20`)
  * Contraseña: `Password_XX!` (ej. `Password_01!`, `Password_02!`, ..., `Password_20!`)

***

### 4. Configuración del Ignition Designer Launcher

1. Abre la aplicación **Ignition Designer Launcher** en tu ordenador.
2. Pulsa en **Add Gateway**.
3. Añade la dirección correspondiente:
   * Para entorno local: `http://localhost:8088`
   * Para entorno Cloud: `http://34.7.33.184:8088`
4. Selecciona el Gateway añadido y pulsa **Open Designer**.
5. Autentícate con tus credenciales asignadas (`alumnoXX` o `admin` en local).
6. Crea tu proyecto personal de prácticas con el nombre: `Proyecto_Alumno_XX`.

***

### 5. Batería de Tests de Validación del Entorno

Realiza los siguientes 5 tests desde tu Designer para certificar que tu entorno está listo para los laboratorios del curso.

***

#### 🧪 Test 1: Conectividad y Autenticación en Designer

* **Objetivo:** Validar la comunicación entre el Designer Launcher y el Gateway de Ignition, confirmando permisos de diseño.
* **Ejecución:**
  1. Abrir el proyecto personal `Proyecto_Alumno_XX` en el Designer.
  2. Verificar que en la esquina inferior derecha el estado de comunicación marque **Connected** con icono verde.
* **Resultado esperado:**
  * El Designer abre el espacio de trabajo sin errores de autenticación ni alertas de desconexión.

***

#### 🧪 Test 2: Validación de la Conexión JDBC `SandboxDB`

* **Objetivo:** Comprobar desde la consola de scripting que la conexión con PostgreSQL está activa y accesible por el motor de scripts.
*   **Ejecución:**

    1. En el Designer, abre el menú superior **Tools $\rightarrow$ Script Console**.
    2. Ejecuta el siguiente script:

    ```python
    res = system.db.runScalarPrepQuery("SELECT 1", [], database="SandboxDB")
    print "Respuesta de PostgreSQL:", res
    ```
* **Resultado esperado:**
  * La consola imprime: `Respuesta de PostgreSQL: 1`.

***

#### 🧪 Test 3: Validación del Esquema de Datos de Laboratorio

* **Objetivo:** Verificar que las tablas necesarias para los casos de estudio (gestor documental, enclavamientos, notas y operadores) contienen datos de prueba.
*   **Ejecución:**

    1. En la **Script Console**, pega y ejecuta:

    ```python
    q = """
    SELECT 
        (SELECT COUNT(*) FROM documents_metadata) AS total_docs,
        (SELECT COUNT(*) FROM enclavaments) AS total_enclavaments,
        (SELECT COUNT(*) FROM notes WHERE Readed = 0) AS notas_pendientes,
        (SELECT COUNT(*) FROM REG_OPERADORS) AS total_operadores
    """
    pyds = system.db.runPrepQuery(q, [], database="SandboxDB")
    for row in pyds:
        print "Documentos:", row["total_docs"]
        print "Enclavamientos:", row["total_enclavaments"]
        print "Notas pendientes:", row["notas_pendientes"]
        print "Registros operadores:", row["total_operadores"]
    ```
* **Resultado esperado:**
  * La consola muestra recuentos válidos (Documentos $\ge 0$, Enclavamientos $\ge 2$, Notas pendientes $\ge 2$, Registros operadores $\ge 3$).

***

#### 🧪 Test 4: Validación del Tag Provider y Lectura en Bloque

* **Objetivo:** Certificar que el Tag Provider `default` permite la lectura agrupada y responde con objetos `QualifiedValue` correctos.
*   **Ejecución:**

    1. En la **Script Console**, ejecuta:

    ```python
    system.tag.writeBlocking(["[default]Test_Signal"], [100.0])
    qv = system.tag.readBlocking(["[default]Test_Signal"])[0]
    print "Valor Tag:", qv.value
    print "Calidad:", qv.quality.toString()
    print "Es Buena Calidad?:", qv.quality.isGood()
    ```
* **Resultado esperado:**
  * La consola imprime `Valor Tag: 100.0`, `Calidad: Good` y `Es Buena Calidad?: True`.

***

#### 🧪 Test 5: Simulación de Named Query con Parámetros Tipados

* **Objetivo:** Validar la capacidad de ejecución de consultas de inserción y lectura parametrizada sobre la tabla de actuaciones del operador.
*   **Ejecución:**

    1. En la **Script Console**, ejecuta una inserción y comprobación parametrizada:

    ```python
    # Inserción de prueba
    ins_q = "INSERT INTO ins_Registre_Actuacions (Equip, Descripcio, Usuari) VALUES (?, ?, ?)"
    system.db.runPrepUpdate(ins_q, ["BOMBA_01", "Test de validación Sesión 0", "Alumno_Test"], database="SandboxDB")

    # Lectura de comprobación
    sel_q = "SELECT Descripcio, Usuari FROM ins_Registre_Actuacions WHERE Equip = ? ORDER BY id DESC LIMIT 1"
    resultado = system.db.runPrepQuery(sel_q, ["BOMBA_01"], database="SandboxDB")
    print "Última actuación registrada:", resultado[0]["Descripcio"], "| Por:", resultado[0]["Usuari"]
    ```
* **Resultado esperado:**
  * La consola imprime: `Última actuación registrada: Test de validación Sesión 0 | Por: Alumno_Test`.

***

### 6. Checklist de Conformidad Final (Pase a Sesión 1)

Marca cada casilla para certificar que tu entorno está 100% operativo:

* [ ] **Designer Launcher** instalado y comunicando con el Gateway (`localhost:8088` o `34.7.33.184:8088`).
* [ ] Conexión de base de datos **`SandboxDB`** en estado **`Valid`**.
* [ ] Base de datos PostgreSQL con todas las tablas creadas y datos de prueba cargados.
* [ ] Los 5 tests de validación ejecutados en la **Script Console** con resultado satisfactorio.
* [ ] Proyecto de Designer `Proyecto_Alumno_XX` creado y guardado.
