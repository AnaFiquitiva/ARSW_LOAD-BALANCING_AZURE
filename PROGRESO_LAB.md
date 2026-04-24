# 📋 PROGRESO DEL LAB 9 - ESCALABILIDAD EN AZURE

## PARTE 1: Escalabilidad Vertical ✅

### Estado Actual de Capturas

| Paso | Descripción | Captura | Estado | Notas |
|------|-------------|---------|--------|-------|
| 1 | Crear VM (configuración básica) | `images/part1/part1-vm-basic-config.png` | ✅ Referenciada | Ya debes tener esta imagen |
| 6 | Inbound port rule (puerto 3000) | `images/part1/part1-vm-3000InboudRule.png` | ✅ Referenciada | Ya debes tener esta imagen |
| 8 | Consumo de CPU (con tamaño B1ls) | `images/part1/part1-vm-cpu.png` | ✅ Referenciada | Necesitas captura actual |
| 10 | Resize VM a B2ms | `images/part1/part1-vm-resize.png` | ✅ Referenciada | Ya debes tener esta imagen |

### Datos que Debes Documentar

| Pregunta | Requiere | Tienes | Estado |
|----------|----------|--------|--------|
| **Pregunta 4** | Tabla de tiempos (n=1000000 a 1090000) | ✅ part1-tiempos-respuesta.png | **✅ AGREGADA AL README** |
| **Pregunta 5** | Imagen consumo CPU (VM B1ls) | ✅ part1-vm-cpu-B1ls.png | **✅ AGREGADA AL README** |
| **Pregunta 6** | Resultado Postman/Newman | ❌ No | **SIGUIENTE PASO** |

---

## PARTE 2: Escalabilidad Horizontal 🔄

### Criar Balanceador de Carga

| Paso | Recurso a Crear | Captura Referenciada | Estado |
|------|-----------------|----------------------|--------|
| 1 | Load Balancer | `images/part2/part2-lb-create.png` | ⏳ Pendiente |
| 2 | Backend Pool | `images/part2/part2-lb-bp-create.png` | ⏳ Pendiente |
| 3 | Health Probe | `images/part2/part2-lb-hp-create.png` | ⏳ Pendiente |
| 4 | Load Balancing Rule | `images/part2/part2-lb-lbr-create.png` | ⏳ Pendiente |
| 5 | Virtual Network | `images/part2/part2-vn-create.png` | ⏳ Pendiente |

### Crear VMs (3 Nodos)

| VM | AZ | Captura 1 | Captura 2 | Captura 3 | Captura 4 | Estado |
|----|----|-----------|-----------|-----------|-----------| -------|
| VM1 | Zone 1 | `part2-vm-create1.png` | `part2-vm-create2.png` | `part2-vm-create3.png` | `part2-vm-create4.png` | ⏳ Pendiente |
| VM2 | Zone 2 | " | " | ← Same NSG | " | ⏳ Pendiente |
| VM3 | Zone 3 | " | " | ← Same NSG | " | ⏳ Pendiente |

### Pruebas Finales

| Punto | Descripción | Requiere | Estado |
|-------|-------------|----------|--------|
| **Punto 2** | Informe comparativo (escalabilidad vertical vs horizontal) | Newman results x2 | ⏳ Pendiente |
| **Punto 3** | Add VM4 + pruebas con 4 procesos de Newman en paralelo | Newman results + CPU graphs | ⏳ Pendiente |
| **Diagrama** | Diagrama de Despliegue final | Mermaid/Visio/Draw.io | ⏳ Pendiente |

---

## ✅ CHECKLIST DE TAREAS INMEDIATAS

### AHORA (Terminar Parte 1)
- [x] Agregar tabla de tiempos a la pregunta 4 en el README ✅
- [x] Obtener captura de **consumo de CPU** en Azure (paso 8 de Parte 1) ✅
- [ ] Ejecutar Newman test **10 iteraciones en 2 procesos paralelos** (paso 9)
- [ ] Guardar captura del resultado de Newman

### DESPUÉS (Pasar a Parte 2)
- [ ] Crear **Load Balancer** en Azure
- [ ] Configurar **Backend Pool**
- [ ] Configurar **Health Probe**
- [ ] Configurar **Load Balancing Rule**
- [ ] Crear **Virtual Network y Subnet**
- [ ] Crear **3 VMs** en diferentes zonas de disponibilidad
- [ ] Instalar FibonacciApp en cada VM
- [ ] Pruebas de carga con Newman (comparación)
- [ ] Crear VM4 y pruebas finales
- [ ] Crear diagrama de despliegue

---

## 📊 CAPTURAS QUE NECESITAS URGENTEMENTE

1. **Consumo de CPU** (Paso 8 - Parte 1)  
   ├─ Ir a Azure Portal → Tu VM → Monitoring → Metrics  
   ├─ Seleccionar métrica "CPU Percentage"  
   └─ Tomar captura

2. **Resultado Newman** (Paso 9 - Parte 1)  
   ├─ Ejecutar: `newman run ... -n 10` (2 procesos en paralelo)  
   └─ Tomar captura del resumen final

3. **Tabla de tiempos** (Ya tienes en Captura_lab9.png)  
   └─ Agregar al README en pregunta 4

---

## 📝 PRÓXIMO PASO

¿Necesitas ayuda para:
1. ✅ **Agregar tu tabla de tiempos** al README?
2. ✅ **Obtener la captura de CPU** en Azure?
3. ✅ **Ejecutar las pruebas de Newman**?

