# Guía Completa de Configuración MikroTik (RouterOS v7)
## Enrutamiento Avanzado, Failover y Conmutación por Umbrales de Ancho de Banda

---

## 1. Resumen del Estado de la Configuración

### Lo que ya está configurado correctamente (¡Vas por buen camino!):
- **Interfaces nombradas:** `CANTV` (ether1), `GALANET` (ether2), `STARLINK` (ether3), `ISP` (ether4) y `LAN` (ether5).
- **Red Local y DHCP Server:** Red `192.168.1.0/24` con DHCP Server y reservas de IP estáticas asignadas correctamente.
- **Tablas FIB creadas:** `usar-cantv` y `usar-galanet` para enrutamiento aislado.
- **Enrutamiento y Mangle base:** Regla de marcado para enviar tráfico local a la tabla `usar-cantv`.
- **Clientes DHCP:** Clientes configurados en `CANTV` (`add-default-route=no`) y `GALANET`.
- **NAT:** Regla genérica de Masquerade.

---

### Lo que FALTA por configurar / corregir:
1. **Creación de la tabla FIB para Starlink:** Tabla `usar-starlink` para mantener el tercer enlace preparado.
2. **NIF/NAT independiente:** Ajustar el NAT general a reglas por interfaz saliente para evitar pérdidas de paquetes al conmutar.
3. **Script de monitoreo por umbrales:** Script automático que mida el tráfico de `CANTV` y active o desactive la regla de Mangle según los límites asignados (bajar de 95 Mbps / subir de 150 Mbps).
4. **Scheduler (Programador):** Automatización del script cada 10 segundos.

---

## 2. Comandos para Copiar y Pegar en MikroTik Terminal

A continuación tienes la guía completa paso a paso. Puedes copiar y pegar cada bloque directamente en la consola (`Terminal`) de tu MikroTik.

### PASO 1: Ajuste de Interfaces y Tablas de Enrutamiento (FIB)
```mikrotik
# Asignar nombres a interfaces físicas
/interface ethernet set [ find default-name=ether1 ] name=CANTV
/interface ethernet set [ find default-name=ether2 ] name=GALANET
/interface ethernet set [ find default-name=ether3 ] name=STARLINK
/interface ethernet set [ find default-name=ether4 ] name=ISP
/interface ethernet set [ find default-name=ether5 ] name=LAN

# Crear tablas de enrutamiento aisladas (FIB)
/routing table add fib name=usar-cantv
/routing table add fib name=usar-galanet
/routing table add fib name=usar-starlink
```

---

### PASO 2: Red Local (LAN) y DHCP Server
```mikrotik
/ip pool add name=dhcp_pool_lan ranges=192.168.1.2-192.168.1.239
/ip dhcp-server add address-pool=dhcp_pool_lan interface=LAN lease-time=10h name=dhcp_lan
/ip address add address=192.168.1.1/24 comment=LAN interface=LAN network=192.168.1.0
/ip dhcp-server network add address=192.168.1.0/24 dns-server=8.8.8.8,1.1.1.1 gateway=192.168.1.1
```

---

### PASO 3: Clientes DHCP de los Proveedores WAN

interface=CANTV: Le indica al router que obtenga IP dinámica por DHCP a través del puerto físico llamado CANTV.

add-default-route=no (La parte más crítica): Por defecto, cuando un proveedor te asigna IP por DHCP, el router inyecta automáticamente una ruta predeterminada (0.0.0.0/0) en la **tabla global (main)**. Al ponerlo en no, bloqueas que CANTV meta sus propias reglas por defecto en la tabla principal, lo que evita que rompa o compita con la navegación de tus otros enlaces. Esto te deja el camino libre para enrutar CANTV manualmente solo por su propia tabla (usar-cantv).

use-peer-dns=no: Evita que el MikroTik acepte y sobrescriba automáticamente los servidores DNS que te mande el módem de CANTV, permitiéndote usar los DNS personalizados que tú prefieras (como 8.8.8.8 o 1.1.1.1).

comment=CANTV_WAN_DHCP: **Una etiqueta** de identificación interna para ubicar rápidamente esta regla.



```mikrotik
# CANTV (Sin inyectar ruta por defecto a la tabla main)
/ip dhcp-client add add-default-route=no comment=CANTV_WAN_DHCP interface=CANTV use-peer-dns=no

# GALANET
/ip dhcp-client add comment=GALANET_WAN_DHCP interface=GALANET

# STARLINK
/ip dhcp-client add comment=STARLINK_WAN_DHCP interface=STARLINK
```

