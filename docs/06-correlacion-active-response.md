# 06 - Correlación de fuerza bruta y respuesta activa (active response)

## Objetivo

Configurar Wazuh para que, tras varios intentos fallidos de SSH desde el mismo origen, genere una alerta de correlación de nivel alto y bloquee automáticamente la IP del atacante mediante `firewall-drop`, cerrando la brecha detectada en la Fase 5 (detección sin correlación ni respuesta).

## Regla de correlación (local_rules.xml)

En `wazuh-server`, se añadió una regla personalizada en `/var/ossec/etc/rules/local_rules.xml`:

```xml
<!-- Local rules - Laboratorio SIEM Wazuh -->
<!-- Autor: Hamid Boughafour -->

<group name="local,syslog,sshd,">

  <rule id="100010" level="10" frequency="4" timeframe="120" ignore="120">
    <if_matched_sid>5760</if_matched_sid>
    <same_source_ip />
    <description>sshd: fuerza bruta detectada - multiples intentos fallidos desde el mismo origen (usuario valido, contraseña incorrecta).</description>
    <mitre>
      <id>T1110.001</id>
    </mitre>
    <group>authentication_failures,brute_force,</group>
  </rule>

</group>
```

- `if_matched_sid>5760`: se basa en la regla de "contraseña incorrecta, usuario válido" vista en la Fase 5.
- `frequency="4" timeframe="120"`: se dispara si la 5760 se repite 4 veces en 120 segundos.
- `same_source_ip`: exige que los 4 fallos vengan de la misma IP, para no mezclar atacantes distintos.
- `ignore="120"`: tras disparar, ignora nuevas coincidencias durante 120s, evitando alertas repetidas en bucle.
- `level="10"`: muy por encima del nivel 5 de la regla 5760, para que la correlación destaque como el evento importante.

## Active response (ossec.conf)

En `/var/ossec/etc/ossec.conf` del manager se añadieron dos bloques `<ossec_config>` nuevos:

```xml
<ossec_config>
  <command>
    <name>firewall-drop</name>
    <executable>firewall-drop</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <active-response>
    <disabled>no</disabled>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>100010</rules_id>
    <timeout>300</timeout>
  </active-response>
</ossec_config>
```

- `<location>local</location>`: el comando se ejecuta en el **agente** que generó la alerta (`victima-linux`), no en el manager.
- `<rules_id>100010</rules_id>`: solo se activa cuando salta la regla de correlación, no con cada fallo individual.
- `<timeout>300</timeout>`: el bloqueo se retira automáticamente a los 5 minutos.

Validación de sintaxis tras cada cambio:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
echo $?   # 0 = configuración correcta
```

## Primer intento: fallo por dependencia (iptables)

Al relanzar el ataque de Hydra, la regla 100010 se disparó correctamente, pero el active response falló:

```
2026/10/13 18:03:48 active-response/bin/firewall-drop: The iptables file 'iptables' is not accessible: No such file or directory (2)
```

**Causa:** `victima-linux` es una instalación mínima de Debian (Fase 3) y no incluye `iptables` por defecto. El resultado fue que el manager detectó la fuerza bruta, disparó la correlación, mandó la orden al agente... pero el agente no tenía la herramienta para ejecutarla. Hydra completó el ataque sin obstáculos y encontró la contraseña válida, igual que en la Fase 5.

## Incidencia añadida: instalación de iptables en un laboratorio aislado

Para instalar `iptables` hizo falta reconectar temporalmente la tarjeta NAT de `victima-linux` (desconectada desde el aislamiento de la Fase 5). Al hacerlo, surgieron dos despistes que merece la pena documentar:

| Incidencia | Causa | Solución |
|---|---|---|
| `ping 8.8.8.8` fallaba tanto en las VMs como en el host Windows tras reconectar la NAT | La red del usuario bloquea ICMP saliente; no es un fallo de conectividad real | Se verificó con `curl`/`wget` sobre HTTPS (TCP) en vez de `ping` (ICMP), confirmando que sí había salida a internet |
| Se intentó copiar el `.deb` de `iptables` desde `atacante` (Kali) a `victima-linux` (Debian) por la red del laboratorio | Las versiones de librerías de Kali (`libxtables12` 1.8.13) no coinciden con las de Debian trixie (1.8.11), y faltaban `libip4tc2`, `libip6tc2`, `libnetfilter-conntrack3`, `libnfnetlink0` | Se purgó el paquete mal instalado (`dpkg --purge iptables`) y se instaló con el propio `apt` de `victima-linux`, que resuelve las versiones correctas desde sus propios repositorios |

Tras instalar `iptables` con `apt install -y iptables`, se **desconectó de nuevo la NAT** en ambas VMs (`atacante` y `victima-linux`), restaurando el aislamiento del laboratorio antes de repetir la prueba.

## Prueba final: ataque con active response ya funcional

Con `iptables` instalado y el laboratorio aislado otra vez, se repitió el mismo ataque de la Fase 5:

```bash
hydra -l hamid -P /home/hamid/passwords.txt ssh://192.168.100.20 -t 1 -V
```

Resultado:

```
[ATTEMPT] ... pass "123456" - 1 of 6
[ATTEMPT] ... pass "password" - 2 of 6
[ATTEMPT] ... pass "qwerty" - 3 of 6
[ATTEMPT] ... pass "admin123" - 4 of 6
[ATTEMPT] ... pass "letmein" - 5 of 6
[ATTEMPT] ... pass "victima" - 6 of 6
1 of 1 target completed, 0 valid password found
Hydra ... finished at 2026-10-13 18:49:05
```

`active-responses.log` en `victima-linux` muestra el bloqueo aplicándose en el mismo segundo del cuarto fallo:

```
2026/10/13 18:48:00 active-response/bin/firewall-drop: Starting
2026/10/13 18:48:00 ... "command":"add" ... "rules_id":"100010" ... "firedtimes":3
2026/10/13 18:48:00 active-response/bin/firewall-drop: Ended
```

Y `iptables -L -n -v` confirma la regla aplicada:

```
Chain INPUT (policy ACCEPT ...)
 pkts bytes target     prot opt in     out     source               destination
   20  2000 DROP       all  --  *      *       192.168.100.30       0.0.0.0/0
