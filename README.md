# RemoteAPP + NPS (RADIUS) + AAA Cisco

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo de la Red](#1-objetivo-de-la-red)
2. [Topología y Direccionamiento](#2-topología-y-direccionamiento)
   - [Diagrama de Topología](#21-diagrama-de-topología)
   - [Tabla de Dispositivos](#22-tabla-de-dispositivos)
3. [Parte 1 — Windows Server](#3-parte-1--windows-server)
   - [3.1 Configuración de IP Estática](#31-configuración-de-ip-estática)
   - [3.2 Instalación de Roles](#32-instalación-de-roles)
   - [3.3 Configurar RemoteAPP](#33-configurar-remoteapp)
   - [3.4 Configurar RemoteAPP Web Client](#34-configurar-remoteapp-web-client)
   - [3.5 Crear Página Personalizada en IIS](#35-crear-página-personalizada-en-iis)
   - [3.6 Publicar la Página en RemoteAPP](#36-publicar-la-página-en-remoteapp)
   - [3.7 Configurar NPS — RADIUS Server](#37-configurar-nps--radius-server)
   - [3.8 Crear Políticas de Acceso por Nivel](#38-crear-políticas-de-acceso-por-nivel)
4. [Parte 2 — Router Cisco (AAA + RADIUS)](#4-parte-2--router-cisco-aaa--radius)
   - [4.1 Configuración Básica e Interfaz](#41-configuración-básica-e-interfaz)
   - [4.2 Usuarios Locales (Fallback)](#42-usuarios-locales-fallback)
   - [4.3 Configurar RADIUS Server](#43-configurar-radius-server)
   - [4.4 Habilitar AAA](#44-habilitar-aaa)
   - [4.5 Habilitar SSH](#45-habilitar-ssh)
5. [Parte 3 — Pruebas desde el Cliente (Host)](#5-parte-3--pruebas-desde-el-cliente-host)
   - [5.1 Página IIS](#51-página-iis)
   - [5.2 Portal RemoteAPP Web (RDWeb)](#52-portal-remoteapp-web-rdweb)
   - [5.3 Conexión RemoteAPP Directo (.rdp)](#53-conexión-remoteapp-directo-rdp)
   - [5.4 SSH al Router via RADIUS](#54-ssh-al-router-via-radius)
   - [5.5 Verificación de AAA en el Router](#55-verificación-de-aaa-en-el-router)
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
              │  Server     │  │   Cisco    │  │  (Host físico) │
              │  2016+      │  │            │  │                │
              │ 20.25.37.10 │  │20.25.37.254│  │  20.25.37.X    │
              │             │  │            │  │  (DHCP o       │
              │ • IIS       │  │ • AAA      │  │   estática)    │
              │ • RemoteAPP │  │ • RADIUS   │  │                │
              │ • NPS/RADIUS│  │ • SSH      │  │                │
              └─────────────┘  └────────────┘  └────────────────┘

  Flujo de autenticación RADIUS:
  Cliente SSH → Router (20.25.37.254) → NPS/RADIUS (20.25.37.10)
                    ↓ RADIUS Access-Accept + Cisco-AVPair priv-lvl
              Router asigna nivel 15 (admin) o nivel 1 (user)

  Flujo RemoteAPP Web:
  Cliente navegador → https://20.25.37.10/rdweb → Portal RDWeb
                           → lanza app RemoteAPP vía RDP
```

### 2.2 Tabla de Dispositivos

| Dispositivo | Rol | Dirección IP | Máscara | Gateway | Notas |
|---|---|---|---|---|---|
| **Windows Server 2016+** | IIS + RemoteAPP + NPS/RADIUS | 20.25.37.10 | /24 | 20.25.37.254 | IP estática |
| **Router Cisco** | AAA client + RADIUS client + SSH | 20.25.37.254 | /24 | — | Fa0/0 o e0/0 |
| **Cliente (Host físico)** | Consumidor de servicios | 20.25.37.X | /24 | 20.25.37.254 | DHCP o estática |

**Usuarios configurados:**

| Usuario | Contraseña | Nivel | Dónde existe |
|---|---|---|---|
| `admin_lab` | `AdminRadius123!` | 15 (Admin) | Windows Server — grupo Administrators |
| `user_lab` | `UserRadius123!` | 1 (User) | Windows Server — usuario local sin grupo admin |
| `admin_local` | `AdminLocal123!` | 15 | Router — fallback local |
| `user_local` | `UserLocal123!` | 1 | Router — fallback local |

**Shared Secret RADIUS:** `RadiusSecret123`

---

## 3. Parte 1 — Windows Server

### 3.1 Configuración de IP Estática

Existen dos formas de configurar la IP en Windows Server. La GUI es la forma natural en Windows — la CLI se incluye como referencia alternativa.

**Opción A — GUI (forma recomendada):**

**Ruta:** `Panel de Control → Centro de redes y recursos compartidos → Cambiar configuración del adaptador`

1. Clic derecho sobre la interfaz de red → `Propiedades`
2. Seleccionar `Protocolo de Internet versión 4 (TCP/IPv4)` → `Propiedades`
3. Seleccionar `Usar la siguiente dirección IP` y completar:

| Campo | Valor |
|---|---|
| Dirección IP | `20.25.37.10` |
| Máscara de subred | `255.255.255.0` |
| Puerta de enlace predeterminada | `20.25.37.254` |
| Servidor DNS preferido | `8.8.8.8` |
| Servidor DNS alternativo | `8.8.4.4` |

4. Clic en **Aceptar** → **Cerrar**

> Ver evidencia: [01_server_ip_gui.png](#01_server_ip_guipng)

**Opción B — PowerShell (alternativa):**

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" `
    -IPAddress 20.25.37.10 `
    -PrefixLength 24 `
    -DefaultGateway 20.25.37.254

Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses 8.8.8.8, 8.8.4.4
```

**Verificar (ambos métodos):**

```powershell
ipconfig /all
```

> Ver evidencia: [01_server_ip_estatica.png](#01_server_ip_estaticapng)

---

### 3.2 Instalación de Roles

En **PowerShell como Administrador**, instalar los tres roles necesarios. Reiniciar entre cada instalación si el sistema lo solicita.

```powershell
# IIS + ASP.NET
Install-WindowsFeature Web-Server, Web-App-Dev, Net-Framework-45-ASPNET, Web-Asp-Net45

# NPS — Network Policy Server (RADIUS)
Install-WindowsFeature NPAS -IncludeManagementTools

# Remote Desktop Services
Install-WindowsFeature RDS-RD-Server, RDS-Licensing
```

Confirmar instalación desde **Server Manager → Dashboard** — los tres roles deben aparecer en verde sin errores.

> Ver evidencia: [02_roles_instalados.png](#02_roles_instaladospng)

---

### 3.3 Configurar RemoteAPP

**Ruta:** `Inicio → Remote Desktop Services → RemoteApp Manager`

**Paso 1 — Publicar una aplicación:**

1. En RemoteApp Manager → clic derecho en **RemoteApps** → `Publish RemoteApp Program`
2. Seleccionar: `notepad.exe` (Bloc de notas)
3. Nombre visible: `Editor de Texto RemoteAPP`
4. Clic en **Publish**

**Paso 2 — Exportar el archivo .rdp:**

1. Clic derecho sobre la app publicada → `Create .rdp File`
2. Guardar en una ubicación accesible para el cliente

> El archivo `.rdp` permite al cliente lanzar la aplicación RemoteAPP directamente sin pasar por el portal web.

> Ver evidencia: [03_remoteapp_publicado.png](#03_remoteapp_publicadopng)

---

### 3.4 Configurar RemoteAPP Web Client

El portal RDWeb permite acceder a las aplicaciones RemoteAPP desde un navegador web mediante HTTPS.

**Ruta:** `Server Manager → Remote Desktop Services → Collections`

**Paso 1 — Crear una Session Collection:**

1. Clic en `Tasks → Create Session Collection`
2. Nombre: `RemoteAPP-Collection`
3. Seleccionar el servidor RD Session Host: el propio servidor
4. Finalizar el asistente

**Paso 2 — Publicar la app en la colección:**

1. En la colección creada → `RemoteApp Programs → Publish RemoteApp Programs`
2. Seleccionar `notepad.exe` y cualquier otra app a publicar
3. Completar el asistente

**Paso 3 — Verificar el portal RDWeb:**

Desde el cliente navegar a:

```
https://20.25.37.10/rdweb
```

> Si el certificado SSL es auto-firmado, aceptar la advertencia del navegador.

> Ver evidencia: [04_rdweb_portal.png](#04_rdweb_portalpng)

---

### 3.5 Crear Página Personalizada en IIS

**Paso 1 — Crear la carpeta y el archivo HTML:**

En **File Explorer**, navegar a `C:\inetpub\wwwroot` y crear:

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

**Paso 2 — Crear Virtual Directory en IIS Manager:**

**Ruta:** `Inicio → IIS Manager → Sites → Default Web Site`

1. Clic derecho sobre `Default Web Site` → `Add Virtual Directory`
2. Alias: `RemoteAPPAccess`
3. Physical path: `C:\inetpub\wwwroot\RemoteAPPAccess`
4. Clic en **OK**

**Paso 3 — Verificar desde el cliente:**

```
http://20.25.37.10/RemoteAPPAccess/
```

> Ver evidencia: [05_iis_pagina_custom.png](#05_iis_pagina_custompng) y [06_iis_virtual_directory.png](#06_iis_virtual_directorypng)

---

### 3.6 Publicar la Página en RemoteAPP

Publicar Internet Explorer (o Microsoft Edge) como aplicación RemoteAPP apuntando a la página IIS, de modo que el cliente pueda abrirla a través del portal RDWeb.

**Ruta:** `RemoteApp Manager → Publish RemoteApp Program`

1. Seleccionar `iexplore.exe` o `msedge.exe`
2. Nombre visible: `Portal IIS RemoteAPP`
3. En **Command-line arguments**: `http://localhost/RemoteAPPAccess/`
4. Publicar

Desde el portal RDWeb (`https://20.25.37.10/rdweb`), el cliente verá esta aplicación y al abrirla se lanzará el navegador directamente apuntando a la página IIS.

> Ver evidencia: [07_remoteapp_iis_publicado.png](#07_remoteapp_iis_publicadopng)

---

### 3.7 Configurar NPS — RADIUS Server

**Ruta:** `Inicio → Network Policy Server`

**Paso 1 — Crear usuarios locales en PowerShell:**

```powershell
# Usuario administrador (nivel 15)
New-LocalUser -Name "admin_lab" `
    -Password (ConvertTo-SecureString "AdminRadius123!" -AsPlainText -Force) `
    -FullName "Administrador Lab"
Add-LocalGroupMember -Group "Administrators" -Member "admin_lab"

# Usuario estándar (nivel 1)
New-LocalUser -Name "user_lab" `
    -Password (ConvertTo-SecureString "UserRadius123!" -AsPlainText -Force) `
    -FullName "Usuario Lab"
```

> Ver evidencia: [08_usuarios_locales.png](#08_usuarios_localespng)

**Paso 2 — Registrar NPS en Active Directory (si aplica):**

En NPS → clic derecho en el nodo raíz → `Register Server in Active Directory`

> Si el servidor no está en un dominio, omitir este paso.

**Paso 3 — Agregar RADIUS Client (el Router):**

En NPS → `RADIUS Clients and Servers → RADIUS Clients` → clic derecho → `New`

| Campo | Valor |
|---|---|
| Friendly name | `Cisco-Router-Lab` |
| Address (IP) | `20.25.37.254` |
| Vendor | `Cisco` |
| Shared Secret | `RadiusSecret123` |
| Confirm Secret | `RadiusSecret123` |

> Ver evidencia: [09_nps_radius_client.png](#09_nps_radius_clientpng)

---

### 3.8 Crear Políticas de Acceso por Nivel

Las políticas NPS determinan qué nivel de privilegio Cisco se devuelve al router según el grupo del usuario que se autentica.

**Política 1 — Nivel 15 (Administrador):**

**Ruta:** `NPS → Policies → Network Policies → New`

| Paso | Campo | Valor |
|---|---|---|
| 1 | Policy name | `Admin_Level_15` |
| 1 | Policy state | `Enabled` |
| 2 | Conditions → Add → `Windows Groups` | `Administrators` |
| 3 | Access Permission | `Access granted` |
| 4 | Authentication Methods | `MS-CHAP v2`, `CHAP` |
| 5 | Settings → RADIUS Attributes → Vendor Specific → Add | |
| 5 | Vendor | `Cisco` |
| 5 | Attribute value | `shell:priv-lvl=15` |

> El atributo `Cisco-AVPair = shell:priv-lvl=15` le indica al router que debe asignar nivel de privilegio 15 al usuario autenticado.

> Ver evidencia: [10_nps_policy_level15.png](#10_nps_policy_level15png)

**Política 2 — Nivel 1 (Usuario estándar):**

Repetir el mismo proceso con:

| Campo | Valor |
|---|---|
| Policy name | `User_Level_1` |
| Conditions → `Windows Groups` | *(ningún grupo específico — aplica a `user_lab`)* |
| Attribute value | `shell:priv-lvl=1` |

> Asegurarse de que la política `Admin_Level_15` esté **por encima** de `User_Level_1` en el orden de procesamiento. NPS aplica la primera política que hace match.

> Ver evidencia: [11_nps_policy_level1.png](#11_nps_policy_level1png)

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

---

### 4.2 Usuarios Locales (Fallback)

Los usuarios locales actúan como mecanismo de respaldo si el servidor RADIUS no está disponible.

```cisco
configure terminal
username admin_local privilege 15 secret AdminLocal123!
username user_local privilege 1 secret UserLocal123!
```

---

### 4.3 Configurar RADIUS Server

```cisco
configure terminal
radius server NPS-Lab
 address ipv4 20.25.37.10 auth-port 1812 acct-port 1813
 key RadiusSecret123
exit
```

---

### 4.4 Habilitar AAA

```cisco
configure terminal
aaa new-model

! Autenticación — primero RADIUS, fallback a local
aaa authentication login default group radius local

! Autorización — niveles de comandos vía RADIUS
aaa authorization console
aaa authorization commands 0 default group radius local
aaa authorization commands 1 default group radius local
aaa authorization commands 15 default group radius local

! Accounting — registrar inicio/fin de sesiones en RADIUS
aaa accounting exec default start-stop group radius

! Contraseña para modo privilegiado (enable)
enable secret EnablePassword123!
```

---

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

**Verificar generación de clave RSA:**

```cisco
show ip ssh
```

> La salida debe mostrar `SSH Enabled - version 2.0`.

---

## 5. Parte 3 — Pruebas desde el Cliente (Host)

### 5.1 Página IIS

Abrir el navegador en el host y navegar a:

```
http://20.25.37.10/RemoteAPPAccess/
```

Debe cargarse la página personalizada con el título "Portal RemoteAPP" y los datos del laboratorio.

> Ver evidencia: [12_cliente_pagina_iis.png](#12_cliente_pagina_iispng)

---

### 5.2 Portal RemoteAPP Web (RDWeb)

Abrir el navegador y navegar a:

```
https://20.25.37.10/rdweb
```

Iniciar sesión con:

| Campo | Valor |
|---|---|
| Usuario | `admin_lab` |
| Contraseña | `AdminRadius123!` |

Deben aparecer las aplicaciones publicadas (Notepad, Portal IIS). Hacer clic en una para lanzarla vía RDP desde el navegador.

> Ver evidencia: [13_rdweb_login.png](#13_rdweb_loginpng) y [14_rdweb_app_lanzada.png](#14_rdweb_app_lanzadapng)

---

### 5.3 Conexión RemoteAPP Directo (.rdp)

Desde el host, abrir el archivo `.rdp` exportado en la sección 3.3. Windows lanzará la aplicación RemoteAPP directamente sin pasar por el portal web.

> Ver evidencia: [15_remoteapp_rdp_directo.png](#15_remoteapp_rdp_directopng)

---

### 5.4 SSH al Router via RADIUS

Abrir una terminal (PowerShell, CMD o PuTTY) en el host y conectarse al router:

**Con usuario nivel 15 (Administrador):**

```bash
ssh admin_lab@20.25.37.254
# Password: AdminRadius123!
```

Una vez conectado, verificar el nivel de privilegio:

```cisco
Lab-Router> show privilege
Current privilege level is 15
Lab-Router# show run    ! debe funcionar
```

**Con usuario nivel 1 (estándar):**

```bash
ssh user_lab@20.25.37.254
# Password: UserRadius123!
```

```cisco
Lab-Router> show privilege
Current privilege level is 1
Lab-Router> show run    ! debe ser denegado — acceso insuficiente
```

> Ver evidencia: [16_ssh_nivel15.png](#16_ssh_nivel15png) y [17_ssh_nivel1.png](#17_ssh_nivel1png)

---

### 5.5 Verificación de AAA en el Router

Ejecutar los siguientes comandos en el router para verificar el funcionamiento completo de AAA y RADIUS:

**Estado del servidor RADIUS:**

```cisco
show aaa servers
```

*Salida esperada:*

```
RADIUS: id 1, priority 1, host 20.25.37.10, auth-port 1812, acct-port 1813
     State: current UP, duration 300s, previous duration 0s
     Dead: total time 0s, count 0
     Authen: request 4, timeouts 0, failover 0, retransmission 0
             Response: accept 3, reject 1, challenge 0
```

> `State: current UP` confirma que el router tiene comunicación con el NPS. Los contadores de `accept` deben subir con cada login exitoso.

**Sesiones AAA activas:**

```cisco
show aaa sessions
```

**Debug en tiempo real (ejecutar antes del login):**

```cisco
debug aaa authentication
debug aaa authorization
debug radius
```

*Salida esperada durante un login de `admin_lab`:*

```
AAA/AUTHEN: create_user user='admin_lab'
RADIUS: Sending Access-Request to 20.25.37.10:1812
RADIUS: Received Access-Accept from 20.25.37.10:1812
AAA/AUTHOR: user='admin_lab' requests service=shell priv-level=15
AAA/AUTHOR: vendor_specific attr Cisco-AVPair shell:priv-lvl=15
```

Para detener el debug:

```cisco
undebug all
```

> Ver evidencia: [18_show_aaa_servers.png](#18_show_aaa_serverspng), [19_show_aaa_sessions.png](#19_show_aaa_sessionspng) y [20_debug_radius_output.png](#20_debug_radius_outputpng)

---

## 6. Capturas de Pantalla

Todas las capturas están en la carpeta [`screenshots/`](screenshots/).

| # | Archivo | Descripción |
|---|---|---|
| 01 | [`01_server_ip_gui.png`](screenshots/01_server_ip_gui.png) | Panel de Control → Propiedades de TCP/IPv4 mostrando la IP `20.25.37.10`, máscara `/24`, gateway `20.25.37.254` y DNS `8.8.8.8` configurados. |
| 01b | [`01_server_ip_estatica.png`](screenshots/01_server_ip_estatica.png) | Salida de `ipconfig /all` en PowerShell confirmando la IP `20.25.37.10/24` y el gateway `20.25.37.254` activos. |
| 02 | [`02_roles_instalados.png`](screenshots/02_roles_instalados.png) | Server Manager → Dashboard mostrando los tres roles instalados: IIS (Web Server), NPS (Network Policy and Access Services) y RDS (Remote Desktop Services) en verde. |
| 03 | [`03_remoteapp_publicado.png`](screenshots/03_remoteapp_publicado.png) | RemoteApp Manager mostrando `notepad.exe` publicado como "Editor de Texto RemoteAPP" con estado activo. |
| 04 | [`04_rdweb_portal.png`](screenshots/04_rdweb_portal.png) | Portal RDWeb (`https://20.25.37.10/rdweb`) cargado en el navegador mostrando la página de login con el logo de Windows Server. |
| 05 | [`05_iis_pagina_custom.png`](screenshots/05_iis_pagina_custom.png) | Navegador mostrando la página personalizada de IIS en `http://20.25.37.10/RemoteAPPAccess/` con el título "Portal RemoteAPP" y los datos del laboratorio. |
| 06 | [`06_iis_virtual_directory.png`](screenshots/06_iis_virtual_directory.png) | IIS Manager mostrando el Virtual Directory `RemoteAPPAccess` creado bajo `Default Web Site`, con la ruta física `C:\inetpub\wwwroot\RemoteAPPAccess`. |
| 07 | [`07_remoteapp_iis_publicado.png`](screenshots/07_remoteapp_iis_publicado.png) | RemoteApp Manager mostrando el navegador (IE/Edge) publicado con el argumento `http://localhost/RemoteAPPAccess/` — la página IIS accesible desde el portal RDWeb. |
| 08 | [`08_usuarios_locales.png`](screenshots/08_usuarios_locales.png) | PowerShell mostrando la creación de `admin_lab` (agregado al grupo Administrators) y `user_lab`, o la vista de `Computer Management → Local Users`. |
| 09 | [`09_nps_radius_client.png`](screenshots/09_nps_radius_client.png) | NPS → RADIUS Clients mostrando el cliente `Cisco-Router-Lab` con IP `20.25.37.254`, vendor Cisco y estado habilitado. |
| 10 | [`10_nps_policy_level15.png`](screenshots/10_nps_policy_level15.png) | NPS → Network Policies mostrando la política `Admin_Level_15` con condición `Windows Groups: Administrators` y el atributo Cisco-AVPair `shell:priv-lvl=15`. |
| 11 | [`11_nps_policy_level1.png`](screenshots/11_nps_policy_level1.png) | NPS → Network Policies mostrando la política `User_Level_1` con el atributo `shell:priv-lvl=1`, posicionada debajo de `Admin_Level_15` en el orden de procesamiento. |
| 12 | [`12_cliente_pagina_iis.png`](screenshots/12_cliente_pagina_iis.png) | Navegador del host mostrando la página IIS personalizada cargada correctamente desde `http://20.25.37.10/RemoteAPPAccess/`. |
| 13 | [`13_rdweb_login.png`](screenshots/13_rdweb_login.png) | Portal RDWeb (`https://20.25.37.10/rdweb`) con las credenciales de `admin_lab` ingresadas y la sesión iniciada exitosamente. |
| 14 | [`14_rdweb_app_lanzada.png`](screenshots/14_rdweb_app_lanzada.png) | Aplicación RemoteAPP (Notepad o Portal IIS) ejecutándose en el host lanzada desde el portal RDWeb — ventana del programa remoto abierta localmente. |
| 15 | [`15_remoteapp_rdp_directo.png`](screenshots/15_remoteapp_rdp_directo.png) | Aplicación RemoteAPP lanzada desde el archivo `.rdp` directamente, sin usar el portal web — ventana del programa remoto en el host. |
| 16 | [`16_ssh_nivel15.png`](screenshots/16_ssh_nivel15.png) | Terminal del host mostrando la conexión SSH como `admin_lab` al router y la salida de `show privilege` confirmando `Current privilege level is 15`. |
| 17 | [`17_ssh_nivel1.png`](screenshots/17_ssh_nivel1.png) | Terminal del host mostrando la conexión SSH como `user_lab` y la salida de `show privilege` confirmando `Current privilege level is 1`, con intento fallido de `show run`. |
| 18 | [`18_show_aaa_servers.png`](screenshots/18_show_aaa_servers.png) | Consola del router mostrando `show aaa servers` con el NPS en `State: current UP` y contadores de `accept` incrementados. |
| 19 | [`19_show_aaa_sessions.png`](screenshots/19_show_aaa_sessions.png) | Consola del router mostrando `show aaa sessions` con las sesiones autenticadas activas. |
| 20 | [`20_debug_radius_output.png`](screenshots/20_debug_radius_output.png) | Consola del router mostrando el output de `debug radius` durante un login de `admin_lab`, con los mensajes `Access-Request` enviado y `Access-Accept` recibido del NPS. |

---

## 7. Video Demostrativo

🎥 **[Ver demostración en YouTube](#)**

**Duración:** máximo 8 minutos

**Contenido del video:**

* ✅ Topología en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica.
* ✅ Navegación a `http://20.25.37.10/RemoteAPPAccess/` — página IIS cargando.
* ✅ Login en `https://20.25.37.10/rdweb` con `admin_lab` y lanzamiento de app RemoteAPP.
* ✅ Apertura de `.rdp` directo — app RemoteAPP lanzada sin portal web.
* ✅ SSH al router como `admin_lab` → `show privilege` → nivel 15.
* ✅ SSH al router como `user_lab` → `show privilege` → nivel 1 → `show run` denegado.
* ✅ `show aaa servers` en el router mostrando NPS UP con contadores activos.
* ✅ Output de `debug radius` durante un login mostrando el flujo Access-Request / Access-Accept.

---

## 8. Referencias

* Microsoft. (2024). *Remote Desktop Services — RemoteApp Programs*. Microsoft Docs.
* Microsoft. (2024). *Network Policy Server (NPS) Configuration Guide*. Microsoft Docs.
* Microsoft. (2024). *Internet Information Services (IIS) Manager*. Microsoft Docs.
* Rigney, C. et al. (2000). *RFC 2865 — Remote Authentication Dial In User Service (RADIUS)*. IETF.
* Cisco Systems. (2024). *Cisco IOS Security Configuration Guide — AAA*. Cisco.
* Cisco Systems. (2024). *Configuring RADIUS with Cisco IOS AAA*. Cisco.
