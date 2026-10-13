# Laboratorio en Packet Tracer: overlay de dos túneles y acceso por aplicación

Laboratorio de conceptos de SD-WAN y ZTNA. No es una implementación de Cisco Catalyst SD-WAN ni de un servicio ZTNA de Cisco: simula dos transportes (MPLS e Internet), un túnel GRE sobre cada uno, un camino preferido con respaldo y una lista de acceso que limita el acceso a una sola aplicación.

## 1. Dispositivos

| Cantidad | Dispositivo | Nombres |
|---|---|---|
| 4 | Router 2911 | HQ-R, BR-R, ISP-MPLS, ISP-INET |
| 2 | Switch 2960 | SW-HQ, SW-BR |
| 2 | Servidor | Server0, Server1 |
| 1 | PC | PC0 |

Cambie los nombres en *Config > Display Name*. Los switches no necesitan configuración.

## 2. Cableado

**Cobre directo (router o PC con switch, switch con equipo final):**

| Origen | Destino |
|---|---|
| HQ-R Gi0/0 | SW-HQ Fa0/1 |
| SW-HQ Fa0/2 | Server0 |
| SW-HQ Fa0/3 | Server1 |
| BR-R Gi0/0 | SW-BR Fa0/1 |
| SW-BR Fa0/2 | PC0 |

**Cobre cruzado (router con router):**

| Origen | Destino | Transporte |
|---|---|---|
| HQ-R Gi0/1 | ISP-MPLS Gi0/0 | MPLS |
| BR-R Gi0/1 | ISP-MPLS Gi0/1 | MPLS |
| HQ-R Gi0/2 | ISP-INET Gi0/0 | Internet |
| BR-R Gi0/2 | ISP-INET Gi0/1 | Internet |

## 3. Plan de direccionamiento

| Dispositivo | Interfaz | Dirección | Conecta con |
|---|---|---|---|
| HQ-R | Gi0/0 | 192.168.10.1/24 | LAN de la sede (SW-HQ) |
| HQ-R | Gi0/1 | 10.1.1.1/30 | ISP-MPLS (Gi0/0) |
| HQ-R | Gi0/2 | 10.2.1.1/30 | ISP-INET (Gi0/0) |
| HQ-R | Tunnel1 | 192.168.101.1/30 | BR-R, por MPLS |
| HQ-R | Tunnel2 | 192.168.102.1/30 | BR-R, por Internet |
| BR-R | Gi0/0 | 192.168.20.1/24 | LAN de la sucursal (SW-BR) |
| BR-R | Gi0/1 | 10.1.2.1/30 | ISP-MPLS (Gi0/1) |
| BR-R | Gi0/2 | 10.2.2.1/30 | ISP-INET (Gi0/1) |
| BR-R | Tunnel1 | 192.168.101.2/30 | HQ-R, por MPLS |
| BR-R | Tunnel2 | 192.168.102.2/30 | HQ-R, por Internet |
| ISP-MPLS | Gi0/0 / Gi0/1 | 10.1.1.2/30 / 10.1.2.2/30 | HQ-R / BR-R |
| ISP-INET | Gi0/0 / Gi0/1 | 10.2.1.2/30 / 10.2.2.2/30 | HQ-R / BR-R |
| Server0 | NIC | 192.168.10.10/24 | Puerta de enlace 192.168.10.1 |
| Server1 | NIC | 192.168.10.20/24 | Puerta de enlace 192.168.10.1 |
| PC0 | NIC | 192.168.20.10/24 | Puerta de enlace 192.168.20.1 |

Los equipos finales se configuran en *Desktop > IP Configuration*.

## 4. Cómo usar los bloques de comandos

- Abra la pestaña **CLI** de cada router y pegue el bloque completo.
- Cada bloque empieza en `enable` y `configure terminal`, y termina en `end`.
- Antes de pegar, compruebe que el prompt sea `Router>` o `Router#`. Si pega comandos de configuración en `#` sin haber entrado a `configure terminal`, todos darán *Invalid input*.
- Si al pegar una línea larga se corta, escríbala a mano. Cuidado especialmente con `eq 80` al final de la línea `permit`.
- Para guardar: `copy running-config startup-config` y pulse Enter cuando pida el nombre del archivo.

## 5. Parte A: overlay y camino de respaldo

La preferencia entre caminos se resuelve con la distancia administrativa: OSPF (110) opera solo sobre Tunnel1 y la ruta estática con distancia 150 sobre Tunnel2 solo se instala si desaparece la ruta OSPF.

### ISP-MPLS

```
enable
configure terminal
hostname ISP-MPLS
interface GigabitEthernet0/0
 ip address 10.1.1.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 10.1.2.2 255.255.255.252
 no shutdown
end
```

### ISP-INET

```
enable
configure terminal
hostname ISP-INET
interface GigabitEthernet0/0
 ip address 10.2.1.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 10.2.2.2 255.255.255.252
 no shutdown
end
```

### HQ-R

```
enable
configure terminal
hostname HQ-R
interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
interface GigabitEthernet0/1
 ip address 10.1.1.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/2
 ip address 10.2.1.1 255.255.255.252
 no shutdown
ip route 10.1.2.0 255.255.255.252 10.1.1.2
ip route 10.2.2.0 255.255.255.252 10.2.1.2
interface Tunnel1
 ip address 192.168.101.1 255.255.255.252
 tunnel source GigabitEthernet0/1
 tunnel destination 10.1.2.1
interface Tunnel2
 ip address 192.168.102.1 255.255.255.252
 tunnel source GigabitEthernet0/2
 tunnel destination 10.2.2.1
router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.101.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0
 exit
ip route 192.168.20.0 255.255.255.0 192.168.102.2 150
end
```

