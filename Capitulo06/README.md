# Simulación de migración: Simulación de migración y Validación post-migración

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 90 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica se simula la migración de la base de datos `MigracionLab_Source` desde una instancia de SQL Server 2017 (`SQL2017SRC`) hacia una instancia de SQL Server 2025 (`SQL2025TGT`). El taller crea esa base de ejemplo en el origen, con tablas, una vista, un procedimiento y un login de prueba. Después se inventaría el origen, se evalúa la compatibilidad con Data Migration Assistant (DMA), se respalda, se transfiere, se restaura como `MigracionLab_2025` y se valida integridad, datos, objetos y funcionamiento antes y después de subir la compatibilidad a 170. A partir del respaldo, `MigracionLab_Source` no se modifica ni se elimina. Con `MigracionLab_2025` ya en SQL Server 2025 se agrega la cadena de respaldos full, diferencial, de log y tail-log, y un trabajo de SQL Server Agent sobre esa base.

## Objetivos de Aprendizaje

- [ ] Crear en `SQL2017SRC` la base de ejemplo `MigracionLab_Source`, con datos, objetos y un login de prueba conocidos.
- [ ] Inventariar la instancia y la base de datos de origen, incluyendo configuración, objetos, dependencias, usuarios y características relevantes.
- [ ] Ejecutar una evaluación de compatibilidad con DMA entre SQL Server 2017 y SQL Server 2025.
- [ ] Crear, verificar y transferir una copia de seguridad con `CHECKSUM` y hash SHA-256.
- [ ] Restaurar la base de datos en SQL Server 2025 conservando inicialmente el nivel de compatibilidad heredado.
- [ ] Validar integridad, datos, objetos, seguridad y rendimiento funcional antes y después de elevar la compatibilidad a nivel 170.
- [ ] Crear en `SQL2025TGT` la cadena full, diferencial, de log y tail-log de `MigracionLab_2025`, y restaurarla hasta dejar la base en línea.
- [ ] Crear y ejecutar en SQL Server Agent un trabajo sobre la base ya migrada.

## Prerrequisitos

**Conocimientos requeridos**

- Conocimientos básicos de T-SQL, bases de datos SQL Server, copias de seguridad y restauración.
- Comprensión básica de niveles de compatibilidad, usuarios de base de datos y logins de instancia.
- Las instancias `SQL2017SRC` (SQL Server 2017) y `SQL2025TGT` (SQL Server 2025) ya están creadas y aceptan autenticación de Windows. Este taller no las instala.

**Acceso requerido**

- Permisos equivalentes a `sysadmin` en `SQL2017SRC` y `SQL2025TGT`.
- Acceso a SQL Server Management Studio (SSMS) 21 y Data Migration Assistant 5.6.6784.0.
- Windows PowerShell 5.1 o PowerShell 7, ejecutado como administrador en cada equipo que aloje una instancia.
- Permisos de modificación para la cuenta de servicio de cada instancia sobre las carpetas que esa instancia usa:
  - En el origen: `C:\LabSQL2025\Backup`, `C:\LabSQL2025\Data` y `C:\LabSQL2025\Logs`.
  - En el destino: `C:\LabSQL2025\Backup`, `C:\SQLLab2025\Data` y `C:\SQLLab2025\Logs`.
- Conectividad entre las instancias de origen y destino mediante los puertos configurados.
- Espacio libre suficiente para los archivos de datos, log y respaldo. La base de ejemplo ocupa pocas decenas de megabytes.

> **Nota de rutas:** esta práctica usa `C:\LabSQL2025\Backup` y `C:\LabSQL2025\Reports` para los artefactos de migración, tal como establece su especificación. Para los archivos restaurados se usan las rutas estándar del laboratorio `C:\SQLLab2025\Data` y `C:\SQLLab2025\Logs`.

## Entorno de Laboratorio

| Componente | Origen | Destino |
|---|---|---|
| Instancia | `SQL2017SRC` | `SQL2025TGT` |
| Motor | SQL Server 2017 Developer Edition | SQL Server 2025 Developer Edition |
| Base de datos | `MigracionLab_Source` | `MigracionLab_2025` |
| Propósito | Base de datos original; no debe modificarse | Base de datos restaurada para validación |
| Herramientas | SSMS 21, DMA 5.6.6784.0, Windows PowerShell 5.1 o PowerShell 7 | SSMS 21, Windows PowerShell 5.1 o PowerShell 7 |

Ejecute PowerShell **como administrador** en cada equipo que aloje `SQL2017SRC` o `SQL2025TGT`. El script crea las carpetas, concede modificación a la cuenta virtual de la instancia que exista en ese equipo y publica el recurso `LabBackup` para la transferencia.

```powershell
$LabFolders = @(
    'C:\LabSQL2025\Backup',
    'C:\LabSQL2025\Reports',
    'C:\LabSQL2025\Data',
    'C:\LabSQL2025\Logs',
    'C:\SQLLab2025\Data',
    'C:\SQLLab2025\Logs'
)

$LabFolders | ForEach-Object {
    New-Item -Path $_ -ItemType Directory -Force | Out-Null
}

$SqlManagedFolders = @(
    'C:\LabSQL2025\Backup',
    'C:\LabSQL2025\Data',
    'C:\LabSQL2025\Logs',
    'C:\SQLLab2025\Data',
    'C:\SQLLab2025\Logs'
)

$ServiceAccounts = @(
    'NT SERVICE\MSSQL$SQL2017SRC',
    'NT SERVICE\MSSQL$SQL2025TGT',
    'NT SERVICE\MSSQLSERVER'
)

$GrantedAccounts = New-Object System.Collections.Generic.List[string]

foreach ($Folder in $SqlManagedFolders) {
    foreach ($Account in $ServiceAccounts) {
        $AccountObject = New-Object System.Security.Principal.NTAccount($Account)
        try {
            $null = $AccountObject.Translate([System.Security.Principal.SecurityIdentifier])
        }
        catch {
            continue
        }

        $Grant = '"{0}:(OI)(CI)M"' -f $Account
        icacls $Folder /grant $Grant | Out-Null
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "No se pudo conceder modificación a $Account en $Folder."
        }
        elseif (-not $GrantedAccounts.Contains($Account)) {
            $GrantedAccounts.Add($Account)
        }
    }
}

if ($GrantedAccounts.Count -eq 0) {
    Write-Warning "Ninguna cuenta virtual de SQL Server recibió permisos. Conceda modificación a la cuenta que devuelva sys.dm_server_services."
}

$BackupFolder = 'C:\LabSQL2025\Backup'
$ExistingShare = Get-SmbShare -Name 'LabBackup' -ErrorAction SilentlyContinue
if (-not $ExistingShare) {
    New-SmbShare `
        -Name 'LabBackup' `
        -Path $BackupFolder `
        -FullAccess 'BUILTIN\Administrators' `
        -ReadAccess 'NT AUTHORITY\Authenticated Users' |
        Out-Null
}
elseif ($ExistingShare.Path -ne $BackupFolder) {
    Write-Warning "El recurso LabBackup ya existe y apunta a $($ExistingShare.Path)."
}

Get-ChildItem C:\LabSQL2025 -Directory
Get-SmbShare -Name 'LabBackup'
```

Compruebe en cada instancia la cuenta de servicio del motor. Use la fila del servicio de base de datos, no la de SQL Server Agent. Si esa cuenta no es una de las cuentas virtuales del script, conceda modificación a esa identidad sobre las carpetas que la instancia va a usar.

```sql
SELECT
    servicename,
    service_account,
    status_desc,
    startup_type_desc
FROM sys.dm_server_services;
```

> **Nota operativa:** en el equipo de origen, SQL Server escribe la base de ejemplo en `C:\LabSQL2025\Data` y `C:\LabSQL2025\Logs`, y el respaldo en `C:\LabSQL2025\Backup`. En el destino, la base restaurada usa `C:\SQLLab2025\Data` y `C:\SQLLab2025\Logs`. Separar esas rutas permite alojar las dos instancias en el mismo equipo sin que los archivos se pisen. El recurso `\\<equipo-origen>\LabBackup` apunta a `C:\LabSQL2025\Backup`.

> **Importante:** no incluya contraseñas de entornos reales en scripts, informes, capturas ni conversaciones con asistentes de IA. La única contraseña de este taller es la del login de ejemplo `lab_migracion`, y solo sirve dentro del laboratorio. Use autenticación de Windows para las tareas administrativas.

## Instrucciones Paso a Paso

### Paso 1: Confirmar conectividad y preparar directorios

**Objetivo:** comprobar que las instancias de origen y destino están disponibles, identificar sus versiones y preparar una ubicación controlada para los artefactos de migración.

**Instrucciones**

1. Abra SSMS y conéctese a `SQL2017SRC` mediante autenticación de Windows.
2. Abra una segunda ventana de SSMS y conéctese a `SQL2025TGT`.
3. En cada instancia, ejecute la siguiente consulta para registrar versión, edición, servidor y fecha de inicio:

```sql
SELECT
    @@SERVERNAME AS nombre_servidor,
    SERVERPROPERTY('MachineName') AS equipo,
    SERVERPROPERTY('InstanceName') AS instancia,
    SERVERPROPERTY('Edition') AS edicion,
    SERVERPROPERTY('ProductVersion') AS version_producto,
    SERVERPROPERTY('ProductLevel') AS nivel_producto,
    SERVERPROPERTY('ProductUpdateLevel') AS nivel_actualizacion,
    SERVERPROPERTY('ProductUpdateReference') AS referencia_actualizacion,
    sqlserver_start_time AS inicio_servicio
FROM sys.dm_os_sys_info;
```

4. En el equipo de origen, confirme las carpetas de la base de ejemplo y del respaldo:

```powershell
Test-Path "C:\LabSQL2025\Backup"
Test-Path "C:\LabSQL2025\Data"
Test-Path "C:\LabSQL2025\Logs"
Test-Path "C:\LabSQL2025\Reports"
Get-SmbShare -Name "LabBackup"
```

5. En el equipo de destino, confirme las rutas de la base restaurada y de los reportes:

```powershell
Test-Path "C:\SQLLab2025\Data"
Test-Path "C:\SQLLab2025\Logs"
Test-Path "C:\LabSQL2025\Backup"
Test-Path "C:\LabSQL2025\Reports"
```

