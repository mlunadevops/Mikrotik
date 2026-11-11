# Guía de Configuración y Documentación Técnica
## Balanceo de Carga y Failover Jerárquico en MikroTik RB750Gr3 (4 WANs)

---

## 1. Resumen Ejecutivo y Especificaciones

Esta guía documenta la implementación de un esquema de **failover y balanceo de carga jerárquico guiado por capacidad de ancho de banda** sobre un enrutador **MikroTik RB750Gr3 (hEX)**. 

### Parámetros de Proveedores (WANs)
| Interfaz | Proveedor | Tipo | Ancho de Banda Contratado / Promedio | Prioridad | Distancia |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `ether1` | **WAN 1 - CANTV** | Principal | 600 Mbps | Alta (1) | `1` |
| `ether3` | **WAN 3 - Starlink** | Respaldo Alto | 200 Mbps | Media-Alta (2) | `2` |
| `ether2` | **WAN 2 - GALANET** | Terciario | 50 Mbps | Media-Baja (3) | `3` |
| `ether4` | **WAN 4 - CANTV 2** | Cuaternario | Por definir | Baja (4) | `4` |
| `ether5` | **LAN** | Red Local | Interna | N/A | N/A |

---

## 2. Definición de Interfaces

El primer paso dentro de RouterOS consiste en asignar un nombre identificable a cada interfaz física para evitar confusión durante la lectura del conjunto de reglas.

```routeros
/interface ethernet
set [ find default-name=ether1 ] name=ether1-WAN1-CANTV
set [ find default-name=ether2 ] name=ether2-WAN2-GALANET
set [ find default-name=ether3 ] name=ether3-WAN3-STARLINK
set [ find default-name=ether4 ] name=ether4-WAN4-CANTV2
set [ find default-name=ether5 ] name=ether5-LAN
```

## 3. Configuración de NAT (Masquerade)

Es indispensable permitir la traducción de direcciones de red (NAT) en cada una de las interfaces WAN independientes de modo que el tráfico pueda salir de la red local sin interrupciones.

```mikrotik
/ip firewall nat
add chain=srcnat out-interface=ether1-WAN1-CANTV action=masquerade comment="NAT CANTV 600M"
add chain=srcnat out-interface=ether3-WAN3-STARLINK action=masquerade comment="NAT Starlink 200M"
add chain=srcnat out-interface=ether2-WAN2-GALANET action=masquerade comment="NAT Galanet 50M"
add chain=srcnat out-interface=ether4-WAN4-CANTV2 action=masquerade comment="NAT CANTV Nueva"
```

## 4. Enrutamiento Recursivo con Control de Distancias

## 4. Enrutamiento Recursivo con Control de Distancias

Para garantizar el salto jerárquico según el ancho de banda contratado, implementamos Rutas Recursivas con comprobación activa (`check-gateway=ping`).

Las rutas hacia servidores DNS públicos sirven como objetivo de monitoreo continuo para verificar conectividad real a internet (no solo enlace físico contra el módem o router de la operadora).

> **Nota:** Reemplazar `[GW_WAN1]`, `[GW_WAN2]`, `[GW_WAN3]` y `[GW_WAN4]` por la dirección IP de la puerta de enlace enviada por tu proveedor (o asignada por DHCP).

/ip route
# 1. Host de monitoreo dedicado para cada WAN
add dst-address=1.1.1.1 gateway=[GW_WAN1] scope=10 comment="Ping Host WAN1"
add dst-address=8.8.8.8 gateway=[GW_WAN3] scope=10 comment="Ping Host WAN3"
add dst-address=9.9.9.9 gateway=[GW_WAN2] scope=10 comment="Ping Host WAN2"
add dst-address=208.67.222.222 gateway=[GW_WAN4] scope=10 comment="Ping Host WAN4"

# 2. Rutas por defecto ordenadas jerárquicamente por capacidad
add dst-address=0.0.0.0/0 gateway=1.1.1.1 target-scope=30 distance=1 check-gateway=ping comment="Ruta Principal WAN1 600M"
add dst-address=0.0.0.0/0 gateway=8.8.8.8 target-scope=30 distance=2 check-gateway=ping comment="Ruta Secundaria WAN3 200M"
add dst-address=0.0.0.0/0 gateway=9.9.9.9 target-scope=30 distance=3 check-gateway=ping comment="Ruta Terciaria WAN2 50M"
add dst-address=0.0.0.0/0 gateway=208.67.222.222 target-scope=30 distance=4 check-gateway=ping comment="Ruta Cuaternaria WAN4"
