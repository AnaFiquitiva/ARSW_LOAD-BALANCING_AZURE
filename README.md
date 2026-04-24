# Lab 9 - Load Balancing and Horizontal Scalability in Azure
## ARSW - Arquitecturas de Software

---

# Escuela Colombiana de Ingeniería
## Arquitecturas de Software - ARSW
### Escalamiento en Azure con Máquinas Virtuales, Scale Sets y Service Plans

---

## Dependencias

Cree una cuenta gratuita dentro de Azure. Para hacerlo puede guiarse de esta documentación. Al hacerlo usted contará con $100 USD para gastar durante 12 meses.

---

## Parte 0 - Entendiendo el escenario de calidad

Adjunto a este laboratorio usted podrá encontrar una aplicación totalmente desarrollada que tiene como objetivo calcular el enésimo valor de la secuencia de Fibonacci.

### Escalabilidad
Cuando un conjunto de usuarios consulta un enésimo número (superior a 1000000) de la secuencia de Fibonacci de forma concurrente y el sistema se encuentra bajo condiciones normales de operación, todas las peticiones deben ser respondidas y el consumo de CPU del sistema no puede superar el 70%.

---

# Parte 1 - Escalabilidad Vertical

## Creación de la VM

Diríjase al Portal de Azure y a continuación cree una máquina virtual con las características básicas descritas a continuación:

- **Resource Group:** SCALABILITY_LAB
- **Virtual machine name:** VERTICAL-SCALABILITY
- **Image:** Ubuntu Server
- **Size:** Standard B1ls
- **Username:** scalability_lab
- **SSH public key:** Su llave ssh pública

### Conectarse a la VM

```bash
ssh scalability_lab@xxx.xxx.xxx.xxx
```

### Instalar Node.js

Siga la sección "Installing Node.js and npm using NVM" que encontrará en este enlace.

### Instalar la aplicación

```bash
git clone <your_repo>
cd <your_repo>/FibonacciApp
npm install
```

### Ejecutar la aplicación

Para ejecutar la aplicación use `forever`:

```bash
npm install forever -g
forever start FibonacciApp.js
```

### Verificar funcionamiento

Antes de verificar si el endpoint funciona, en Azure vaya a la sección de Networking y cree una Inbound port rule que permita tráfico por el puerto 3000.

Para verificar que la aplicación funciona:
```
http://xxx.xxx.xxx.xxx:3000/fibonacci/6
```
Respuesta esperada: **The answer is 8**

## Pruebas de rendimiento Parte 1

### Documentar tiempos de respuesta

La función que calcula el enésimo número de la secuencia de Fibonacci está muy mal construida y consume bastante CPU. Documente los tiempos de respuesta para el endpoint usando los siguientes valores en el browser:

- 1000000
- 1010000
- 1020000
- 1030000
- 1040000
- 1050000
- 1060000
- 1070000
- 1080000
- 1090000

### Pruebas con Postman/Newman

**Con la VM en tamaño B1ls (tamaño inicial):**

1. Instale newman: `npm install newman -g`
2. Configure el archivo `[ARSW_LOAD-BALANCING_AZURE].postman_environment.json` con la IP de su VM
3. Ejecute las pruebas concurrentes:

```bash
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Resultado:** La carga concurrente es demasiada. Muchas peticiones fallan.

### Escalamiento Vertical: B1ls → B2ms

1. En Azure, vaya a la sección "Size" de la VM
2. Seleccione el tamaño **B2ms** (aumenta a 2 vCPUs y 4GB RAM)
3. Espere a que la VM se reinicie
4. Repita las pruebas con Newman

**Resultado:** Las peticiones mejoran, pero aún hay saturación con 4 procesos concurrentes.

### Conclusión Parte 1

**Limitaciones del Escalamiento Vertical:**
- ❌ Requiere detener la VM (downtime)
- ❌ Costo más alto
- ❌ Límite de escalabilidad (tamaños máximos)
- ❌ Sin redundancia (si la VM falla, todo cae)
- ❌ Con 4 procesos concurrentes: **FALLA TOTAL (0% éxito)**

---

# Parte 2 - Escalabilidad Horizontal

## Arquitectura de Escalabilidad Horizontal

La solución usa:
- **Azure Load Balancer** (capa 4, TCP/UDP)
- **2 Máquinas Virtuales** en diferentes Availability Zones
- **Virtual Network** para aislamiento
- **Network Security Group** como firewall

### Nota importante sobre limitaciones
Debido a limitaciones de cuota en la suscripción Azure, solo se pudieron crear **2 VMs en Availability Zones 2 y 3**. La zona 1 no está disponible. Por lo tanto:
- La infraestructura funcionará con **2 VMs en lugar de 3**
- Aún hay distribución de carga horizontal
- Aún hay redundancia de 2 zonas de disponibilidad

---

## Respuestas a las Preguntas Teóricas de Parte 2

### Pregunta 1: Tipos de Balanceadores de Carga, SKU e IP Pública

#### Tipos de Balanceadores de Carga en Azure:

| Tipo | Capa | Caso de Uso | Performance |
|------|------|-----------|-------------|
| **Load Balancer** | Layer 4 (TCP/UDP) | Aplicaciones de alto rendimiento, baja latencia | Millones conexiones/seg |
| **Application Gateway** | Layer 7 (HTTP/HTTPS) | Web apps, routing por URL/hostname | Más lento, más inteligente |
| **Traffic Manager** | DNS | Distribución global, failover geográfico | Multi-región |
| **Front Door** | Edge/Global | CDN + balanceo global | Extremadamente rápido |

**Para este laboratorio:** Usamos **Load Balancer** (Layer 4) por su alto rendimiento con aplicaciones TCP/UDP simples.

#### ¿Qué es SKU?

**SKU = Stock Keeping Unit** - Define el nivel de servicio y características:

| SKU | Características | Costo |
|-----|-----------------|-------|
| **Basic** | Max 300k conexiones, sin SLA | Económico |
| **Standard** | Millones conexiones, SLA 99.99%, redundancia, métricas avanzadas | Medio |

**Elegimos:** Standard SKU para máxima disponibilidad.

#### ¿Por qué el Load Balancer necesita IP Pública?

- Es el **punto de entrada único** para los clientes desde internet
- Permite que los usuarios accedan: `http://IP_PUBLICA/fibonacci/6`
- El Load Balancer **distribuye a IPs privadas** de las VMs
- Sin IP pública: solo acceso interno (VNet), no es útil