6. Anote el valor de `equipo` devuelto por la consulta del punto 3. Ese nombre se usa en la ruta `\\<equipo>\LabBackup` cuando origen y destino están en equipos distintos.
7. Registre en su informe el nombre de cada instancia, versión exacta del motor, edición y ubicación física de cada carpeta.

**Resultado esperado**

- `SQL2017SRC` responde con información de SQL Server 2017.
- `SQL2025TGT` responde con información de SQL Server 2025.
- Todas las rutas requeridas existen.
- El recurso compartido `LabBackup` existe en el equipo de origen y su ruta es `C:\LabSQL2025\Backup`.

**Verificación**

Ejecute en ambas instancias:

```sql
SELECT
    SERVERPROPERTY('ServerName') AS servidor,
    SERVERPROPERTY('ProductVersion') AS version,
    SERVERPROPERTY('Edition') AS edicion;
```

Registre la salida en `C:\LabSQL2025\Reports\06-00-01_RegistroEntorno.txt`.

---

### Paso 2: Crear la base de ejemplo en el origen

**Objetivo:** crear en `SQL2017SRC` la base `MigracionLab_Source`, con archivos, datos, objetos y un login conocidos, para migrarla en los pasos siguientes.

**Instrucciones**

1. En una ventana conectada a `SQL2017SRC`, ejecute el script completo. Si el primer lote informa que la instancia no es SQL Server 2017, pulse **Cancelar** en SSMS y no deje que continúen los lotes siguientes.
2. El script sustituye `MigracionLab_Source` si ya existe en esa instancia. No lo ejecute en `SQL2025TGT`.
3. La contraseña de `lab_migracion` es solo de este taller.

```sql
USE [master];
GO

IF CAST(SERVERPROPERTY('ProductMajorVersion') AS int) <> 14
    THROW 50001, N'Ejecute este script conectado a SQL2017SRC (SQL Server 2017).', 1;

IF DB_ID(N'MigracionLab_Source') IS NOT NULL
BEGIN
    ALTER DATABASE [MigracionLab_Source] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
    DROP DATABASE [MigracionLab_Source];
END
GO

CREATE DATABASE [MigracionLab_Source]
ON PRIMARY
(
    NAME = N'MigracionLab_Source_Data',
    FILENAME = N'C:\LabSQL2025\Data\MigracionLab_Source.mdf'
)
LOG ON
(
    NAME = N'MigracionLab_Source_Log',
    FILENAME = N'C:\LabSQL2025\Logs\MigracionLab_Source_log.ldf'
);
GO

ALTER DATABASE [MigracionLab_Source] SET COMPATIBILITY_LEVEL = 140;
ALTER DATABASE [MigracionLab_Source] SET RECOVERY FULL;
ALTER DATABASE [MigracionLab_Source] SET PAGE_VERIFY CHECKSUM;
GO

IF NOT EXISTS
(
    SELECT 1
    FROM sys.server_principals
    WHERE name = N'lab_migracion'
)
BEGIN
    CREATE LOGIN [lab_migracion]
        WITH PASSWORD = N'LabMigracion2025!ChangeMe',
             CHECK_POLICY = ON;
END
GO

USE [MigracionLab_Source];
GO

CREATE SCHEMA Sales;
GO

CREATE SCHEMA Reporting;
GO

CREATE TABLE Sales.Customers
(
    CustomerId int NOT NULL CONSTRAINT PK_Customers PRIMARY KEY,
    CustomerCode varchar(12) NOT NULL CONSTRAINT UQ_Customers_CustomerCode UNIQUE,
    CustomerName nvarchar(80) NOT NULL
);
GO

CREATE TABLE Sales.Orders
(
    OrderId int NOT NULL CONSTRAINT PK_Orders PRIMARY KEY,
    CustomerId int NOT NULL,
    OrderDate date NOT NULL,
    OrderStatus char(1) NOT NULL,
    TotalAmount decimal(12,2) NOT NULL,
    CONSTRAINT FK_Orders_Customers
        FOREIGN KEY (CustomerId) REFERENCES Sales.Customers (CustomerId),
    CONSTRAINT CK_Orders_Status CHECK (OrderStatus IN ('C', 'P', 'X'))
);
GO

CREATE TABLE Sales.OrderLines
(
    OrderLineId int NOT NULL CONSTRAINT PK_OrderLines PRIMARY KEY,
    OrderId int NOT NULL,
    ProductCode varchar(12) NOT NULL,
    Quantity int NOT NULL,
    UnitPrice decimal(12,2) NOT NULL,
    LineAmount AS (CONVERT(decimal(12,2), Quantity * UnitPrice)) PERSISTED,
    CONSTRAINT FK_OrderLines_Orders
        FOREIGN KEY (OrderId) REFERENCES Sales.Orders (OrderId),
    CONSTRAINT CK_OrderLines_Quantity CHECK (Quantity > 0),
    CONSTRAINT CK_OrderLines_UnitPrice CHECK (UnitPrice >= 0)
);
GO

INSERT INTO Sales.Customers (CustomerId, CustomerCode, CustomerName)
VALUES
    (1, 'C0001', N'Ana Ruiz'),
    (2, 'C0002', N'Luis Mora'),
    (3, 'C0003', N'Carla Diaz'),
    (4, 'C0004', N'Pedro Solis'),
    (5, 'C0005', N'Elena Cruz');

INSERT INTO Sales.Orders (OrderId, CustomerId, OrderDate, OrderStatus, TotalAmount)
VALUES
    (1, 1, '20240615', 'C', 175.00),
    (2, 1, '20240701', 'C', 80.00),
    (3, 2, '20240620', 'P', 45.00),
    (4, 2, '20240802', 'C', 210.00),
    (5, 3, '20240511', 'C', 60.00),
    (6, 3, '20240909', 'X', 30.00),
    (7, 4, '20240418', 'C', 500.00),
    (8, 5, '20241001', 'C', 95.00);

INSERT INTO Sales.OrderLines (OrderLineId, OrderId, ProductCode, Quantity, UnitPrice)
VALUES
    (1,  1, 'P-100', 2,  25.00),
    (2,  1, 'P-200', 1, 125.00),
    (3,  2, 'P-100', 1,  80.00),
    (4,  3, 'P-300', 3,  15.00),
    (5,  4, 'P-200', 1, 125.00),
    (6,  4, 'P-400', 1,  85.00),
    (7,  5, 'P-100', 2,  30.00),
    (8,  6, 'P-500', 1,  30.00),
    (9,  7, 'P-400', 4,  85.00),
    (10, 7, 'P-200', 1, 160.00),
    (11, 8, 'P-300', 1,  15.00),
    (12, 8, 'P-100', 2,  40.00);
GO

CREATE VIEW Reporting.v_OrderSummary
AS
SELECT
    o.OrderId,
    c.CustomerCode,
    c.CustomerName,
    o.OrderDate,
    o.OrderStatus,
    o.TotalAmount
FROM Sales.Orders AS o
INNER JOIN Sales.Customers AS c
    ON c.CustomerId = o.CustomerId;
GO

CREATE PROCEDURE Reporting.usp_OrdersByCustomer
    @CustomerCode varchar(12)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        v.OrderId,
        v.CustomerCode,
        v.CustomerName,
        v.OrderDate,
        v.OrderStatus,
        v.TotalAmount
    FROM Reporting.v_OrderSummary AS v
    WHERE v.CustomerCode = @CustomerCode
    ORDER BY v.OrderDate, v.OrderId;
END
GO

CREATE USER [lab_migracion] FOR LOGIN [lab_migracion];
ALTER ROLE db_datareader ADD MEMBER [lab_migracion];
GRANT EXECUTE ON Reporting.usp_OrdersByCustomer TO [lab_migracion];
GO

USE [msdb];
GO

IF NOT EXISTS (SELECT 1 FROM dbo.sysjobs WHERE name = N'MigracionLab_Inventario')
BEGIN
    EXEC dbo.sp_add_job
        @job_name = N'MigracionLab_Inventario',
        @enabled = 0,
        @description = N'Trabajo de ejemplo. No se transfiere con RESTORE DATABASE.';

    EXEC dbo.sp_add_jobstep
        @job_name = N'MigracionLab_Inventario',
        @step_name = N'Conteo de pedidos',
        @subsystem = N'TSQL',
        @database_name = N'MigracionLab_Source',
        @command = N'SELECT COUNT(*) FROM Sales.Orders;';

    EXEC dbo.sp_add_jobserver
        @job_name = N'MigracionLab_Inventario';
END
GO
```

4. Si el lote de SQL Server Agent falla porque el servicio no está instalado, anote esa ausencia y continúe. El inventario de trabajos quedará vacío y esa ausencia también se documenta.
5. A partir de aquí no modifique objetos ni datos de `MigracionLab_Source`.

**Resultado esperado**

- `MigracionLab_Source` queda en línea, con compatibilidad `140`, recuperación `FULL` y verificación de páginas `CHECKSUM`.
- Los nombres lógicos son `MigracionLab_Source_Data` y `MigracionLab_Source_Log`.
- `Sales.Customers` tiene 5 filas, `Sales.Orders` tiene 8 y `Sales.OrderLines` tiene 12.
- Existen `Reporting.v_OrderSummary` y `Reporting.usp_OrdersByCustomer`.
- El login y el usuario `lab_migracion` existen en el origen. El usuario es miembro de `db_datareader` y puede ejecutar el procedimiento.
- El trabajo `MigracionLab_Inventario` queda deshabilitado en `msdb`. No se ejecuta durante el taller.

**Verificación**

```sql
USE [MigracionLab_Source];
GO

SELECT
    d.name AS base_datos,
    d.compatibility_level,
    d.recovery_model_desc,
    d.page_verify_option_desc,
    f.name AS nombre_logico,
    f.type_desc,
    f.physical_name
FROM sys.databases AS d
CROSS JOIN sys.database_files AS f
WHERE d.name = DB_NAME()
ORDER BY f.type, f.file_id;

SELECT
    SCHEMA_NAME(t.schema_id) AS esquema,
    t.name AS tabla,
    SUM(ps.row_count) AS filas
FROM sys.tables AS t
INNER JOIN sys.dm_db_partition_stats AS ps
    ON t.object_id = ps.object_id
WHERE t.is_ms_shipped = 0
  AND ps.index_id IN (0, 1)
GROUP BY t.schema_id, t.name
ORDER BY esquema, tabla;

SELECT
    dp.name AS usuario,
    sp.name AS login_asociado
FROM sys.database_principals AS dp
LEFT JOIN sys.server_principals AS sp
    ON dp.sid = sp.sid
WHERE dp.name = N'lab_migracion';

EXEC Reporting.usp_OrdersByCustomer
    @CustomerCode = 'C0001';
```

