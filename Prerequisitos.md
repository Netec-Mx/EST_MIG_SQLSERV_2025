# Prerrequisitos de la máquina virtual

Este documento es para quien prepara la imagen del curso. No es un laboratorio: los estudiantes no lo ejecutan. Con la máquina lista, los capítulos 1 a 6 pueden recorrerse en `SQLLAB-WS2022` sin instalar componentes adicionales.

Los estudiantes crean durante las prácticas las bases `Sql2025Lab`, `MigracionLab_Source` y `MigracionLab_2025`, sus objetos, los logins de cada taller y los respaldos. Esta preparación deja el sistema operativo, las instancias, las herramientas, las carpetas y la cuenta de acceso.

## Hardware

Una sola máquina virtual aloja las tres instancias.

| Recurso | Mínimo | Recomendado |
|---|---|---|
| Procesadores virtuales | 8 vCPU | 12 vCPU |
| Memoria RAM | 24 GB | 32 GB |
| Disco libre | 120 GB en SSD o NVMe | 200 GB |
| Red | Acceso local a TCP 1433, 1435 y 1436; salida HTTPS por TCP 443 | La misma |

`max server memory` de la instancia predeterminada queda en 16384 MB dentro del capítulo 1. Por eso la máquina necesita más memoria que esa cifra, para Windows, SSMS, Visual Studio Code y las otras dos instancias.

## Sistema operativo y cuenta de acceso

| Elemento | Valor |
|---|---|
| Nombre del equipo | `SQLLAB-WS2022` |
| Sistema operativo | Windows Server 2022 Datacenter, versión 21H2 |
| Compilación | `20348.2402` |
| Cuenta de preparación y de los laboratorios | `.\SqlLabAdmin` |
| Pertenencia | Grupo local `Administrators` |
| PowerShell | Windows PowerShell 5.1, incluido en el sistema. PowerShell 7 puede coexistir |

La contraseña de `.\SqlLabAdmin` se entrega con la imagen. No se documenta en el repositorio.

Inicie sesión con esa cuenta para instalar el motor, SSMS y las extensiones. Los laboratorios comprueban el nombre del equipo y la compilación con `winver`.

## Instancias de SQL Server

Instale el motor de base de datos y SQL Server Agent. La edición Developer es obligatoria: Resource Governor, la compresión de respaldos y el resto de prácticas del curso dependen de ella. Use la intercalación `Latin1_General_CI_AS` en las tres instancias.

Habilite TCP/IP en cada instancia. En **IPAll**, fije el puerto estático y deje vacío el puerto dinámico. Reinicie el servicio después del cambio. Conceda a cada cuenta de servicio el derecho **Perform volume maintenance tasks**, para que Instant File Initialization quede activo.

| Conexión que usa el laboratorio | Instancia | Motor | Puerto TCP | Servicio del motor | Servicio del Agent |
|---|---|---|---|---|---|
| `LOCALHOST,1433` | Predeterminada `MSSQLSERVER` | SQL Server 2025 Developer Edition `17.0.1000.7` | 1433 | `MSSQLSERVER` | `SQLSERVERAGENT` |
| `SQL2017SRC` | Con nombre `SQL2017SRC` | SQL Server 2017 Developer Edition | 1435 | `MSSQL$SQL2017SRC` | `SQLAgent$SQL2017SRC` |
| `SQL2025TGT` | Con nombre `SQL2025TGT` | SQL Server 2025 Developer Edition `17.0.1000.7` | 1436 | `MSSQL$SQL2025TGT` | `SQLAgent$SQL2025TGT` |

Los capítulos 1 a 5 usan solo `MSSQLSERVER`. El capítulo 6 se conecta a los nombres `SQL2017SRC` y `SQL2025TGT`. Cree alias de cliente de 32 y 64 bits para que esos nombres lleguen a los puertos de esta misma máquina:

| Alias | Valor |
|---|---|
| `SQL2017SRC` | `DBMSSOCN,127.0.0.1,1435` |
| `SQL2025TGT` | `DBMSSOCN,127.0.0.1,1436` |

Puede crearlos en el Administrador de configuración de SQL Server, en **SQL Native Client Configuration** y en **SQL Server Native Client 11.0 Configuration** si ambas ramas existen, o con el registro siguiente. Ejecútelo en PowerShell como administrador y reinicie SSMS después.