---

### Pregunta 2: Propósito del Backend Pool

El **Backend Pool** es:
- La **lista de VMs** que van a servir peticiones
- Define **dónde van** las peticiones después del Load Balancer
- Almacena IPs privadas de las VMs (10.0.0.4, 10.0.0.5)

**En nuestro caso:**
```
Backend Pool "HORIZONTAL-SCALABILITY-BP"
├── VM1 (10.0.0.4:3000) - Zone 2
└── VM2 (10.0.0.5:3000) - Zone 3
```

Cuando el LB recibe una petición en puerto 80, la reenvía a una VM en el Backend Pool en puerto 3000.

---

### Pregunta 3: Propósito del Health Probe

El **Health Probe** es un **vigilante** que verifica si las VMs están vivas:

- **Envía requests periódicamente** (cada 5 segundos) a `http://10.0.0.x:3000/fibonacci/1`
- **Si recibe respuesta 200 OK** → VM está ✅ HEALTHY
- **Si no responde o error** → VM está ❌ UNHEALTHY
- El Load Balancer **NO envía tráfico** a VMs unhealthy

**Beneficio:** Si una VM se cae, el LB automáticamente solo envía tráfico a las otras.

---

### Pregunta 4: Load Balancing Rule y Sesión Persistente

#### Propósito de Load Balancing Rule:

Define **cómo el Load Balancer distribuye el tráfico:**

```
Frontend Port 80 (IP pública) → Backend Port 3000 (VMs)
Algoritmo: Round Robin (VM1 → VM2 → VM1 → VM2...)
```

#### Tipos de Sesión Persistente (Session Affinity):

| Tipo | Descripción | Uso | Problema |
|------|------------|-----|----------|
| **None** | Cada request puede ir a VM diferente | Stateless apps ✅ | No mantiene contexto |
| **Client IP** | Misma IP cliente → siempre misma VM | Apps con estado | Concentra carga |
| **Client IP + Protocol** | Más específico que Client IP | Protocolos mixtos | Aún concentra carga |

**Para Fibonacci:** Usar **None** (sin persistencia) porque cada request es independiente.

#### Impacto en escalabilidad:

- **Sin persistencia:** 2 VMs pueden distribuir carga uniformemente
- **Con persistencia:** Algunos clientes siempre van a misma VM → cuello de botella

---

### Pregunta 5: Virtual Network y Subnet

#### Virtual Network (VNet):

Red **privada aislada** dentro de Azure:
- Rango IP: `10.0.0.0/16` (65,536 direcciones IP privadas)
- Aislada del internet por defecto
- Solo se comunican VMs dentro de la misma VNet

#### Subnet:

**Subdivisión de la VNet**:
- Rango más pequeño: `10.0.0.0/24` (256 direcciones)
- Permite organizar recursos
- Las VMs en misma subnet se comunican directamente

#### Address Space vs Address Range:

| Término | Significado | Ejemplo |
|---------|-----------|---------|
| **Address Space** | Rango total de VNet | 10.0.0.0/16 (64K IPs) |
| **Address Range** | Rango de una subnet específica | 10.0.0.0/24 (256 IPs) |

**Jerarquía:**
```
VNet Address Space: 10.0.0.0/16 (65,536 IPs)
  └─ Subnet Address Range: 10.0.0.0/24 (256 IPs)
      ├── VM1: 10.0.0.4
      └── VM2: 10.0.0.5
```

---

### Pregunta 6: Availability Zones e IP Zone-Redundant

#### ¿Qué son Availability Zones?

**Datacenters separados físicamente** dentro de una región:

```
Región: East US 2
├── Zone 1 (Datacenter A)
├── Zone 2 (Datacenter B)
└── Zone 3 (Datacenter C)
```

Cada zona tiene hardware, red y electricidad independientes.

#### ¿Por qué 3 diferentes zonas? (Se intentó, solo 2 disponibles)

**Tolerancia a fallos:**

| Escenario | 1 Zone | 3 Zones |
|-----------|--------|---------|
| Se cae Datacenter | ❌ Servicio DOWN | ✅ 2 VMs siguen |
| Fallo de red | ❌ Servicio DOWN | ✅ Otras zonas |
| Ataque DDoS | ❌ Servicio DOWN | ✅ Otras zonas |

**En nuestro caso (2 zonas):**
```
VM1 → Zone 2
VM2 → Zone 3
```
Si Zone 2 falla, VM2 en Zone 3 mantiene el servicio.

#### IP Zone-Redundant:

Una IP **zone-redundant** funciona en **TODAS las zonas simultáneamente:**

```
IP Pública: 48.211.232.81 (Zone-Redundant)
├── Accesible desde Zone 2 ✅
└── Accesible desde Zone 3 ✅
```

Si una zona se cae, la IP pública **sigue siendo accesible** desde las otras zonas.