El procedimiento debe devolver dos filas: el pedido 1 del `2024-06-15` por `175.00` y el pedido 2 del `2024-07-01` por `80.00`.

---

### Paso 3: Inventariar el entorno de origen

**Objetivo:** recopilar una línea base técnica de la base `MigracionLab_Source` creada en el paso anterior, antes de la evaluación de compatibilidad y la migración.

**Instrucciones**

1. Conéctese a `SQL2017SRC`.
2. Ejecute la siguiente consulta para inventariar la base de datos de origen:

```sql
SELECT
    name AS nombre_base_datos,
    compatibility_level AS nivel_compatibilidad,
    recovery_model_desc AS modelo_recuperacion,
    state_desc AS estado,
    user_access_desc AS acceso_usuarios,
    page_verify_option_desc AS verificacion_paginas,
    collation_name AS intercalacion,
    create_date AS fecha_creacion
FROM sys.databases
WHERE name = N'MigracionLab_Source';
```

3. Cambie el contexto a la base de datos de origen y registre tamaño, ubicación y espacio utilizado de sus archivos:

```sql
USE [MigracionLab_Source];
GO

SELECT
    DB_NAME() AS base_datos,
    name AS nombre_archivo,
    type_desc AS tipo_archivo,
    physical_name AS ruta_fisica,
    CAST(size / 128.0 AS decimal(18,2)) AS tamano_mb,
    CAST(FILEPROPERTY(name, 'SpaceUsed') / 128.0 AS decimal(18,2)) AS espacio_usado_mb,
    CAST((size - FILEPROPERTY(name, 'SpaceUsed')) / 128.0 AS decimal(18,2)) AS espacio_libre_mb
FROM sys.database_files;
```

4. Obtenga la configuración relevante de la instancia de origen:

```sql
SELECT
    name,
    value_in_use AS valor_configurado,
    value AS valor_configurado_sp_configure,
    description
FROM sys.configurations
WHERE name IN
(
    'max server memory (MB)',
    'max degree of parallelism',
    'cost threshold for parallelism',
    'backup compression default'
)
ORDER BY name;
```

5. Inventaríe tablas, vistas, procedimientos almacenados, funciones y triggers:

```sql
USE [MigracionLab_Source];
GO

SELECT
    SCHEMA_NAME(o.schema_id) AS esquema,
    o.name AS objeto,
    o.type_desc AS tipo_objeto,
    o.create_date AS fecha_creacion,
    o.modify_date AS fecha_modificacion
FROM sys.objects AS o
WHERE o.is_ms_shipped = 0
ORDER BY
    o.type_desc,
    esquema,
    objeto;
```

6. Identifique módulos programables y registre una huella SHA-256 de sus definiciones para facilitar la comparación posterior:

```sql
USE [MigracionLab_Source];
GO

SELECT
    SCHEMA_NAME(o.schema_id) AS esquema,
    o.name AS objeto,
    o.type_desc AS tipo_objeto,
    CONVERT(varchar(64), HASHBYTES(
        'SHA2_256',
        CONVERT(varbinary(max), sm.definition)
    ), 2) AS hash_definicion
FROM sys.objects AS o
INNER JOIN sys.sql_modules AS sm
    ON o.object_id = sm.object_id
WHERE o.is_ms_shipped = 0
ORDER BY
    esquema,
    objeto;
```

7. Inventaríe usuarios, roles y asociaciones con logins:

```sql
USE [MigracionLab_Source];
GO

SELECT
    dp.name AS usuario_base_datos,
    dp.type_desc AS tipo_usuario,
    dp.authentication_type_desc AS tipo_autenticacion,
    sp.name AS login_asociado,
    dp.default_schema_name
FROM sys.database_principals AS dp
LEFT JOIN sys.server_principals AS sp
    ON dp.sid = sp.sid
WHERE dp.type NOT IN ('A', 'G', 'R', 'X')
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
ORDER BY dp.name;
```

8. Revise dependencias de objetos dentro de la base de datos:

```sql
USE [MigracionLab_Source];
GO

SELECT
    OBJECT_SCHEMA_NAME(d.referencing_id) AS esquema_origen,
    OBJECT_NAME(d.referencing_id) AS objeto_origen,
    d.referenced_schema_name AS esquema_referenciado,
    d.referenced_entity_name AS objeto_referenciado,
    d.referenced_database_name AS base_referenciada,
    d.referenced_server_name AS servidor_referenciado
FROM sys.sql_expression_dependencies AS d
WHERE d.referencing_id IS NOT NULL
ORDER BY
    esquema_origen,
    objeto_origen;
```

9. Inventaríe trabajos de SQL Server Agent. Recuerde que los trabajos de `msdb` no se transfieren mediante una restauración de base de datos:

```sql
USE [msdb];
GO

SELECT
    j.name AS nombre_trabajo,
    j.enabled AS habilitado,
    j.description AS descripcion,
    SUSER_SNAME(j.owner_sid) AS propietario,
    j.date_created AS fecha_creacion,
    j.date_modified AS fecha_modificacion
FROM dbo.sysjobs AS j
ORDER BY j.name;
```

10. Revise características persistentes de SQL Server utilizadas por la base:

```sql
USE [MigracionLab_Source];
GO

SELECT
    feature_name,
    feature_id
FROM sys.dm_db_persisted_sku_features
ORDER BY feature_name;
```

**Resultado esperado**

- Se dispone de un inventario de versión, compatibilidad, recuperación, archivos, objetos, usuarios, dependencias y trabajos.
- La compatibilidad de `MigracionLab_Source` es 140 y el modelo de recuperación es `FULL`.
- Los archivos lógicos son `MigracionLab_Source_Data` y `MigracionLab_Source_Log`.
- Los conteos son `Sales.Customers` = 5, `Sales.Orders` = 8 y `Sales.OrderLines` = 12.
- Existen `Reporting.v_OrderSummary` y `Reporting.usp_OrdersByCustomer`.
- El usuario `lab_migracion` aparece asociado a su login.
- El trabajo `MigracionLab_Inventario` aparece en `msdb` del origen y está deshabilitado. Los trabajos de Agent no se trasladan con `BACKUP DATABASE` y `RESTORE DATABASE`. Tampoco se trasladan los logins, los linked servers ni la configuración de la instancia.
- La base de datos de origen permanece sin modificaciones después de este inventario.

**Verificación**

Exporte los resultados de las consultas a archivos CSV o texto y guárdelos con el prefijo `06-00-01_Origen_` en:

```text
C:\LabSQL2025\Reports
```

Como mínimo, conserve:

```text
06-00-01_Origen_BaseDatos.csv
06-00-01_Origen_Archivos.csv
06-00-01_Origen_Objetos.csv
06-00-01_Origen_Usuarios.csv
06-00-01_Origen_Dependencias.csv
06-00-01_Origen_TrabajosAgent.csv
```

---

### Paso 4: Evaluar compatibilidad con Data Migration Assistant

**Objetivo:** identificar bloqueadores, cambios importantes y recomendaciones de compatibilidad antes de transferir la base de datos.

**Instrucciones**

1. Abra **Data Migration Assistant 5.6.6784.0**.
2. Seleccione **New** y cree un proyecto de tipo **Assessment**.
3. Asigne un nombre descriptivo, por ejemplo:

```text
DMA_MigracionLab_Source_2017_a_2025
```

4. Seleccione **SQL Server** como tipo de servidor de origen.
5. Configure la instancia de origen como `SQL2017SRC` y autentíquese con una cuenta administrativa del laboratorio.
6. Seleccione la base de datos `MigracionLab_Source`.
7. Seleccione **SQL Server 2025** como plataforma de destino, si está disponible en la versión instalada de DMA.
8. Incluya las categorías de evaluación disponibles, especialmente:
   - Problemas de compatibilidad.
   - Cambios importantes.
   - Características obsoletas.
   - Recomendaciones de rendimiento y comportamiento.
9. Ejecute la evaluación.
10. Revise cada hallazgo y clasifíquelo en una de las siguientes categorías:
    - **Bloqueador:** impide o desaconseja la migración hasta aplicar una corrección.
    - **Advertencia:** requiere prueba, revisión o mitigación.
    - **Información:** no impide la migración, pero debe documentarse.
11. Exporte el informe de DMA en formato HTML o JSON y guárdelo en:

```text
C:\LabSQL2025\Reports\06-00-01_DMA_MigracionLab_Source.html
```

12. Si DMA no ofrece explícitamente SQL Server 2025 como destino, documente esta limitación en el informe y use la versión de destino más reciente disponible únicamente como evaluación complementaria. No interprete la ausencia de hallazgos como garantía de compatibilidad.

**Resultado esperado**

- Se genera un informe de compatibilidad para `MigracionLab_Source`.
- Se identifican los elementos que requieren corrección o pruebas adicionales.
- Se dispone de evidencia exportable para la decisión técnica de migración.

**Verificación**

Complete una tabla de hallazgos en `C:\LabSQL2025\Reports\06-00-01_MatrizHallazgos.csv`:

| ID | Hallazgo | Severidad | Impacto potencial | Acción | Estado |
|---|---|---|---|---|---|
| DMA-01 | Ejemplo: característica obsoleta | Advertencia | Cambio de comportamiento | Probar consulta o reemplazar característica | Pendiente/Resuelto |
| DMA-02 | Ejemplo: dependencia externa | Información | Error de conectividad | Validar servidor y credenciales | Pendiente |

No continúe con una migración real si DMA identifica un bloqueador no evaluado. En este laboratorio, documente el bloqueador y continúe solo si el instructor autoriza la simulación.

---

### Paso 5: Crear y verificar el respaldo consistente del origen

**Objetivo:** crear una copia de seguridad consistente de `MigracionLab_Source`, verificar su legibilidad y calcular un hash para comprobar su integridad durante la transferencia.

**Instrucciones**

1. En `SQL2017SRC`, confirme que no se está ejecutando una operación de respaldo o restauración sobre la base:

```sql
SELECT
    r.session_id,
    r.command,
    r.status,
    r.percent_complete,
    r.start_time,
    r.estimated_completion_time / 1000.0 AS segundos_estimados
FROM sys.dm_exec_requests AS r
WHERE r.command IN ('BACKUP DATABASE', 'RESTORE DATABASE');
```

