# Capturas de pantalla — IPSec VPN Basado en Políticas (IKEv1)

Capturas del laboratorio en orden de demostración.

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