```powershell
$Aliases = @{
    'SQL2017SRC' = 'DBMSSOCN,127.0.0.1,1435'
    'SQL2025TGT' = 'DBMSSOCN,127.0.0.1,1436'
}

$Paths = @(
    'HKLM:\SOFTWARE\Microsoft\MSSQLServer\Client\ConnectTo',
    'HKLM:\SOFTWARE\WOW6432Node\Microsoft\MSSQLServer\Client\ConnectTo'
)

foreach ($Path in $Paths) {
    if (-not (Test-Path $Path)) {
        New-Item -Path $Path -Force | Out-Null
    }
    foreach ($Name in $Aliases.Keys) {
        New-ItemProperty -Path $Path -Name $Name -PropertyType String -Value $Aliases[$Name] -Force | Out-Null
    }
}
```

`.\SqlLabAdmin` debe ser `sysadmin` en las tres instancias. Deje los tres motores y los tres Agent en inicio automático y en ejecución.

En cada instancia, el modo de autenticación es mixto: Windows y SQL Server. El cambio exige reiniciar el servicio del motor.

## Login de conectividad

En `MSSQLSERVER` debe existir el login de SQL Server `lab_sa`, con permiso para conectar a `master`. El capítulo 1 solo comprueba que la autenticación SQL responde.

Defina su contraseña al preparar la imagen y entréguela al instructor por fuera del repositorio. No la escriba en scripts, informes ni en este archivo.

Estos logins no se crean en la imagen. Los laboratorios los generan:

| Login | Dónde se crea |
|---|---|
| `lab_reader`, `lab_analyst` | Capítulo 5, en `MSSQLSERVER` |
| `lab_reports` | Capítulo 4, en `MSSQLSERVER`; el mismo paso lo elimina |
| `lab_migracion` | Capítulo 6, en `SQL2017SRC` |

## Software de cliente

Instálelo en `SQLLAB-WS2022` e inicie sesión con `.\SqlLabAdmin` donde el producto lo pida.

| Software | Versión del curso | Uso |
|---|---|---|
| SQL Server Management Studio | 21.3.2 | Capítulos 1 a 6 |
| Data Migration Assistant | 5.6.6784.0 | Capítulo 6 |
| Visual Studio Code | 1.96.4 | Capítulo 3 |
| Extensión `mssql` de Visual Studio Code | 1.28.0 | Capítulo 3 |
| GitHub Copilot | 1.250.0 | Capítulo 3 |
| GitHub Copilot Chat | 0.23.2 | Capítulo 3 |

La ruta prevista de SSMS es:

```text
C:\Program Files\Microsoft SQL Server Management Studio 21\Release\Common7\IDE\SSMS.exe
```

Antes de entregar la imagen, abra Visual Studio Code con `.\SqlLabAdmin` y complete el inicio de sesión de GitHub Copilot. La salida HTTPS por el puerto 443 debe estar permitida. No guarde tokens ni contraseñas en el repositorio del curso.

## Carpetas y permisos

Ejecute PowerShell como administrador. El script crea las carpetas del curso y concede modificación a la cuenta virtual de cada instancia que exista en el equipo.

```powershell
$LabFolders = @(
    'C:\SQLLab2025\Scripts',
    'C:\SQLLab2025\Scripts\Lab02',
    'C:\SQLLab2025\Scripts\Lab03',
    'C:\SQLLab2025\Reports',
    'C:\SQLLab2025\Data',
    'C:\SQLLab2025\Logs',
    'C:\SQLLab2025\Backups',
    'C:\SQLLab2025\Audit',
    'C:\SQLLab2025\Keys',
    'C:\SQLLab2025\XEvents',
    'C:\LabSQL2025\Backup',
    'C:\LabSQL2025\Reports',
    'C:\LabSQL2025\Data',
    'C:\LabSQL2025\Logs'
)

$LabFolders | ForEach-Object {
    New-Item -Path $_ -ItemType Directory -Force | Out-Null
}

$SqlManagedFolders = @(
    'C:\SQLLab2025\Data',
    'C:\SQLLab2025\Logs',
    'C:\SQLLab2025\Backups',
    'C:\SQLLab2025\Audit',
    'C:\SQLLab2025\XEvents',
    'C:\LabSQL2025\Backup',
    'C:\LabSQL2025\Data',
    'C:\LabSQL2025\Logs'
)

$ServiceAccounts = @(
    'NT SERVICE\MSSQLSERVER',
    'NT SERVICE\MSSQL$SQL2017SRC',
    'NT SERVICE\MSSQL$SQL2025TGT',
    'NT SERVICE\SQLSERVERAGENT',
    'NT SERVICE\SQLAgent$SQL2017SRC',
    'NT SERVICE\SQLAgent$SQL2025TGT'
)

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
    }
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
```