2. Cree una copia de seguridad completa de solo copia. Conserve este nombre de archivo en todos los pasos posteriores:

```sql
BACKUP DATABASE [MigracionLab_Source]
TO DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak'
WITH
    COPY_ONLY,
    INIT,
    CHECKSUM,
    COMPRESSION,
    STATS = 10,
    NAME = N'MigracionLab_Source - Respaldo de migración';
GO
```

`INIT` reemplaza un respaldo anterior en ese mismo archivo si repite el paso. `COPY_ONLY` no altera una cadena de respaldos diferenciales del origen.

3. Verifique el archivo sin restaurarlo:

```sql
RESTORE VERIFYONLY
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak'
WITH CHECKSUM;
GO
```

4. En PowerShell, en el equipo de origen, calcule el hash SHA-256 y guárdelo junto al respaldo y en la carpeta de reportes:

```powershell
$origenBak = "C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak"
$hash = Get-FileHash -Path $origenBak -Algorithm SHA256
$lineas = @(
    "Algorithm=$($hash.Algorithm)"
    "Hash=$($hash.Hash)"
    "Path=$($hash.Path)"
)
$lineas | Set-Content -Path "C:\LabSQL2025\Reports\06-00-01_HashOrigen_SHA256.txt" -Encoding ascii
$lineas | Set-Content -Path "C:\LabSQL2025\Backup\06-00-01_HashOrigen_SHA256.txt" -Encoding ascii
Get-Content "C:\LabSQL2025\Reports\06-00-01_HashOrigen_SHA256.txt"
```

5. Registre el tamaño del archivo y la hora de finalización del respaldo:

```powershell
Get-Item "C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak" |
    Select-Object Name, Length, CreationTime, LastWriteTime
```

**Resultado esperado**

- El respaldo se completa sin errores.
- `RESTORE VERIFYONLY` confirma que el conjunto de respaldo es válido.
- Se obtiene una huella SHA-256 del archivo de respaldo.

**Verificación**

La salida de `RESTORE VERIFYONLY` debe indicar:

```text
The backup set on file 1 is valid.
```

El archivo `06-00-01_HashOrigen_SHA256.txt` debe contener una línea `Hash=` seguida de 64 caracteres hexadecimales.

> **Nota:** `COPY_ONLY` evita alterar la secuencia normal de respaldos diferenciales. `INIT` deja un solo conjunto de respaldo dentro del archivo `.bak`, que es el que se restaura después.

---

### Paso 6: Transferir el respaldo y comprobar su integridad en el destino

**Objetivo:** transferir el respaldo al entorno destino sin alterar el archivo original y demostrar que el archivo recibido es idéntico al generado en origen.

**Instrucciones**

1. Si `SQL2017SRC` y `SQL2025TGT` están en el mismo equipo, el archivo ya está en `C:\LabSQL2025\Backup`. No ejecute `robocopy` sobre la misma carpeta. Continúe en el punto 3.

2. Si las instancias están en equipos distintos, ejecute este comando en el equipo de destino. Sustituya `SQL2017SRC` por el nombre de equipo anotado en el paso 1 cuando sea diferente:

```powershell
robocopy `
  "\\SQL2017SRC\LabBackup" `
  "C:\LabSQL2025\Backup" `
  "MigracionLab_Source_2025-09-22.bak" `
  "06-00-01_HashOrigen_SHA256.txt" `
  /Z /R:2 /W:5 /COPY:DAT

if ($LASTEXITCODE -ge 8) {
    throw "robocopy no pudo copiar el respaldo. Código $LASTEXITCODE."
}

Copy-Item `
  "C:\LabSQL2025\Backup\06-00-01_HashOrigen_SHA256.txt" `
  "C:\LabSQL2025\Reports\06-00-01_HashOrigen_SHA256.txt" `
  -Force
```

Un código de salida de `robocopy` entre 0 y 7 indica que la copia terminó. 8 o superior es un error.

3. En el equipo de destino, calcule el hash SHA-256 del archivo recibido:

```powershell
$destinoBak = "C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak"
$hash = Get-FileHash -Path $destinoBak -Algorithm SHA256
@(
    "Algorithm=$($hash.Algorithm)"
    "Hash=$($hash.Hash)"
    "Path=$($hash.Path)"
) | Set-Content -Path "C:\LabSQL2025\Reports\06-00-01_HashDestino_SHA256.txt" -Encoding ascii
```

4. Compare los dos archivos de hash. En el mismo equipo, el archivo de origen ya está en `C:\LabSQL2025\Reports`. En equipos distintos, el punto 2 lo copió allí.

```powershell
function Get-LabHashValue {
    param([string]$Path)
    $line = Get-Content -Path $Path | Where-Object { $_ -like "Hash=*" } | Select-Object -First 1
    if (-not $line) {
        throw "No se encontró la línea Hash= en $Path"
    }
    return $line.Substring(5).Trim()
}

$hashOrigen = Get-LabHashValue "C:\LabSQL2025\Reports\06-00-01_HashOrigen_SHA256.txt"
$hashDestino = Get-LabHashValue "C:\LabSQL2025\Reports\06-00-01_HashDestino_SHA256.txt"
$hashOrigen -eq $hashDestino
```

5. En `SQL2025TGT`, ejecute una segunda validación del respaldo:

```sql
RESTORE VERIFYONLY
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak'
WITH CHECKSUM;
GO
```

**Resultado esperado**

- El archivo de destino tiene el mismo hash SHA-256 que el archivo del origen.
- El archivo puede ser leído correctamente por SQL Server 2025.
- Se conserva el archivo original en el origen como evidencia y como elemento del plan de reversión.

**Verificación**

La comparación de hash debe devolver:

```text
True
```

No continúe si el hash difiere o si `RESTORE VERIFYONLY` devuelve errores. Elimine la copia incompleta del destino, repita la transferencia y vuelva a verificar.

---

### Paso 7: Restaurar la base de datos en SQL Server 2025

**Objetivo:** restaurar la base de datos con un nuevo nombre, manteniendo inicialmente el nivel de compatibilidad heredado.

**Instrucciones**

1. En `SQL2025TGT`, identifique los nombres lógicos de los archivos contenidos en el respaldo:

```sql
RESTORE FILELISTONLY
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak';
GO
```

2. Confirme que `LogicalName` muestra `MigracionLab_Source_Data` para el archivo de datos y `MigracionLab_Source_Log` para el archivo de log.

3. Compruebe que no existe previamente la base `MigracionLab_2025`:

```sql
SELECT
    name,
    state_desc,
    compatibility_level
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

4. Restaure la base. Los nombres lógicos son los definidos al crear `MigracionLab_Source`. Confírmelos con `RESTORE FILELISTONLY` antes de ejecutar este lote:

```sql
RESTORE DATABASE [MigracionLab_2025]
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak'
WITH
    MOVE N'MigracionLab_Source_Data'
        TO N'C:\SQLLab2025\Data\MigracionLab_2025.mdf',
    MOVE N'MigracionLab_Source_Log'
        TO N'C:\SQLLab2025\Logs\MigracionLab_2025_log.ldf',
    STATS = 10,
    RECOVERY;
GO
```

La base de ejemplo tiene un archivo de datos y un archivo de log. No agregue más cláusulas `MOVE`. Si `FILELISTONLY` muestra otro `LogicalName`, la restauración debe incluir un `MOVE` para ese nombre.

5. Consulte la configuración de la base restaurada:

```sql
SELECT
    name,
    state_desc,
    compatibility_level,
    recovery_model_desc,
    collation_name,
    page_verify_option_desc
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

6. Habilite Query Store en modo de lectura y escritura. La captura `ALL` registra las pocas consultas de este taller. No cambie aún el nivel de compatibilidad:

```sql
ALTER DATABASE [MigracionLab_2025]
SET QUERY_STORE = ON;
GO

ALTER DATABASE [MigracionLab_2025]
SET QUERY_STORE
(
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = ALL,
    DATA_FLUSH_INTERVAL_SECONDS = 60,
    INTERVAL_LENGTH_MINUTES = 1
);
GO
```

**Resultado esperado**

- La base de datos se restaura como `MigracionLab_2025`.
- El nivel de compatibilidad inicial es `140`, heredado de `MigracionLab_Source`.
- Query Store queda en `READ_WRITE` y el modo de captura es `ALL`.

**Verificación**

Ejecute:

```sql
USE [MigracionLab_2025];
GO

SELECT
    d.name,
    d.state_desc,
    d.compatibility_level,
    d.recovery_model_desc,
    q.actual_state_desc AS query_store_estado,
    q.query_capture_mode_desc AS modo_captura
FROM sys.databases AS d
CROSS JOIN sys.database_query_store_options AS q
WHERE d.name = N'MigracionLab_2025';
```

La base debe estar `ONLINE`, con compatibilidad `140`, Query Store en `READ_WRITE` y modo de captura `ALL`. `sys.database_query_store_options` describe solo la base de datos actual y no tiene columna `database_id`; por eso la consulta se ejecuta dentro de `MigracionLab_2025`.

---

### Paso 8: Validar integridad, datos, objetos y seguridad antes del cambio de compatibilidad

**Objetivo:** demostrar que la base restaurada es coherente y funcional antes de cambiar a nivel de compatibilidad 170.

**Instrucciones**

1. Ejecute una comprobación de integridad completa en la base restaurada:

```sql
DBCC CHECKDB (N'MigracionLab_2025')
WITH NO_INFOMSGS, ALL_ERRORMSGS;
GO
```

2. Valide restricciones y referencias declaradas:

```sql
USE [MigracionLab_2025];
GO

DBCC CHECKCONSTRAINTS WITH ALL_CONSTRAINTS;
GO
```

3. Obtenga los conteos de filas. En `SQL2017SRC`:

```sql
USE [MigracionLab_Source];
GO

SELECT
    SCHEMA_NAME(t.schema_id) AS esquema,
    t.name AS tabla,
    SUM(ps.row_count) AS filas
FROM sys.tables AS t
INNER JOIN sys.dm_db_partition_stats AS ps
    ON t.object_id = ps.object_id
WHERE t.is_ms_shipped = 0
  AND ps.index_id IN (0, 1)
GROUP BY
    t.schema_id,
    t.name
ORDER BY
    esquema,
    tabla;
```

En `SQL2025TGT`:

