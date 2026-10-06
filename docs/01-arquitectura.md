# 01 - Arquitectura y preparación del laboratorio

## Objetivo

Preparar un laboratorio aislado con tres máquinas virtuales: un servidor Wazuh (SIEM), una víctima Linux con el agente instalado y una máquina atacante para generar tráfico malicioso controlado.

## Equipo anfitrión

| Componente | Detalle |
|---|---|
| CPU | AMD Ryzen AI 7 350 (8 núcleos, 16 hilos) |
| RAM | 32 GB |
| Almacenamiento | SSD NVMe de 1 TB |
| Hipervisor | VMware Workstation 17.5.2 |

## Máquinas virtuales

| Máquina | Sistema | vCPU | RAM | Disco | Estado |
|---|---|---|---|---|---|
| `wazuh-server` | Ubuntu Server 24.04.5 LTS | 4 | 8 GB | 50 GB | Instalada |
| `victima-linux` | Debian 13.7.0 | 2 | 2 GB | 20 GB | Pendiente |
| `atacante` | Kali Linux 2026.2 | 2 | 3 GB | 40 GB | Pendiente |

Los discos son de crecimiento dinámico, así que ocupan menos espacio real que el indicado.

### Por qué estas versiones y estos recursos

- La documentación oficial de Wazuh recomienda Ubuntu 22.04 y 24.04 para los componentes centrales. Se eligió 24.04 LTS.
- Se usa la rama estable 4.14 de Wazuh. La rama 5.0 está en beta y se descartó.
- Para hasta 25 agentes, Wazuh recomienda 4 vCPU, 8 GiB de RAM y 50 GB de almacenamiento. El servidor sigue esa recomendación.

## Red

La red del laboratorio es una red solo anfitrión (host-only) llamada **VMnet10**, aislada de la red doméstica.

| Parámetro | Valor |
|---|---|
| Red | VMnet10 (host-only) |
| Subred | 192.168.100.0/24 |
| DHCP de VMware | Desactivado (IP fijas) |

| Equipo | IP |
|---|---|
| PC anfitrión (Windows) | 192.168.100.1 |
| `wazuh-server` | 192.168.100.10 |
| `victima-linux` | 192.168.100.20 (pendiente) |
| `atacante` | 192.168.100.30 (pendiente) |

Cada máquina lleva además una segunda tarjeta en NAT, **temporal**, solo para instalar sistemas y paquetes. Se desconectará antes de ejecutar los ataques para mantener el laboratorio aislado.

En `wazuh-server`, `ens33` corresponde a VMnet10 y `ens34` a la NAT. La ruta por defecto sale por `ens34`, y la red del laboratorio va por `ens33`.

Configuración de `ens33` (`/etc/netplan/60-lab.yaml`):

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.100.10/24
```

## Verificación de las imágenes ISO

Las ISO se descargaron desde las webs oficiales y se comprobó su suma SHA256 contra la publicada por cada proyecto.

| ISO | SHA256 | Resultado |
|---|---|---|
| ubuntu-24.04.5-live-server-amd64.iso | `97f3d7ffb032c3eb3b23d2c8be9cc76e60c2c1f2c0146ba5ba9fe01cafae0fd8` | Coincide |
| debian-13.7.0-amd64-netinst.iso | `a7ef94ac2fb9a7fec454552abd629b7cc9d5155c886165a45649f5ce6167e355` | Coincide |
| kali-linux-2026.2-installer-amd64.iso | `6dbefacc95e3b556c19c48e8bae39b8b505e2d3a1aba0bfb7ab62b036c3d2ba3` | Coincide |

## Decisiones de instalación de `wazuh-server`

- **Disco completo con ext4 y sin LVM**, para aprovechar los 50 GB en una única partición raíz.
- **OpenSSH instalado con autenticación por contraseña**, suficiente en un laboratorio local. Mejora prevista: pasar a claves SSH.
- **Sin snaps opcionales**, para mantener el sistema mínimo.
- Nombre de equipo `wazuh-server` y usuario no privilegiado con `sudo`.

## Incidencias y soluciones

| Incidencia | Causa | Solución |
|---|---|---|
| La descarga de la ISO de Ubuntu pesaba unos KB | El navegador guardó el archivo `.torrent` con nombre `.iso` | Descarga directa con `curl` y verificación SHA256 |
| El guion no se escribía en el instalador | El instalador usaba distribución de teclado inglesa | Se usó la tecla equivalente y después `loadkeys es` |
| El `ping` a internet no respondía | La NAT filtra ICMP | Prueba con `curl -I` sobre HTTP, que sí respondió `200 OK` |

## Estado de la fase

- [x] Red VMnet10 creada y DHCP desactivado
- [x] ISO descargadas y verificadas
- [x] `wazuh-server` instalado con red configurada
- [ ] Conexión SSH desde Windows a 192.168.100.10
- [ ] Instantánea inicial de `wazuh-server`
- [ ] `victima-linux` instalada
- [ ] `atacante` instalada
