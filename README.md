# RemoteAPP + NPS (RADIUS) + AAA Cisco

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo de la Red](#1-objetivo-de-la-red)
2. [Topología y Direccionamiento](#2-topología-y-direccionamiento)
3. [Parte 1 — Windows Server](#3-parte-1--windows-server)
   - [3.1 Configuración de IP Estática](#31-configuración-de-ip-estática)
   - [3.2 Promover el Servidor a Controlador de Dominio](#32-promover-el-servidor-a-controlador-de-dominio)
   - [3.3 Instalación de Roles (IIS, NPS, RDS)](#33-instalación-de-roles-iis-nps-rds)
   - [3.4 Crear el Despliegue de RDS](#34-crear-el-despliegue-de-rds)
   - [3.5 Corregir el Certificado de RDWeb](#35-corregir-el-certificado-de-rdweb)
   - [3.6 Crear la Colección de Sesiones y Publicar RemoteAPP](#36-crear-la-colección-de-sesiones-y-publicar-remoteapp)
   - [3.7 Crear Página Personalizada en IIS](#37-crear-página-personalizada-en-iis)
   - [3.8 Publicar la Página IIS en RemoteAPP](#38-publicar-la-página-iis-en-remoteapp)
   - [3.9 Configurar NPS — RADIUS Server](#39-configurar-nps--radius-server)
   - [3.10 Crear Políticas de Acceso por Nivel](#310-crear-políticas-de-acceso-por-nivel)
4. [Parte 2 — Router Cisco (AAA + RADIUS)](#4-parte-2--router-cisco-aaa--radius)
5. [Parte 3 — Pruebas desde el Cliente (Host)](#5-parte-3--pruebas-desde-el-cliente-host)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Video Demostrativo](#7-video-demostrativo)
8. [Referencias](#8-referencias)

---

## 1. Objetivo de la Red

Esta topología integra tres servicios en un único entorno de laboratorio:

* **RemoteAPP:** Publicar aplicaciones Windows de forma remota, permitiendo al cliente ejecutar programas del servidor como si fueran locales, tanto mediante cliente RDP directo como mediante el portal web RDWeb (HTTPS).
* **IIS + Página personalizada:** Servir una página web institucional desde el Windows Server, publicada también como recurso en el portal RemoteAPP Web.
* **NPS (RADIUS):** Centralizar la autenticación de red — el Windows Server actúa como servidor RADIUS que valida usuarios con distintos niveles de privilegio (nivel 15 para administradores, nivel 1 para usuarios estándar).
* **AAA en el Router Cisco:** Delegar la autenticación SSH del router al servidor RADIUS, de modo que las credenciales se validan en el NPS y el router asigna el nivel de privilegio correspondiente según la política de acceso devuelta. Los usuarios locales actúan como fallback si el RADIUS no responde.

> ⚠️ **Nota importante:** El portal RDWeb (RemoteAPP Web Client) requiere **RD Connection Broker**, el cual en Windows Server 2022 **no se puede desplegar correctamente en modo workgroup** (sin dominio) — el proceso depende de PowerShell Remoting con autenticación Kerberos entre el propio servidor y sí mismo, algo que un workgroup no resuelve de forma confiable. Por eso este lab **promueve el servidor a Controlador de Dominio** (`lab.local`) antes de tocar RDS. Es un dominio de un solo servidor, exclusivamente para que RDS funcione — no hacen falta más DCs ni clientes unidos al dominio para el resto del lab.

---

## 2. Topología y Direccionamiento

### 2.1 Diagrama de Topología

```
                    ┌─────────────────────────────────┐
                    │         Red del Laboratorio      │
                    │         20.25.37.0/24            │
                    │    (Host-Only / PNETLab Cloud)   │
                    └────┬───────────┬────────────┬────┘
                         │           │            │
              ┌──────────┴──┐  ┌─────┴──────┐  ┌─┴──────────────┐
              │  Windows    │  │   Router   │  │    Cliente     │
              │  Server 2022│  │   Cisco    │  │  (Host físico) │
              │  (DC lab.   │  │            │  │                │
              │   local)    │  │            │  │                │
              │ 20.25.37.10 │  │20.25.37.254│  │  20.25.37.X    │
              │             │  │            │  │  (DHCP o       │
              │ • AD DS     │  │ • AAA      │  │   estática)    │
              │ • IIS       │  │ • RADIUS   │  │                │
              │ • RemoteAPP │  │ • SSH      │  │                │
              │ • NPS/RADIUS│  │            │  │                │
              └─────────────┘  └────────────┘  └────────────────┘

  Flujo de autenticación RADIUS:
  Cliente SSH → Router (20.25.37.254) → NPS/RADIUS (20.25.37.10)
                    ↓ RADIUS Access-Accept + Cisco-AVPair priv-lvl
              Router asigna nivel 15 (admin) o nivel 1 (user)

  Flujo RemoteAPP Web:
  Cliente navegador → https://SERVER-LOCAL.lab.local/rdweb → Portal RDWeb
                           → lanza app RemoteAPP vía RDP
```

### 2.2 Tabla de Dispositivos

| Dispositivo | Rol | Dirección IP | Máscara | Gateway | Notas |
|---|---|---|---|---|---|
| **Windows Server 2022** | DC + IIS + RemoteAPP + NPS/RADIUS | 20.25.37.10 | /24 | 20.25.37.254 | IP estática |
| **Router Cisco** | AAA client + RADIUS client + SSH | 20.25.37.254 | /24 | — | Fa0/0 o e0/0 |
| **Cliente (Host físico)** | Consumidor de servicios | 20.25.37.X | /24 | 20.25.37.254 | DHCP o estática |

**Datos del dominio:**

| Campo | Valor |
|---|---|
| Nombre del dominio (FQDN) | `lab.local` |
| Nombre NetBIOS | `LAB` |
| Nombre del servidor | `SERVER-LOCAL` |
| FQDN del servidor | `SERVER-LOCAL.lab.local` |

**Usuarios configurados:**

| Usuario | Contraseña | Nivel | Dónde existe |
|---|---|---|---|
| `admin_lab` | `AdminRadius123!` | 15 (Admin) | Windows Server — grupo de dominio (Domain Admins) |
| `user_lab` | `UserRadius123!` | 1 (User) | Windows Server — usuario de dominio sin privilegios |
| `admin_local` | `AdminLocal123!` | 15 | Router — fallback local |
| `user_local` | `UserLocal123!` | 1 | Router — fallback local |

**Shared Secret RADIUS:** `RadiusSecret123`

---

## 3. Parte 1 — Windows Server

### 3.1 Configuración de IP Estática

**Ruta GUI:** `Panel de control → Centro de redes y recursos compartidos → Cambiar configuración del adaptador`

1. Clic derecho sobre la interfaz de red → `Propiedades`
2. Seleccionar `Protocolo de Internet versión 4 (TCP/IPv4)` → `Propiedades`
3. Seleccionar `Usar la siguiente dirección IP` y completar:

| Campo | Valor |
|---|---|
| Dirección IP | `20.25.37.10` |
| Máscara de subred | `255.255.255.0` |
| Puerta de enlace predeterminada | `20.25.37.254` |
| Servidor DNS preferido | `127.0.0.1` *(una vez sea DC, se apunta a sí mismo)* |

4. Clic en **Aceptar** → **Cerrar**

**Alternativa PowerShell:**

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress 20.25.37.10 `
    -PrefixLength 24 `
    -DefaultGateway 20.25.37.254

Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1
```

**Verificar:**

```powershell
ipconfig /all
```

> Ver evidencia: [01_server_ip_estatica.png](#01_server_ip_estaticapng)

---

### 3.2 Promover el Servidor a Controlador de Dominio

Este paso es **obligatorio** para que RDS (RD Connection Broker / RDWeb) funcione correctamente en Server 2022. Sin esto, `New-RDSessionDeployment` falla con errores de WinRM/Kerberos imposibles de resolver de forma estable en workgroup.

**Paso 1 — Instalar el rol AD DS:**

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

**Paso 2 — Promover el servidor y crear el bosque/dominio nuevo:**

```powershell
Import-Module ADDSDeployment

Install-ADDSForest `
    -DomainName "lab.local" `
    -DomainNetbiosName "LAB" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "Segura123!" -AsPlainText -Force) `
    -Force:$true
```

> El servidor se reiniciará automáticamente al terminar. Espera 2-3 minutos extra tras el login para que AD/DNS terminen de inicializar en segundo plano.

**Paso 3 — Verificar el dominio tras el reinicio:**

```powershell
Get-ADDomain
Resolve-DnsName SERVER-LOCAL.lab.local -Type A
ipconfig /all
```

* `Get-ADDomain` debe mostrar `DNSRoot: lab.local`.
* `Resolve-DnsName` debe devolver `20.25.37.10`.
* `ipconfig /all` debe mostrar `Sufijo DNS principal: lab.local`.

> A partir de aquí inicia sesión como `LAB\Administrador` (misma contraseña que tenías como admin local).

> Ver evidencia: [02_dominio_creado.png](#02_dominio_creadopng)

---

### 3.3 Instalación de Roles (IIS, NPS, RDS)

**Paso 1 — IIS y NPS (roles independientes, sin problema):**

```powershell
Install-WindowsFeature Web-Server, Web-App-Dev, Net-Framework-45-ASPNET, Web-Asp-Net45
Install-WindowsFeature NPAS -IncludeManagementTools
```

**Paso 2 — Roles de RDS, instalados uno por uno vía PowerShell:**

> ⚠️ **No uses el asistente GUI "Instalación de Servicios de Escritorio remoto → Implementación estándar"**. Ese wizard tiene un bug conocido en Server 2022 (falla con `ArgumentNotValid: MultiPointServerRole` y cancela toda la instalación). Instala los role services por separado en su lugar:

```powershell
Install-WindowsFeature -Name RDS-RD-Server -IncludeManagementTools
Install-WindowsFeature -Name RDS-Connection-Broker -IncludeManagementTools
Install-WindowsFeature -Name RDS-Web-Access -IncludeManagementTools
```

**Paso 3 — Reiniciar y verificar:**

```powershell
Restart-Computer
```

Después del reinicio:

```powershell
Get-WindowsFeature RDS-RD-Server, RDS-Connection-Broker, RDS-Web-Access
```

Los tres deben mostrar `Installed`.

> Ver evidencia: [03_roles_instalados.png](#03_roles_instaladospng)

---

### 3.4 Crear el Despliegue de RDS

Con el dominio ya creado, este paso debería completarse sin errores. Usa el **FQDN real** del servidor:

```powershell
New-RDSessionDeployment -ConnectionBroker "SERVER-LOCAL.lab.local" `
    -WebAccessServer "SERVER-LOCAL.lab.local" `
    -SessionHost "SERVER-LOCAL.lab.local"
```

> Este comando puede tardar varios minutos. No lo interrumpas aunque la consola parezca colgada.

**Si el comando falla con un error de RDMS o WID**, verifica y arranca estos servicios en orden antes de reintentar:

```powershell
Get-Service "MSSQL`$MICROSOFT##WID" | Select Status, StartType
Start-Service "MSSQL`$MICROSOFT##WID"

Get-Service RDMS | Select Status, StartType
Start-Service RDMS
```

Si `RDMS` sigue sin arrancar después de esto, reinstala el rol de Connection Broker:

```powershell
Uninstall-WindowsFeature RDS-Connection-Broker
Restart-Computer
# Después del reinicio:
Install-WindowsFeature RDS-Connection-Broker -IncludeManagementTools
Restart-Computer
```

Y repite el `New-RDSessionDeployment`.

**Verificar que quedó bien:**

```powershell
Get-RDServer
```

Debe listar `SERVER-LOCAL.LAB.LOCAL` con los tres roles: `RDS-RD-SERVER`, `RDS-CONNECTION-BROKER`, `RDS-WEB-ACCESS`.

> En **Administrador del servidor**, si la sección "Servicios de Escritorio remoto" aún muestra "No existe una implementación" pese a que `Get-RDServer` ya la confirma, es solo caché de la consola — cierra y vuelve a abrir Administrador del servidor (o `Restart-Service RDMS` y reabre).

> Ver evidencia: [04_get_rdserver.png](#04_get_rdserverpng)

---

### 3.5 Corregir el Certificado de RDWeb

Al probar `https://SERVER-LOCAL.lab.local/rdweb`, navegadores basados en Chromium (Edge, Chrome) rechazan el certificado autofirmado que RDWeb genera por defecto, mostrando:

```
ERR_SSL_KEY_USAGE_INCOMPATIBLE
```

Esto ocurre porque el certificado por defecto no tiene el `Key Usage` que Chromium exige explícitamente para autenticación de servidor. La solución es generar un certificado nuevo con las extensiones correctas y asignarlo al sitio.

**Paso 1 — Generar el certificado:**

```powershell
$cert = New-SelfSignedCertificate -DnsName "SERVER-LOCAL.lab.local", "20.25.37.10" `
    -CertStoreLocation "cert:\LocalMachine\My" `
    -KeyUsage DigitalSignature, KeyEncipherment `
    -KeyExportPolicy Exportable `
    -NotAfter (Get-Date).AddYears(5)
```

**Paso 2 — Asignarlo como binding SSL del sitio (puerto 443):**

```powershell
Import-Module WebAdministration
Remove-Item IIS:\SslBindings\0.0.0.0!443 -ErrorAction SilentlyContinue
New-Item IIS:\SslBindings\0.0.0.0!443 -Value $cert
```

**Paso 3 — Reiniciar IIS:**

```powershell
iisreset
```

**Paso 4 — Probar de nuevo:**

```
https://SERVER-LOCAL.lab.local/rdweb
```

> El navegador seguirá mostrando advertencia de "certificado no confiable" (porque sigue siendo autofirmado), pero ya no debe dar el error `ERR_SSL_KEY_USAGE_INCOMPATIBLE`. Acepta la excepción de seguridad para continuar — es esperado en un lab sin CA corporativa.

> Ver evidencia: [05_rdweb_certificado_ok.png](#05_rdweb_certificado_okpng)

---

### 3.6 Crear la Colección de Sesiones y Publicar RemoteAPP

**Ruta GUI:** `Administrador del servidor → Servicios de Escritorio remoto → Colecciones`

**Paso 1 — Crear la colección:**

1. `Tareas → Crear colección de sesiones`
2. Nombre: `RemoteAPP-Collection`
3. Seleccionar el servidor RD Session Host: `SERVER-LOCAL.lab.local`
4. Finalizar el asistente

**Paso 2 — Publicar Notepad como RemoteAPP de prueba:**

**Ruta GUI:** `Administrador del servidor → Herramientas → Servicios de Escritorio remoto → Administrador de RemoteApp`

1. Clic derecho en **Programas RemoteApp** → `Publicar programa RemoteApp`
2. Seleccionar `notepad.exe`
3. Nombre visible: `Editor de Texto RemoteAPP`
4. Publicar

**Paso 3 — Exportar el archivo .rdp (para conexión directa sin portal):**

1. Clic derecho sobre la app publicada → `Crear archivo .rdp`
2. Guardar en una ubicación accesible para el cliente

> Ver evidencia: [06_collection_creada.png](#06_collection_creadapng) y [07_remoteapp_publicado.png](#07_remoteapp_publicadopng)

---

### 3.7 Crear Página Personalizada en IIS

**Paso 1 — Crear la carpeta y el archivo HTML:**

En **Explorador de archivos**, navegar a `C:\inetpub\wwwroot` y crear:

```
C:\inetpub\wwwroot\RemoteAPPAccess\index.html
```

Contenido del archivo `index.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Portal RemoteAPP - RADIUS Lab</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            margin: 0; padding: 20px;
        }
        .container {
            max-width: 600px;
            margin: 100px auto;
            background: white;
            padding: 40px;
            border-radius: 10px;
            box-shadow: 0 10px 25px rgba(0,0,0,0.2);
        }
        h1 { color: #333; text-align: center; }
        p { text-align: center; color: #666; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🖥️ Portal RemoteAPP</h1>
        <p>Bienvenido al laboratorio — Jordy Rosario · 20250737</p>
        <p>Seguridad de Redes 2026-C-2 · ITLA</p>
    </div>
</body>
</html>
```

**Paso 2 — Crear Directorio Virtual en IIS Manager:**

**Ruta:** `Administrador del servidor → Herramientas → Administrador de Internet Information Services (IIS) → Sitios → Sitio web predeterminado`

1. Clic derecho sobre `Sitio web predeterminado` → `Agregar directorio virtual`
2. Alias: `RemoteAPPAccess`
3. Ruta de acceso física: `C:\inetpub\wwwroot\RemoteAPPAccess`
4. Clic en **Aceptar**

**Paso 3 — Verificar desde el cliente:**

```
http://20.25.37.10/RemoteAPPAccess/
```

> Ver evidencia: [08_iis_pagina_custom.png](#08_iis_pagina_custompng) y [09_iis_virtual_directory.png](#09_iis_virtual_directorypng)

---

### 3.8 Publicar la Página IIS en RemoteAPP

Publicar Microsoft Edge (o Internet Explorer) como aplicación RemoteAPP apuntando a la página IIS, de modo que el cliente pueda abrirla a través del portal RDWeb — esto cumple el requisito de "consultar la página mediante los dos servicios de RDP RemoteAPP" (portal web + .rdp directo).

**Ruta:** `Administrador de RemoteApp → Publicar programa RemoteApp`

1. Seleccionar `msedge.exe` (o `iexplore.exe` si está disponible)
2. Nombre visible: `Portal IIS RemoteAPP`
3. En **Argumentos de línea de comandos**: `http://localhost/RemoteAPPAccess/`
4. Publicar

Desde el portal RDWeb (`https://SERVER-LOCAL.lab.local/rdweb`), el cliente verá esta aplicación y al abrirla se lanzará el navegador apuntando a la página IIS.

> Ver evidencia: [10_remoteapp_iis_publicado.png](#10_remoteapp_iis_publicadopng)

---

### 3.9 Configurar NPS — RADIUS Server

**Ruta:** `Administrador del servidor → Herramientas → Servidor de directivas de redes (NPS)`

**Paso 1 — Crear usuarios (ahora como usuarios de dominio, ya que el servidor es DC):**

```powershell
New-ADUser -Name "admin_lab" -SamAccountName "admin_lab" `
    -UserPrincipalName "admin_lab@lab.local" `
    -AccountPassword (ConvertTo-SecureString "AdminRadius123!" -AsPlainText -Force) `
    -Enabled $true -PasswordNeverExpires $true
Add-ADGroupMember -Identity "Domain Admins" -Members "admin_lab"

New-ADUser -Name "user_lab" -SamAccountName "user_lab" `
    -UserPrincipalName "user_lab@lab.local" `
    -AccountPassword (ConvertTo-SecureString "UserRadius123!" -AsPlainText -Force) `
    -Enabled $true -PasswordNeverExpires $true
```

> Nota: usamos `Domain Admins` como grupo de nivel 15 porque ya existe y es equivalente a "Administradores" local en un DC. Si prefieres un grupo dedicado, créalo con `New-ADGroup -Name "RADIUS-Admins" -GroupScope Global` y agrega ahí a `admin_lab`.

**Paso 2 — Agregar RADIUS Client (el Router):**

En NPS → `Clientes y servidores RADIUS → Clientes RADIUS` → clic derecho → `Nuevo`

| Campo | Valor |
|---|---|
| Nombre descriptivo | `Cisco-Router-Lab` |
| Dirección (IP) | `20.25.37.254` |
| Proveedor | `Cisco` |
| Secreto compartido | `RadiusSecret123` |
| Confirmar secreto | `RadiusSecret123` |

> Ver evidencia: [11_usuarios_ad.png](#11_usuarios_adpng) y [12_nps_radius_client.png](#12_nps_radius_clientpng)

---

### 3.10 Crear Políticas de Acceso por Nivel

**Política 1 — Nivel 15 (Administrador):**

**Ruta:** `NPS → Directivas → Directivas de redes → Nueva`

| Paso | Campo | Valor |
|---|---|---|
| 1 | Nombre de la directiva | `Admin_Level_15` |
| 1 | Estado | `Habilitada` |
| 2 | Condiciones → Agregar → `Grupos de Windows` | `LAB\Domain Admins` |
| 3 | Permiso de acceso | `Acceso concedido` |
| 4 | Métodos de autenticación | `MS-CHAP v2`, `CHAP` |
| 5 | Configuración → Atributos RADIUS → Específico del proveedor → Agregar → Proveedor `Cisco` | `shell:priv-lvl=15` |

**Política 2 — Nivel 1 (Usuario estándar):**

| Campo | Valor |
|---|---|
| Nombre de la directiva | `User_Level_1` |
| Condiciones → `Grupos de Windows` | *(ningún grupo específico — aplica a `user_lab`)* |
| Valor del atributo | `shell:priv-lvl=1` |

> `Admin_Level_15` debe estar **por encima** de `User_Level_1` en el orden de procesamiento (NPS aplica la primera política que hace match).

> Ver evidencia: [13_nps_policy_level15.png](#13_nps_policy_level15png) y [14_nps_policy_level1.png](#14_nps_policy_level1png)

---

## 4. Parte 2 — Router Cisco (AAA + RADIUS)

### 4.1 Configuración Básica e Interfaz

```cisco
enable
configure terminal
hostname Lab-Router
ip domain-name lab.local

interface FastEthernet0/0
 ip address 20.25.37.254 255.255.255.0
 no shutdown
end
```

### 4.2 Usuarios Locales (Fallback)

```cisco
configure terminal
username admin_local privilege 15 secret AdminLocal123!
username user_local privilege 1 secret UserLocal123!
```

### 4.3 Configurar RADIUS Server

```cisco
configure terminal
radius server NPS-Lab
 address ipv4 20.25.37.10 auth-port 1812 acct-port 1813
 key RadiusSecret123
exit
```

### 4.4 Habilitar AAA

```cisco
configure terminal
aaa new-model

aaa authentication login default group radius local

aaa authorization console
aaa authorization commands 0 default group radius local
aaa authorization commands 1 default group radius local
aaa authorization commands 15 default group radius local

aaa accounting exec default start-stop group radius

enable secret EnablePassword123!
```

### 4.5 Habilitar SSH

```cisco
configure terminal
ip ssh version 2
crypto key generate rsa modulus 2048

line vty 0 4
 login authentication default
 transport input ssh
 logging synchronous
exit

write memory
```

**Verificar:**

```cisco
show ip ssh
```

> La salida debe mostrar `SSH Enabled - version 2.0`.

---

## 5. Parte 3 — Pruebas desde el Cliente (Host)

### 5.1 Página IIS

```
http://20.25.37.10/RemoteAPPAccess/
```

> Ver evidencia: [15_cliente_pagina_iis.png](#15_cliente_pagina_iispng)

### 5.2 Portal RemoteAPP Web (RDWeb)

```
https://SERVER-LOCAL.lab.local/rdweb
```

Iniciar sesión con `admin_lab` / `AdminRadius123!`. Deben aparecer las apps publicadas (Notepad, Portal IIS). Lanzar el "Portal IIS RemoteAPP" desde aquí satisface la prueba de "consultar la página vía RDWeb".

> Ver evidencia: [16_rdweb_login.png](#16_rdweb_loginpng) y [17_rdweb_app_lanzada.png](#17_rdweb_app_lanzadapng)

### 5.3 Conexión RemoteAPP Directo (.rdp)

Abrir el archivo `.rdp` exportado en la sección 3.6/3.8 (versión Notepad o versión Edge apuntando a la página IIS). Esto cubre el segundo de "los dos servicios de RDP RemoteAPP" que pide la tarea.

> Ver evidencia: [18_remoteapp_rdp_directo.png](#18_remoteapp_rdp_directopng)

### 5.4 SSH al Router via RADIUS

**Nivel 15:**

```bash
ssh admin_lab@20.25.37.254
# Password: AdminRadius123!
```

```cisco
Lab-Router> show privilege
Current privilege level is 15
Lab-Router# show run    ! debe funcionar
```

**Nivel 1:**

```bash
ssh user_lab@20.25.37.254
# Password: UserRadius123!
```

```cisco
Lab-Router> show privilege
Current privilege level is 1
Lab-Router> show run    ! debe ser denegado
```

> Ver evidencia: [19_ssh_nivel15.png](#19_ssh_nivel15png) y [20_ssh_nivel1.png](#20_ssh_nivel1png)

### 5.5 Verificación de AAA en el Router

```cisco
show aaa servers
show aaa sessions
debug aaa authentication
debug aaa authorization
debug radius
```

*Salida esperada de `show aaa servers`:*

```
RADIUS: id 1, priority 1, host 20.25.37.10, auth-port 1812, acct-port 1813
     State: current UP, duration 300s, previous duration 0s
     Dead: total time 0s, count 0
     Authen: request 4, timeouts 0, failover 0, retransmission 0
             Response: accept 3, reject 1, challenge 0
```

Detener el debug:

```cisco
undebug all
```

> Ver evidencia: [21_show_aaa_servers.png](#21_show_aaa_serverspng), [22_show_aaa_sessions.png](#22_show_aaa_sessionspng) y [23_debug_radius_output.png](#23_debug_radius_outputpng)

---

## 6. Capturas de Pantalla

Todas las capturas están en la carpeta [`screenshots/`](screenshots/).

| # | Archivo | Descripción |
|---|---|---|
| 01 | `01_server_ip_estatica.png` | `ipconfig /all` confirmando IP `20.25.37.10/24` y gateway `20.25.37.254`. |
| 02 | `02_dominio_creado.png` | `Get-ADDomain` mostrando `DNSRoot: lab.local` tras la promoción a DC. |
| 03 | `03_roles_instalados.png` | `Get-WindowsFeature` confirmando RDS-RD-Server, RDS-Connection-Broker y RDS-Web-Access como `Installed`. |
| 04 | `04_get_rdserver.png` | `Get-RDServer` mostrando los tres roles asignados a `SERVER-LOCAL.LAB.LOCAL`. |
| 05 | `05_rdweb_certificado_ok.png` | Portal RDWeb cargando sin el error `ERR_SSL_KEY_USAGE_INCOMPATIBLE` tras el nuevo certificado. |
| 06 | `06_collection_creada.png` | Colección de sesiones `RemoteAPP-Collection` creada. |
| 07 | `07_remoteapp_publicado.png` | Administrador de RemoteApp con Notepad publicado. |
| 08 | `08_iis_pagina_custom.png` | Página personalizada de IIS cargando en el navegador. |
| 09 | `09_iis_virtual_directory.png` | Directorio virtual `RemoteAPPAccess` en IIS Manager. |
| 10 | `10_remoteapp_iis_publicado.png` | Edge publicado como RemoteApp apuntando a la página IIS. |
| 11 | `11_usuarios_ad.png` | Usuarios `admin_lab` y `user_lab` creados en AD. |
| 12 | `12_nps_radius_client.png` | Cliente RADIUS `Cisco-Router-Lab` en NPS. |
| 13 | `13_nps_policy_level15.png` | Política `Admin_Level_15` con atributo `shell:priv-lvl=15`. |
| 14 | `14_nps_policy_level1.png` | Política `User_Level_1` con atributo `shell:priv-lvl=1`. |
| 15 | `15_cliente_pagina_iis.png` | Cliente accediendo a la página IIS directamente. |
| 16 | `16_rdweb_login.png` | Login exitoso en el portal RDWeb con `admin_lab`. |
| 17 | `17_rdweb_app_lanzada.png` | App RemoteAPP lanzada desde RDWeb. |
| 18 | `18_remoteapp_rdp_directo.png` | App RemoteAPP lanzada desde archivo `.rdp` directo. |
| 19 | `19_ssh_nivel15.png` | SSH como `admin_lab`, `show privilege` = 15. |
| 20 | `20_ssh_nivel1.png` | SSH como `user_lab`, `show privilege` = 1, `show run` denegado. |
| 21 | `21_show_aaa_servers.png` | `show aaa servers` con NPS en `State: current UP`. |
| 22 | `22_show_aaa_sessions.png` | `show aaa sessions` con sesiones activas. |
| 23 | `23_debug_radius_output.png` | `debug radius` mostrando Access-Request / Access-Accept. |

---

## 7. Video Demostrativo

🎥 **[Ver demostración en YouTube](#)**

**Duración:** máximo 8 minutos

**Contenido del video:**

* ✅ Topología en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica.
* ✅ Navegación a `http://20.25.37.10/RemoteAPPAccess/` — página IIS cargando.
* ✅ Login en `https://SERVER-LOCAL.lab.local/rdweb` con `admin_lab` y lanzamiento de la app "Portal IIS RemoteAPP" (primer servicio de RemoteAPP).
* ✅ Apertura de `.rdp` directo — app RemoteAPP lanzada sin portal web (segundo servicio de RemoteAPP).
* ✅ SSH al router como `admin_lab` → `show privilege` → nivel 15.
* ✅ SSH al router como `user_lab` → `show privilege` → nivel 1 → `show run` denegado.
* ✅ `show aaa servers` en el router mostrando NPS UP con contadores activos.
* ✅ Output de `debug radius` durante un login mostrando el flujo Access-Request / Access-Accept.

---

## 8. Referencias

* Microsoft. (2024). *Remote Desktop Services — RemoteApp Programs*. Microsoft Docs.
* Microsoft. (2024). *Network Policy Server (NPS) Configuration Guide*. Microsoft Docs.
* Microsoft. (2024). *Internet Information Services (IIS) Manager*. Microsoft Docs.
* Microsoft. (2024). *Install-ADDSForest*. Microsoft Docs.
* Rigney, C. et al. (2000). *RFC 2865 — Remote Authentication Dial In User Service (RADIUS)*. IETF.
* Cisco Systems. (2024). *Cisco IOS Security Configuration Guide — AAA*. Cisco.
* Cisco Systems. (2024). *Configuring RADIUS with Cisco IOS AAA*. Cisco.
