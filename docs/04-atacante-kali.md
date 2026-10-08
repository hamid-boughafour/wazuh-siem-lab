# 04 - Máquina atacante (Kali Linux)

## Objetivo

Crear la máquina `atacante`, conectarla a la red aislada del laboratorio y dejarla preparada para lanzar ataques controlados contra `victima-linux`.

## Máquina `atacante`

| Parámetro | Valor |
|---|---|
| Sistema | Kali Linux 2026.2 (instalador, versión de la ISO sin actualizar) |
| Recursos | 2 vCPU, 3 GB de RAM, 40 GB de disco |
| Red 1 | VMnet10 (`eth0`), IP fija `192.168.100.30/24` |
| Red 2 | NAT (`eth1`), temporal, por DHCP |
| Escritorio | Xfce (el de Kali por defecto) |
| Herramientas | Conjuntos `top10` y `default` de Kali |
| Usuario | `hamid` |

Se creó una máquina nueva con la ISO verificada de Kali 2026.2 en lugar de reutilizar una instalación anterior, para que el laboratorio sea reproducible: cualquiera puede montarlo con las versiones documentadas.

No se actualizó el sistema con `apt full-upgrade`, de modo que la versión de Kali coincide con la documentada.

### Particionado

Disco completo, una única partición ext4 y swap, sin LVM, igual que en las otras máquinas.

## Configuración de red

Kali gestiona la red con NetworkManager, y el instalador solo crea un perfil para la tarjeta NAT (`eth1`). La tarjeta del laboratorio se configuró con un perfil nuevo:

```bash
sudo nmcli con add type ethernet ifname eth0 con-name lab ipv4.method manual ipv4.addresses 192.168.100.30/24
sudo nmcli con up lab
```

No se define puerta de enlace en `eth0`; la salida a internet sigue yendo por `eth1`.

Comprobación:

```
eth0   UP   192.168.100.30/24
eth1   UP   192.168.244.132/24
```

Conectividad con el resto del laboratorio:

```bash
ping -c 3 192.168.100.20   # victima-linux
ping -c 3 192.168.100.10   # wazuh-server
```

Resultado: todos los paquetes recibidos en ambos casos.

## Acceso por SSH

El servicio SSH viene instalado en Kali pero desactivado. Se activó con:

```bash
sudo systemctl enable --now ssh
```

Desde el equipo anfitrión:

```bash
ssh hamid@192.168.100.30
```

## Herramientas de ataque verificadas

| Herramienta | Uso en el proyecto | Versión |
|---|---|---|
| `nmap` | Escaneo de puertos y servicios (reconocimiento) | 7.99 |
| `hydra` | Fuerza bruta contra SSH | instalada |

## Nota sobre la ética y el alcance

Todos los ataques se lanzan **solo** contra `victima-linux`, dentro de la red aislada `VMnet10` y con el consentimiento del propietario del laboratorio. Antes de ejecutarlos se desconectará la tarjeta NAT para que el tráfico no pueda salir del laboratorio.

## Incidencias y soluciones

| Incidencia | Causa | Solución |
|---|---|---|
| El instalador mostraba `eth0` y `eth1` en lugar de `ens33` y `ens34` | Kali nombra las interfaces de forma distinta a Debian y Ubuntu | Se mantuvo el mismo orden: `eth0` = VMnet10, `eth1` = NAT |
| `eth0` no tenía IP tras la instalación | NetworkManager solo crea perfil para la tarjeta con DHCP | Perfil nuevo `lab` con IP manual |

## Estado de la fase

- [x] `atacante` instalada (Kali 2026.2)
- [x] IP fija `192.168.100.30` en VMnet10
- [x] Conectividad con `victima-linux` y `wazuh-server`
- [x] SSH activo y herramientas verificadas
- [x] Instantánea `01-kali-base-red-ok`
- [ ] Primer ataque: escaneo de puertos con `nmap`
- [ ] Segundo ataque: fuerza bruta SSH con `hydra`