```sql
USE [MigracionLab_2025];
GO

SELECT
    SCHEMA_NAME(t.schema_id) AS esquema,
    t.name AS tabla,
    SUM(ps.row_count) AS filas
FROM sys.tables AS t
INNER JOIN sys.dm_db_partition_stats AS ps
    ON t.object_id = ps.object_id
WHERE t.is_ms_shipped = 0
  AND ps.index_id IN (0, 1)
GROUP BY
    t.schema_id,
    t.name
ORDER BY
    esquema,
    tabla;
```

4. Compare la definición de los objetos programables. En origen use `MigracionLab_Source` y en destino `MigracionLab_2025`:

```sql
USE [MigracionLab_2025];
GO

SELECT
    SCHEMA_NAME(o.schema_id) AS esquema,
    o.name AS objeto,
    o.type_desc AS tipo_objeto,
    CONVERT(varchar(64), HASHBYTES(
        'SHA2_256',
        CONVERT(varbinary(max), sm.definition)
    ), 2) AS hash_definicion
FROM sys.objects AS o
INNER JOIN sys.sql_modules AS sm
    ON o.object_id = sm.object_id
WHERE o.is_ms_shipped = 0
ORDER BY
    esquema,
    objeto;
```

5. Compruebe el propietario de la base restaurada:

```sql
SELECT
    name AS base_datos,
    SUSER_SNAME(owner_sid) AS propietario
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

6. Si el propietario no es el esperado para el entorno de laboratorio, asígnelo de manera controlada:

```sql
ALTER AUTHORIZATION ON DATABASE::[MigracionLab_2025] TO [sa];
GO
```

7. Detecte usuarios huérfanos. Un usuario SQL puede existir dentro de la base restaurada, pero su login asociado no se transfiere mediante el respaldo:

```sql
USE [MigracionLab_2025];
GO

SELECT
    dp.name AS usuario_huerfano,
    dp.type_desc AS tipo_usuario,
    dp.authentication_type_desc AS tipo_autenticacion
FROM sys.database_principals AS dp
LEFT JOIN sys.server_principals AS sp
    ON dp.sid = sp.sid
WHERE dp.type = 'S'
  AND dp.authentication_type = 1
  AND dp.name NOT IN ('dbo', 'guest', 'sys', 'INFORMATION_SCHEMA')
  AND sp.sid IS NULL
ORDER BY dp.name;
```

8. La consulta anterior debe devolver `lab_migracion`. En `SQL2025TGT`, cree el login de laboratorio y asócielo al usuario restaurado. Esta contraseña solo existe para el taller:

```sql
USE [master];
GO

IF NOT EXISTS
(
    SELECT 1
    FROM sys.server_principals
    WHERE name = N'lab_migracion'
)
BEGIN
    CREATE LOGIN [lab_migracion]
        WITH PASSWORD = N'LabMigracion2025!ChangeMe',
             CHECK_POLICY = ON;
END
GO

USE [MigracionLab_2025];
GO

ALTER USER [lab_migracion] WITH LOGIN = [lab_migracion];
GO
```

9. Vuelva a ejecutar la consulta de usuarios huérfanos. El resultado debe quedar vacío. Compruebe también que el trabajo de Agent del origen no llegó al destino:

```sql
USE [msdb];
GO

SELECT name
FROM dbo.sysjobs
WHERE name = N'MigracionLab_Inventario';
```

10. Active el plan de ejecución real en SSMS con **Ctrl+M** y ejecute las pruebas funcionales:

```sql
USE [MigracionLab_2025];
GO

SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

SELECT
    OrderId,
    CustomerCode,
    CustomerName,
    OrderDate,
    OrderStatus,
    TotalAmount
FROM Reporting.v_OrderSummary
ORDER BY OrderId;
GO

EXEC Reporting.usp_OrdersByCustomer
    @CustomerCode = 'C0001';
GO

SELECT
    OrderId,
    CustomerId,
    OrderDate,
    OrderStatus,
    TotalAmount
FROM Sales.Orders
ORDER BY OrderId;
GO

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
GO
```

**Resultado esperado**

- `DBCC CHECKDB` y `DBCC CHECKCONSTRAINTS` no informan errores.
- Los conteos coinciden: `Sales.Customers` = 5, `Sales.Orders` = 8 y `Sales.OrderLines` = 12.
- `Reporting.v_OrderSummary` y `Reporting.usp_OrdersByCustomer` tienen la misma huella en origen y destino.
- Antes de corregir el login, el usuario huérfano es `lab_migracion`. Después, la consulta no devuelve filas.
- `MigracionLab_Inventario` no existe en `msdb` de `SQL2025TGT`.
- `Reporting.v_OrderSummary` devuelve 8 filas. La primera es el pedido 1, cliente `C0001`, fecha `2024-06-15`, estado `C` e importe `175.00`.
- `Reporting.usp_OrdersByCustomer` con `C0001` devuelve 2 filas: pedidos 1 y 2, importes `175.00` y `80.00`.
- `Sales.Orders` devuelve 8 filas.

**Verificación**

Conserve como evidencia:

```text
06-00-01_Destino_CHECKDB.txt
06-00-01_Conteos_Origen.csv
06-00-01_Conteos_Destino.csv
06-00-01_ComparacionObjetos.csv
06-00-01_UsuariosHuerfanos.csv
06-00-01_PruebasFuncionales_Pre170.txt
```

Si existen diferencias de datos, definiciones, permisos o resultados funcionales, clasifique el estado como **Corrección requerida** y no eleve aún el nivel de compatibilidad.

---

### Paso 9: Elevar la compatibilidad a 170 y repetir pruebas representativas

**Objetivo:** cambiar de forma controlada el nivel de compatibilidad de la base restaurada a 170 y evaluar posibles cambios de comportamiento o rendimiento.

**Instrucciones**

1. Registre el nivel de compatibilidad actual:

```sql
SELECT
    name,
    compatibility_level
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

2. Revise las conexiones activas a la base. Programe el cambio durante una ventana de mantenimiento del laboratorio:

```sql
SELECT
    session_id,
    login_name,
    host_name,
    program_name,
    status
FROM sys.dm_exec_sessions
WHERE database_id = DB_ID(N'MigracionLab_2025')
  AND session_id <> @@SPID;
```

3. Cambie el nivel de compatibilidad:

```sql
ALTER DATABASE [MigracionLab_2025]
SET COMPATIBILITY_LEVEL = 170;
GO
```

4. Compruebe el cambio:

```sql
SELECT
    name,
    compatibility_level
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

5. Active el plan de ejecución real en SSMS con **Ctrl+M** y repita el mismo lote funcional. Conserve parámetros y orden:

```sql
USE [MigracionLab_2025];
GO

SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

SELECT
    OrderId,
    CustomerCode,
    CustomerName,
    OrderDate,
    OrderStatus,
    TotalAmount
FROM Reporting.v_OrderSummary
ORDER BY OrderId;
GO

EXEC Reporting.usp_OrdersByCustomer
    @CustomerCode = 'C0001';
GO

SELECT
    OrderId,
    CustomerId,
    OrderDate,
    OrderStatus,
    TotalAmount
FROM Sales.Orders
ORDER BY OrderId;
GO

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
GO
```

6. Compare con la ejecución previa al cambio:
   - Duración total.
   - Lecturas lógicas.
   - Uso de CPU informado por `STATISTICS TIME`.
   - Número de filas.
   - Operadores nuevos o cambios relevantes del plan.

7. Vuelque Query Store y revise las consultas capturadas:

```sql
USE [MigracionLab_2025];
GO

EXEC sys.sp_query_store_flush_db;
GO

SELECT TOP (20)
    q.query_id,
    p.plan_id,
    rs.count_executions,
    CAST(rs.avg_duration / 1000.0 AS decimal(18,2)) AS duracion_promedio_ms,
    CAST(rs.avg_cpu_time / 1000.0 AS decimal(18,2)) AS cpu_promedio_ms,
    rs.avg_logical_io_reads,
    qt.query_sql_text
FROM sys.query_store_query AS q
INNER JOIN sys.query_store_query_text AS qt
    ON q.query_text_id = qt.query_text_id
INNER JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
INNER JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
ORDER BY rs.avg_duration DESC;
```

8. Revise los planes de la vista y del procedimiento de este taller:

```sql
USE [MigracionLab_2025];
GO

SELECT
    p.query_id,
    p.plan_id,
    p.is_forced_plan,
    p.last_execution_time,
    qt.query_sql_text
FROM sys.query_store_plan AS p
INNER JOIN sys.query_store_query AS q
    ON q.query_id = p.query_id
INNER JOIN sys.query_store_query_text AS qt
    ON qt.query_text_id = q.query_text_id
WHERE qt.query_sql_text LIKE N'%v_OrderSummary%'
   OR qt.query_sql_text LIKE N'%usp_OrdersByCustomer%'
ORDER BY p.last_execution_time DESC;
```

9. Documente los resultados en una tabla comparativa.

| Consulta o módulo | Resultado pre-170 | Resultado post-170 | Duración pre-170 | Duración post-170 | Plan cambió | Decisión |
|---|---|---|---:|---:|---|---|
| `Reporting.v_OrderSummary` | 8 filas | 8 filas | Registrar | Registrar | Sí/No | Aceptar/Investigar |
| `Reporting.usp_OrdersByCustomer` con `C0001` | 2 filas | 2 filas | Registrar | Registrar | Sí/No | Aceptar/Investigar |

**Resultado esperado**

- `MigracionLab_2025` queda en nivel de compatibilidad `170`.
- La vista sigue devolviendo 8 filas y el procedimiento sigue devolviendo 2 filas para `C0001`.
- Query Store muestra al menos un plan de `v_OrderSummary` o de `usp_OrdersByCustomer`. Si esa consulta no devuelve filas, ejecute otra vez el lote funcional y vuelva a lanzar `sys.sp_query_store_flush_db`.
- Cualquier regresión de rendimiento o cambio funcional queda documentado.

**Verificación**

Ejecute:

```sql
USE [MigracionLab_2025];
GO

SELECT
    d.name,
    d.compatibility_level,
    q.actual_state_desc AS query_store_estado,
    q.desired_state_desc AS query_store_estado_deseado,
    q.query_capture_mode_desc AS modo_captura