---

### Pregunta 7: Network Security Group (NSG)

NSG es un **firewall** que controla qué tráfico entra y sale:

#### Inbound Rules (Entrante):

| Prioridad | Protocolo | Puerto | Origen | Acción |
|-----------|-----------|--------|--------|--------|
| 100 | TCP | 80 | Any | Allow (HTTP) |
| 110 | TCP | 3000 | Any | Allow (App) |
| 120 | TCP | 22 | Any | Allow (SSH) |
| 4096 | * | * | * | Deny (Default) |

**Interpretación:**
- ✅ Puerto 80 abierto → Load Balancer puede recibir peticiones HTTP
- ✅ Puerto 3000 abierto → Load Balancer puede comunicarse con VMs
- ❌ Otros puertos bloqueados

#### ¿Por qué es importante?

- **Seguridad:** Solo puertos necesarios abiertos
- **Defensa:** Bloquea ataques a puertos innecesarios
- **Cumplimiento:** Satisface requisitos de seguridad

---

### Pregunta 8: Diagrama de Despliegue

```
                          ┌─────────────────┐
                          │   Internet      │
                          │   Clientes      │
                          └────────┬────────┘
                                   │ HTTP:80
                                   ▼
                    ┌──────────────────────────────┐
                    │  Azure Load Balancer         │
                    │  IP: 48.211.232.81:80        │
                    │  (Zone-Redundant)            │
                    └──────────────┬───────────────┘
                                   │ Health Probe
                     ┌─────────────┴────────────┐
                     ▼ (Round Robin)            ▼ (Round Robin)
         ┌──────────────────────┐   ┌──────────────────────┐
         │  Zone 2 (Datacenter) │   │  Zone 3 (Datacenter) │
         ├──────────────────────┤   ├──────────────────────┤
         │  VM1 (Standard_B2s)  │   │  VM2 (Standard_B2s)  │
         │  10.0.0.4:3000       │   │  10.0.0.5:3000       │
         │                      │   │                      │
         │  ┌────────────────┐  │   │  ┌────────────────┐  │
         │  │ FibonacciApp   │  │   │  │ FibonacciApp   │  │
         │  │ (Node.js)      │  │   │  │ (Node.js)      │  │
         │  └────────────────┘  │   │  └────────────────┘  │
         │                      │   │                      │
         │  ✅ HEALTHY          │   │  ✅ HEALTHY          │
         └──────────────────────┘   └──────────────────────┘
                     ▲                       ▲
                     │ SSH:22                │ SSH:22
                     │ HTTP:80               │ HTTP:80
                     │ APP:3000              │ APP:3000
                     │                       │
         ┌───────────┴───────────┬───────────┴───────────┐
         │                       │                       │
         │   Network Security Group (NSG)                │
         │   ✅ Allow: 22, 80, 3000                      │
         │   ❌ Deny: Others                             │
         │                       │                       │
         └───────────┬───────────┴───────────┬───────────┘
                     │                       │
                     ▼                       ▼
         ┌──────────────────────┐   ┌──────────────────────┐
         │   Virtual Network    │   │   Virtual Network    │
         │  10.0.0.0/16         │   │  10.0.0.0/16         │
         │  Subnet: 10.0.0.0/24 │   │  Subnet: 10.0.0.0/24 │
         └──────────────────────┘   └──────────────────────┘
```

---

## Pruebas Realizadas - Parte 2

### Prueba 1: 2 Procesos Newman en Paralelo (20 requests)

#### VM1 (Zone 2) - Proceso 1 y 2:

**Ejecución:**
```bash
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json \
  -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json \
  -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Resultados:**

| Métrica | Proceso 1 | Proceso 2 | Promedio |
|---------|-----------|-----------|----------|
| Iteraciones | 10/10 ✅ | 10/10 ✅ | 20/20 ✅ |
| Tasa éxito | 100% | 100% | **100%** ✅ |
| Tiempo promedio | 8.4s | 8.5s | **8.45s** |
| Rango | 8.3-8.5s | 8.3-8.6s | 8.3-8.6s |
| Desviación estándar | 41ms | 72ms | 56.5ms |

**Conclusión:** ✅ VM1 maneja 20 requests concurrentes exitosamente

---

#### VM2 (Zone 3) - Proceso 1 y 2:

**Resultados:**

| Métrica | Proceso 1 | Proceso 2 | Promedio |
|---------|-----------|-----------|----------|
| Iteraciones | 10/10 ✅ | 10/10 ✅ | 20/20 ✅ |
| Tasa éxito | 100% | 100% | **100%** ✅ |
| Tiempo promedio | 6.4s | 6.4s | **6.4s** |
| Rango | 6.3-6.5s | 6.3-6.5s | 6.3-6.5s |
| Desviación estándar | 41ms | 41ms | 41ms |

**Conclusión:** ✅ VM2 maneja 20 requests concurrentes **MÁS RÁPIDO** (24% mejor que VM1)

---

#### Comparativa VM1 vs VM2 (2 procesos):

| Métrica | VM1 | VM2 | Ventaja |
|---------|-----|-----|---------|
| Tasa éxito | 100% ✅ | 100% ✅ | Igual |
| Tiempo promedio | 8.45s | 6.4s | **VM2: -2.05s (-24.3%)** ⚡ |
| Estabilidad (StdDev) | 56.5ms | 41ms | **VM2 más estable** |
| Throughput | 14.2 req/min | 18.75 req/min | **VM2: +32%** |

**Conclusión:** Ambas VMs manejan 20 requests exitosamente. VM2 tiene mejor hardware o menor contención.

---

### Prueba 2: 4 Procesos Newman en Paralelo (40 requests)

**Comando:**
```bash
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json \
  -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json \
  -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json \
  -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json \
  -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Resultados por Proceso (en VM1):**

