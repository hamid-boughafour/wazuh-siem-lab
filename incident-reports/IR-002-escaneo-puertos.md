# IR-002 - Informe de incidente: escaneo de puertos contra victima-linux

| Campo | Valor |
|---|---|
| ID de incidente | IR-002 |
| Técnica MITRE ATT&CK | T1046 (Network Service Discovery) |
| Táctica asociada | Discovery |
| Severidad | Informativa (reconocimiento, sin explotación) |
| Sistema afectado | `victima-linux` (192.168.100.20) |
| Origen | `atacante` / Kali Linux (192.168.100.30) |
| Analista | Abdelhamid Boughafour |
| Fecha del informe | 15/10/2026 |
| Estado | Cerrado — brecha de detección documentada, sin remediar en esta fase |

## 1. Resumen ejecutivo

Se lanzó un escaneo de puertos con `nmap` (`-sS -sV`) contra `victima-linux` desde la IP `192.168.100.30`, como fase de reconocimiento previa a un posible ataque. El escaneo identificó el único puerto abierto del sistema (22/SSH). Pese a que la conexión usada para detectar la versión del servicio quedó registrada en el log del propio sistema operativo, **Wazuh no generó ninguna alerta ni indexó el evento**, dejando el reconocimiento completamente invisible desde el panel de monitorización. Se trata de una limitación conocida y esperable de la detección basada en logs de host, no de un fallo de configuración. El incidente se cierra como hallazgo documentado, sin acción de remediación en esta fase.

## 2. Línea temporal

| Hora (CEST) | Evento |
|---|---|
| ~18:08 | Verificado el aislamiento del laboratorio (sin salida a internet, con conectividad interna) |
| 18:09 | Se lanza `sudo nmap -sS -sV 192.168.100.20` desde `atacante` |
| 18:09:59 | `sshd-session` en `victima-linux` registra `Connection closed by 192.168.100.30 port 47838` (conexión real abierta por `-sV` para leer el banner del servicio SSH) |
| 18:09-18:10 | Escaneo finalizado: 1 puerto abierto (22/tcp, OpenSSH 10.0p2), 999 cerrados |
| — | Revisión en Wazuh (Threat Hunting → Events): sin resultados para ese rango horario, con o sin filtro por IP origen |
| — | Revisión directa en `victima-linux` (`journalctl -t sshd-session`): confirma que el evento sí existe en el sistema |

## 3. Indicadores de compromiso (IOCs)

| Tipo | Valor |
|---|---|
| IP origen (atacante) | 192.168.100.30 |
| Herramienta | nmap 7.99, técnica SYN scan + detección de versión (`-sS -sV`) |
| Sistema objetivo | `victima-linux` (192.168.100.20) |
| Puerto identificado | 22/tcp (SSH), OpenSSH 10.0p2 |
| Log generado en el host | `sshd-session[1711]: Connection closed by 192.168.100.30 port 47838` |
| Alertas generadas en Wazuh | Ninguna |

## 4. Impacto

Ninguno directo: un escaneo de puertos no compromete el sistema por sí mismo, es una fase de reconocimiento. El impacto real es **la pérdida de visibilidad**: si este reconocimiento hubiera sido el primer paso de un ataque dirigido (por ejemplo, previo a explotar una vulnerabilidad del servicio SSH detectado), el equipo de seguridad no habría tenido ninguna alerta temprana para investigar o anticiparse.

## 5. Análisis técnico

- El escaneo SYN (`-sS`) no completa la conexión TCP, por lo que normalmente no llega a generar ningún log a nivel de aplicación (`sshd` nunca "ve" la conexión, solo el núcleo del sistema operativo).
- El modificador `-sV` sí obligó a `nmap` a completar una conexión real al único puerto abierto, para leer la versión del servicio. Esa conexión quedó registrada como `Connection closed by [IP] port [N]`.
- Wazuh no generó ninguna alerta porque **no existe ninguna regla en el conjunto por defecto que reconozca ese mensaje como sospechoso**. No es un fallo de autenticación, ni un usuario inválido, ni ningún patrón que el sistema de reglas esté buscando — es, a todos los efectos, tráfico "normal" desde la perspectiva de las reglas actuales.
- Se confirmó mediante `journalctl` directamente en `victima-linux` que el evento sí ocurrió y quedó registrado a nivel de sistema operativo, descartando que se tratara de un problema de recolección de logs por parte del agente.

## 6. Acciones de respuesta

No se tomó ninguna acción de bloqueo o contención en este incidente: se decidió, de forma consciente, documentar la brecha de detección tal como se encontró, sin ampliar el alcance de esta fase a construir una solución (a diferencia de IR-001, donde sí se cerró el hueco con una regla de correlación y respuesta activa). Los motivos:

- `victima-linux` solo tiene un puerto/servicio activo, lo que limita mucho el volumen de señal disponible para construir una regla de correlación fiable basada únicamente en logs de `sshd` (un enfoque por frecuencia, como el de IR-001, necesitaría varios eventos del mismo tipo para distinguir un patrón real de una conexión normal, y aquí solo se genera uno).
- Una detección fiable de escaneos de red requiere, por diseño, una fuente de datos distinta a los logs de aplicación: típicamente logs de firewall con todas las conexiones entrantes (abiertas y cerradas, en cualquier puerto) o una herramienta de detección de red (NIDS) como Suricata, que analiza el tráfico directamente en lugar de depender de que una aplicación registre algo.

## 7. Lecciones aprendidas y recomendaciones

- **Detectar y vigilar son cosas distintas según la capa.** Wazuh, basado en logs de host, es muy eficaz contra patrones que las aplicaciones registran explícitamente (intentos de login, cambios de configuración), pero tiene un punto ciego estructural ante actividad de red que nunca llega a tocar una aplicación — como un escaneo sigiloso.
- **Un log ausente en el panel no siempre significa que el log no exista.** Antes de concluir "esto no se ve", conviene comprobar la fuente original (en este caso, `journalctl` en el propio host) para distinguir entre "no se generó ningún registro" y "se generó pero no se alertó ni indexó".
- **Recomendación para una futura fase:** incorporar una fuente de datos de red (logs de `iptables` con registro de conexiones entrantes, o un NIDS como Suricata) específicamente para cubrir este tipo de reconocimiento, en lugar de forzar una regla poco fiable sobre logs de aplicación que no fueron pensados para ese propósito.

## 8. Referencias

- `docs/07-escaneo-puertos-nmap.md` — procedimiento completo del escaneo y la verificación de detección.
- `docs/06-correlacion-active-response.md` — antecedente directo: metodología usada para cerrar una brecha similar (IR-001) cuando sí había señal suficiente para una regla de correlación fiable.
