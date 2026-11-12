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

comment=CANTV_WAN_DHCP: **Una etiqueta** de identificación interna para ubicar rápidamente esta regla.

**3.1)** CANTV (Sin inyectar ruta por defecto a la tabla main)

**interface=CANTV**: Obtenga IP por DHCP en puerto físico llamado CANTV.

**add-default-route=no** (La parte más crítica): Por defecto, la configuracion IP por DHCP, se inyecta automáticamente una ruta  (0.0.0.0/0) en la **tabla global (main)**. 
Al ponerlo en **no**, bloqueas que la config DHCP de CANTV se agregue en la tabla principal, lo que evita que rompa o compita con la navegación de tus otros enlaces. 
Esto te deja el camino libre para enrutar CANTV manualmente solo por su propia tabla **(usar-cantv)**.

**use-peer-dns=no:** Evita que el MikroTik sobrescriba los servidores DNS DHCP de CANTV, permitiéndote usar los DNS personalizados que tú prefieras (como 8.8.8.8 o 1.1.1.1).

```mikrotik
/ip dhcp-client add add-default-route=no comment=CANTV_WAN_DHCP interface=CANTV use-peer-dns=no
```

**3.2)** GALANET: Inyecta ruta por defecto en 'main' con Prioridad 1 (Respaldo Inmediato)

A diferencia de CANTV, aquí no se especificó **`add-default-route=no`**. Esto significa que Galanet opera de forma estándar: recibe su IP por DHCP y sí inyecta su ruta por defecto en la tabla global **(`main`)**, sirviendo como la pasarela base o respaldo predeterminado del sistema.

```mikrotik
/ip dhcp-client add comment=GALANET_WAN_DHCP default-route-distance=1 interface=GALANET
```

**3.3)** STARLINK Inyecta ruta por defecto en 'main' con Prioridad 2 (Respaldo de Emergencia)

A diferencia de CANTV, aquí no se especificó **`add-default-route=no`**. Esto significa que Galanet opera de forma estándar: recibe su IP por DHCP y sí inyecta su ruta por defecto en la tabla global **(`main`)**, sirviendo como la pasarela base o respaldo predeterminado del sistema.

```mikrotik
/ip dhcp-client add comment=STARLINK_WAN_DHCP default-route-distance=2 interface=STARLINK
```

**PASO 4: Rutas Estáticas por Tabla:**

**Ruta para GALANET y STARLINK:** No se requiere ninguna ruta estática manual. Al haber configurado sus clientes DHCP en los pasos anteriores sin bloquear la ruta por defecto, ambos proveedores inyectan y actualizan sus pasarelas de forma completamente automática en la tabla global **(main)**.

**4.1)**  Ruta para CANTV en su tabla dedicada (gateway asignado por el módem CANTV)

```mikrotik
/ip route add distance=1 gateway=CANTV routing-table=usar-cantv
```
---

### PASO 5: Firewall NAT Independiente por Interfaz


Eliminar reglas NAT genéricas previas

```mikrotik
/ip firewall nat remove [find]
```


**Crear reglas de Masquerade por interfaz de salida:**

```mikrotik
/ip firewall nat add action=masquerade chain=srcnat out-interface=CANTV comment="NAT CANTV"
/ip firewall nat add action=masquerade chain=srcnat out-interface=GALANET comment="NAT GALANET"
/ip firewall nat add action=masquerade chain=srcnat out-interface=STARLINK comment="NAT STARLINK"
```

---

**PASO 6:** Regla de Mangle (Marcado de Enrutamiento):

**¿Para qué se utiliza el Mangle en este escenario?**

En MikroTik, la tabla de enrutamiento principal (`main`) normalmente decide por dónde sale todo basándose únicamente en las rutas por defecto. Sin embargo, cuando tienes múltiples proveedores de internet (Multi-WAN) y quieres que los paquetes de tus dispositivos locales viajen por un camino específico (por ejemplo, por CANTV en lugar de Galanet o Starlink), las reglas normales de enrutamiento no bastan.

Aquí es donde entra **Mangle**: su función es interceptar los paquetes al vuelo y "etiquetarlos" o marcarles una ruta específica antes de que el router decida por dónde enviarlos.

**Desglose detallado de cada parte de tu comando:**