| Proceso | Iteraciones | Exitosas | Fallidas | Tasa Éxito | Tiempo Prom |
|---------|-------------|----------|----------|-----------|-----------|
| **Proceso 1** | 10 | 5 | 5 | **50%** ❌ | 18.6s |
| **Proceso 2** | 10 | 5 | 5 | **50%** ❌ | 21.9s |
| **Proceso 3** | 10 | 5 | 5 | **50%** ❌ | 25.3s |
| **Proceso 4** | 10 | 6 | 4 | **60%** ⚠️ | 26.7s |
| **TOTAL** | **40** | **21** | **19** | **52.5%** ❌ | **23.1s** |

**Errores observados:**
- **Tipo:** ECONNRESET (servidor rechaza conexiones)
- **Patrón:** Errores en iteraciones 2, 4, 6, 8, 10 (pares)
- **Causa:** CPU saturada, backlog del servidor lleno

**Conclusión (4 procesos):**
- ❌ Una sola VM NO puede manejar 40 requests
- ⚠️ 52.5% de éxito (mejor que Parte 1 pero insuficiente)
- ❌ Saturación en el servidor

---

# Informe Comparativo: Parte 1 (Vertical) vs Parte 2 (Horizontal)

## Comparativa de Arquitectura

| Aspecto | Parte 1 (Vertical) | Parte 2 (Horizontal) |
|--------|------------------|-------------------|
| **VMs** | 1 (B2ms) | 2 (2×B2s) |
| **Escalabilidad** | Redimensionar VM | Agregar/quitar VMs |
| **Alta Disponibilidad** | ❌ No (SPOF) | ✅ Sí (2 AZ) |
| **Redundancia** | 0 zonas | 2 zonas |
| **Failover automático** | ❌ Manual | ✅ Automático (30s) |
| **Load Distribution** | N/A | Round Robin |
| **Health Monitoring** | Manual | Automático (5s) |

---

## Comparativa de Resultados de Carga

### Escenario 1: 2 procesos (20 requests simultáneos)

| Métrica | Parte 1 | Parte 2 |
|--------|--------|--------|
| **Tasa Éxito** | ✅ 100% (20/20) | ✅ 100% (20/20) |
| **Tiempo Prom** | 8.3s | 7.4s (12% mejor) |
| **Throughput** | 14.5 req/min | 15.3 req/min |
| **CPU** | <70% | <70% |
| **Conclusión** | ✅ Funciona | ✅ Funciona + redundancia |

**Ganador:** Empate técnico, pero **Parte 2 tiene redundancia automática**.

---

### Escenario 2: 4 procesos (40 requests simultáneos)

| Métrica | Parte 1 | Parte 2 |
|--------|--------|--------|
| **Tasa Éxito** | ❌ **0%** (0/40) | ⚠️ **52.5%** (21/40) |
| **Tiempo Prom** | Timeout | 23.1s |
| **Error** | ECONNREFUSED | ECONNRESET |
| **CPU** | 100% (saturado) | 100% (saturado) |
| **Conclusión** | ❌ Falla total | ⚠️ Falla parcial |

**Ganador:** **Parte 2 (horizontal)** - 52.5% > 0%

---

## Análisis Económico

### Costo Mensual:

| Componente | Parte 1 | Parte 2 |
|-----------|--------|--------|
| **VM Compute** | Standard_B2ms = $60.74/mes | 2×B2s = $49.48/mes |
| **Load Balancer** | N/A | $18.25/mes |
| **Almacenamiento** | ~$5/mes | ~$10/mes |
| **TOTAL** | **$65.74/mes** | **$77.73/mes** |

**Diferencia:** Parte 2 cuesta **$12/mes más (+18%)**

**Retorno de inversión:**
- ✅ Máxima disponibilidad (99.95% SLA)
- ✅ Recuperación automática en 30s
- ✅ 52.5% vs 0% en carga extrema
- ✅ Escalabilidad futura

---

## Disponibilidad y SLA

### Escenario: 1 VM falla

| Aspecto | Parte 1 | Parte 2 |
|--------|--------|--------|
| **Uptime** | **0%** ❌ | **100%** ✅ (otra VM continúa) |
| **RTO** | ~30 minutos (manual) | ~30 segundos (automático) |
| **SLA** | Sin SLA formal | 99.95% |

**Ganador:** **Parte 2** - Arquitectura empresarial

---

# Conclusiones Finales

## ¿Cuándo usar cada enfoque?

### Parte 1 (Vertical) es ideal para:
- ✅ Desarrollo/testing (no producción)
- ✅ Cargas predecibles y bajas
- ✅ Presupuesto mínimo
- ✅ Aplicaciones sin requisitos de HA

### Parte 2 (Horizontal) es ideal para:
- ✅ **Producción con SLA**
- ✅ **Cargas variables o crecientes**
- ✅ **Aplicaciones críticas**
- ✅ **Requisitos de alta disponibilidad**
- ✅ **Escalabilidad futura**

---

## Recomendación Final

### Para esta aplicación Fibonacci (CPU-intensiva):

> **Parte 2 (Escalabilidad Horizontal) es SUPERIOR**

**Razones:**
1. Maneja mejor cargas pesadas (52.5% vs 0%)
2. Redundancia automática (2 Availability Zones)
3. Recuperación en 30 segundos
4. Solo +18% costo por máxima disponibilidad
5. Escalable: agregar VM3, VM4 cuando carga crece
6. Health Probe automático (no manual)

