# RESPUESTAS PREGUNTAS PARTE 1 - ESCALABILIDAD VERTICAL

## PREGUNTA 1: ¿Cuántos y cuáles recursos crea Azure junto con la VM?

**Respuesta:**
Azure crea automáticamente los siguientes recursos junto con la VM:

1. **Network Interface (NIC - Interfaz de Red)**
   - Para conectar la VM a la red virtual
   - Asignada a la subnet

2. **Public IP Address (Dirección IP Pública)**
   - Para acceder a la VM desde internet
   - Tipo: Standard SKU (zona-redundante)

3. **Virtual Network (VNet)**
   - Red virtual donde se encuentra la VM
   - Proporciona aislamiento lógico

4. **Subnet (Subred)**
   - Subnet dentro de la VNet
   - Define el rango de direcciones IP

5. **Network Security Group (NSG)**
   - Controla el tráfico de entrada y salida
   - Actúa como firewall

6. **OS Disk (Disco del Sistema Operativo)**
   - Contenedor de almacenamiento para Ubuntu Server
   - Gestionado automáticamente

7. **Storage Account (Implícito)**
   - Para diagnósticos de la VM
   - Almacenamiento de logs

---

## PREGUNTA 2: ¿Brevemente describa para qué sirve cada recurso?

| Recurso | Función |
|---------|---------|
| **Network Interface (NIC)** | Conecta la VM a la red, gestiona direcciones MAC e IP internas |
| **Public IP** | Permite acceso desde internet a la VM (ssh, HTTP, etc.) |
| **Virtual Network (VNet)** | Crea una red privada virtual para aislar recursos en Azure |
| **Subnet** | Segmenta la VNet en bloques de direcciones IP lógicos |
| **Network Security Group** | Actúa como firewall, permitiendo/bloqueando tráfico por puertos |
| **OS Disk** | Almacena el sistema operativo (Ubuntu Server) y datos persistentes |
| **Storage Account** | Almacena logs de diagnóstico y monitoreo de la VM |

---

## PREGUNTA 3: ¿Al cerrar la conexión ssh con la VM, por qué se cae la aplicación? ¿Por qué necesitamos Inbound port rule?

**¿Por qué se cae la aplicación al cerrar SSH?**

Cuando ejecutamos `node FibonacciApp.js` desde SSH sin usar `&` (background) o `forever`:
- El proceso Node.js está vinculado a la sesión SSH
- Al cerrar SSH, la sesión termina completamente
- Todos los procesos hijo (incluyendo Node) reciben SIGHUP (hangup signal)
- La aplicación se detiene inmediatamente

**Solución:**
- Usar `node FibonacciApp.js &` para ejecutar en background
- O usar `forever` para reinicio automático
- O crear un systemd service

**¿Por qué necesitamos Inbound port rule?**

- **Por defecto, Azure bloquea TODO el tráfico entrante** en la VM
- El NSG (Network Security Group) actúa como firewall
- Sin una regla de entrada en puerto 3000, nadie puede acceder a la aplicación web
- La regla permite tráfico TCP desde cualquier fuente (0.0.0.0/0) al puerto 3000
- Sin esta regla: conexión rechazada (ECONNREFUSED)

---

## PREGUNTA 7: ¿Cuál es la diferencia entre B2ms y B1ls?

### Diferencias de Especificaciones:

| Característica | B1ls | B2ms | Diferencia |
|---|---|---|---|
| **vCPUs** | 1 | 2 | +100% |
| **RAM (GiB)** | 0.5 | 8 | +1500% |
| **Data disks** | 2 | 4 | +100% |
| **Max IOPS** | 320 | 1920 | +500% |
| **Local storage (GiB)** | 4 | 16 | +300% |
| **Costo/mes** | $30.37 | $60.74 | 2x más caro |

### Diferencias en Rendimiento (Real según datos):

| Métrica | B1ls | B2ms |
|---|---|---|
| **Tiempos Fibonacci** | 7.76-9.02s | 8.16-9.48s |
| **Velocidad** | Más rápida | **+5% más lenta** ⚠️ |
| **CPU Utilization %** | 2.27% | 3.4% |
| **Newman promedio** | 7.5s | 7.5s |

**Conclusión:** Aunque B2ms tiene el **doble de recursos**, la aplicación Fibonacci no se beneficia porque:
- El algoritmo es **single-threaded** (usa solo 1 CPU)
- Más RAM no ayuda (la aplicación no es memory-intensive)
- B1ls está más optimizado para cargas ligeras

---

## PREGUNTA 8: ¿Es aumentar el tamaño de la VM una buena solución? ¿Qué pasa con FibonacciApp?

**Respuesta: NO es una buena solución en este caso**

**¿Qué pasa al cambiar el tamaño?**
1. La VM se **detiene automáticamente** (downtime ~1-2 minutos)
2. **Se reasigna a nuevo hardware** con recursos diferentes
3. La **IP privada cambia**, pero IP pública se mantiene
4. Todos los procesos se detienen y reinician
5. La aplicación no se reinicia automáticamente (sin `forever` o daemon)

**Problemas observados:**
- ✗ Tiempos de respuesta **empeoraron** (5% más lento)
- ✗ Consumo de CPU **subió** (3.4% vs 2.27%)
- ✗ **No hay mejora** en capacidad de carga
- ✗ **Costo se duplicó** ($60.74 vs $30.37)
- ✓ Pero requiere **mantenimiento y downtime**