FROM sys.databases AS d
CROSS JOIN sys.database_query_store_options AS q
WHERE d.name = N'MigracionLab_2025';
```

El resultado debe indicar compatibilidad `170`, Query Store en `READ_WRITE` y modo de captura `ALL`.

---

### Paso 10: Elaborar la decisión técnica y el plan de reversión

**Objetivo:** consolidar la evidencia recolectada y decidir si la simulación de migración es aceptable, requiere corrección o debe revertirse.

**Instrucciones**

1. Cree el archivo:

```text
C:\LabSQL2025\Reports\06-00-01_InformeDecisionMigracion.md
```

2. Incluya los siguientes apartados:
   - Identificación de origen y destino.
   - Fecha y hora de respaldo, transferencia y restauración.
   - Hash SHA-256 en origen y destino.
   - Resultado de DMA.
   - Resultado de `RESTORE VERIFYONLY`.
   - Resultado de `DBCC CHECKDB`.
   - Comparación de conteos por tabla.
   - Comparación de objetos programables.
   - Estado de usuarios, logins, roles y permisos.
   - Resultado de pruebas funcionales.
   - Resultado de pruebas previas y posteriores a compatibilidad 170.
   - Riesgos y acciones de mitigación.
   - Decisión final.

3. Registre riesgos comunes de migración y su tratamiento:

| Riesgo | Evidencia o señal | Mitigación |
|---|---|---|
| Usuarios huérfanos | Usuario de base sin login asociado | Crear o asociar login autorizado con `ALTER USER ... WITH LOGIN` |
| Trabajos no migrados | Trabajos presentes en `msdb` de origen | Script y recreación controlada de SQL Server Agent Jobs |
| Dependencias externas | Linked servers, referencias entre bases o servidores | Validar red, nombres, credenciales y permisos |
| Cambio de planes | Diferencias en plan o duración tras compatibilidad 170 | Analizar Query Store, estadísticas, índices y consultas |
| Archivo transferido incorrectamente | Hash SHA-256 diferente | Repetir transferencia y verificar nuevamente |
| Problema de integridad | Errores de `DBCC CHECKDB` | Detener aceptación, investigar y restaurar una copia válida |

4. Use los siguientes criterios de aceptación:
   - DMA no contiene bloqueadores sin resolver o aceptar formalmente.
   - El hash del respaldo es idéntico en origen y destino.
   - `RESTORE VERIFYONLY` y `DBCC CHECKDB` finalizan correctamente.
   - Los conteos de datos y objetos coinciden o las diferencias están justificadas.
   - No existen usuarios huérfanos sin tratamiento.
   - Las pruebas funcionales críticas son correctas.
   - Las consultas representativas no presentan regresiones inaceptables.
   - El plan de reversión está documentado y es viable.

5. Registre una de las decisiones siguientes:
   - **Aceptación:** todos los criterios esenciales se cumplen.
   - **Corrección requerida:** existen hallazgos reparables antes de una migración real.
   - **Reversión:** se detectó un problema crítico de integridad, funcionalidad, seguridad o rendimiento.

6. Documente el plan de reversión:
   - Mantener `MigracionLab_Source` como base original y sin modificaciones.
   - Conservar el archivo `.bak` original y su hash.
   - No redirigir aplicaciones de producción al destino hasta aprobar los criterios de aceptación.
   - Si se hubiese realizado un cambio de aplicación, redirigir la conexión nuevamente a la instancia origen.
   - Investigar el hallazgo en una nueva copia restaurada, sin modificar la evidencia original.

**Resultado esperado**

- Se genera una decisión técnica basada en evidencias.
- Los riesgos, acciones y responsables quedan documentados.
- Existe un plan claro de retorno al origen.

**Verificación**

El informe debe concluir con una declaración similar a una de las siguientes:

```text
Decisión: Aceptación condicionada.
Justificación: la integridad, conteos, objetos y pruebas funcionales fueron correctos.
Acción pendiente: recrear y validar los trabajos de SQL Server Agent antes de una migración productiva.
```

```text
Decisión: Corrección requerida.
Justificación: se detectaron usuarios huérfanos y una regresión de duración en una consulta crítica tras elevar la compatibilidad a 170.
Acción: corregir asociación de logins y analizar el plan en Query Store antes de aceptar.
```

---

### Paso 11: Crear el respaldo completo y el diferencial en SQL Server 2025

**Objetivo:** iniciar en `SQL2025TGT` una cadena de recuperación de `MigracionLab_2025`, distinta del respaldo `COPY_ONLY` usado para migrar.

Ejecute este paso y los dos siguientes en `SQL2025TGT`, con la base en línea. El archivo `MigracionLab_Source_2025-09-22.bak` no forma parte de esta cadena. Si repite el paso, empiece de nuevo por el respaldo completo: un full nuevo deja inservibles el diferencial y los logs anteriores.

**Instrucciones**

1. Confirme que `MigracionLab_2025` está en línea, en recuperación `FULL` y en compatibilidad `170`:

```sql
SELECT
    name,
    state_desc,
    recovery_model_desc,
    compatibility_level
FROM sys.databases
WHERE name = N'MigracionLab_2025';
GO
```

2. Si el modelo de recuperación no es `FULL`, cámbielo antes del respaldo completo. El cambio solo protege el log a partir de ese full:

```sql
IF EXISTS
(
    SELECT 1
    FROM sys.databases
    WHERE name = N'MigracionLab_2025'
      AND recovery_model_desc <> N'FULL'
)
    ALTER DATABASE [MigracionLab_2025] SET RECOVERY FULL;
GO
```

3. Deje la base con los ocho pedidos originales. Así cada respaldo posterior captura un pedido nuevo y comprobable:

```sql
USE [MigracionLab_2025];
GO

DELETE FROM Sales.Orders
WHERE OrderId IN (9, 10, 11);
GO
```

4. Tome el respaldo completo. No use `COPY_ONLY`:

```sql
BACKUP DATABASE [MigracionLab_2025]
TO DISK = N'C:\LabSQL2025\Backup\MigracionLab_2025_FULL.bak'
WITH
    INIT,
    CHECKSUM,
    COMPRESSION,
    STATS = 10,
    NAME = N'MigracionLab_2025 - Full';
GO
```

5. Agregue el pedido que debe quedar solo en el diferencial y en los respaldos de log posteriores:

```sql
USE [MigracionLab_2025];
GO

IF NOT EXISTS (SELECT 1 FROM Sales.Orders WHERE OrderId = 9)
    INSERT INTO Sales.Orders (OrderId, CustomerId, OrderDate, OrderStatus, TotalAmount)
    VALUES (9, 1, '20241101', 'C', 40.00);
GO
```

6. Tome el respaldo diferencial:

```sql
BACKUP DATABASE [MigracionLab_2025]
TO DISK = N'C:\LabSQL2025\Backup\MigracionLab_2025_DIFF.bak'
WITH
    DIFFERENTIAL,
    INIT,
    CHECKSUM,
    COMPRESSION,
    STATS = 10,
    NAME = N'MigracionLab_2025 - Diferencial';
GO
```

7. Registre el tipo de cada respaldo en `msdb`. `D` es completo e `I` es diferencial:

```sql
SELECT
    database_name,
    type,
    name,
    backup_start_date,
    backup_finish_date
FROM msdb.dbo.backupset
WHERE database_name = N'MigracionLab_2025'
  AND name IN (N'MigracionLab_2025 - Full', N'MigracionLab_2025 - Diferencial')
ORDER BY backup_start_date;
GO
```

**Resultado esperado**

- Los dos archivos existen en `C:\LabSQL2025\Backup`.
- `msdb` muestra una fila `D` y una fila `I`.
- `Sales.Orders` contiene el pedido 9.
- `MigracionLab_2025` sigue `ONLINE`.

**Verificación**

```sql
USE [MigracionLab_2025];
GO

SELECT OrderId, CustomerId, OrderDate, OrderStatus, TotalAmount
FROM Sales.Orders
WHERE OrderId = 9;
GO
```

La consulta debe devolver una fila: pedido 9, cliente 1, fecha `2024-11-01`, estado `C`, importe `40.00`.

---

### Paso 12: Crear el respaldo de log y el tail-log, y restaurar la cadena

**Objetivo:** cerrar la cadena con un log convencional y un tail-log, y devolver `MigracionLab_2025` a línea con los tres pedidos agregados.

No se detenga después del tail-log. Ese respaldo usa `NORECOVERY` y deja la base en estado `RESTORING` hasta el final de este paso.

**Instrucciones**

1. Agregue el pedido que debe quedar en el log convencional, no en el diferencial:

```sql
USE [MigracionLab_2025];
GO

IF NOT EXISTS (SELECT 1 FROM Sales.Orders WHERE OrderId = 10)
    INSERT INTO Sales.Orders (OrderId, CustomerId, OrderDate, OrderStatus, TotalAmount)
    VALUES (10, 1, '20241115', 'C', 55.00);
GO
```

2. Respalde el log de transacciones:

```sql
BACKUP LOG [MigracionLab_2025]
TO DISK = N'C:\LabSQL2025\Backup\MigracionLab_2025_LOG.trn'
WITH
    INIT,
    CHECKSUM,
    COMPRESSION,
    STATS = 10,
    NAME = N'MigracionLab_2025 - Log';
GO
```

3. Agregue el pedido que solo puede recuperar el tail-log:

```sql
USE [MigracionLab_2025];
GO

IF NOT EXISTS (SELECT 1 FROM Sales.Orders WHERE OrderId = 11)
    INSERT INTO Sales.Orders (OrderId, CustomerId, OrderDate, OrderStatus, TotalAmount)
    VALUES (11, 1, '20241201', 'C', 70.00);
GO
```

4. Compruebe los nombres lógicos del full. Deben ser `MigracionLab_Source_Data` y `MigracionLab_Source_Log`. Si `RESTORE FILELISTONLY` muestra otros, use esos valores en el `MOVE` del punto 6:

```sql
RESTORE FILELISTONLY
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_2025_FULL.bak';
GO
```

5. Tome el tail-log desde `master`. `NORECOVERY` corta la actividad y deja la base lista para restaurar:

```sql
USE [master];
GO

ALTER DATABASE [MigracionLab_2025]
SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

BACKUP LOG [MigracionLab_2025]
TO DISK = N'C:\LabSQL2025\Backup\MigracionLab_2025_TAIL.trn'
WITH
    INIT,
    NORECOVERY,
    CHECKSUM,
    COMPRESSION,
    STATS = 10,
    NAME = N'MigracionLab_2025 - Tail-log';
