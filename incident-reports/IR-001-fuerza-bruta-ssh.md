# IR-001 - Informe de incidente: fuerza bruta SSH contra victima-linux

| Campo | Valor |
|---|---|
| ID de incidente | IR-001 |
| Técnica MITRE ATT&CK | T1110.001 (Brute Force: Password Guessing) |
| Tácticas asociadas | Credential Access, Lateral Movement (T1021.004) |
| Severidad | Alta (acceso con credenciales válidas, mitigado) |
| Sistema afectado | `victima-linux` (192.168.100.20) |
| Origen | `atacante` / Kali Linux (192.168.100.30) |
| Analista | Abdelhamid Boughafour |
| Fecha del informe | 13/10/2026 |
| Estado | Cerrado — mitigado mediante respuesta activa |

## 1. Resumen ejecutivo

Se detectó un ataque de fuerza bruta contra el servicio SSH de `victima-linux`, originado desde la IP `192.168.100.30`, usando la herramienta Hydra contra el usuario `hamid`. En una primera prueba, realizada con la configuración por defecto de Wazuh (sin reglas de correlación ni respuesta activa), el atacante consiguió acceso válido tras 6 intentos, sin que se generara ninguna alerta de mayor severidad ni se tomara ninguna acción de bloqueo. Tras implementar una regla de correlación personalizada y una respuesta activa (bloqueo automático por firewall), se repitió el ataque: esta vez el sistema bloqueó la IP origen tras el cuarto intento fallido, impidiendo que el atacante llegara a autenticarse con éxito (`0 valid password found`). El incidente se considera **cerrado y mitigado**.

## 2. Línea temporal

### 2.1 Prueba inicial (configuración por defecto, sin correlación ni respuesta activa)

Realizada en una sesión anterior (documentada en `docs/05-fuerza-bruta-ssh.md`). Sin marcas de tiempo exactas registradas, pero con la siguiente secuencia confirmada:

| Evento | Detalle |
|---|---|
| Inicio del ataque | Hydra lanza 6 intentos secuenciales (`-t 1`) contra `hamid@192.168.100.20` |
| Duración total | ~17 segundos |
| Alertas generadas | 5 alertas individuales, regla 5760 (nivel 5, "authentication failed") |
| Resultado | Contraseña válida encontrada en el 6º intento (`1 valid password found`) |
| Respuesta del sistema | Ninguna — no hubo correlación ni bloqueo |

### 2.2 Remediación

| Fecha/hora | Acción |
|---|---|
| 13/10/2026 | Se crea la regla de correlación `100010` en `local_rules.xml` (frecuencia 4 repeticiones de la regla 5760 en 120s, misma IP origen) |
| 13/10/2026 | Se configura la respuesta activa `firewall-drop` en `ossec.conf`, ligada a la regla `100010`, con bloqueo de 300s |
| 13/10/2026 18:03 | Primer intento de verificación: la regla se dispara correctamente, pero el bloqueo falla — `iptables` no está instalado en `victima-linux` |
| 13/10/2026 | Se instala `iptables` en `victima-linux` (incidencia documentada en `docs/06-correlacion-active-response.md`) |

### 2.3 Incidente verificado (ataque final, con respuesta activa operativa)

| Hora (CEST) | Evento |
|---|---|
| 18:47:50 | Hydra inicia el ataque desde `192.168.100.30` contra `hamid@192.168.100.20` |
| 18:47:52 | Intento fallido nº2 (`password`) — regla 5760 |
| 18:47:54 | Intento fallido nº3 (`qwerty`) — regla 5760 |
| 18:47:56 | Intento fallido nº4 (`admin123`) — regla 5760 |
| 18:48:00 | Intento fallido nº5 (`letmein`) — regla 5760. 4ª coincidencia en la ventana de 120s → **se dispara la regla 100010** (nivel 10) |
| 18:48:00 | Respuesta activa `firewall-drop` ejecutada en `victima-linux`: se añade regla `DROP` en `iptables` para `192.168.100.30` |
| 18:48:0x | Intento nº6 (`victima`, contraseña válida) — la conexión queda bloqueada por el firewall, no se completa |
| 18:49:05 | Hydra finaliza: `1 of 1 target completed, 0 valid password found` |
| ~18:53 (estimado) | Expiración automática del bloqueo tras el `timeout` de 300s configurado (comportamiento verificado en una ejecución de prueba anterior a este incidente) |

