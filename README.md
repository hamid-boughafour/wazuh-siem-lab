# Wazuh SIEM Lab

Laboratorio de ciberseguridad defensiva: despliegue de Wazuh, detección de ataques mapeados a MITRE ATT&CK y respuesta a incidentes en un entorno aislado.

## Objetivos

- Desplegar un SIEM funcional con agentes en Linux
- Detectar técnicas reales de ataque y documentar la evidencia
- Crear reglas personalizadas y respuesta activa
- Redactar informes de incidente como lo haría un analista SOC

## Entorno

| Componente | Versión |
|---|---|
| Wazuh | Pendiente |
| Sistema operativo | Debian 13 |
| Hipervisor | Pendiente |

## Escenarios de detección

| ID | Técnica MITRE ATT&CK | Estado | Informe |
|---|---|---|---|
| IR-001 | T1110 Fuerza bruta SSH | Pendiente | - |
| IR-002 | T1046 Escaneo de puertos | Pendiente | - |
| IR-003 | T1565 Modificación de ficheros críticos | Pendiente | - |
| IR-004 | T1190 Ataques web a Apache | Pendiente | - |

## Estado del proyecto

- [x] Fase 0: repositorio y estructura
- [ ] Fase 1: máquinas virtuales y red aislada
- [ ] Fase 2: instalación de Wazuh
- [ ] Fase 3: despliegue del agente
- [ ] Fase 4: escenarios de ataque y detección
- [ ] Fase 5: respuesta activa y regla personalizada
- [ ] Fase 6: cierre y versión 1.0

## Estructura del repositorio

- `docs/`: guía paso a paso de cada fase
- `config/`: reglas, decodificadores y configuración de Wazuh
- `scripts/`: scripts de apoyo
- `incident-reports/`: informes de incidente

## Aviso

Todas las pruebas se realizan en un laboratorio aislado y propio, sobre máquinas creadas para este proyecto.

## Autor

Abdelhamid Boughafour · [LinkedIn](https://www.linkedin.com/in/abdelhamid-boughafour/)