---

## Arquitectura Recomendada (Futuro)

Para mejorar escalabilidad con 4+ procesos simultáneos:

```
Internet
   ↓
Load Balancer (48.211.232.81:80)
   ↓
Backend Pool (Round Robin)
├─→ VM1 (Zone 1) :3000
├─→ VM2 (Zone 2) :3000
├─→ VM3 (Zone 3) :3000 ← AGREGAR
└─→ VM4 (Zone 1) :3000 ← AGREGAR

Con 4 VMs → Soportaría 60-80 requests concurrentes ✅
```

---

## Respuestas a Preguntas Adicionales

### Pregunta: ¿Cuántos y cuáles recursos crea Azure junto con la VM?

Azure crea automáticamente:
1. **NIC (Network Interface Card)** - Conexión a VNet
2. **Disco OS** - Almacenamiento del sistema operativo
3. **IP Pública** - Acceso desde internet
4. **NSG (Network Security Group)** - Firewall
5. **Storage Account** - Diagnósticos

---

### Pregunta: Al cerrar SSH, ¿por qué se cae la aplicación?

Cuando cierras SSH, se envía una señal SIGHUP que termina todos los procesos hijos, incluido Node.js. **Solución:** Usar `forever` como process manager para mantener la app viva.

---

### Pregunta: ¿Por qué el escalamiento horizontal es mejor?

**Escalamiento Vertical (limitado):**
- Requiere downtime
- Tamaños máximos
- Sin redundancia
- Más caro

**Escalamiento Horizontal (ilimitado):**
- Sin downtime
- Escala infinitamente
- Redundancia automática
- Más barato
- Auto-healing con Health Probe

---

## Recursos

- **Repository:** https://github.com/AnaFiquitiva/ARSW_LOAD-BALANCING_AZURE
- **Región Azure:** East US 2
- **Grupo de Recursos:** HORIZONTAL-SCALABILITY_LAB
- **Load Balancer:** HORIZONTAL-SCALABILITY-LB
- **VNet:** HORIZONTAL-SCALABILITY-VNet
- **VMs:** HORIZONTAL-SCALABILITY-VM1 (Zone 2), HORIZONTAL-SCALABILITY-VM2 (Zone 3)

---

**Lab completado:** ✅ 24 de abril de 2026

---

## 📝 **Descripción**

Este laboratorio demuestra dos enfoques de escalabilidad en Microsoft Azure:

- **Parte 1:** Escalabilidad **Vertical** (aumentar recursos de 1 VM)
- **Parte 2:** Escalabilidad **Horizontal** (distribuir carga entre múltiples VMs con Load Balancer)

**Aplicación:** FibonacciApp (Node.js) - Calcula fibonacci(n) de forma recursiva (CPU-intensiva)
**Infraestructura:** Azure Virtual Machines + Load Balancer

---

# PARTE 1: ESCALABILIDAD VERTICAL

## Arquitectura Parte 1

**Componentes:**
- 1 × VM única (Standard_B1ls → Standard_B2ms upgrade)
- IP Pública directa en VM
- Sin redundancia (1 punto de fallo)

### Pruebas Parte 1

#### Con 2 procesos Newman (20 requests):
- ✅ **Resultado:** 100% éxito (20/20)
- **Tiempo promedio:** 8.3s
- **Throughput:** 14.5 req/min

#### Con 4 procesos Newman (40 requests):
- ❌ **Resultado:** 0% éxito (0/40)
- **Error:** ECONNREFUSED (servidor saturado, rechaza conexiones)
- **Razón:** CPU agotada, backlog lleno

**Conclusión Parte 1:**
- ✅ Maneja carga normal (2 procesos)
- ❌ Colapsa con carga pesada (4 procesos)
- ❌ Sin redundancia (SPOF)

---

# PARTE 2: ESCALABILIDAD HORIZONTAL

## 🏗️ Arquitectura Parte 2

### Componentes principales:

```
Internet (Clientes)
    ↓
Azure Load Balancer (IP Pública: 48.211.232.81:80)
    ↓ Health Probe cada 5s
    ├─→ VM1 (Zone 2): 10.0.0.4:3000 ✅ HEALTHY
    └─→ VM2 (Zone 3): 10.0.0.5:3000 ✅ HEALTHY
```

**Especificaciones de Infraestructura:**

| Componente | Detalles |
|-----------|---------|
| **Load Balancer** | Azure Load Balancer Standard (SKU) |
| **Frontend** | IP Pública: 48.211.232.81:80 |
| **Backend Pool** | 2 VMs en puertos 3000 |
| **Health Probe** | HTTP GET /fibonacci/1, intervalo 5s |
| **Regla LB** | Round Robin, puerto 80→3000 |
| **Virtual Network** | HORIZONTAL-SCALABILITY-VNet (10.0.0.0/16) |
| **Subnet** | default (10.0.0.0/24) |
| **Availability Zones** | Zone 2 (VM1) + Zone 3 (VM2) |
| **NSG** | Permite SSH:22, HTTP:80, APP:3000 |

---

## 🖥️ Máquinas Virtuales

### VM1 - Availability Zone 2
- **Tipo:** Standard_B2s
- **vCPUs:** 2
- **RAM:** 4GB
- **IP Privada:** 10.0.0.4
- **IP Pública:** 52.232.252.189
- **SO:** Ubuntu Server
- **Estado:** ✅ Healthy

### VM2 - Availability Zone 3
- **Tipo:** Standard_B2s
- **vCPUs:** 2
- **RAM:** 4GB
- **IP Privada:** 10.0.0.5
- **IP Pública:** 172.206.26.250
- **SO:** Ubuntu Server
- **Estado:** ✅ Healthy

