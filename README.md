# Ataque DHCP Starvation
**Nombre:** Henry Vicente Quezada | **Matrícula:** 2025-1332 | **Fecha:** Junio 2026

---

## 🎬 Video Demostrativo
https://youtu.be/1omdjGjq42A?list=PLhmycmsx2nBtM_kPjpLdj-vl3sWWjShMS
---

## 1. Objetivo del Laboratorio
Demostrar cómo un atacante puede agotar el pool de direcciones IP del servidor DHCP enviando miles de solicitudes con MACs aleatorias, dejando sin servicio a los clientes legítimos, y aplicar DHCP Snooping con rate limiting como contramedida.

---

## 2. Objetivo del Script
Enviar masivamente solicitudes DHCP Discover con MACs falsas aleatorias para consumir todas las IPs disponibles en el pool del servidor DHCP, impidiendo que dispositivos legítimos obtengan direccionamiento.

### 2.1 Parámetros Usados
| Parámetro | Descripción | Valor por defecto |
|---|---|---|
| `interfaz` | Interfaz de red del atacante | Requerido (ej: `eth0`) |
| `cantidad` | Número de solicitudes a enviar | 500 |
| `delay` | Tiempo entre paquetes (segundos) | 0.1 |

### 2.2 Requisitos
- Sistema operativo: **Kali Linux**
- Python 3.x
- Librería Scapy: `pip install scapy`
- Permisos de root: `sudo`
- Conectividad en la misma red que el servidor DHCP

---

## 3. Funcionamiento del Script
1. Genera una MAC aleatoria por cada solicitud
2. Construye un paquete DHCP Discover con esa MAC como dirección de hardware
3. Envía el paquete como broadcast UDP (puerto 68 → 67)
4. El servidor DHCP asigna una IP a cada MAC recibida
5. Repite hasta agotar el pool completo
6. Muestra el progreso cada 50 paquetes enviados

```bash
sudo python3 dhcp_starvation.py <interfaz> [cantidad]
sudo python3 dhcp_starvation.py eth0 300
```

---

## 4. Documentación de la Red

### Topología
<img width="648" height="556" alt="image" src="https://github.com/user-attachments/assets/ecade9a5-9d38-4994-a388-962dad06f919" />

### Tabla de Direccionamiento
| Dispositivo | Interfaz | VLAN | IP | Máscara | Rol |
|---|---|---|---|---|---|
| R1 | e0/0.10 | 10 | 10.13.32.1 | /24 | **Servidor DHCP** VLAN10 |
| R1 | e0/0.20 | 20 | 10.13.33.1 | /24 | Servidor DHCP VLAN20 |
| R1 | e0/0.99 | 99 | 10.13.99.1 | /24 | Gateway Management |
| SW1 | vlan 99 | 99 | 10.13.99.2 | /24 | Gestión Switch |
| VPC10 | eth0 | 10 | DHCP | /24 | **Cliente víctima** |
| VPC20 | eth0 | 20 | DHCP | /24 | Cliente VLAN20 |
| Kali | eth0 | 10 | 10.13.32.5 | /24 | **Atacante** |

### Pool DHCP en R1
| Pool | Red | Rango disponible | Excluidos |
|---|---|---|---|
| VLAN10_TI | 10.13.32.0/24 | .11 - .254 | .1 - .10 |
| VLAN20_GERENCIA | 10.13.33.0/24 | .11 - .254 | .1 - .10 |

### Interfaces SW1
| Puerto | Modo | VLAN | Conectado a |
|---|---|---|---|
| e0/0 | Trunk 802.1q | 10,20,99 | R1 |
| e0/1 | Access | 10 | VPC10 |
| e0/2 | Access | 20 | VPC20 |
| e0/3 | Access | 10 | **Kali Linux** |

---

## 5. Capturas de Pantalla

### Antes del ataque
<img width="731" height="493" alt="image" src="https://github.com/user-attachments/assets/c280306d-acb6-4ce8-8f5b-95f7aa87f573" />

> 📷 `R1# show ip dhcp binding` — pocas IPs asignadas

### Script en ejecución
 <img width="639" height="475" alt="image" src="https://github.com/user-attachments/assets/fe176c2a-43ec-4ec2-a91b-867be5d51a38" />

> 📷 Kali ejecutando `dhcp_starvation.py` mostrando solicitudes enviadas

### Pool agotado
<img width="630" height="469" alt="image" src="https://github.com/user-attachments/assets/cb452888-4bf7-4cce-a4ac-07aa92ef8616" />

> 📷 `R1# show ip dhcp binding` — pool agotado con MACs falsas

### Cliente sin IP
<img width="537" height="430" alt="image" src="https://github.com/user-attachments/assets/48517c61-50a6-45eb-aa5d-ef9b5791b725" />

> 📷 `VPC10# ip dhcp` — falla por pool agotado

### Contramedida aplicada
<img width="572" height="363" alt="image" src="https://github.com/user-attachments/assets/db7c01c6-7cb4-47ec-b9b7-6756a4caeeb1" />

> 📷 `SW1# show ip dhcp snooping statistics` — bloqueando solicitudes excesivas

---

## 6. Contramedidas

```
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10
SW1(config)# ip dhcp snooping vlan 20
SW1(config)# no ip dhcp snooping information option
SW1(config)# interface e0/0
SW1(config-if)# ip dhcp snooping trust
SW1(config)# interface e0/3
SW1(config-if)# ip dhcp snooping limit rate 15
SW1(config)# end
SW1# write memory

! Verificación
SW1# show ip dhcp snooping statistics
SW1# show ip dhcp snooping
```

El rate limiting limita a 15 solicitudes DHCP por segundo en el puerto e0/3. Si Kali supera ese límite, el puerto se deshabilita automáticamente, impidiendo que el atacante agote el pool del servidor DHCP.