## 3. Indicadores de compromiso (IOCs)

| Tipo | Valor |
|---|---|
| IP origen (atacante) | 192.168.100.30 |
| Sistema atacante | Kali Linux, herramienta Hydra v9.7 |
| Usuario objetivo | `hamid` |
| Servicio objetivo | SSH (puerto 22) en `victima-linux` (192.168.100.20) |
| Reglas Wazuh involucradas | 5760 (autenticación fallida, nivel 5), 100010 (correlación personalizada, nivel 10) |
| Patrón de ataque | 6 intentos secuenciales, 1 conexión concurrente (`-t 1`), lista de contraseñas de 6 entradas |

## 4. Impacto

- **Prueba inicial (sin mitigación):** acceso SSH obtenido con éxito por el atacante usando credenciales válidas (`hamid` / contraseña real del laboratorio). Impacto equivalente a una toma de control de la cuenta del usuario.
- **Incidente final (con mitigación):** ningún acceso conseguido. El atacante quedó bloqueado a nivel de firewall antes de poder usar la contraseña correcta. No se detectó ninguna alerta de autenticación exitosa (regla 5715) durante este ataque.

## 5. Acciones de respuesta

1. **Detección:** regla por defecto de Wazuh (5760) identificó cada intento fallido correctamente desde el principio; el problema no era de detección sino de ausencia de correlación y respuesta.
2. **Contención automatizada:** se diseñó la regla 100010 para generar una alerta de alta severidad al detectar el patrón de fuerza bruta (4 fallos en 120s desde el mismo origen), y se ligó a una respuesta activa (`firewall-drop`) que bloquea la IP origen durante 5 minutos mediante `iptables` en el propio agente afectado.
3. **Incidencia durante la implementación:** el primer intento de bloqueo falló porque `victima-linux`, al ser una instalación mínima de Debian, no tenía `iptables` instalado. Esto se detectó revisando `/var/ossec/logs/active-responses.log` y se corrigió instalando el paquete, sin necesidad de cambiar la configuración de Wazuh.
4. **Verificación:** se repitió el ataque en condiciones controladas y se confirmó, con evidencia de `iptables -L -n -v` y del propio log de active response, que el bloqueo se aplica y se retira correctamente.

## 6. Lecciones aprendidas y recomendaciones

- **La detección por defecto no implica respuesta.** Wazuh, de fábrica, registra cada evento de autenticación fallida como una alerta aislada de severidad media, pero no correlaciona múltiples fallos ni actúa sobre ellos. Cualquier despliegue real necesita reglas de correlación personalizadas para los patrones de ataque que se quieran contener automáticamente.
- **Una respuesta activa depende de que el agente tenga las herramientas necesarias.** La orden de bloqueo se generó y se transmitió correctamente desde el primer momento; el fallo estuvo en una dependencia del sistema operativo del agente (`iptables` ausente). Recomendación: añadir la verificación de `iptables` (o el backend de firewall que se use) como parte del proceso de preparación de cualquier nuevo agente en este laboratorio.
- **El `ping` no es una prueba fiable de conectividad en esta red.** Durante la remediación se perdió tiempo diagnosticando una supuesta falta de internet que en realidad era un bloqueo de ICMP en la red del propio host; las pruebas de conectividad para instalar paquetes deben hacerse con herramientas que usen TCP real (`curl`/`wget`), no con `ping`.
- **Mejora futura:** considerar un segundo nivel de contención (por ejemplo, a nivel de red/firewall perimetral) para no depender únicamente del firewall del host afectado, y ampliar la regla de correlación para cubrir también fuerza bruta con usuarios inexistentes (regla 5710, vista en la Fase 3), actualmente fuera del alcance de la regla 100010.

## 7. Referencias

- `docs/05-fuerza-bruta-ssh.md` — ataque inicial sin correlación ni respuesta.
- `docs/06-correlacion-active-response.md` — diseño e implementación de la regla 100010 y la respuesta activa `firewall-drop`, incluyendo la resolución de la incidencia con `iptables`.