| Ruta | Uso |
|---|---|
| `C:\SQLLab2025\Scripts` | Scripts de los capítulos 1 a 5. `Lab02` y `Lab03` son subcarpetas. |
| `C:\SQLLab2025\Reports` | Informes de los capítulos 1 a 5 |
| `C:\SQLLab2025\Data` | Datos de `tempdb` y de `MigracionLab_2025` |
| `C:\SQLLab2025\Logs` | Logs de `tempdb` y de `MigracionLab_2025` |
| `C:\SQLLab2025\Backups` | Respaldo TDE del capítulo 5 |
| `C:\SQLLab2025\Audit` | Archivos de auditoría del capítulo 5 |
| `C:\SQLLab2025\Keys` | Certificado y clave privada de TDE |
| `C:\SQLLab2025\XEvents` | Reservada para el laboratorio |
| `C:\LabSQL2025\Data` y `C:\LabSQL2025\Logs` | Archivos de `MigracionLab_Source` en `SQL2017SRC` |
| `C:\LabSQL2025\Backup` | Respaldo de migración y cadena full, diferencial, log y tail-log |
| `C:\LabSQL2025\Reports` | Informes del capítulo 6 |
| Recurso `LabBackup` | Comparte `C:\LabSQL2025\Backup` como `\\SQLLAB-WS2022\LabBackup` |

En esta imagen de una sola máquina, el capítulo 6 detecta que el origen y el destino comparten disco y no depende de copiar el archivo por la red. El recurso queda creado para que la ruta del taller exista.

## Red

Abra en el firewall local, para los perfiles usados por el laboratorio:

| Puerto | Protocolo | Motivo |
|---|---|---|
| 1433 | TCP | Instancia `MSSQLSERVER` |
| 1435 | TCP | Instancia `SQL2017SRC` |
| 1436 | TCP | Instancia `SQL2025TGT` |
| 445 | TCP | Recurso `LabBackup`, si se valida el uso compartido |
| 443 | TCP de salida | GitHub Copilot |

SQL Server Browser puede permanecer detenido. Los alias usan puerto fijo.

## Comprobación de la imagen

Ejecute esta revisión una vez, con `.\SqlLabAdmin`, antes de entregar la máquina.

```powershell
$env:COMPUTERNAME
(Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion').CurrentBuildNumber
Get-Service MSSQLSERVER, SQLSERVERAGENT, 'MSSQL$SQL2017SRC', 'SQLAgent$SQL2017SRC', 'MSSQL$SQL2025TGT', 'SQLAgent$SQL2025TGT' |
    Format-Table Name, Status, StartType -AutoSize
Test-NetConnection -ComputerName 127.0.0.1 -Port 1433
Test-NetConnection -ComputerName 127.0.0.1 -Port 1435
Test-NetConnection -ComputerName 127.0.0.1 -Port 1436
Get-SmbShare -Name LabBackup
Test-Path 'C:\Program Files\Microsoft SQL Server Management Studio 21\Release\Common7\IDE\SSMS.exe'
```

En cada instancia, una conexión de Windows de `.\SqlLabAdmin` debe mostrar edición Developer y `sysadmin = 1`. En `MSSQLSERVER`, la versión de producto es `17.0.1000.7` y existe el login `lab_sa`.

```sql
SELECT
    @@SERVERNAME AS servidor,
    SERVERPROPERTY('Edition') AS edicion,
    SERVERPROPERTY('ProductVersion') AS version,
    SERVERPROPERTY('IsIntegratedSecurityOnly') AS solo_windows,
    IS_SRVROLEMEMBER(N'sysadmin') AS es_sysadmin;
```

`solo_windows` debe ser `0`.

## Qué usa cada capítulo

| Capítulo | Instancia | Qué debe existir antes de empezarlo |
|---|---|---|
| 1 | `MSSQLSERVER` | Windows, SQL Server 2025, SSMS, carpetas `C:\SQLLab2025`, `lab_sa` y autenticación mixta |
| 2 | `MSSQLSERVER` | La base `Sql2025Lab` creada en el capítulo 1 |
| 3 | `MSSQLSERVER` | Visual Studio Code, extensión `mssql`, Copilot con sesión iniciada y salida por 443 |
| 4 | `MSSQLSERVER` | SQL Server Agent en ejecución y edición Developer |
| 5 | `MSSQLSERVER` | Carpetas `Audit`, `Keys` y `Backups` con permiso para la cuenta del servicio |
| 6 | `SQL2017SRC` y `SQL2025TGT` | Ambas instancias, DMA 5.6.6784.0, carpetas `C:\LabSQL2025` y recurso `LabBackup` |
