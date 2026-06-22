# IPSec VPN — Basado en Políticas (IKEv1)

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [¿Qué es IPSec?](#21-qué-es-ipsec)
   - [IKEv1 — Negociación en Dos Fases](#22-ikev1--negociación-en-dos-fases)
   - [VPN Basada en Políticas vs. Basada en Rutas](#23-vpn-basada-en-políticas-vs-basada-en-rutas)
   - [Parámetros Criptográficos Utilizados](#24-parámetros-criptográficos-utilizados)
3. [Documentación de la Red](#3-documentación-de-la-red)
   - [Topología](#31-topología)
   - [Tabla de Dispositivos y Direccionamiento IP](#32-tabla-de-dispositivos-y-direccionamiento-ip)
4. [Scripts de Configuración](#4-scripts-de-configuración)
   - [Router R1 — Site A](#41-router-r1--site-a)
   - [Router R2 — Site B](#42-router-r2--site-b)
   - [Configuración de PCs (Hosts)](#43-configuración-de-pcs-hosts)
5. [Verificación del Túnel](#5-verificación-del-túnel)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Contramedidas y Consideraciones de Seguridad](#7-contramedidas-y-consideraciones-de-seguridad)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo del Laboratorio

El objetivo de este laboratorio es **implementar y verificar una VPN Site-to-Site basada en políticas utilizando IPSec con IKEv1** sobre infraestructura Cisco IOS. A través de esta práctica se busca demostrar:

* La configuración del proceso de negociación de claves en dos fases de IKEv1 (ISAKMP Phase 1 y Quick Mode Phase 2) entre dos routers Cisco situados en sitios geográficamente separados.
* La protección criptográfica del tráfico de datos entre dos subredes LAN privadas (`20.25.37.128/25` y `20.25.37.0/25`) a través de un segmento de red pública simulado (`192.168.1.0/24`).
* El uso de **Crypto Maps** y **ACLs de tráfico interesante** como mecanismo de selección de flujos a cifrar, propio del modelo *policy-based* de IPSec.
* La validación del túnel mediante comandos de verificación nativos de IOS y la comprobación de conectividad extremo a extremo con `ping` extendido entre hosts.

Este laboratorio se realiza en un entorno controlado con fines **exclusivamente educativos** dentro del curso de Seguridad de Redes del ITLA.

---

## 2. Marco Teórico

### 2.1 ¿Qué es IPSec?

IPSec (Internet Protocol Security — RFC 4301) es un conjunto de protocolos de la capa de red (Capa 3) que proporciona tres servicios fundamentales de seguridad sobre redes IP no confiables:

| Servicio | Descripción | Protocolo responsable |
|---|---|---|
| **Confidencialidad** | Cifrado de la carga útil del paquete IP. | ESP (Encapsulating Security Payload) |
| **Integridad** | Verificación de que el paquete no fue alterado en tránsito. | ESP / AH (Authentication Header) |
| **Autenticación de origen** | Garantiza que el paquete proviene del peer legítimo. | IKE + ESP/AH |

IPSec opera en dos modos:

* **Modo Transporte:** Solo cifra el payload del paquete IP original — el header IP queda expuesto. Utilizado para comunicaciones host-a-host.
* **Modo Túnel:** Cifra el paquete IP completo (header + payload) y lo encapsula dentro de un nuevo paquete IP. Es el modo estándar para VPNs Site-to-Site. **Este laboratorio utiliza modo túnel.**

### 2.2 IKEv1 — Negociación en Dos Fases

IKEv1 (RFC 2409) es el protocolo de gestión de claves que automatiza el establecimiento de las **Security Associations (SAs)** entre los dos peers IPSec. Opera en dos fases secuenciales:

#### Fase 1 — ISAKMP SA (Main Mode)

Objetivo: establecer un canal seguro y autenticado entre los dos peers para proteger las negociaciones posteriores.

El **Main Mode** intercambia 6 mensajes en 3 pares:

| Par de mensajes | Función |
|---|---|
| Mensajes 1 y 2 | Negociación de algoritmos (policy exchange): cifrado, hash, autenticación, grupo DH, lifetime. |
| Mensajes 3 y 4 | Intercambio Diffie-Hellman y nonces — generan el material de clave compartido. |
| Mensajes 5 y 6 | Autenticación de los peers (intercambiados ya cifrados con la clave derivada). |

El resultado es la **ISAKMP SA**: un canal IKE bidireccional cifrado con los parámetros negociados.

#### Fase 2 — IPSec SA (Quick Mode)

Objetivo: negociar los parámetros específicos del túnel de datos IPSec (las **IPSec SAs**).

Usando el canal seguro establecido en Fase 1, Quick Mode intercambia 3 mensajes para acordar:

* **Transform Set**: algoritmos de cifrado (AES) y autenticación (SHA) para ESP.
* **Tráfico interesante**: qué flujos IP serán protegidos (definidos por la ACL de la Crypto Map).
* **Lifetime de la SA**: tiempo o volumen de datos tras el cual se renegocian las claves.

El resultado son **dos IPSec SAs unidireccionales** (una por sentido del tráfico) con sus respectivos **SPIs (Security Parameter Index)**.

### 2.3 VPN Basada en Políticas vs. Basada en Rutas

| Característica | Policy-Based (este lab) | Route-Based |
|---|---|---|
| **Mecanismo de selección** | ACL de "tráfico interesante" en la Crypto Map | Interfaz de túnel lógica (Virtual-Tunnel Interface) |
| **Enrutamiento dinámico** | No soportado directamente sobre el túnel | Soportado (OSPF, EIGRP, BGP sobre el túnel) |
| **Configuración** | Crypto Map aplicada a la interfaz física WAN | `tunnel protection ipsec profile` en interfaz Tunnel |
| **Escalabilidad** | Baja (una Crypto Map entry por par de redes) | Alta |
| **IOS soporte** | Todas las versiones IOS | IOS 12.3(14)T+ |
| **Compatibilidad** | Universal — soportado por prácticamente todos los vendors | Depende del vendor |

La VPN basada en políticas es el modelo **más compatible y universalmente soportado**, lo que lo convierte en el estándar de facto para interoperabilidad entre equipos de distintos fabricantes.

### 2.4 Parámetros Criptográficos Utilizados

| Parámetro | Valor configurado | Propósito |
|---|---|---|
| **Cifrado IKE (Fase 1)** | AES-256 | Cifrado simétrico del canal IKE |
| **Hash IKE (Fase 1)** | SHA-256 | Integridad de mensajes IKE |
| **Autenticación** | Pre-Shared Key (PSK) | Autenticación mutua de los peers |
| **Grupo Diffie-Hellman** | Grupo 14 (2048-bit MODP) | Intercambio seguro de clave simétrica |
| **Lifetime ISAKMP SA** | 86400 segundos (24 horas) | Duración del canal IKE antes de renegociar |
| **Cifrado ESP (Fase 2)** | AES-256 | Cifrado del tráfico de datos |
| **Autenticación ESP (Fase 2)** | SHA-256 HMAC | Integridad del tráfico de datos |
| **Modo IPSec** | Tunnel | VPN Site-to-Site completa |
| **Lifetime IPSec SA** | 3600 segundos (1 hora) | Duración del túnel de datos antes de renegociar |

---

## 3. Documentación de la Red

### 3.1 Topología

El laboratorio simula la interconexión segura de dos sitios corporativos a través de Internet. El segmento `192.168.1.0/24` representa la nube pública/ISP. Cada sitio tiene su propia subred LAN privada, derivadas de la matrícula `20250737`.

```
                              [ INTERNET / ISP ]
                              192.168.1.0/24
                              (Router ISP: 192.168.1.2)
                                    │
                   ┌────────────────┴────────────────┐
                   │ e0/0: 192.168.1.10              │ e0/0: 192.168.1.20
           ┌───────┴───────┐                 ┌───────┴───────┐
           │  Router R1    │◄═══ TÚNEL ══════►  Router R2    │
           │  (Site A)     │   IPSec IKEv1   │  (Site B)     │
           └───────┬───────┘  Policy-Based   └───────┬───────┘
                   │ e0/1: 20.25.37.129/25           │ e0/1: 20.25.37.1/25
                   │                                 │
           ┌───────┴───────┐                 ┌───────┴───────┐
           │  Switch SW1   │                 │  Switch SW2   │
           └───┬───────┬───┘                 └───┬───────┬───┘
               │       │                         │       │
           ┌───┴──┐ ┌──┴───┐               ┌────┴─┐ ┌───┴──┐
           │ PC1  │ │ PC2  │               │ PC3  │ │ PC4  │
           │.130  │ │.131  │               │ .2   │ │ .3   │
           └──────┘ └──────┘               └──────┘ └──────┘
            20.25.37.128/25                 20.25.37.0/25
               SITE A                          SITE B

  ════════════════════════════════════════════════════════
  Flujo del túnel IPSec IKEv1:
    1. PC1 (20.25.37.130) hace ping a PC3 (20.25.37.2).
    2. R1 detecta tráfico interesante (ACL_IPSEC_TRAFFIC).
    3. R1 inicia IKE Fase 1 (Main Mode) con R2 → ISAKMP SA.
    4. IKE Fase 2 (Quick Mode) → IPSec SAs (ESP bidireccional).
    5. El tráfico viaja cifrado por Internet: 192.168.1.1 → 192.168.1.2.
    6. R2 desencapsula y entrega a PC3. Conexión segura establecida.
  ════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Gateway | Rol |
|---|---|---|---|---|---|---|
| **ISP** | Router ISP | e0/0 | 192.168.1.2 | /24 | — | Enlace hacia R2 |
| | | e0/1 | 192.168.1.2 | /24 | — | Enlace hacia R2 |
| **R1** | Cisco IOS (Router) | e0/0 | 192.168.1.1 | /24 | 192.168.1.254 | Gateway WAN — Site A |
| | | e0/1 | 20.25.37.129 | /25 | — | Gateway LAN — Site A |
| **R2** | Cisco IOS (Router) | e0/0 | 192.168.1.2 | /24 | 192.168.1.254 | Gateway WAN — Site B |
| | | e0/1 | 20.25.37.1 | /25 | — | Gateway LAN — Site B |
| **SW1** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Site A |
| **SW2** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Site B |
| **PC1** | Host Linux / VPC | eth0 | 20.25.37.130 | /25 | 20.25.37.129 | Host Site A |
| **PC2** | Host Linux / VPC | eth0 | 20.25.37.131 | /25 | 20.25.37.129 | Host Site A |
| **PC3** | Host Linux / VPC | eth0 | 20.25.37.2 | /25 | 20.25.37.1 | Host Site B |
| **PC4** | Host Linux / VPC | eth0 | 20.25.37.3 | /25 | 20.25.37.1 | Host Site B |

---

## 4. Scripts de Configuración

### 4.1 Router R1 — Site A

```cisco
! ══════════════════════════════════════════════════════
! R1 — Site A | IPSec IKEv1 Policy-Based VPN
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R1

! ─── Interfaces ────────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.1.1 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteA
 ip address 20.25.37.129 255.255.255.128
 no shutdown

! ─── Ruta por defecto hacia Internet (ISP) ─────────────
ip route 0.0.0.0 0.0.0.0 192.168.1.254

! ─── PASO 1: ISAKMP Policy — IKE Fase 1 ───────────────
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! ─── Pre-Shared Key del peer remoto (R2) ───────────────
crypto isakmp key J0rdyITLA2026! address 192.168.1.2

! ─── PASO 2: Transform Set — IKE Fase 2 (ESP) ─────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ─── PASO 3: ACL — Tráfico Interesante ─────────────────
! Define qué flujos IP activan y son protegidos por IPSec
ip access-list extended ACL_IPSEC_TRAFFIC
 permit ip 20.25.37.128 0.0.0.127 20.25.37.0 0.0.0.127

! ─── PASO 4: Crypto Map ────────────────────────────────
crypto map CMAP_SITEA 10 ipsec-isakmp
 description Tunel-IPSec-hacia-SiteB-R2
 set peer 192.168.1.2
 set transform-set TS_AES256_SHA256
 match address ACL_IPSEC_TRAFFIC
 set security-association lifetime seconds 3600

! ─── PASO 5: Aplicar Crypto Map a interfaz WAN ─────────
interface Ethernet0/0
 crypto map CMAP_SITEA
```

### 4.2 Router R2 — Site B

```cisco
! ══════════════════════════════════════════════════════
! R2 — Site B | IPSec IKEv1 Policy-Based VPN
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R2

! ─── Interfaces ────────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.1.2 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteB
 ip address 20.25.37.1 255.255.255.128
 no shutdown

! ─── Ruta por defecto hacia Internet (ISP) ─────────────
ip route 0.0.0.0 0.0.0.0 192.168.1.254

! ─── PASO 1: ISAKMP Policy — IKE Fase 1 ───────────────
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! ─── Pre-Shared Key del peer remoto (R1) ───────────────
crypto isakmp key J0rdyITLA2026! address 192.168.1.1

! ─── PASO 2: Transform Set — IKE Fase 2 (ESP) ─────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ─── PASO 3: ACL — Tráfico Interesante ─────────────────
! La ACL debe ser el ESPEJO exacto de la ACL en R1
ip access-list extended ACL_IPSEC_TRAFFIC
 permit ip 20.25.37.0 0.0.0.127 20.25.37.128 0.0.0.127

! ─── PASO 4: Crypto Map ────────────────────────────────
crypto map CMAP_SITEB 10 ipsec-isakmp
 description Tunel-IPSec-hacia-SiteA-R1
 set peer 192.168.1.1
 set transform-set TS_AES256_SHA256
 match address ACL_IPSEC_TRAFFIC
 set security-association lifetime seconds 3600

! ─── PASO 5: Aplicar Crypto Map a interfaz WAN ─────────
interface Ethernet0/0
 crypto map CMAP_SITEB
```

### 4.3 Configuración de PCs (Hosts)

**PC1 y PC2 — Site A (`20.25.37.128/25`)**

```bash
# PC1
ip 20.25.37.130 255.255.255.128 20.25.37.129

# PC2
ip 20.25.37.131 255.255.255.128 20.25.37.129
```

**PC3 y PC4 — Site B (`20.25.37.0/25`)**

```bash
# PC3
ip 20.25.37.2 255.255.255.128 20.25.37.1

# PC4
ip 20.25.37.3 255.255.255.128 20.25.37.1
```

> **Nota:** Si los hosts son VPCs de PNETLab se usa el comando `ip` como se muestra. Si son máquinas Linux se configura con `ip addr add` e `ip route add default via`.

---

## 5. Verificación del Túnel

### 5.1 Verificar el estado de IKE Fase 1 (ISAKMP SA)

```cisco
R1# show crypto isakmp sa
```

*Salida esperada — túnel activo:*

```
IPv4 Crypto ISAKMP SA
dst            src            state          conn-id status
192.168.1.2    192.168.1.1    QM_IDLE        1001    ACTIVE
```

> `QM_IDLE` indica que la Fase 1 completó exitosamente y el canal IKE está listo. Si muestra `MM_NO_STATE` o `AG_NO_STATE`, la negociación falló.

---

### 5.2 Verificar las IPSec SAs activas (Fase 2)

```cisco
R1# show crypto ipsec sa
```

*Salida esperada (fragmento):*

```
interface: Ethernet0/0
    Crypto map tag: CMAP_SITEA, local addr 192.168.1.1

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (20.25.37.128/255.255.255.128/0/0)
   remote ident (addr/mask/prot/port): (20.25.37.0/255.255.255.128/0/0)
   current_peer 192.168.1.2 port 500
    PERMIT, flags={origin_is_acl,}

    #pkts encaps: 24, #pkts encrypt: 24, #pkts digest: 24
    #pkts decaps: 24, #pkts decrypt: 24, #pkts verify: 24
```

> Los contadores `#pkts encaps` y `#pkts decaps` deben incrementarse con cada ping. Si están en 0 después de un ping, el tráfico no está siendo capturado por la ACL.

---

### 5.3 Verificar conectividad extremo a extremo

Ejecutar desde **PC1** hacia **PC3**:

```bash
PC1> ping 20.25.37.2
```

*Resultado esperado:*

```
84 bytes from 20.25.37.2 icmp_seq=1 ttl=62 time=X ms
84 bytes from 20.25.37.2 icmp_seq=2 ttl=62 time=X ms
84 bytes from 20.25.37.2 icmp_seq=3 ttl=62 time=X ms
```

> El TTL de 62 (en lugar de 64) confirma que el paquete atravesó 2 routers (R1 y R2). Un TTL de 64 indicaría que no se está pasando por los routers correctamente.

---

### 5.4 Verificar que el tráfico viaja cifrado (Wireshark / debug)

Desde **R1**:

```cisco
R1# debug crypto isakmp
R1# debug crypto ipsec
```

Para detener el debug:

```cisco
R1# undebug all
```

---

### 5.5 Tabla de comandos de verificación

| Comando | Qué muestra |
|---|---|
| `show crypto isakmp sa` | Estado de la SA de IKE Fase 1. Debe ser `QM_IDLE`. |
| `show crypto ipsec sa` | SAs de Fase 2 activas, SPIs, contadores de paquetes. |
| `show crypto isakmp policy` | Confirma los parámetros configurados en la política ISAKMP. |
| `show crypto map` | Muestra las Crypto Maps aplicadas y sus parámetros. |
| `show crypto session` | Resumen rápido del estado de todas las sesiones IPSec. |
| `show ip access-lists ACL_IPSEC_TRAFFIC` | Verifica hits en la ACL de tráfico interesante. |

---

## 6. Capturas de Pantalla

A continuación se detalla el índice de evidencias correspondientes a las fases de configuración, verificación y funcionamiento del túnel IPSec IKEv1, alojadas en la carpeta [`screenshots/`](screenshots/README.md):

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_ping_sin_vpn.png`](screenshots/02_ping_sin_vpn.png) | Ping desde PC1 (`20.25.37.130`) hacia PC3 (`20.25.37.2`) **antes** de aplicar la configuración IPSec — demuestra conectividad base. |
| 3 | [`03_config_r1_isakmp.png`](screenshots/03_config_r1_isakmp.png) | Consola de R1 mostrando la configuración de la ISAKMP policy (`crypto isakmp policy 10`) y la pre-shared key. |
| 4 | [`04_config_r1_cryptomap.png`](screenshots/04_config_r1_cryptomap.png) | Consola de R1 mostrando el transform set, la ACL de tráfico interesante y la Crypto Map aplicada a `Ethernet0/0`. |
| 5 | [`05_config_r2_completa.png`](screenshots/05_config_r2_completa.png) | Configuración equivalente de R2 (Site B) mostrando la simetría de la ACL espejo y el peer apuntando a R1. |
| 6 | [`06_isakmp_sa_qmidle.png`](screenshots/06_isakmp_sa_qmidle.png) | Salida de `show crypto isakmp sa` en R1 mostrando estado `QM_IDLE` — Fase 1 completada exitosamente. |
| 7 | [`07_ipsec_sa_contadores.png`](screenshots/07_ipsec_sa_contadores.png) | Salida de `show crypto ipsec sa` mostrando los contadores `#pkts encaps/decaps` incrementando con tráfico activo. |
| 8 | [`08_ping_con_vpn_activa.png`](screenshots/08_ping_con_vpn_activa.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC4 (`20.25.37.3`) con el túnel IPSec activo, TTL=62. |

---

## 7. Contramedidas y Consideraciones de Seguridad

### 7.1 Debilidades de IKEv1 en este laboratorio

Aunque este laboratorio usa parámetros robustos (AES-256, SHA-256, DH14), IKEv1 tiene limitaciones conocidas:

* **Aggressive Mode es inseguro:** Expone el hash de la PSK sin cifrar — permite ataques de diccionario offline. Este lab usa **Main Mode**, que es seguro.
* **PSK simple:** Las pre-shared keys son vulnerables si se reutilizan o son débiles. En producción se recomienda migrar a certificados digitales (PKI con IKEv1) o a IKEv2.
* **No soporta MOBIKE:** IKEv1 no maneja cambios de IP del peer durante la sesión.

### 7.2 Hardening recomendado

```cisco
! Deshabilitar DH groups débiles
no crypto isakmp policy 65535   ! Elimina la policy por defecto de Cisco (DES)

! Forzar Perfect Forward Secrecy en Fase 2
crypto map CMAP_SITEA 10 ipsec-isakmp
 set pfs group14

! Limitar intentos de negociación IKE (anti-DoS)
crypto isakmp aggressive-mode disable   ! Deshabilitar Aggressive Mode globalmente
```

### 7.3 Migración a IKEv2

Para un entorno de producción se recomienda migrar a IKEv2 (`crypto ikev2 proposal` / `crypto ikev2 profile`), que resuelve las limitaciones de IKEv1: negociación más eficiente (4 mensajes vs 9), soporte nativo de EAP, MOBIKE y mejor resistencia a ataques de DoS.

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](#)**

> *(Enlace disponible en `videos.txt` en la raíz del repositorio)*

**Duración:** máximo 8 minutos

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Demostración de los scripts de configuración aplicados en R1 y R2.
* ✅ Verificación de `show crypto isakmp sa` mostrando estado `QM_IDLE`.
* ✅ Verificación de `show crypto ipsec sa` con contadores de paquetes activos.
* ✅ Ping exitoso extremo a extremo entre PC1 (`20.25.37.130`) y PC3/PC4 (`20.25.37.2/3`).

---

## 9. Referencias

* Kent, S. & Seo, K. (2005). *RFC 4301 — Security Architecture for the Internet Protocol*. IETF.
* Harkins, D. & Carrel, D. (1998). *RFC 2409 — The Internet Key Exchange (IKEv1)*. IETF.
* Cisco Systems. (2024). *IPSec VPN Design Guide — Policy-Based vs. Route-Based VPN*.
* Cisco Systems. (2024). *Cisco IOS Security Command Reference — crypto isakmp / crypto map*.
* Doraswamy, N. & Harkins, D. (2003). *IPSec: The New Security Standard for the Internet, Intranets, and Virtual Private Networks (2nd Ed.)*. Prentice Hall.
