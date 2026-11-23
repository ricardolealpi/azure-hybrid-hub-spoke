# Arquitectura Híbrida Privada y Segmentada en Azure: Hub-and-Spoke, DNS Cruzado e Identidad

## 1. Caso de Estudio y Contexto Empresarial

Una organización con infraestructura tradicional *on-premises* —basada en Active Directory y DNS interno sobre Windows Server (`hl.internal`)— inicia su expansión hacia Microsoft Azure para desplegar entornos aislados de **Producción** y **Desarrollo**.

El departamento de sistemas requiere construir una arquitectura híbrida en Azure mediante Infraestructura como Código (Terraform) que cumpla cinco requisitos operativos:

1. **Exposición mínima a internet:** Ninguna máquina virtual de aplicación puede tener dirección IP pública. El único punto de entrada externo es un Network Virtual Appliance (NVA) endurecido que termina el túnel VPN cifrado.
2. **Segmentación estricta entre entornos:** Los entornos de Producción y Desarrollo deben estar aislados entre sí para impedir movimientos laterales, enrutando únicamente el tráfico autorizado a través de la red central (Hub).
3. **Resolución de nombres bidireccional:** Los servidores en Azure deben resolver registros del dominio corporativo local (`hl.internal`) y los equipos *on-premises* deben resolver zonas privadas de Azure (`azure.hl.internal` / `privatelink`) sin salir a internet.
4. **Control de acceso granular:** Aplicación del principio de mínimo privilegio mediante roles RBAC personalizados en Azure e integración posterior de identidad híbrida.
5. **Eficiencia de costes (FinOps) y cuota:** Maximizar el aprovechamiento de recursos evitando servicios gestionados de alto coste fijo (como Azure VPN Gateway o Azure DNS Private Resolver) mediante soluciones ligeras auditadas con Azure Cost Management.

---

## 2. Alcance y Fases de Implementación

* **Infraestructura base (Prerrequisitos):** Se parte de un entorno local operativo con un Controlador de Dominio en **Windows Server 2022 Core** (`DC01-SRV`) bajo el dominio `hl.internal`, una estación de administración con herramientas RSAT (`ADM01-CLI`) y una VM **Ubuntu Server** (`UBUNTU01-SRV`) que actuará como gateway VPN local.
* **Fases del proyecto:**
  * **Fase 1 — Enrutamiento local y preparación VPN:** Configuración de interfaces de red, reenvío IP en kernel Linux, túnel WireGuard saliente (`PersistentKeepalive`) y rutas estáticas en `DC01-SRV`.
  * **Fase 2 — Red Hub-and-Spoke con Terraform:** Despliegue de `vnet-hybrid-core` (Hub) y dos VNets Spoke (`vnet-spoke-prod` y `vnet-spoke-dev`), *VNet Peering*, reglas NSG de aislamiento entre Spokes, tablas de rutas (UDR) y NVA con IP Forwarding habilitado en Azure.
  * **Fase 3 — DNS Híbrido y Gobernanza RBAC:** Configuración de *Azure Private DNS Zones*, despliegue de reenviador DNS ligero (`dnsmasq`) en el NVA, reenviadores condicionales en Windows Server y creación de roles RBAC personalizados en formato JSON.
  * **Fase 4 — Validación, Troubleshooting y Auditoría FinOps:** Pruebas de conectividad y aislamiento extremo a extremo, resolución de incidencias documentadas, sincronización opcional con *Microsoft Entra Cloud Sync* y análisis de consumo real en Azure Cost Management.

---

## 3. Decisiones de Arquitectura

| Componente | Decisión adoptada | Alternativa descartada | Justificación técnica y financiera |
| :--- | :--- | :--- | :--- |
| **Topología de red** | **Hub-and-Spoke** (3 VNets con Peering) | VNet única con múltiples subredes | Permite aislar Producción y Desarrollo a nivel de red, centralizar el tráfico híbrido en `vnet-hybrid-core` y escalar nuevas cargas sin solapamiento CIDR. |
| **Túnel Site-to-Site** | **NVA Linux con WireGuard** (iniciado desde local) | Azure VPN Gateway (SKU VpnGw1) | Reduce drásticamente el coste por hora y el tiempo de despliegue frente a un gateway gestionado. Al iniciar la conexión desde *on-premises* con `PersistentKeepalive`, funciona detrás de NAT/CGNAT sin abrir puertos en el router local. |
| **Aislamiento inter-Spoke** | **NSG Deny explícito + reglas `iptables` en NVA** | Confiar solo en la falta de peering entre Spokes | Al apuntar las UDR de ambos Spokes hacia el NVA, el NVA podría enrutar tráfico entre ellos. El bloqueo en NSG e `iptables` garantiza aislamiento real. |
| **Resolución DNS híbrida** | **Azure Private DNS + `dnsmasq` en el NVA** | Azure DNS Private Resolver | Un Private Resolver supone un coste fijo elevado. Ejecutar `dnsmasq` en el NVA reenvía consultas hacia `168.63.129.16` (Azure) y `192.168.10.10` (AD local) sin consumir más vCPUs ni presupuesto. |
| **Infraestructura como Código** | **Terraform** | Despliegue manual en Portal | Garantiza despliegues reproducibles, control de versiones en GitHub y destrucción limpia de recursos tras cada sesión de pruebas. |

---

## 4. Topología y Plan de Direccionamiento IP (CIDR)

| Entorno | Red / VNet | Bloque CIDR | Subred / Función | Host / Recurso | IP Privada |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **On-Premises** | Red Local (`hl.internal`) | `192.168.10.0/24` | Red interna de servidores | `DC01-SRV` (Win Server 2022 Core) | `192.168.10.10` |
| **On-Premises** | Red Local (`hl.internal`) | `192.168.10.0/24` | Red interna de servidores | `UBUNTU01-SRV` (Gateway WireGuard) | `192.168.10.20` |
| **On-Premises** | Red Local (`hl.internal`) | `192.168.10.0/24` | Administración (RSAT) | `ADM01-CLI` (Windows 10) | `192.168.10.30` |
| **Túnel VPN** | Enlace Punto a Punto | `172.16.0.0/30` | Interfaz cifrada `wg0` | Local (`.1`) ↔ Azure NVA (`.2`) | `172.16.0.1` / `.2` |
| **Azure Hub** | `vnet-hybrid-core` | `10.0.0.0/16` | `snet-nva` (`10.0.1.0/24`) | `vm-nva-vpn` (WireGuard + `dnsmasq`) | `10.0.1.4` |
| **Azure Hub** | `vnet-hybrid-core` | `10.0.0.0/16` | `snet-shared` (`10.0.2.0/24`) | Reservada / VM temporal Sync | `10.0.2.4` |
| **Azure Spoke 1** | `vnet-spoke-prod` | `10.1.0.0/16` | `snet-workload` (`10.1.1.0/24`) | `vm-prod-01` (Carga Producción) | `10.1.1.4` |
| **Azure Spoke 2** | `vnet-spoke-dev` | `10.2.0.0/16` | `snet-workload` (`10.2.1.0/24`) | `vm-dev-01` (Carga Desarrollo) | `10.2.1.4` |