**Razón principal:** La función Fibonacci es **CPU-bound single-threaded**
- Solo usa 1 núcleo de los 2 disponibles en B2ms
- Extra CPU/RAM no se aprovecha
- El cuello de botella es el algoritmo recursivo ineficiente

---

## PREGUNTA 9: ¿Qué pasa con la infraestructura al cambiar tamaño? ¿Qué efectos negativos?

**Cambios en infraestructura:**

1. **Redeployment (Redesplegue)**
   - VM se detiene completamente
   - Se libera del hardware actual
   - Se asigna a nuevo hardware con recursos B2ms

2. **Tiempo de inactividad (Downtime)**
   - Típicamente 1-3 minutos
   - La aplicación no responde durante este tiempo

3. **Cambio de dirección IP privada** (en algunos casos)
   - La IP pública se mantiene
   - Pero conexiones internas se pierden

**Efectos negativos:**

| Efecto | Impacto |
|--------|---------|
| **Downtime** | Usuarios no pueden acceder a la app (~2 min) |
| **Pérdida de estado en memoria** | Node reinicia, cache se pierde |
| **Mayor costo** | 2x más caro mensualmente |
| **Sin beneficio real** | Algoritmo no se optimiza con más CPU |
| **Complejidad operacional** | Necesita mantenimiento y monitoreo |

**Mejor alternativa:** Escalabilidad Horizontal (Parte 2)
- Agregar más VMs pequeñas en lugar de hacer 1 grande
- Distribuir carga entre múltiples instancias
- Sin downtime
- Más económico

---

## PREGUNTA 10: ¿Hubo mejora en CPU o tiempos de respuesta? Si/No ¿Por qué?

**Respuesta: NO, no hubo mejora**

### Datos Comparativos:

**Tiempos de Respuesta:**
```
B1ls:   7.76-9.02s  (promedio 7.5s)
B2ms:   8.16-9.48s  (promedio 7.5s)
Resultado: +5% MÁS LENTO ❌
```

**Consumo de CPU:**
```
B1ls:   2.27%
B2ms:   3.4%
Resultado: +50% MÁS CONSUMO ❌
```

**Newman Tests (paralelismo):**
```
B1ls:   7.5s promedio
B2ms:   7.5s promedio
Resultado: SIN CAMBIO ➖
```

### ¿Por qué NO mejoró?

1. **Algoritmo Fibonacci es single-threaded**
   - Solo usa 1 CPU core
   - No saca ventaja de los 2 cores de B2ms

2. **No es memory-bound**
   - De 0.5GB en B1ls a 8GB en B2ms
   - La RAM extra no ayuda

3. **Bottleneck es el algoritmo recursivo ineficiente**
   - Recalcula valores múltiples veces
   - Complejidad O(2^n) terrible
   - Doblando recursos CPU/RAM no lo arregla

4. **Overhead de infraestructura**
   - B2ms usa más recursos incluso sin carga
   - VM más "pesada" consume más en baseline

### Conclusión:
B2ms es **peor** que B1ls para esta aplicación:
- ❌ Más lento
- ❌ Más CPU consumption
- ❌ 2x más caro
- ✓ El único beneficio sería escalar aplicaciones multi-threaded

---

## PREGUNTA 11: Aumente a 4 ejecuciones paralelas de Newman. ¿Mejor comportamiento?

**Estado:** ⏳ **PENDIENTE DE EJECUCIÓN**

**Plan para completar:**

Se necesita ejecutar en la VM B1ls (para comparación):
```bash
# 4 procesos en paralelo
newman run ... -n 10 &
newman run ... -n 10 &
newman run ... -n 10 &
newman run ... -n 10
```

**Métricas esperadas a recolectar:**
1. Tasa de éxito (vs 100% actual con 2 paralelos)
2. Tiempos de respuesta (probablemente aumentarán)
3. CPU utilization (debe alcanzar ~100%)
4. Posibles timeouts o errores

**Hipótesis:**
- Con 4 procesos paralelos, ambas VMs probablemente fallen
- B1ls (1 CPU): sobrecarga extrema, muchos timeouts
- B2ms (2 CPU): mejor, pero aún limitado

---

## ✅ RESUMEN: REQUISITOS PENDIENTES DE PARTE 1

| Pregunta | Estado | Qué falta |
|----------|--------|-----------|
| 1 | ✅ Respondida | - |
| 2 | ✅ Respondida | - |
| 3 | ✅ Respondida | - |
| 4 | ✅ Respondida | Tabla + interpretación |
| 5 | ✅ Respondida | Imagen + interpretación |
| 6 | ✅ Respondida | 3 Imágenes Newman |
| 7 | ✅ Respondida | - |
| 8 | ✅ Respondida | - |
| 9 | ✅ Respondida | - |
| 10 | ✅ Respondida | - |
| 11 | ⏳ Incompleta | **Ejecutar 4 procesos Newman** |
| 12 | ⏳ Incompleta | Evaluación de escalabilidad |
| 13 | ⏳ Pendiente | **Cambiar VM a B1ls** |

### PRÓXIMO PASO:
Ejecutar la pregunta 11 con 4 procesos en paralelo y capturar resultados.