---

### PASO 4: Rutas Estáticas por Tabla
```mikrotik
# Ruta para GALANET en su tabla dedicada
/ip route add disabled=no distance=1 dst-address=0.0.0.0/0 gateway=172.19.24.101 routing-table=usar-galanet scope=30 target-scope=10

# Ruta para CANTV en su tabla dedicada (gateway asignado por el módem CANTV)
/ip route add distance=1 gateway=192.168.100.1 routing-table=usar-cantv
```

---

### PASO 5: Firewall NAT Independiente por Interfaz
```mikrotik
# Eliminar reglas NAT genéricas previas
/ip firewall nat remove [find]

# Crear reglas de Masquerade por interfaz de salida
/ip firewall nat add action=masquerade chain=srcnat out-interface=CANTV comment="NAT CANTV"
/ip firewall nat add action=masquerade chain=srcnat out-interface=GALANET comment="NAT GALANET"
/ip firewall nat add action=masquerade chain=srcnat out-interface=STARLINK comment="NAT STARLINK"
```

---

### PASO 6: Regla de Mangle (Marcado de Enrutamiento)
```mikrotik
# Eliminar reglas mangle previas para evitar duplicados
/ip firewall mangle remove [find]

# Marcar tráfico LAN para forzar salida por la tabla CANTV
/ip firewall mangle add action=mark-routing chain=prerouting comment="Ruta por defecto hacia CANTV" \
    dst-address=!192.168.1.0/24 new-routing-mark=usar-cantv passthrough=no src-address=192.168.1.0/24
```

---

### PASO 7: Script de Monitoreo Dinámico por Umbral de Ancho de Banda
> **Lógica:** El script mide la velocidad en la interfaz `CANTV`. Si el rendimiento baja de **95 Mbps**, desactiva la regla de Mangle para que todo el tráfico caiga automáticamente hacia `GALANET`. Cuando la interfaz `CANTV` supera los **150 Mbps**, vuelve a activar la regla de Mangle para restablecer a CANTV como enlace principal.

```mikrotik
/system script add name="check_cantv_bandwidth" source="
:local iface \"CANTV\";
:local rx1 [/interface get \$iface rx-byte];
:local tx1 [/interface get \$iface tx-byte];
:delay 2s;
:local rx2 [/interface get \$iface rx-byte];
:local tx2 [/interface get \$iface tx-byte];

:local rxBps (\$rx2 - \$rx1);
:local txBps (\$tx2 - \$tx1);
:local totalBps (\$rxBps + \$txBps);
:local totalMbps (\$totalBps * 8 / 2000000);

:local mangleId [/ip firewall mangle find comment=\"Ruta por defecto hacia CANTV\"];
:local mangleDisabled [/ip firewall mangle get \$mangleId disabled];

:if (\$totalMbps < 95 && \$mangleDisabled = false) do={
    /ip firewall mangle set \$mangleId disabled=yes;
    :log warning (\"CANTV por debajo del umbral (\" . \$totalMbps . \" Mbps). Conmutando trafico a GALANET.\");
}

:if (\$totalMbps > 150 && \$mangleDisabled = true) do={
    /ip firewall mangle set \$mangleId disabled=no;
    :log info (\"CANTV recuperado (\" . \$totalMbps . \" Mbps). Restableciendo CANTV como principal.\");
}
"
```

---

### PASO 8: Programador Automático (Scheduler)
```mikrotik
# Configurar la ejecución automática del script cada 10 segundos
/system scheduler add interval=10s name="sched_check_bandwidth" on-event=check_cantv_bandwidth policy=read,write,test start-time=startup

# Ajustar hora y zona horaria del sistema
/system clock set time-zone-name=America/Caracas
```

---

## 3. Comprobación y Verificación

Una vez aplicados los comandos, puedes verificar la correcta ejecución mediante:

1. **Estado del Script y Logs:**
   ```mikrotik
   /log print follow-only
   ```
   *Verifica en tiempo real los mensajes emitidos cuando cambia la velocidad o commuta el enlace.*

2. **Revisión de Mangle:**
   ```mikrotik
   /ip firewall mangle print
   ```
   *Comprueba que la regla alterne entre habilitada/deshabilitada según la actividad de tráfico.*