* `/ip firewall mangle add`: Le indica al MikroTik que cree una nueva regla de marcado.
* `chain=prerouting`: Es el momento exacto en el que actúa la regla. Se ejecuta antes de que el router tome la decisión de enrutamiento, interceptando los paquetes apenas entran desde tu red local hacia el MikroTik.
* `src-address=192.168.1.0/24`: Filtra y aplica la regla únicamente a los paquetes que provienen de tu red local (tus computadoras, teléfonos, etc.).
* `dst-address=!192.168.1.0/24`: El signo de exclamación (`!`) significa "diferente de". Esto le dice al router: aplica esta marca a todo el tráfico, excepto si va dirigido a la misma red local. (Esto es vital para que las impresoras de la red o el intercambio de archivos locales sigan funcionando sin ser desviados a internet).
* `action=mark-routing`: Es la acción que debe realizar. Le asigna una etiqueta de enrutamiento especial al paquete.
* `new-routing-mark=usar-cantv`: Es el nombre de la etiqueta que se le imprime al paquete. Como esta etiqueta coincide con la tabla FIB `usar-cantv` que creamos antes, el MikroTik dice: *"Este paquete tiene la marca 'usar-cantv', así que debo buscar su salida única y exclusivamente en la tabla de enrutamiento de CANTV"*.
* `passthrough=no`: Indica que esta es la última regla de Mangle que debe evaluar este paquete (detiene el procesamiento posterior para ahorrar recursos del router).
* `comment="Ruta por defecto hacia CANTV"`: Una etiqueta informativa interna para identificar fácilmente la regla en la consola.

 **Eliminar reglas mangle previas para evitar duplicados**

```mikrotik
/ip firewall mangle remove [find]
```

**Marcar tráfico LAN para forzar salida por la tabla CANTV**

```mikrotik
/ip firewall mangle add action=mark-routing chain=prerouting comment="Ruta por defecto hacia CANTV" \
    dst-address=!192.168.1.0/24 new-routing-mark=usar-cantv passthrough=no src-address=192.168.1.0/24
```

---

### PASO 7: Script de Monitoreo Dinámico por Umbral de Ancho de Banda
> **Lógica:** El script mide la velocidad en la interfaz `CANTV`. Si el rendimiento baja de **95 Mbps**, desactiva la regla de Mangle para que todo el tráfico caiga automáticamente hacia `GALANET`. Cuando la interfaz `CANTV` supera los **150 Mbps**, vuelve a activar la regla de Mangle para restablecer a CANTV como enlace principal.


### Interacción entre la regla de Mangle y el script (Failover Automatizado)

La interacción entre la regla de Mangle y el script es el mecanismo exacto que permite automatizar el *failover*. Trabajan en equipo como un interruptor de luz inteligente: el script es el sensor y la regla de Mangle es el interruptor físico.

Así es como se comunican paso a paso:

**1.** El estado normal (Todo funciona por CANTV)
* La regla de Mangle está habilitada (`disabled=no`).
* Por lo tanto, intercepta todo el tráfico de tu red local (`192.168.1.0/24`) y le coloca la etiqueta `new-routing-mark=usar-cantv`.
* El MikroTik lee esa etiqueta y envía todo el tráfico a través del enlace principal de CANTV.

**2.** ¿Qué pasa cuando CANTV cae o baja del umbral (< 95 Mbps)?
Cada 10 segundos, el *Scheduler* ejecuta el script de monitoreo. El script hace lo siguiente:
1. Mide cuántos bytes están pasando por la interfaz CANTV en ese instante y calcula la velocidad actual en Mbps.
2. Revisa la regla de Mangle (identificándola mediante su comentario `"Ruta por defecto hacia CANTV"`) para ver si está activa o apagada.
3. Si el cálculo da menos de 95 Mbps y la regla de Mangle está encendida, el script ejecuta la orden:

```mikrotik
/ip firewall mangle set [find comment="Ruta por defecto hacia CANTV"] disabled=yes


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


--------------------------------------------------------------------------------------------------------------------------------


### Paso 1: Configurar Netwatch para vigilar CANTV

Netwatch se encarga de lanzar *pings* constantes (por ejemplo, a un DNS confiable como `1.1.1.1` o `8.8.8.8`) forzando obligatoriamente la salida por la tabla de CANTV, para evaluar únicamente la salud de ese enlace.

Ejecuta el siguiente bloque en la terminal de tu MikroTik:

```mikrotik
/tool netwatch
# Configurar Netwatch para hacer ping a 1.1.1.1 usando la tabla FIB de CANTV
/tool netwatch
add interval=10s timeout=3s packet-count=3 \
    comment="Monitoreo Enlace CANTV" \
    host=1.1.1.1 \
    routing-table=usar-cantv \
    up-script="cantv_up" \
    down-script="cantv_down"
```