```

## Comparativa Fase 5 vs Fase 6

| | Fase 5 (sin correlación) | Fase 6 (con regla 100010 + active response) |
|---|---|---|
| Alertas generadas | 5 individuales (nivel 5, regla 5760) | 5 individuales + 1 de correlación (nivel 10, regla 100010) |
| Bloqueo de la IP atacante | No | Sí, automático vía `firewall-drop` |
| Duración del ataque completo | ~18 segundos | ~75 segundos (las conexiones tras el bloqueo hacen timeout) |
| Contraseñas válidas encontradas | 1 de 1 (`victima`) | **0 de 1** |
| Momento del bloqueo | — | Tras el 4º intento fallido, antes de llegar a la contraseña correcta |
| Retirada del bloqueo | — | Automática a los 300s (`timeout`), verificado con `iptables -L` |

## Hallazgo clave

La regla por defecto de Wazuh (5760) detecta cada fallo de autenticación como un evento aislado, pero no correlaciona varios fallos en un patrón de ataque ni actúa sobre él. Añadiendo una regla de correlación por frecuencia (`frequency`/`timeframe`/`same_source_ip`) y un active response (`firewall-drop`) ligado a esa regla, Wazuh pasa de ser puramente detectivo a tener una respuesta automática: en la prueba final, el atacante fue bloqueado **antes** de poder usar la contraseña correcta, reduciendo de 1 contraseña comprometida a 0.

El punto débil real no estuvo en la lógica de Wazuh, sino en una dependencia del sistema operativo (`iptables` ausente en la instalación mínima de Debian): la detección y la orden de bloqueo funcionaron perfectamente desde el primer intento, pero el mecanismo de aplicación (el firewall del propio agente) no estaba disponible. Es un recordatorio de que un active response depende tanto de la configuración de Wazuh como de que el agente tenga las herramientas necesarias para ejecutar la acción.

## Estado de la fase

- [x] Regla de correlación 100010 creada y validada (`wazuh-analysisd -t`)
- [x] Active response `firewall-drop` configurado en el manager, ligado a la regla 100010
- [x] Incidencia detectada y resuelta: `iptables` no instalado en `victima-linux`
- [x] Laboratorio reaislado (NAT desconectada de nuevo) tras instalar la dependencia
- [x] Ataque de fuerza bruta repetido con el active response funcionando
- [x] Bloqueo confirmado en `iptables -L` y en `active-responses.log`
- [x] Retirada automática del bloqueo verificada tras el `timeout` de 300s
- [x] Comparativa Fase 5 vs Fase 6 documentada (0 vs 1 contraseñas comprometidas)
- [ ] Capturas de pantalla del panel (alerta de correlación, nivel 10, regla 100010)
- [ ] Instantánea de las máquinas tras la Fase 6
