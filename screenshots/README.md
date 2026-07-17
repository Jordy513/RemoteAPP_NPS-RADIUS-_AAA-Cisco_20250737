# Capturas de pantalla — RemoteAPP + NPS (RADIUS) + AAA Cisco

Capturas del laboratorio en orden de demostración.

| # | Archivo | Descripción |
|---|---|---|
| 01 | [01_server_ip_estatica.png](/screenshots/01_server_ip_estatica.png) | `ipconfig /all` confirmando IP `20.25.37.10/24` y gateway `20.25.37.254`. |
| 02 | [02_dominio_creado.png](/screenshots/02_dominio_creado.png) | `Get-ADDomain` mostrando `DNSRoot: lab.local` tras la promoción a DC. |
| 03 | [03_roles_instalados.png](/screenshots/03_roles_instalados.png) | `Get-WindowsFeature` confirmando RDS-RD-Server, RDS-Connection-Broker y RDS-Web-Access como `Installed`. |
| 04 | [04_get_rdserver.png](/screenshots/04_get_rdserver.png) | `Get-RDServer` mostrando los tres roles asignados a `SERVER-LOCAL.LAB.LOCAL`. |
| 05 | [05_rdweb_certificado_ok.png](/screenshots/05_rdweb_certificado_ok.png) | Portal RDWeb cargando sin el error `ERR_SSL_KEY_USAGE_INCOMPATIBLE` tras el nuevo certificado. |
| 06 | [06_collection_creada.png](/screenshots/06_collection_creada.png) | Colección de sesiones `RemoteAPP-Collection` creada. |
| 07 | [07_remoteapp_publicado.png](/screenshots/07_remoteapp_publicado.png) | Administrador de RemoteApp con Notepad publicado. |
| 08 | [08_iis_pagina_custom.png](/screenshots/08_iis_pagina_custom.png) | Página personalizada de IIS cargando en el navegador. |
| 09 | [09_iis_virtual_directory.png](/screenshots/09_iis_virtual_directory.png) | Directorio virtual `RemoteAPPAccess` en IIS Manager. |
| 10 | [10_remoteapp_iis_publicado.png](/screenshots/10_remoteapp_iis_publicado.png) | Edge publicado como RemoteApp apuntando a la página IIS. |
| 11 | [11_usuarios_ad.png](/screenshots/11_usuarios_ad.png) | Usuarios `admin_lab` y `user_lab` creados en AD. |
| 12 | [12_nps_radius_client.png](/screenshots/12_nps_radius_client.png) | Cliente RADIUS `Cisco-Router-Lab` en NPS. |
| 13 | [13_nps_policy_level15.png](/screenshots/13_nps_policy_level15.png) | Política `Admin_Level_15` con atributo `shell:priv-lvl=15`. |
| 14 | [14_nps_policy_level1.png](/screenshots/14_nps_policy_level1.png) | Política `User_Level_1` con atributo `shell:priv-lvl=1`. |
| 15 | [15_cliente_pagina_iis.png](/screenshots/15_cliente_pagina_iis.png) | Cliente accediendo a la página IIS directamente. |
| 16 | [16_rdweb_login.png](/screenshots/16_rdweb_login.png) | Login exitoso en el portal RDWeb con `admin_lab`. |
| 17 | [17_rdweb_app_lanzada.png](/screenshots/17_rdweb_app_lanzada.png) | App RemoteAPP lanzada desde RDWeb. |
| 18 | [18_remoteapp_rdp_directo.png](/screenshots/18_remoteapp_rdp_directo.png) | App RemoteAPP lanzada desde archivo `.rdp` directo. |
| 19 | [19_ssh_nivel15.png](/screenshots/19_ssh_nivel15.png) | SSH como `admin_lab`, `show privilege` = 15. |
| 20 | [20_ssh_nivel1.png](/screenshots/20_ssh_nivel1.png) | SSH como `user_lab`, `show privilege` = 1, `show run` denegado. |
| 21 | [21_show_aaa_servers.png](/screenshots/21_show_aaa_servers.png) | `show aaa servers` con NPS en `State: current UP`. |
| 22 | [22_show_aaa_sessions.png](/screenshots/22_show_aaa_sessions.png) | `show aaa sessions` con sesiones activas. |
| 23 | [23_debug_radius_output.png](/screenshots/23_debug_radius_output.png) | `debug radius` mostrando Access-Request / Access-Accept. |
