# 📊 CHECKLIST CAPTURAS - PARTE 1

## ✅ PARTE 1 - PASOS INICIALES (Completos con B1ls)

### Pasos de Configuración (1-9)
| Paso | Descripción | Archivo | Estado |
|------|-------------|---------|--------|
| 1 | Crear VM B1ls (config básica) | `part1-vm-basic-config.png` | ✅ TIENES |
| 6 | Inbound port rule (puerto 3000) | `part1-vm-3000InboudRule.png` | ✅ TIENES |
| 8 | CPU monitoring (B1ls) | `part1-vm-cpu-B1ls.png` | ✅ TIENES |
| 10 | Resize VM a B2ms | `part1-vm-resize.png` | ✅ TIENES |

### Preguntas con Datos B1ls (4-6)
| Pregunta | Dato Requerido | Archivo | Estado |
|----------|----------------|---------|--------|
| 4 | Tabla tiempos B1ls | `part1-tiempos-respuesta.png` | ✅ AGREGADA |
| 5 | Gráfica CPU B1ls | `part1-vm-cpu-B1ls.png` | ✅ AGREGADA |
| 6 | Newman tests B1ls | `part1-newman-test{1,2,3}.png` | ✅ AGREGADAS (3 tests) |

**Resumen B1ls:**
- ✅ Tiempos: 7.5-9.0 segundos
- ✅ CPU: ~2.27% (bajo)
- ✅ Newman: 0 fallos, 100% éxito

---

## ⏳ PARTE 1 - ESCALAMIENTO VERTICAL (B2ms) - FALTA

Según paso 11: *"Una vez el cambio se vea reflejado, repita el paso 7, 8 y 9"*

### Datos Necesarios CON TAMAÑO B2ms

| Paso | Descripción | Archivo Esperado | Prioridad |
|------|-------------|------------------|-----------|
| **7** | Tabla tiempos (n=1000000-1090000) | `part1-tiempos-respuesta-B2ms.png` | 🔴 URGENTE |
| **8** | Gráfica CPU con B2ms | `part1-vm-cpu-B2ms.png` | 🔴 URGENTE |
| **9** | Newman tests B2ms | `part1-newman-B2ms-test{1,2,3}.png` | 🔴 URGENTE |

---

## 📋 PLAN DE ACCIÓN INMEDIATO

### AHORA:
1. ✅ Cambiar VM a tamaño **B2ms** en Azure
   - Ir a: Portal → Tu VM → Size → B2ms
   - Esperar a que se complete (5-10 min)

2. 🔴 **Ejecutar PASO 7 (Tiempos)** con B2ms
   - Acceder a `http://localhost:3000/fibonacci/{valor}`
   - Probar con: 1000000, 1010000... 1090000
   - **Tomar captura de la tabla de resultados**

3. 🔴 **Ejecutar PASO 8 (CPU)** con B2ms
   - Ir a Azure Monitoring → Metrics
   - **Tomar captura de la gráfica de CPU**

4. 🔴 **Ejecutar PASO 9 (Newman)** con B2ms
   - Ejecutar 3 tests iguales como antes (10 iteraciones c/u)
   - **Tomar 3 capturas de los resultados**

### DESPUÉS:
- Responder preguntas 7-11 con comparación B1ls vs B2ms
- Pregunta 7: Diferencia de specs entre tamaños
- Pregunta 10: ¿Hubo mejora? (esperado: SÍ, más velocidad)
- Pregunta 11: Performance con 4 procesos paralelos

---

## 🎯 COMPARATIVA ESPERADA (B1ls vs B2ms)

| Métrica | B1ls | B2ms | Esperado |
|---------|------|------|----------|
| **CPU cores** | 1 | 2 | ↑ Mejor |
| **Memoria RAM** | 0.5GB | 4GB | ↑ Mejor |
| **Tiempos respuesta** | ~7.5s | ~? | ↓ Más rápido |
| **Consumo CPU** | ~2.27% | ~? | Variable |
| **Newman éxito** | 100% | Esperado 100% | ✅ |

---

## ✍️ NOTAS IMPORTANTES

- **paso 13**: después de completar todo, vuelve a B1ls para evitar cobros
- No elimines la VM, solo cambia el tamaño
- Los tiempos pueden mejorar porque hay más recursos
- La CPU % puede ser más baja porque se distribuye en 2 cores