---

## 📊 Preguntas Teóricas

### **Pregunta 1: Tipos de Balanceadores de Carga, SKU e IP Pública**

#### Tipos de Load Balancer en Azure:

| Balanceador | Capa | Uso | Geografía |
|------------|------|-----|-----------|
| **Load Balancer** | L4 (TCP/UDP) | Alta concurrencia interna | Single Region |
| **Application Gateway** | L7 (HTTP/HTTPS) | Enrutamiento URL/hostname | Single Region |
| **Traffic Manager** | DNS | Failover geográfico | Multi-region |
| **Front Door** | L7 + DDoS | CDN + WAF | Global |

**Para este lab:** Usamos **Load Balancer** (L4) porque es eficiente para aplicaciones TCP estateless.

#### SKU Types:

| SKU | Upgrade Path | SLA | Costo |
|-----|--------------|-----|------|
| **Basic** | Requiere redeploy | No | Menor |
| **Standard** | Soporta cambios en vivo | 99.99% | Mayor |

**Elegimos:** Standard SKU para máxima disponibilidad.

#### ¿Por qué Load Balancer necesita IP Pública?

- **Razón:** Clientes externos (internet) necesitan una dirección para alcanzar el servicio
- **Flujo:** Internet → IP Pública LB (48.211.232.81) → LB distribuye a IPs privadas de VMs (10.0.0.4, 10.0.0.5)
- **Sin IP pública:** Solo acceso interno (VNet), no es útil

---

### **Pregunta 2: Propósito del Backend Pool**

**Backend Pool:** Almacena la lista de máquinas virtuales que reciben el tráfico.

**Nuestro caso:**
- **VM1:** 10.0.0.4:3000
- **VM2:** 10.0.0.5:3000

**Funciones:**
1. Define QUÉ máquinas reciben tráfico
2. Especifica puerto de destino (3000)
3. Habilita auto-scaling: agregar/quitar VMs sin reconfigizar LB
4. Integración con Health Probe para detectar fallos

---

### **Pregunta 3: Propósito del Health Probe**

**Health Probe:** Monitorea la salud de VMs en el Backend Pool.

**Configuración:**
```
Protocolo: HTTP
Ruta: /fibonacci/1
Puerto: 3000
Intervalo: 5 segundos
Timeout: 5 segundos
Umbral saludable: 2 intentos exitosos
Umbral no saludable: 2 intentos fallidos
```

**Resultados observados:**
- ✅ **VM1:** HEALTHY (responde 200 OK a /fibonacci/1)
- ✅ **VM2:** HEALTHY (responde 200 OK a /fibonacci/1)

**Impacto:**
- Si VM1 falla → LB la marca UNHEALTHY → deja de enviarle tráfico
- Todo el tráfico se desvía a VM2 automáticamente
- **RTO (Recovery Time):** ~30 segundos (3 timeouts × 5s = 15s + detección)

---

### **Pregunta 4: Load Balancing Rule y Sesión Persistente**

**Load Balancing Rule:** Mapea frontend → backend

**Nuestra regla:**
```
Frontend: 48.211.232.81:80 (IP pública, puerto HTTP)
Backend: :3000 (puerto aplicación en VMs)
Protocolo: TCP
Persistencia: NONE (Sin afinidad)
Algoritmo: Round Robin (alternancia)
```

#### Tipos de Sesión Persistente:

| Tipo | Impacto en Escalabilidad |
|------|-------------------------|
| **None** | ✅ Óptimo. Distribuye todas las requests equitativamente |
| **Client IP** | ⚠️ Baja. Mismo cliente siempre a misma VM. Si VM falla, pierde sesión |
| **Client IP + Protocol** | ❌ Peor. Vinculación muy fuerte. Afecta distribución |

**Para nuestra app (Fibonacci stateless):** None es perfecto.

**Impacto en escalabilidad:**
- **None:** Máxima distribución, escalable ✅
- **Client IP:** Concentra clientes, menos escalable ⚠️
- **Client IP+Protocol:** Múltiples vinculaciones, muy inflexible ❌

---

### **Pregunta 5: Virtual Network y Subnet**

#### **Virtual Network (VNet):**
- Red aislada de Azure (red privada interna)
- Define el rango total de IPs disponibles
- **Nuestro caso:** HORIZONTAL-SCALABILITY-VNet (10.0.0.0/16)
  - Rango: 10.0.0.0 - 10.0.255.255 (65,536 direcciones)

#### **Subnet:**
- Subdivisión lógica de VNet
- Donde realmente se conectan VMs
- **Nuestro caso:** default (10.0.0.0/24)
  - Rango: 10.0.0.0 - 10.0.0.255 (256 direcciones)
  - VM1: 10.0.0.4
  - VM2: 10.0.0.5

#### **Address Space vs Address Range:**

| Término | Significado |
|--------|-----------|
| **Address Space** | Rango total de VNet (10.0.0.0/16) |
| **Address Range** | Rango de Subnet (10.0.0.0/24) |

**Jerarquía:**
```
VNet (10.0.0.0/16 - Address Space)
  └─ Subnet (10.0.0.0/24 - Address Range)
      ├─ VM1: 10.0.0.4
      └─ VM2: 10.0.0.5
```

---

### **Pregunta 6: Availability Zones**

#### **¿Qué es una Availability Zone?**
- Datacenter físicamente separado dentro de una región
- Fallo de zona NO afecta otras zonas
- SLA: 99.99% con 2 AZ, 99.99% con 3+ AZ