### BR-R

```
enable
configure terminal
hostname BR-R
interface GigabitEthernet0/0
 ip address 192.168.20.1 255.255.255.0
 no shutdown
interface GigabitEthernet0/1
 ip address 10.1.2.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/2
 ip address 10.2.2.1 255.255.255.252
 no shutdown
ip route 10.1.1.0 255.255.255.252 10.1.2.2
ip route 10.2.1.0 255.255.255.252 10.2.2.2
interface Tunnel1
 ip address 192.168.101.2 255.255.255.252
 tunnel source GigabitEthernet0/1
 tunnel destination 10.1.1.1
interface Tunnel2
 ip address 192.168.102.2 255.255.255.252
 tunnel source GigabitEthernet0/2
 tunnel destination 10.2.1.1
router ospf 1
 router-id 2.2.2.2
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.101.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0
 exit
ip route 192.168.10.0 255.255.255.0 192.168.102.1 150
end
```

### Pruebas de la Parte A

1. **Transporte.** Desde HQ-R: `ping 10.1.2.1` y `ping 10.2.2.1`. El primer ping suele fallar por el ARP; repítalo.
2. **Túneles.** Desde HQ-R: `ping 192.168.101.2` y `ping 192.168.102.2`.
3. **Vecino OSPF.** En HQ-R: `show ip ospf neighbor` debe mostrar a BR-R por Tunnel1.
4. **Camino preferido.** En HQ-R: `show ip route` debe mostrar `192.168.20.0/24` como ruta `O` por 192.168.101.2. La ruta estática de respaldo no aparece mientras exista la ruta OSPF.
5. **Extremo a extremo.** Desde PC0: `ping 192.168.10.10` y `tracert 192.168.10.10`, que debe pasar por 192.168.20.1 y 192.168.101.1.

### Simulación de falla

1. En **ISP-MPLS** apague las dos interfaces:
   ```
   enable
   configure terminal
   interface GigabitEthernet0/0
    shutdown
   interface GigabitEthernet0/1
    shutdown
   end
   ```
2. Espere hasta unos 40 segundos, el tiempo de espera predeterminado de OSPF.
3. En HQ-R: `show ip route` debe mostrar `192.168.20.0/24` como ruta `S` `[150/0]` por 192.168.102.2.
4. Desde PC0: `tracert 192.168.10.10` debe pasar por 192.168.102.1, es decir, por Internet.
5. Para volver: `no shutdown` en las dos interfaces de ISP-MPLS. Pasados unos segundos, el tráfico regresa por 192.168.101.1.

## 6. Parte B: acceso por aplicación (mínimo privilegio)

Se restringe el acceso a una sola aplicación: el servicio web de Server0. La lista se aplica a la **salida de Gi0/0 de HQ-R**, porque todo el tráfico de la sucursal hacia la sede, por el túnel que sea, sale por esa interfaz ya desencapsulado.

### HQ-R

```
enable
configure terminal
ip access-list extended ACCESO-APP
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.10.10 eq 80
 deny ip any any
 exit
interface GigabitEthernet0/0
 ip access-group ACCESO-APP out
end
```

Si necesita rehacer la lista, borre la anterior antes de crearla de nuevo, desde `HQ-R(config)#`:

```
no ip access-list extended ACCESO-APP
```

### Pruebas de la Parte B

1. En HQ-R: `show access-lists` debe mostrar solo dos líneas, el `permit` y el `deny`. IOS muestra el puerto 80 como `www`.
2. Desde el navegador de PC0, `http://192.168.10.10` debe abrir.
3. Desde PC0, el ping a 192.168.10.10, `http://192.168.10.20` y el FTP a Server0 deben fallar.

## 7. Limitaciones de Packet Tracer observadas

Durante las pruebas, Packet Tracer rechazó estos comandos bajo las interfaces `Tunnel`, por lo que el diseño los evita:

| Comando rechazado | Alternativa usada |
|---|---|
| `ip ospf cost` | Ruta estática de respaldo con distancia administrativa 150 |
| `ip access-group` | Lista aplicada a la salida de `Gi0/0` en HQ-R |

Además, la palabra `log` no fue aceptada al final de `deny ip any any`, por lo que se omite.

## 8. Errores frecuentes

| Síntoma | Causa probable |
|---|---|
| *Invalid input* en la primera palabra de cada línea | Está en `Router#` o `Router>`. Entre con `enable` y `configure terminal` |
| *Incomplete command* en `permit ospf any` | Faltan origen y destino (`permit ospf any any`). No se usa en este diseño |
| *Invalid input* en `eq 80` | Escribió `eq 80` en otra línea. Debe ir al final de la línea `permit tcp ...` |
| El túnel no sube | Verifique que la interfaz física esté arriba y que el ping al otro extremo del transporte funcione |
| Hay túnel pero no vecino OSPF | Revise las líneas `network` y que Tunnel2 **no** esté en OSPF |
| Luz roja en el cable | Cruzado entre routers, directo hacia switches. O haga `shutdown` y `no shutdown` |
| Tras la falla, el `tracert` sigue por 192.168.101.1 | Espere los 40 segundos de OSPF y confirme que apagó **ambas** interfaces de ISP-MPLS |