GO
```

6. Restaure la cadena sobre la misma base, en este orden: full, diferencial, log y tail-log. El último respaldo es el único que usa `RECOVERY`:

```sql
RESTORE DATABASE [MigracionLab_2025]
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_2025_FULL.bak'
WITH
    MOVE N'MigracionLab_Source_Data'
        TO N'C:\SQLLab2025\Data\MigracionLab_2025.mdf',
    MOVE N'MigracionLab_Source_Log'
        TO N'C:\SQLLab2025\Logs\MigracionLab_2025_log.ldf',
    NORECOVERY,
    REPLACE,
    STATS = 10;
GO

RESTORE DATABASE [MigracionLab_2025]
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_2025_DIFF.bak'
WITH NORECOVERY, STATS = 10;
GO

RESTORE LOG [MigracionLab_2025]
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_2025_LOG.trn'
WITH NORECOVERY, STATS = 10;
GO

RESTORE LOG [MigracionLab_2025]
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_2025_TAIL.trn'
WITH RECOVERY, STATS = 10;
GO
```

7. Confirme el estado y los tres pedidos:

```sql
SELECT
    name,
    state_desc,
    recovery_model_desc,
    compatibility_level
FROM sys.databases
WHERE name = N'MigracionLab_2025';
GO

USE [MigracionLab_2025];
GO

SELECT OrderId, OrderDate, OrderStatus, TotalAmount
FROM Sales.Orders
WHERE OrderId IN (9, 10, 11)
ORDER BY OrderId;
GO
```

**Resultado esperado**

- Existen `MigracionLab_2025_LOG.trn` y `MigracionLab_2025_TAIL.trn`.
- Durante el tail-log la base pasa a `RESTORING`.
- Al terminar la restauración vuelve a `ONLINE`, en recuperación `FULL` y compatibilidad `170`.
- Los pedidos 9, 10 y 11 están presentes. `Sales.Orders` queda con 11 filas.

**Verificación**

```sql
SELECT
    type,
    name,
    backup_start_date
FROM msdb.dbo.backupset
WHERE database_name = N'MigracionLab_2025'
  AND name LIKE N'MigracionLab_2025 - %'
ORDER BY backup_start_date;
GO

USE [MigracionLab_2025];
GO

SELECT COUNT(*) AS pedidos
FROM Sales.Orders;
GO
```

`msdb` debe mostrar `D`, `I` y dos filas `L`. El conteo de pedidos debe ser `11`.

---

### Paso 13: Crear un trabajo de SQL Server Agent sobre la base migrada

**Objetivo:** programar en `SQL2025TGT` un trabajo que registre el conteo de `Sales.Orders` en `MigracionLab_2025`. Este trabajo se crea en el destino; no viajó con el respaldo de SQL Server 2017.

**Instrucciones**

1. En el equipo de `SQL2025TGT`, identifique el servicio del Agent e inícielo si está detenido. Use `SQLSERVERAGENT` cuando `SQL2025TGT` sea la instancia predeterminada, o `SQLAgent$SQL2025TGT` cuando sea una instancia con nombre:

```powershell
Get-Service -Name 'SQLSERVERAGENT','SQLAgent$SQL2025TGT' -ErrorAction SilentlyContinue |
    Format-Table Name, Status, StartType -AutoSize

$agent = Get-Service -Name 'SQLAgent$SQL2025TGT' -ErrorAction SilentlyContinue
if (-not $agent) {
    $agent = Get-Service -Name 'SQLSERVERAGENT' -ErrorAction SilentlyContinue
}
if (-not $agent) {
    Write-Warning "No se encontró el servicio del Agent. Inicie el de la instancia SQL2025TGT."
}
else {
    if ($agent.Status -ne 'Running') {
        Start-Service -Name $agent.Name
    }
    Get-Service -Name $agent.Name
}
```

2. Compruebe el servicio desde la instancia:

```sql
SELECT
    servicename,
    status_desc,
    startup_type_desc,
    service_account
FROM sys.dm_server_services
WHERE servicename LIKE N'SQL Server Agent%';
GO
```

3. Cree la tabla donde el trabajo deja evidencia:

```sql
USE [MigracionLab_2025];
GO