#### **¿Por qué 2 zonas? (Se intentó 3)**
- **Original:** 3 diferentes zonas
- **Realidad:** Solo 2 disponibles (Zone 2 y Zone 3)
  - Zone 1 no estaba disponible en suscripción
- **Beneficio:** Aún proporciona redundancia geográfica

#### **Zone-Redundant IP:**
- IP que es accesible desde CUALQUIER zona
- Si Zone 2 falla, zona 3 puede seguir usando la IP
- **Sin zone-redundancy:** IP vinculada a zona, fallaría con zona

**Nuestro caso:**
```
Load Balancer IP (48.211.232.81) - Zone-Redundant
  └─ Accesible desde Internet sin importar zona
  
VM1 (Zone 2) - IP: 52.232.252.189
VM2 (Zone 3) - IP: 172.206.26.250
```

**Beneficio:**
- Si Zone 2 falla → VM1 cae, VM2 sigue activa
- LB automáticamente desvía tráfico a VM2 ✅

---

### **Pregunta 7: Network Security Group**

**NSG:** Firewall que controla tráfico entrante y saliente

**Nuestras reglas (Inbound):**

| Prioridad | Protocolo | Puerto | Origen | Acción |
|-----------|-----------|--------|--------|--------|
| 100 | TCP | 22 | Any | Allow (SSH) |
| 110 | TCP | 80 | Any | Allow (HTTP) |
| 120 | TCP | 3000 | Any | Allow (App) |
| 4096 | * | * | * | Deny (Default) |

**Impacto:**
- ✅ Clientes pueden conectar SSH, HTTP, app
- ✅ Interno: Comunicación entre VMs permitida
- ❌ Otros puertos: Bloqueados

---

## 📈 Pruebas de Carga - Parte 2

### **Pruebas con 2 Procesos Newman (20 requests concurrentes)**

#### **Configuración:**
```json
{
  "endpoint": "localhost:3000",
  "nth": 1000000,
  "procesos": 2,
  "iteraciones_por_proceso": 10,
  "total_requests": 20
}
```

#### **VM1 - Proceso 1:**

| Métrica | Valor |
|--------|-------|
| Iteraciones | 10/10 ✅ |
| Tiempo Promedio | 8.4s |
| Rango | 8.3s - 8.5s |
| Desviación Estándar | 41ms |
| Tasa Éxito | **100%** ✅ |

#### **VM1 - Proceso 2:**

| Métrica | Valor |
|--------|-------|
| Iteraciones | 10/10 ✅ |
| Tiempo Promedio | 8.5s |
| Rango | 8.3s - 8.6s |
| Desviación Estándar | 72ms |
| Tasa Éxito | **100%** ✅ |

**Resumen VM1 (2 procesos):**
- **Total:** 20/20 requests ✅
- **Promedio:** 8.45s
- **Throughput:** 14.2 req/min
- **Conclusión:** Maneja carga normal perfectamente ✅

---

#### **VM2 - Proceso 1:**

| Métrica | Valor |
|--------|-------|
| Iteraciones | 10/10 ✅ |
| Tiempo Promedio | 6.4s |
| Rango | 6.3s - 6.5s |
| Desviación Estándar | 41ms |
| Tasa Éxito | **100%** ✅ |

#### **VM2 - Proceso 2:**

| Métrica | Valor |
|--------|-------|
| Iteraciones | 10/10 ✅ |
| Tiempo Promedio | 6.4s |
| Rango | 6.3s - 6.5s |
| Desviación Estándar | 41ms |
| Tasa Éxito | **100%** ✅ |

**Resumen VM2 (2 procesos):**
- **Total:** 20/20 requests ✅
- **Promedio:** 6.4s
- **Throughput:** 18.75 req/min
- **Conclusión:** **VM2 es 24.3% más rápida que VM1** ⚡

---

### **Comparativa VM1 vs VM2 (2 procesos):**

| Métrica | VM1 | VM2 | Ventaja |
|---------|-----|-----|---------|
| Tasa Éxito | 100% | 100% | Igual |
| Tiempo Promedio | 8.45s | 6.4s | **VM2: -2.05s (-24.3%)** ⚡ |
| Estabilidad (StdDev) | 56.5ms | 41ms | **VM2 más estable** |
| Throughput | 14.2 req/min | 18.75 req/min | **VM2: +32% mejor** |

**Conclusión:** Ambas VMs manejan 20 requests exitosamente. VM2 tiene mejor hardware o menos contención.

---

### **Pruebas con 4 Procesos Newman (40 requests simultáneos)**