IF OBJECT_ID(N'dbo.Lab06_AgentRun', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.Lab06_AgentRun
    (
        AgentRunId int IDENTITY(1,1) NOT NULL
            CONSTRAINT PK_Lab06_AgentRun PRIMARY KEY,
        CapturedAt datetime2(0) NOT NULL
            CONSTRAINT DF_Lab06_AgentRun_CapturedAt DEFAULT SYSDATETIME(),
        OrderCount int NOT NULL,
        Note nvarchar(80) NOT NULL
    );
END
GO
```

4. Cree el operador, el trabajo, el paso y el horario diario de las 23:00. La notificación por correo queda solo para fallos. Sin Database Mail no sale el mensaje, y el historial del trabajo sí se registra:

```sql
USE [msdb];
GO

IF NOT EXISTS
(
    SELECT 1
    FROM dbo.sysoperators
    WHERE name = N'OperadorLaboratorio'
)
    EXEC dbo.sp_add_operator
        @name = N'OperadorLaboratorio',
        @enabled = 1,
        @email_address = N'lab-sql@localhost';
GO

IF NOT EXISTS
(
    SELECT 1
    FROM dbo.sysjobs
    WHERE name = N'MigracionLab_2025_Resumen'
)
BEGIN
    EXEC dbo.sp_add_job
        @job_name = N'MigracionLab_2025_Resumen',
        @enabled = 1,
        @description = N'Resumen de pedidos de MigracionLab_2025. Creado en SQL2025TGT; no proviene del respaldo de 2017.',
        @notify_level_email = 2,
        @notify_email_operator_name = N'OperadorLaboratorio';

    EXEC dbo.sp_add_jobstep
        @job_name = N'MigracionLab_2025_Resumen',
        @step_name = N'Registrar conteo de pedidos',
        @subsystem = N'TSQL',
        @database_name = N'MigracionLab_2025',
        @command = N'INSERT INTO dbo.Lab06_AgentRun (OrderCount, Note)
SELECT COUNT(*), N''Ejecucion del Agent en SQL2025TGT''
FROM Sales.Orders;';

    EXEC dbo.sp_add_schedule
        @schedule_name = N'MigracionLab_2025_Diario_2300',
        @freq_type = 4,
        @freq_interval = 1,
        @active_start_time = 230000;

    EXEC dbo.sp_attach_schedule
        @job_name = N'MigracionLab_2025_Resumen',
        @schedule_name = N'MigracionLab_2025_Diario_2300';

    EXEC dbo.sp_add_jobserver
        @job_name = N'MigracionLab_2025_Resumen';
END
GO
```

5. Ejecute el trabajo una vez y espere a que termine:

```sql
USE [msdb];
GO

EXEC dbo.sp_start_job
    @job_name = N'MigracionLab_2025_Resumen';
GO

WAITFOR DELAY '00:00:10';
GO
```

6. Revise el horario, el historial y la fila escrita en la base migrada. En el historial, `step_id = 0` es el resultado del trabajo y `run_status = 1` es éxito:

```sql
SELECT
    j.name AS trabajo,
    j.enabled,
    s.name AS horario,
    s.active_start_time
FROM msdb.dbo.sysjobs AS j
INNER JOIN msdb.dbo.sysjobschedules AS js
    ON js.job_id = j.job_id
INNER JOIN msdb.dbo.sysschedules AS s
    ON s.schedule_id = js.schedule_id
WHERE j.name = N'MigracionLab_2025_Resumen';
GO

SELECT TOP (5)
    j.name AS trabajo,
    h.step_id,
    h.run_status,
    h.run_date,
    h.run_time,
    h.message
FROM msdb.dbo.sysjobs AS j
INNER JOIN msdb.dbo.sysjobhistory AS h
    ON h.job_id = j.job_id
WHERE j.name = N'MigracionLab_2025_Resumen'
ORDER BY h.instance_id DESC;
GO

USE [MigracionLab_2025];
GO

SELECT
    AgentRunId,
    CapturedAt,
    OrderCount,
    Note
FROM dbo.Lab06_AgentRun
ORDER BY AgentRunId;
GO
```

**Resultado esperado**

- El servicio de SQL Server Agent de `SQL2025TGT` está en ejecución.
- Existe el trabajo habilitado `MigracionLab_2025_Resumen`, con un paso sobre `MigracionLab_2025` y horario diario a las 23:00.
- El historial muestra `run_status = 1` en el paso `0`.
- `dbo.Lab06_AgentRun` tiene al menos una fila con `OrderCount = 11`.
- `MigracionLab_Inventario` sigue sin existir en este `msdb`. Ese trabajo permaneció en el origen.

**Verificación**

```sql
USE [msdb];
GO

SELECT name
FROM dbo.sysjobs
WHERE name IN (N'MigracionLab_2025_Resumen', N'MigracionLab_Inventario');
GO
```

La consulta debe devolver solo `MigracionLab_2025_Resumen`.

---

## Validación y Pruebas

Utilice la siguiente lista de comprobación final:

| Validación | Comando o evidencia | Criterio de aprobación |
|---|---|---|
| Base de ejemplo | `MigracionLab_Source` en `SQL2017SRC` | Compatibilidad 140; conteos 5, 8 y 12; vista y procedimiento creados |
| Inventario de origen | Informes de consultas de `sys.databases`, objetos, usuarios y dependencias | Archivos exportados y revisados |
| Evaluación DMA | Informe HTML o JSON | Bloqueadores resueltos, aceptados o documentados |
| Respaldo | `BACKUP DATABASE ... WITH CHECKSUM` | Finaliza sin error |
| Verificación del respaldo | `RESTORE VERIFYONLY ... WITH CHECKSUM` | Respaldo válido en origen y destino |
| Integridad de transferencia | `Get-FileHash -Algorithm SHA256` | Hashes idénticos |
| Restauración | `RESTORE DATABASE ... WITH MOVE` | Base `ONLINE` en destino |
| Integridad lógica y física | `DBCC CHECKDB` | Sin errores |
| Integridad referencial | `DBCC CHECKCONSTRAINTS` | Sin errores de restricciones |
| Datos de la migración | Conteos del paso 8, antes de la cadena de respaldos | Coinciden con el origen: 5, 8 y 12 |
| Cadena de respaldos | Full, diferencial, log y tail-log restaurados en `SQL2025TGT` | `MigracionLab_2025` queda `ONLINE` y `Sales.Orders` tiene 11 filas |
| SQL Server Agent | Trabajo `MigracionLab_2025_Resumen` | Historial con `run_status = 1` y una fila en `dbo.Lab06_AgentRun` con `OrderCount = 11` |
| Objetos | Hash SHA-256 de módulos | Coinciden con origen o hay justificación |
| Seguridad | Consulta de usuarios huérfanos | Sin usuarios sin tratamiento |
| Compatibilidad | `compatibility_level = 170` | Cambio registrado |
| Query Store | `actual_state_desc = READ_WRITE` y captura `ALL` | Consultas del taller registradas |
| Rendimiento funcional | Estadísticas, planes y Query Store | Sin regresiones no aceptadas |
| Decisión | Informe final | Aceptación, corrección o reversión documentada |

Ejecute esta consulta final en `SQL2025TGT` después del paso 13, con `MigracionLab_2025` en línea:

```sql
USE [MigracionLab_2025];
GO

SELECT
    d.name AS base_datos,
    d.state_desc AS estado,
    d.compatibility_level AS nivel_compatibilidad,
    d.recovery_model_desc AS modelo_recuperacion,
    q.actual_state_desc AS query_store,
    q.query_capture_mode_desc AS modo_captura
FROM sys.databases AS d
CROSS JOIN sys.database_query_store_options AS q
WHERE d.name = N'MigracionLab_2025';
```

El resultado esperado es:

- Base de datos: `MigracionLab_2025`.
- Estado: `ONLINE`.
- Nivel de compatibilidad: `170`.
- Query Store: `READ_WRITE`.
- Modo de captura: `ALL`.

## Solución de Problemas

### Problema 1: La restauración falla porque no se pueden crear los archivos en la ruta de destino

**Síntomas**

- Error similar a `Operating system error 3` o `Access is denied`.
- Error que indica que el archivo físico no puede crearse.
- La restauración falla al ejecutar `RESTORE DATABASE ... WITH MOVE`.

**Causa**

La carpeta de destino no existe, o la cuenta de servicio de SQL Server no tiene permisos de escritura. El mismo error aparece al crear `MigracionLab_Source` si falta permiso sobre `C:\LabSQL2025\Data`, `C:\LabSQL2025\Logs` o `C:\LabSQL2025\Backup`. También ocurre si un nombre de `MOVE` no coincide con `RESTORE FILELISTONLY`.

**Corrección**

1. Ejecute `RESTORE FILELISTONLY` y copie exactamente los valores de `LogicalName`.
2. Cree las carpetas requeridas en el servidor destino.
3. Compruebe la cuenta de servicio con `sys.dm_server_services`.
4. Conceda permisos de modificación a la cuenta de servicio del origen sobre `C:\LabSQL2025\Data`, `C:\LabSQL2025\Logs` y `C:\LabSQL2025\Backup`.
5. Conceda permisos de modificación a la cuenta de servicio del destino sobre `C:\SQLLab2025\Data`, `C:\SQLLab2025\Logs` y `C:\LabSQL2025\Backup`.
6. Ejecute nuevamente la creación de la base o la restauración. En `RESTORE`, incluya una cláusula `MOVE` por cada archivo del respaldo.

### Problema 2: Se detectan usuarios huérfanos después de la restauración

**Síntomas**

- La consulta de usuarios huérfanos devuelve filas.
- Un usuario puede existir en `MigracionLab_2025`, pero no puede iniciar sesión o acceder a la base.
- Una aplicación recibe errores de autenticación o de usuario no asociado a un login.

**Causa**

Los usuarios de base de datos se incluyen en el respaldo, pero los logins son objetos de instancia y no se restauran automáticamente desde SQL Server 2017 al servidor SQL Server 2025.

**Corrección**

1. Identifique el login equivalente autorizado en `SQL2025TGT`.
2. Cree o migre el login mediante el procedimiento de seguridad aprobado, preservando el SID cuando sea necesario.
3. Asocie el usuario al login:

```sql
USE [master];
GO

IF NOT EXISTS
(
    SELECT 1
    FROM sys.server_principals
    WHERE name = N'lab_migracion'
)
BEGIN
    CREATE LOGIN [lab_migracion]
        WITH PASSWORD = N'LabMigracion2025!ChangeMe',
             CHECK_POLICY = ON;
END
GO

USE [MigracionLab_2025];
GO

ALTER USER [lab_migracion] WITH LOGIN = [lab_migracion];
GO
```

4. Vuelva a ejecutar la consulta de detección de usuarios huérfanos.
5. Pruebe los permisos con una cuenta de prueba autorizada; no otorgue `sysadmin` como solución rápida.

### Problema 3: La copia del respaldo no llega al destino o los hashes no se pueden comparar

**Síntomas**

- `robocopy` no encuentra `\\SQL2017SRC\LabBackup`.
- El código de salida de `robocopy` es 8 o superior.
- La comparación de hash termina con el mensaje de que no existe la línea `Hash=`.

**Causa**

El nombre del equipo de origen no es `SQL2017SRC`, el recurso `LabBackup` no se creó, o el archivo de hash se guardó con otro formato. Si las dos instancias están en el mismo equipo, no hace falta `robocopy`.

**Corrección**

1. Use el nombre de equipo anotado en el paso 1 dentro de la ruta UNC.
2. En el equipo de origen, compruebe `Get-SmbShare -Name LabBackup`.
3. Vuelva a ejecutar el cálculo de hash del paso 5. La línea debe empezar por `Hash=`.
4. En el mismo equipo, omita `robocopy` y compare los dos archivos de `C:\LabSQL2025\Reports`.

### Problema 4: La base queda en RESTORING o el diferencial no se puede crear

**Síntomas**

- `MigracionLab_2025` aparece como `RESTORING` y no acepta consultas.
- El respaldo diferencial informa que no existe un respaldo completo actual.
- La restauración se detiene en el diferencial o en un log.

**Causa**

El tail-log se tomó con `NORECOVERY` y todavía no se restauró la cadena. Un diferencial también falla si el único full existente es el `COPY_ONLY` de la migración, o si se tomó un full nuevo después del diferencial.

**Corrección**

1. Confirme que los cuatro archivos de la cadena están en `C:\LabSQL2025\Backup`: `MigracionLab_2025_FULL.bak`, `MigracionLab_2025_DIFF.bak`, `MigracionLab_2025_LOG.trn` y `MigracionLab_2025_TAIL.trn`.
2. Ejecute de nuevo, en ese orden, los cuatro `RESTORE` del paso 12. El tail-log es el único que lleva `RECOVERY`.
3. Si falta alguno de esos archivos, borre los que hayan quedado y repita los pasos 11 y 12 desde el respaldo completo.

### Problema 5: El trabajo de Agent no se ejecuta

**Síntomas**

- `sp_start_job` informa que SQL Server Agent no está en ejecución.
- El historial no muestra filas o `run_status` es distinto de `1`.
- `dbo.Lab06_AgentRun` no recibe filas.

**Causa**

El servicio del Agent de `SQL2025TGT` está detenido, el paso apunta a otra base o la tabla `dbo.Lab06_AgentRun` no existe.

**Corrección**

1. Inicie el servicio indicado en el paso 13 y compruebe `status_desc = Running` en `sys.dm_server_services`.
2. Confirme que el paso del trabajo usa la base `MigracionLab_2025`.
3. Cree la tabla si falta y vuelva a ejecutar `sp_start_job`.
4. Lea `message` en `msdb.dbo.sysjobhistory` para el paso `1`.

## Limpieza

1. No elimine `MigracionLab_Source`; es la referencia de origen y forma parte del plan de reversión.
2. No elimine el respaldo original, los cuatro archivos de la cadena de `MigracionLab_2025` ni los archivos de hash mientras la práctica esté en revisión.
3. Mantenga `MigracionLab_2025` y el trabajo `MigracionLab_2025_Resumen` para análisis posterior, salvo que el instructor solicite su eliminación. El trabajo vive en `msdb` de `SQL2025TGT`; eliminar la base no lo borra.
4. Si se requiere retirar la base restaurada al finalizar el laboratorio, confirme primero que los informes estén guardados y ejecute en `SQL2025TGT`:

```sql
USE [master];
GO

ALTER DATABASE [MigracionLab_2025]
SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

DROP DATABASE [MigracionLab_2025];
GO
```

5. Si se elimina la base, conserve como mínimo:
   - El informe DMA.
   - Los hashes SHA-256.
   - El informe de decisión.
   - Los resultados de `DBCC CHECKDB`.
   - Las comparaciones de conteos, objetos y seguridad.
6. Elimine copias temporales incompletas del respaldo únicamente después de comprobar que la copia validada y su evidencia permanecen disponibles.

## Resumen

En esta práctica se creó `MigracionLab_Source` en SQL Server 2017 y se simuló su migración hacia SQL Server 2025. Se inventarió el entorno de origen, se evaluó la compatibilidad con DMA, se creó un respaldo verificable con `CHECKSUM` y SHA-256, se restauró la base en un destino aislado y se validaron integridad, datos, objetos, permisos y funcionamiento. Sobre `MigracionLab_2025` se construyó después una cadena full, diferencial, de log y tail-log, se restauró hasta dejar la base en línea con 11 pedidos y se ejecutó el trabajo `MigracionLab_2025_Resumen` en el Agent del destino.

La elevación del nivel de compatibilidad a 170 se realizó únicamente después de una validación inicial satisfactoria. Query Store, estadísticas de ejecución y planes reales permitieron comparar el comportamiento funcional y de rendimiento antes y después del cambio. La migración no debe considerarse aceptada solo porque la restauración finalizó correctamente: la decisión técnica debe basarse en evidencia de integridad, seguridad, dependencias, funcionalidad, rendimiento y reversibilidad.

**Recursos recomendados**

- [Documentación de Microsoft sobre actualización de SQL Server](https://learn.microsoft.com/sql/database-engine/install-windows/upgrade-sql-server)
- [Documentación de BACKUP DATABASE](https://learn.microsoft.com/sql/t-sql/statements/backup-transact-sql)
- [Documentación de RESTORE VERIFYONLY](https://learn.microsoft.com/sql/t-sql/statements/restore-statements-verifyonly-transact-sql)
- [Documentación de DBCC CHECKDB](https://learn.microsoft.com/sql/t-sql/database-console-commands/dbcc-checkdb-transact-sql)
- [Documentación de Query Store](https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
- [Documentación de Data Migration Assistant](https://learn.microsoft.com/sql/dma/dma-overview)