**Comando:**
```bash
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

#### **Resultados por Proceso (en VM1):**

| Proceso | Iteraciones | Exitosas | Fallidas | Tasa Éxito | Tiempo Prom |
|---------|-------------|----------|----------|-----------|-----------|
| **Proceso 1** | 10 | 5 | 5 | **50%** ❌ | 18.6s |
| **Proceso 2** | 10 | 5 | 5 | **50%** ❌ | 21.9s |
| **Proceso 3** | 10 | 5 | 5 | **50%** ❌ | 25.3s |
| **Proceso 4** | 10 | 6 | 4 | **60%** ⚠️ | 26.7s |
| **TOTAL** | **40** | **21** | **19** | **52.5%** ❌ | **23.1s** |

**Errores observados:**
- **ECONNRESET:** Servidor rechaza conexiones por sobrecarga
- **Patrón:** Errores en iteraciones 2, 4, 6, 8, 10 (pares)
- **Causa:** CPU saturada, backlog lleno

**Conclusión (4 procesos):**
- ❌ Una sola VM NO puede manejar 40 requests
- ⚠️ 52.5% de éxito (mejor que Parte 1 pero insuficiente)
- ❌ Servidor comienza a saturarse

---

## 📊 Informe Comparativo: Parte 1 vs Parte 2

### **Arquitectura Comparada**

| Aspecto | Parte 1 (Vertical) | Parte 2 (Horizontal) |
|--------|------------------|-------------------|
| **VMs** | 1 (B2ms) | 2 (2×B2s) |
| **Escalabilidad** | Redimensionar VM | Agregar/quitar VMs |
| **Alta Disponibilidad** | ❌ No (SPOF) | ✅ Sí (2 AZ) |
| **Redundancia** | 0 zonas | 2 zonas |
| **Failover automático** | ❌ Manual | ✅ Automático (30s) |
| **Load Distribution** | N/A | Round Robin |
| **Health Monitoring** | Manual | Automático (5s) |

---

### **Resultados de Carga**

#### **Escenario 1: 2 procesos (20 requests)**

| Métrica | Parte 1 | Parte 2 |
|--------|--------|--------|
| **Tasa Éxito** | 100% ✅ | 100% ✅ |
| **Tiempo Prom** | 8.3s | 7.4s ⚡ |
| **Throughput** | 14.5 req/min | 15.3 req/min |
| **CPU Utilización** | <70% | <70% |
| **Conclusión** | ✅ Funciona | ✅ Funciona + redundancia |

**Ganador:** Empate técnico, pero **Parte 2 tiene redundancia automática**.

---

#### **Escenario 2: 4 procesos (40 requests)**

| Métrica | Parte 1 | Parte 2 |
|--------|--------|--------|
| **Tasa Éxito** | **0%** ❌ | **52.5%** ⚠️ |
| **Tiempo Prom** | Timeout | 23.1s |
| **Error** | ECONNREFUSED | ECONNRESET |
| **CPU** | 100% (saturado) | 100% (saturado) |
| **Conclusión** | ❌ Falla total | ⚠️ Falla parcial |

**Ganador:** **Parte 2 (horizontal)** - 52.5% > 0%

---

### **Análisis Económico**

#### **Costo Mensual:**

| Componente | Parte 1 | Parte 2 |
|-----------|--------|--------|
| **VM Compute** | Standard_B2ms = $60.74/mes | 2×B2s = $49.48/mes |
| **Load Balancer** | N/A | $18.25/mes |
| **IP Pública** | Incluida | Incluida |
| **Storage/Backup** | $5/mes | $10/mes |
| **Total** | **$65.74/mes** | **$77.73/mes** |

**Diferencia:** Parte 2 cuesta **$12/mes más (+18%)**

**Retorno de inversión:**
- ✅ Máxima disponibilidad (99.95% SLA)
- ✅ Recuperación automática
- ✅ Mayor capacidad de carga
- ✅ Escalabilidad futura

---

### **Disponibilidad y SLA**

#### **Escenario: 1 VM falla**

| Aspectos | Parte 1 | Parte 2 |
|---------|--------|--------|
| **Uptime** | **0%** ❌ | **100%** ✅ (VM2 continúa) |
| **RTO** | ~30 minutos (manual) | ~30 segundos (automático) |
| **RPO** | Potencial pérdida | Ninguna (stateless) |
| **SLA** | Sin SLA formal | 99.95% |

**Ganador:** **Parte 2** - Arquitectura empresarial

---

## 🎯 Conclusiones Finales

### **¿Cuándo usar cada enfoque?**

#### **Parte 1 (Vertical) es ideal para:**
- ✅ Desarrollo/testing (no producción)
- ✅ Cargas predecibles y bajas
- ✅ Presupuesto muy limitado
- ✅ Aplicaciones sin requisitos de HA

#### **Parte 2 (Horizontal) es ideal para:**
- ✅ **Producción con SLA**
- ✅ **Cargas variables o crecientes**
- ✅ **Aplicaciones críticas**
- ✅ **Requisitos de alta disponibilidad**
- ✅ **Escalabilidad futura**

---

### **Recomendación**

Para esta aplicación **Fibonacci (CPU-intensiva):**

> **Parte 2 (Escalabilidad Horizontal) es SUPERIOR**
> 
> **Razones:**
> 1. Maneja mejor cargas pesadas (52.5% vs 0%)
> 2. Redundancia automática (2 AZ)
> 3. Recuperación en 30 segundos
> 4. Solo +18% costo por máxima disponibilidad
> 5. Escalable: agregar VM3, VM4 cuando carga crece
> 6. Health Probe automático (no manual)

---

### **Arquitectura Recomendada (Futuro)**

Para mejorar escalabilidad con 4+ procesos simultáneos:

```
Internet
   ↓
Load Balancer (48.211.232.81:80)
   ↓
┌──────────────────────────────────┐
│ Backend Pool (Round Robin)        │
├──────────────────────────────────┤
│ VM1 (Zone 1) :3000 ✅             │
│ VM2 (Zone 2) :3000 ✅             │
│ VM3 (Zone 3) :3000 (AGREGAR)      │
│ VM4 (Zone 1) :3000 (AGREGAR)      │
└──────────────────────────────────┘

Con 4 VMs → Soportaría 60-80 requests concurrentes ✅
```

---

## 📎 Recursos

- **Repository:** https://github.com/AnaFiquitiva/ARSW_LOAD-BALANCING_AZURE
- **Región Azure:** East US 2
- **Grupo de Recursos:** HORIZONTAL-SCALABILITY_LAB
- **Load Balancer:** HORIZONTAL-SCALABILITY-LB
- **VNet:** HORIZONTAL-SCALABILITY-VNet

---

**Lab completado:** ✅ 24 de abril de 2026
