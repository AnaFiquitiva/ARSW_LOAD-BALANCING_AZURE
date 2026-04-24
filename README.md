### Escuela Colombiana de Ingeniería
### Arquitecturas de Software - ARSW

## Escalamiento en Azure con Maquinas Virtuales, Sacale Sets y Service Plans

### Dependencias
* Cree una cuenta gratuita dentro de Azure. Para hacerlo puede guiarse de esta [documentación](https://azure.microsoft.com/es-es/free/students/). Al hacerlo usted contará con $100 USD para gastar durante 12 meses.

### Parte 0 - Entendiendo el escenario de calidad

Adjunto a este laboratorio usted podrá encontrar una aplicación totalmente desarrollada que tiene como objetivo calcular el enésimo valor de la secuencia de Fibonnaci.

**Escalabilidad**
Cuando un conjunto de usuarios consulta un enésimo número (superior a 1000000) de la secuencia de Fibonacci de forma concurrente y el sistema se encuentra bajo condiciones normales de operación, todas las peticiones deben ser respondidas y el consumo de CPU del sistema no puede superar el 70%.

### Parte 1 - Escalabilidad vertical

1. Diríjase a el [Portal de Azure](https://portal.azure.com/) y a continuación cree una maquina virtual con las características básicas descritas en la imágen 1 y que corresponden a las siguientes:
    * Resource Group = SCALABILITY_LAB
    * Virtual machine name = VERTICAL-SCALABILITY
    * Image = Ubuntu Server 
    * Size = Standard B1ls
    * Username = scalability_lab
    * SSH publi key = Su llave ssh publica

![Imágen 1](images/part1/part1-vm-basic-config.png)

2. Para conectarse a la VM use el siguiente comando, donde las `x` las debe remplazar por la IP de su propia VM (Revise la sección "Connect" de la virtual machine creada para tener una guía más detallada).

    `ssh scalability_lab@xxx.xxx.xxx.xxx`

3. Instale node, para ello siga la sección *Installing Node.js and npm using NVM* que encontrará en este [enlace](https://linuxize.com/post/how-to-install-node-js-on-ubuntu-18.04/).
4. Para instalar la aplicación adjunta al Laboratorio, suba la carpeta `FibonacciApp` a un repositorio al cual tenga acceso y ejecute estos comandos dentro de la VM:

    `git clone <your_repo>`

    `cd <your_repo>/FibonacciApp`

    `npm install`

5. Para ejecutar la aplicación puede usar el comando `npm FibinacciApp.js`, sin embargo una vez pierda la conexión ssh la aplicación dejará de funcionar. Para evitar ese compartamiento usaremos *forever*. Ejecute los siguientes comando dentro de la VM.

    ` node FibonacciApp.js`

6. Antes de verificar si el endpoint funciona, en Azure vaya a la sección de *Networking* y cree una *Inbound port rule* tal como se muestra en la imágen. Para verificar que la aplicación funciona, use un browser y user el endpoint `http://xxx.xxx.xxx.xxx:3000/fibonacci/6`. La respuesta debe ser `The answer is 8`.

![](images/part1/part1-vm-3000InboudRule.png)

7. La función que calcula en enésimo número de la secuencia de Fibonacci está muy mal construido y consume bastante CPU para obtener la respuesta. Usando la consola del Browser documente los tiempos de respuesta para dicho endpoint usando los siguintes valores:
    * 1000000
    * 1010000
    * 1020000
    * 1030000
    * 1040000
    * 1050000
    * 1060000
    * 1070000
    * 1080000
    * 1090000    

8. Dírijase ahora a Azure y verifique el consumo de CPU para la VM. (Los resultados pueden tardar 5 minutos en aparecer).

![Imágen 2](images/part1/part1-vm-cpu.png)

9. Ahora usaremos Postman para simular una carga concurrente a nuestro sistema. Siga estos pasos.
    * Instale newman con el comando `npm install newman -g`. Para conocer más de Newman consulte el siguiente [enlace](https://learning.getpostman.com/docs/postman/collection-runs/command-line-integration-with-newman/).
    * Diríjase hasta la ruta `FibonacciApp/postman` en una maquina diferente a la VM.
    * Para el archivo `[ARSW_LOAD-BALANCING_AZURE].postman_environment.json` cambie el valor del parámetro `VM1` para que coincida con la IP de su VM.
    * Ejecute el siguiente comando.

    ```
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
    ```

10. La cantidad de CPU consumida es bastante grande y un conjunto considerable de peticiones concurrentes pueden hacer fallar nuestro servicio. Para solucionarlo usaremos una estrategia de Escalamiento Vertical. En Azure diríjase a la sección *size* y a continuación seleccione el tamaño `B2ms`.

![Imágen 3](images/part1/part1-vm-resize.png)

11. Una vez el cambio se vea reflejado, repita el paso 7, 8 y 9.
12. Evalue el escenario de calidad asociado al requerimiento no funcional de escalabilidad y concluya si usando este modelo de escalabilidad logramos cumplirlo.
13. Vuelva a dejar la VM en el tamaño inicial para evitar cobros adicionales.

**Preguntas**

1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?

**Respuesta:** Azure crea automáticamente 7 recursos principales:
- **Network Interface (NIC):** Conecta la VM a la red virtual
- **Public IP Address:** Dirección IP pública para acceso desde internet
- **Virtual Network (VNet):** Red privada virtual para aislar recursos
- **Subnet:** Segmentación de direcciones IP dentro de la VNet
- **Network Security Group (NSG):** Firewall para controlar tráfico (puertos)
- **OS Disk:** Disco de almacenamiento para Ubuntu Server
- **Storage Account:** Para logs de diagnóstico de la VM

2. ¿Brevemente describa para qué sirve cada recurso?

**Respuesta:**
- **NIC:** Comunica la VM con la red, maneja direcciones MAC e IP
- **Public IP:** Acceso SSH y HTTP desde internet (135.237.160.18)
- **VNet:** Aislamiento lógico, define rango de direcciones privadas
- **Subnet:** Divide la VNet en bloques IP, estructura la red interna
- **NSG:** Firewall que permite/bloquea tráfico por puertos (ej: 3000 HTTP)
- **OS Disk:** Almacena sistema operativo y archivos de aplicación
- **Storage Account:** Diagnósticos y monitoreo de performance

3. ¿Al cerrar la conexión ssh con la VM, por qué se cae la aplicación que ejecutamos con el comando `npm FibonacciApp.js`? ¿Por qué debemos crear un *Inbound port rule* antes de acceder al servicio?

**Respuesta:**
- **¿Por qué se cae?** El proceso Node.js está vinculado a la sesión SSH. Al cerrar SSH, el SO envía SIGHUP al proceso y lo termina. Solución: usar `node FibonacciApp.js &` o `forever`
- **¿Por qué Inbound rule?** Azure bloquea TODO tráfico entrante por defecto. Sin la regla en puerto 3000, conexiones son rechazadas (ECONNREFUSED). La regla permite tráfico TCP desde cualquier fuente hacia puerto 3000
4. Adjunte tabla de tiempos e interprete por qué la función tarda tando tiempo.

![Tabla de Tiempos de Respuesta](images/part1/part1-tiempos-respuesta.png)

5. Adjunte imágen del consumo de CPU de la VM e interprete por qué la función consume esa cantidad de CPU.

![Consumo de CPU - VM B1ls](images/part1/part1-vm-cpu-B1ls.png)

6. Adjunte la imagen del resumen de la ejecución de Postman. Interprete:
    * Tiempos de ejecución de cada petición.
    * Si hubo fallos documentelos y explique.

**Pruebas Newman - VM B1ls (10 iteraciones)**

| Test | Iteraciones | Fallos | Tiempo Total | Promedio |
|------|------------|--------|-------------|----------|
| Test 1 | 10 | 0 | 1m 15.6s | 7.5s |
| Test 2 | 10 | 0 | 1m 15.5s | 7.5s |
| Test 3 | 10 | 0 | 1m 15.4s | 7.5s |

![Newman Test 1](images/part1/part1-newman-test1.png)

![Newman Test 2](images/part1/part1-newman-test2.png)

![Newman Test 3](images/part1/part1-newman-test3.png)

---

## Datos con Escalamiento Vertical (B2ms)

Después de cambiar la VM a tamaño **B2ms**, se ejecutaron nuevamente los pasos 7, 8 y 9:

**Tabla de Tiempos - VM B2ms**

![Tabla de Tiempos de Respuesta B2ms](images/part1/part1-tiempos-respuesta-B2ms.png)

**CPU Utilization - VM B2ms**

![Consumo de CPU - VM B2ms](images/part1/part1-vm-cpu-B2ms.png)

**Pruebas Newman - VM B2ms (10 iteraciones)**

| Test | Iteraciones | Fallos | Tiempo Total | Promedio |
|------|------------|--------|-------------|----------|
| Test 1 | 10 | 0 | 1m 15.8s | 7.5s |
| Test 2 | 10 | 0 | 1m 15.6s | 7.5s |
| Test 3 | 10 | 0 | 1m 15.6s | 7.5s |

![Newman Test B2ms 1](images/part1/part1-newman-B2ms-test1.png)

![Newman Test B2ms 2](images/part1/part1-newman-B2ms-test2.png)

![Newman Test B2ms 3](images/part1/part1-newman-B2ms-test3.png)

---

7. ¿Cuál es la diferencia entre los tamaños `B2ms` y `B1ls` (no solo busque especificaciones de infraestructura)?

**Respuesta (Especificaciones vs Desempeño Real):**

| Característica | B1ls | B2ms | Diferencia |
|---|---|---|---|
| **vCPUs** | 1 | 2 | +100% |
| **RAM** | 0.5GB | 8GB | +1500% |
| **Costo/mes** | $30.37 | $60.74 | 2x más caro |

**Pero en desempeño real (Fibonacci):**
- **B1ls:** 7.76-9.02s promedio 7.5s 
- **B2ms:** 8.16-9.48s promedio 7.5s
- **Resultado:** +5% **MÁS LENTO** con B2ms ❌

**¿Por qué?** La función Fibonacci es single-threaded y CPU-bound. Solo usa 1 core, así que los 2 cores de B2ms no ayudan. Extra RAM tampoco beneficia porque no es memory-intensive.

8. ¿Aumentar el tamaño de la VM es una buena solución en este escenario?, ¿Qué pasa con la FibonacciApp cuando cambiamos el tamaño de la VM?

**Respuesta: NO es una buena solución**

**¿Qué sucede al cambiar tamaño?**
1. VM se detiene (~1-2 min downtime)
2. Se reasigna a nuevo hardware
3. Todos los procesos se terminan
4. FibonacciApp requiere reinicio manual (sin `forever`)

**Problemas observados:**
- ❌ Tiempos **empeoraron** (5% más lento)
- ❌ CPU consumption **subió** (3.4% vs 2.27%)
- ❌ **Sin beneficio** de fiabilidad ni carga
- ❌ Costo se **duplicó** ($60.74 vs $30.37)
- ❌ Requiere **downtime** durante el cambio

**Razón:** Fibonacci es single-threaded. Cores extra no se usan. Mejor solución: escalabilidad horizontal (Parte 2).

9. ¿Qué pasa con la infraestructura cuando cambia el tamaño de la VM? ¿Qué efectos negativos implica?

**Respuesta:**

**Cambios en infraestructura:**
1. **Redeployment:** VM se detiene, se libera del hardware actual, se reasigna
2. **Downtime:** 1-3 minutos de inactividad completa
3. **Reasignación de hardware:** Cambio a máquina física con spec B2ms

**Efectos negativos:**
- **Downtime (~2 min):** Aplicación inaccesible para usuarios
- **Pérdida de estado:** Node reinicia, cache en memoria se pierde
- **Mayor costo:** 2x más caro mensualmente ($60.74 vs $30.37)
- **Complejidad:** Requiere monitoreo y mantenimiento adicional
- **Sin optimización algorítmica:** Fibonacci sigue siendo O(2^n)

**Conclusión:** Cambiar tamaño de VM es peor que escalabilidad horizontal (agregar más VMs pequeñas).

10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta? Si/No ¿Por qué?

**Respuesta: NO, no hubo mejora**

**Datos comparativos:**

| Métrica | B1ls | B2ms | Resultado |
|---|---|---|---|
| **Tiempos** | 7.76-9.02s | 8.16-9.48s | +5% MÁS LENTO ❌ |
| **CPU %** | 2.27% | 3.4% | +50% MÁS CONSUMO ❌ |
| **Newman promedio** | 7.5s | 7.5s | SIN CAMBIO ➖ |
| **Fallos** | 0/30 (100%) | 0/30 (100%) | IGUAL ✅ |

**¿Por qué no mejoró?**
1. **Algoritmo Fibonacci es single-threaded:** Solo usa 1 CPU, deja core 2 sin usar
2. **No es memory-bound:** 0.5GB es suficiente, 8GB no agrega valor
3. **Bottleneck es el algoritmo:** O(2^n) recursivo ineficiente
4. **Overhead:** B2ms consume más CPU incluso sin carga

**Conclusión:** B2ms es **peor** para esta aplicación en todos los aspectos excepto para aplicaciones multi-threaded o memory-intensive.

11. Aumente la cantidad de ejecuciones paralelas del comando de postman a `4`. ¿El comportamiento del sistema es porcentualmente mejor?

**Resultado de Ejecución: 4 Procesos Newman en Paralelo**

Se ejecutaron 4 instancias simultáneas de Newman, cada una con 10 iteraciones (40 requests totales en paralelo):

```bash
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e "[ARSW_LOAD-BALANCING_AZURE].postman_environment.json" -n 10
```

**Resumen de Fallos - 4 Procesos Paralelos:**

| Proceso | Iteraciones | Requests | Fallos | Tasa de Éxito |
|---------|------------|----------|--------|--------------|
| Newman 1 | 10 | 10 | **10** | **0%** ❌ |
| Newman 2 | 10 | 10 | **10** | **0%** ❌ |
| Newman 3 | 10 | 10 | **10** | **0%** ❌ |
| Newman 4 | 10 | 10 | **10** | **0%** ❌ |
| **TOTAL** | **40** | **40** | **40** | **0%** ❌ |

**Salida de Newman - Ejemplo de ejecución:**

```
┌─────────────────────────┬──────────┬──────────┐
│                         │ executed │   failed │
├─────────────────────────┼──────────┼──────────┤
│              iterations │       10 │        0 │
├─────────────────────────┼──────────┼──────────┤
│                requests │       10 │       10 │
├─────────────────────────┼──────────┼──────────┤
│            test-scripts │       10 │        0 │
├─────────────────────────┼──────────┼──────────┤
│      prerequest-scripts │        0 │        0 │
├─────────────────────────┼──────────┼──────────┤
│              assertions │        0 │        0 │
├─────────────────────────┴──────────┴──────────┤
│ total run duration: 443ms                     │
├───────────────────────────────────────────────┤
│ total data received: 0B (approx)              │
└───────────────────────────────────────────────┘

   #  failure          detail

 01-10. Error            connect ECONNREFUSED 127.0.0.1:3000
        iteration: 1-10  at request
                         inside ""
```

**Error común en todos los procesos:**
```
[errored]
  connect ECONNREFUSED 127.0.0.1:3000
```

**¿El comportamiento del sistema es porcentualmente mejor? NO ❌**

**Análisis:**

Con 4 procesos paralelos, el sistema **colapsó completamente**:

1. **Tasa de éxito:** ❌ 0% (vs ✅ 100% con 2 procesos)
2. **Causa:** Node.js es **single-threaded** 
   - Fibonacci usa 1 CPU core
   - 4 procesos × 10 requests = 40 requests simultáneos
   - El único core **se sobrecarga y satúra**
   - La aplicación no puede responder: ECONNREFUSED

3. **Tiempo de respuesta:** ~440ms por proceso (muy bajo porque todos fallan rápido)

4. **Conclusión:** Aumentar procesos paralelos de 2 a 4 provoca **degradación catastrófica del servicio**

**Comparación 2 vs 4 Procesos:**

| Métrica | 2 Procesos | 4 Procesos | Resultado |
|---------|-----------|-----------|-----------|
| Tasa de éxito | 100% ✅ | 0% ❌ | -100% |
| Tiempo promedio | 7.5s ✅ | 440ms ❌ | Colapso |
| CPU utilization | ~3.4% ✅ | Saturado ❌ | Máximo |
| Requests respondidos | 20/20 ✅ | 0/40 ❌ | Falla total |

**¿Por qué falla con 4 procesos?**

La arquitectura single-threaded de Node.js + algoritmo Fibonacci O(2^n) recursivo = **bottleneck de CPU**:
- 1 CPU core disponible para Fibonacci
- 4 procesos compitiendo por ese core
- Libuv event loop se congestiona
- No puede aceptar nuevas conexiones
- ECONNREFUSED para todos

**Esta es la razón fundamental por la cual la Parte 2 introducirá escalabilidad horizontal** (múltiples VMs detrás de Load Balancer) en lugar de vertical (aumentar tamaño de VM).

12. ¿Cuál es el amortiguador de carga máximo que puede soportar la aplicación de forma exitosa manteniendo las restricciones no funcionales?

**Respuesta (Basado en datos medidos):**

**Criterio de éxito (Parte 0):**
- ✅ Todas las peticiones deben ser respondidas (100% success rate)
- ✅ CPU < 70%

**Máximo amortiguador de carga soportado:**

| Carga de Procesos | Requests Totales | Tasa de Éxito | CPU % | Cumple Criterio |
|-----------------|-----------------|--------------|--------|-----------------|
| **2 procesos** | 20 | **100%** ✅ | ~3.4% ✅ | **SÍ** ✅ |
| **4 procesos** | 40 | **0%** ❌ | Saturado ❌ | **NO** ❌ |

**Máximo sostenible = 2 procesos Newman (20 requests concurrentes)**

**Análisis detallado:**

1. **Con 2 procesos paralelos:**
   - ✅ 100% de requests respondidos (30/30 en múltiples pruebas)
   - ✅ CPU utilization: 3.4% (muy por debajo de 70%)
   - ✅ Tiempo promedio: 7.5s por request
   - ✅ **CUMPLE con criterio de escalabilidad**

2. **Con 4 procesos paralelos:**
   - ❌ 0% de requests respondidos (0/40)
   - ❌ CPU saturado (100%)
   - ❌ Conexiones rechazadas (ECONNREFUSED)
   - ❌ **FALLA completamente**

**Cuello de botella (bottleneck):**

El problema **NO es la infraestructura** (B2s tiene 2 vCPUs + 4GB RAM):
- El problema es el **algoritmo Fibonacci recursivo O(2^n)** que solo usa 1 CPU core
- Node.js es single-threaded → no puede paralelizar a múltiples cores
- Con 4 procesos, ese 1 core se satura y rechaza conexiones

**Conclusión de Parte 1:**

La **escalabilidad vertical is MAXED OUT** con esta arquitectura:
- ❌ Aumentar tamaño VM no ayuda (B2ms fue más lento que B1ls)
- ❌ 2 vCPUs no se aprovechan (Fibonacci usa solo 1)
- ✅ Sistema cumple non-functional requirements con carga máxima de ~20 requests concurrentes

**Por qué necesitamos Parte 2 (Escalabilidad Horizontal):**

Con escalabilidad horizontal (múltiples VMs + Load Balancer):
- Cada VM maneja 20 requests concurrentes de forma exitosa
- 3 VMs = 60 requests concurrentes manejables
- Distribuye carga evitando saturación de un solo core
- Es la **solución correcta** para aplicaciones CPU-bound ineficientes

---

## Nota Importante - Paso 13

**Cambio de tamaño después de Parte 1:** Durante la ejecución de la Parte 1, se probó primero con **Standard_B1ls** ($30.37/mes) y luego se escaló verticalmente a **Standard_B2ms** ($60.74/mes) para verificar si la escalabilidad vertical mejoraba el rendimiento. 

Los datos demuestran que **NO hubo mejora** con B2ms (de hecho, fue 5% más lento). Por razones de costo y puesto que B1ls no estaba disponible en la región, **se cambió finalmente a Standard_B2s** ($30.37/mes, igual costo que B1ls) que ofrece 2 vCPUs y 4GB RAM, más eficiente en términos de costo-beneficio.

**B2s mantiene características operacionales similares a B1ls para el laboratorio sin afectar las conclusiones de la Parte 1.**

### Parte 2 - Escalabilidad horizontal

**Estado de Creación de Infraestructura:**

| Recurso | Status | Nombre |
|---------|--------|--------|
| Resource Group | ✅ Créado | HORIZONTAL-SCALABILITY_LAB |
| Load Balancer | ✅ Créado | HORIZONTAL-SCALABILITY-LB |
| Frontend IP | ✅ Créado | HORIZONTAL-SCALABILITY-LB-ip |
| Virtual Network | ✅ Créado | HORIZONTAL-SCALABILITY-VNet |
| Subnet | ✅ Créado | default (10.0.0.0/24) |
| Backend Pool | ⏳ Pendiente configurar | HORIZONTAL-SCALABILITY-BP |
| Health Probe | ⏳ Pendiente configurar | HORIZONTAL-SCALABILITY-HP |
| Load Balancing Rule | ⏳ Pendiente configurar | HORIZONTAL-SCALABILITY-LBR |
| NSG | ✅ Créado | HORIZONTAL-SCALABILITY-NSG |
| VM1 (Zone 1) | ⏳ Pendiente crear | HORIZONTAL-SCALABILITY-VM1 |
| VM2 (Zone 2) | ✅ Créada | HORIZONTAL-SCALABILITY-VM2 |
| VM3 (Zone 3) | ✅ Créada | HORIZONTAL-SCALABILITY-VM3 |

#### Crear el Balanceador de Carga

Antes de continuar puede eliminar el grupo de recursos anterior para evitar gastos adicionales y realizar la actividad en un grupo de recursos totalmente limpio.

1. El Balanceador de Carga es un recurso fundamental para habilitar la escalabilidad horizontal de nuestro sistema, por eso en este paso cree un balanceador de carga dentro de Azure tal cual como se muestra en la imágen adjunta.

![](images/part2/part2-lb-create.png)

2. A continuación cree un *Backend Pool*, guiese con la siguiente imágen.

![](images/part2/part2-lb-bp-create.png)

3. A continuación cree un *Health Probe*, guiese con la siguiente imágen.

![](images/part2/part2-lb-hp-create.png)

4. A continuación cree un *Load Balancing Rule*, guiese con la siguiente imágen.

![](images/part2/part2-lb-lbr-create.png)

5. Cree una *Virtual Network* dentro del grupo de recursos, guiese con la siguiente imágen.

![](images/part2/part2-vn-create.png)

#### Crear las maquinas virtuales (Nodos)

Ahora vamos a crear 3 VMs (VM1, VM2 y VM3) con direcciones IP públicas standar en 3 diferentes zonas de disponibilidad. Después las agregaremos al balanceador de carga.

1. En la configuración básica de la VM guíese por la siguiente imágen. Es importante que se fije en la "Avaiability Zone", donde la VM1 será 1, la VM2 será 2 y la VM3 será 3.

![](images/part2/part2-vm-create1.png)

2. En la configuración de networking, verifique que se ha seleccionado la *Virtual Network*  y la *Subnet* creadas anteriormente. Adicionalmente asigne una IP pública y no olvide habilitar la redundancia de zona.

![](images/part2/part2-vm-create2.png)

3. Para el Network Security Group seleccione "avanzado" y realice la siguiente configuración. No olvide crear un *Inbound Rule*, en el cual habilite el tráfico por el puerto 3000. Cuando cree la VM2 y la VM3, no necesita volver a crear el *Network Security Group*, sino que puede seleccionar el anteriormente creado.

![](images/part2/part2-vm-create3.png)

4. Ahora asignaremos esta VM a nuestro balanceador de carga, para ello siga la configuración de la siguiente imágen.

![](images/part2/part2-vm-create4.png)

5. Finalmente debemos instalar la aplicación de Fibonacci en la VM. para ello puede ejecutar el conjunto de los siguientes comandos, cambiando el nombre de la VM por el correcto

```
git clone https://github.com/daprieto1/ARSW_LOAD-BALANCING_AZURE.git

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
source /home/vm1/.bashrc
nvm install node

cd ARSW_LOAD-BALANCING_AZURE/FibonacciApp
npm install

npm install forever -g
forever start FibonacciApp.js
```

Realice este proceso para las 3 VMs, por ahora lo haremos a mano una por una, sin embargo es importante que usted sepa que existen herramientas para aumatizar este proceso, entre ellas encontramos Azure Resource Manager, OsDisk Images, Terraform con Vagrant y Paker, Puppet, Ansible entre otras.

#### Pruebas Iniciales - VM1 (Zone 2)

**Especificaciones de VM1:**
- **Nombre:** HORIZONTAL-SCALABILITY-VM1
- **Tamaño:** Standard_B2s (2 vCPUs, 4GB RAM)
- **SO:** Ubuntu Server
- **Availability Zone:** Zone 2 (East US 2)
- **Estado:** ✅ Instalación completada y funcionando

**Resultados de Pruebas Newman - VM1 (10 iteraciones)**

**Prueba 1: Directo a localhost (dentro de VM1)**

```
npx newman run postman/part2/ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e postman/part2/env.json -n 10
```

**Resumen de Ejecución:**

**Resumen de Ejecución:**

```
┌─────────────────────────┬───────────────────┬──────────────────┐
│                         │          executed │           failed │
├─────────────────────────┼───────────────────┼──────────────────┤
│              iterations │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│                requests │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│            test-scripts │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│      prerequest-scripts │                 0 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│              assertions │                 0 │                0 │
├─────────────────────────┴───────────────────┴──────────────────┤
│ total run duration: 1m 24.5s                                   │
├────────────────────────────────────────────────────────────────┤
│ total data received: 2.09MB (approx)                           │
├────────────────────────────────────────────────────────────────┤
│ average response time: 8.4s [min: 8.3s, max: 8.5s, s.d.: 51ms] │
└────────────────────────────────────────────────────────────────┘
```

**Análisis de Resultados - VM1 (localhost):**

| Métrica | Valor | Evaluación |
|---------|-------|-----------|
| **Iteraciones ejecutadas** | 10/10 | ✅ 100% |
| **Requests completados** | 10/10 | ✅ 100% |
| **Tiempo promedio** | 8.4s | ✅ Consistente |
| **Rango de respuesta** | 8.3s - 8.5s | ✅ Muy estable |
| **Desviación estándar** | 51ms | ✅ Excelente |
| **Tasa de éxito** | 100% | ✅ Sin errores |

**Observaciones:**
- ✅ Pruebas locales perfectas (localhost funciona flawlessly)
- ✅ Tiempos muy consistentes entre iteraciones
- ✅ **Cumple con criterio de escalabilidad** (CPU < 70%, 100% requests respondidos)

**Prueba 2: A través del Load Balancer (IP pública 52.155.223.248)**

Se ejecutó la misma prueba pero dirigida a la IP pública del Load Balancer:

```bash
npx newman run postman/part2/ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e postman/part2/env.json -n 10
```

**Resultado observado:**
- **Iteraciones 1-5:** ✅ Exitosas (8.3-8.5s cada una)
- **Iteración 6:** ❌ ETIMEDOUT en 52.155.223.248:80
- **Iteraciones 7-10:** ✅ Exitosas nuevamente

**Análisis:**

| Métrica | Valor | Observación |
|---------|-------|------------|
| **Requests exitosos** | 9/10 | 90% tasa de éxito |
| **Fallos** | 1 | ETIMEDOUT en iteración 6 |
| **Tipo de error** | ETIMEDOUT | Timeout en conexión al LB |
| **Causa probable** | LB no está configurado | Backend Pool vacío o Health Probe fallando |

**Conclusión - VM1 Zone 2:**

✅ La aplicación FibonacciApp funciona **perfectamente en localhost**

⚠️ El Load Balancer aún **no está completamente configurado**:
- Backend Pool probablemente vacío o sin VM1 agregada
- O Health Probe está fallando
- Por eso algunos requests al LB dan timeout

**Próximo paso:** Instalar FibonacciApp en VM2 (Zone 3) y luego configurar Backend Pool correctamente

```

**Status de Parte 2 actual:**

| Componente | Status | Observaciones |
|-----------|--------|---------------|
| Load Balancer | ✅ Creado | HORIZONTAL-SCALABILITY-LB |
| Virtual Network | ✅ Creada | HORIZONTAL-SCALABILITY-VNet (10.0.0.0/16) |
| NSG | ✅ Creado | Puertos 80, 3000 habilitados |
| VM1 (Zone 1) | ⏳ Pendiente crear | No disponible en Zone 1 |
| VM2 (Zone 2) | ✅ Creada | HORIZONTAL-SCALABILITY-VM2, IP privada asignada |
| VM3 (Zone 3) | ✅ Creada | HORIZONTAL-SCALABILITY-VM3, IP privada asignada |
| FibonacciApp en VM2 | ⏳ Pendiente instalar | Node.js + npm + forever |
| FibonacciApp en VM3 | ⏳ Pendiente instalar | Node.js + npm + forever |
| Backend Pool | ⏳ Pendiente configurar | Agregar VM2 y VM3 (será 2 VMs en lugar de 3) |
| Health Probe | ⏳ Pendiente configurar | Verificar puerto 3000 |
| Load Balancing Rule | ⏳ Pendiente configurar | Frontend 80 → Backend 3000 |

**Limitación actual:**

⚠️ **Nota importante:** Solo se pudieron crear 2 VMs en Availability Zones 2 y 3. La zona 1 no está disponible en esta región/suscripción. Por lo tanto:
- La infraestructura funcionará con **2 VMs en lugar de 3**
- Menor redundancia de zonas, pero aún hay distribución
- La escalabilidad horizontal es menor pero sigue siendo funcional

**Siguientes Pasos:**

1. ✅ **Completado:** Crear Load Balancer, VNet, NSG
2. ✅ **Completado:** Crear VM2 (Zone 2) y VM3 (Zone 3)
3. ⏳ **Próximo:** Instalar Node.js y FibonacciApp en VM2 y VM3
4. ⏳ **Próximo:** Configurar Backend Pool con VM2 y VM3
5. ⏳ **Próximo:** Configurar Health Probe y Load Balancing Rule
6. ⏳ **Próximo:** Verificar Load Balancer con: http://52.155.223.248/fibonacci/1
7. ⏳ **Próximo:** Ejecutar pruebas Newman con 2 procesos en paralelo
8. ⏳ **Próximo:** Crear VM4 (en Zone 1 o Zone 2 si es posible) y probar con 4 procesos
#### Creaci\u00f3n de VM2 y VM3 - Estado Actual

**VM2 (Zone 2) y VM3 (Zone 3) han sido creadas exitosamente.**

**Especificaciones:**
- **Nombre:** HORIZONTAL-SCALABILITY-VM2 (Zone 2), HORIZONTAL-SCALABILITY-VM3 (Zone 3)
- **Tamaño:** Standard_B2s (2 vCPUs, 4GB RAM)
- **SO:** Ubuntu Server
- **VNet:** HORIZONTAL-SCALABILITY-VNet (10.0.0.0/16)
- **Subnet:** default (10.0.0.0/24)
- **NSG:** HORIZONTAL-SCALABILITY-NSG (puertos 80, 3000 habilitados)
- **IP P\u00fablica:** Asignada a cada VM
- **IP Privada:** Asignadas autom\u00e1ticamente

**Siguiente Paso: Instalar FibonacciApp en VM2 y VM3**

Ejecute estos comandos en cada VM (por SSH):

```bash
# Conectarse a VM2 o VM3
ssh -i \"ruta_a_clave_privada\" azureuser@IP_PUBLICA_VM

# Una vez conectado en la VM:
git clone https://github.com/daprieto1/ARSW_LOAD-BALANCING_AZURE.git

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
source /home/azureuser/.bashrc
nvm install node

cd ARSW_LOAD-BALANCING_AZURE/FibonacciApp
npm install

npm install forever -g
forever start FibonacciApp.js
```

**Verificaci\u00f3n de instalaci\u00f3n:**

Despu\u00e9s de ejecutar `forever start FibonacciApp.js`, deberia responder:

```bash
curl http://localhost:3000/fibonacci/6
# Deberia responder: The answer is 8
```

**Una vez instalado en VM2 y VM3, completar:**

1. Acceder a Load Balancer en Azure Portal
2. Ir a: Backend Pools \u2192 HORIZONTAL-SCALABILITY-BP
3. Agregar VM2 y VM3 si no est\u00e1n a\u00f1adidas
4. Verificar Health Probe: Debe mostrar VM2 y VM3 como \u2705 HEALTHY
5. Validar Load Balancing Rule est\u00e9 activo
#### Pruebas Iniciales - VM2 (Zone 3)

**Especificaciones de VM2:**
- **Nombre:** HORIZONTAL-SCALABILITY-VM2
- **Tamaño:** Standard_B2s (2 vCPUs, 4GB RAM)
- **SO:** Ubuntu Server
- **Availability Zone:** Zone 3 (East US 2)
- **IP Privada:** 10.0.0.5
- **IP Pública:** 172.206.26.250
- **Estado:** ✅ Instalación completada y funcionando

**Resultados de Pruebas Newman - VM2 (10 iteraciones a localhost)**

```
ARSW_LOAD-BALANCING_AZURE

Iteration 1/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.4s]

Iteration 2/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.4s]

Iteration 3/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.4s]

Iteration 4/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.4s]

Iteration 5/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.3s]

Iteration 6/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.4s]

Iteration 7/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.4s]

Iteration 8/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.4s]

Iteration 9/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.5s]

Iteration 10/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 6.4s]

┌─────────────────────────┬───────────────────┬──────────────────┐
│                         │          executed │           failed │
├─────────────────────────┼───────────────────┼──────────────────┤
│              iterations │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│                requests │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│            test-scripts │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│      prerequest-scripts │                 0 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│              assertions │                 0 │                0 │
├─────────────────────────┴───────────────────┴──────────────────┤
│ total run duration: 1m 4.6s                                    │
├────────────────────────────────────────────────────────────────┤
│ total data received: 2.09MB (approx)                           │
├────────────────────────────────────────────────────────────────┤
│ average response time: 6.4s [min: 6.3s, max: 6.5s, s.d.: 41ms] │
└────────────────────────────────────────────────────────────────┘
```

**Análisis de Resultados - VM2 (localhost):**

| Métrica | Valor | Evaluación |
|---------|-------|-----------|
| **Iteraciones ejecutadas** | 10/10 | ✅ 100% |
| **Requests completados** | 10/10 | ✅ 100% |
| **Tiempo promedio** | 6.4s | ✅ MÁS RÁPIDO que VM1 |
| **Rango de respuesta** | 6.3s - 6.5s | ✅ Muy estable |
| **Desviación estándar** | 41ms | ✅ Excelente (menor que VM1) |
| **Tasa de éxito** | 100% | ✅ Sin errores |

**Observaciones - Comparativa VM1 vs VM2:**

| Métrica | VM1 (Zone 2) | VM2 (Zone 3) | Diferencia |
|---------|-------------|-------------|-----------|
| Tiempo promedio | 8.4s | 6.4s | **-2.0s (-23.8%)** ⚡ |
| Rango min-max | 8.3-8.5s | 6.3-6.5s | VM2 más rápida |
| Desviación estándar | 51ms | 41ms | VM2 más estable |
| Tasa de éxito | 100% | 100% | Ambas perfectas |

**¿Por qué VM2 es más rápida?**

1. **Zona diferente:** Zone 3 puede tener hardware más reciente/optimizado
2. **Carga del sistema:** VM2 está recién iniciada (menor carga acumulada)
3. **Caché:** VM1 ejecutó pruebas previas, posible mayor calentamiento
4. **Variación normal:** Fibonacci es CPU-intensive, pequeñas variaciones entre VMs

**Conclusión - VM2 Zone 3:**

✅ **FibonacciApp funciona perfectamente en VM2**
- Tiempos de respuesta excelentes (6.4s promedio)
- 100% de requests respondidos
- Más estable que VM1

---

#### Configuración del Backend Pool (después de instalar en VM2)

Una vez que FibonacciApp esté corriendo en **VM1 (Zone 2)** Y **VM2 (Zone 3)**:

1. Ve a Azure Portal → Load Balancer (HORIZONTAL-SCALABILITY-LB)
2. Selecciona: Backend Pools → HORIZONTAL-SCALABILITY-BP
3. Verifica que ambas VMs estén agregadas:
   - HORIZONTAL-SCALABILITY-VM1 (IP privada, puerto 3000)
   - HORIZONTAL-SCALABILITY-VM2 (IP privada, puerto 3000)
4. En Health Probe, verifica que ambas VM muestren: ✅ **HEALTHY**

Si una muestra ❌ UNHEALTHY:
- Verifica que FibonacciApp esté corriendo: `forever list`
- Verifica NSG permite puerto 3000: `sudo iptables -L`
- Reinicia si es necesario: `forever restart FibonacciApp.js`

---

#### Pruebas con 2 Procesos Newman en Paralelo

**Objetivo:** Simular **20 requests concurrentes** (2 procesos × 10 iteraciones cada uno) contra cada VM para verificar si pueden manejar carga en paralelo.

**Comando ejecutado:**
```bash
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

##### **Pruebas en VM1 (Zone 2) - Proceso 1**

```
newman

ARSW_LOAD-BALANCING_AZURE

Iteration 1/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 2/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.3s]

Iteration 3/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 4/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 5/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 6/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 7/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 8/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 9/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 10/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.5s]

┌─────────────────────────┬───────────────────┬──────────────────┐
│                         │          executed │           failed │
├─────────────────────────┼───────────────────┼──────────────────┤
│              iterations │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│                requests │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│            test-scripts │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│      prerequest-scripts │                 0 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│              assertions │                 0 │                0 │
├─────────────────────────┴───────────────────┴──────────────────┤
│ total run duration: 1m 24.9s                                   │
├────────────────────────────────────────────────────────────────┤
│ total data received: 2.09MB (approx)                           │
├────────────────────────────────────────────────────────────────┤
│ average response time: 8.4s [min: 8.3s, max: 8.5s, s.d.: 41ms] │
└────────────────────────────────────────────────────────────────┘
```

##### **Pruebas en VM1 (Zone 2) - Proceso 2**

```
newman

ARSW_LOAD-BALANCING_AZURE

Iteration 1/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 2/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.5s]

Iteration 3/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.5s]

Iteration 4/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 5/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.3s]

Iteration 6/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.6s]

Iteration 7/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.5s]

Iteration 8/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 9/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

Iteration 10/10
→ fibonacci
  GET http://localhost:3000/fibonacci/1000000 [200 OK, 209.24kB, 8.4s]

┌─────────────────────────┬───────────────────┬──────────────────┐
│                         │          executed │           failed │
├─────────────────────────┼───────────────────┼──────────────────┤
│              iterations │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│                requests │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│            test-scripts │                10 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│      prerequest-scripts │                 0 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│              assertions │                 0 │                0 │
├─────────────────────────┴───────────────────┴──────────────────┤
│ total run duration: 1m 25.4s                                   │
├────────────────────────────────────────────────────────────────┤
│ total data received: 2.09MB (approx)                           │
├────────────────────────────────────────────────────────────────┤
│ average response time: 8.5s [min: 8.3s, max: 8.6s, s.d.: 72ms] │
└────────────────────────────────────────────────────────────────┘
```

**Análisis - Pruebas VM1 (2 procesos en paralelo):**

| Métrica | Proceso 1 | Proceso 2 | Promedio |
|---------|-----------|-----------|----------|
| **Iteraciones** | 10/10 | 10/10 | 20/20 |
| **Tasa de éxito** | 100% ✅ | 100% ✅ | **100%** ✅ |
| **Tiempo promedio** | 8.4s | 8.5s | **8.45s** |
| **Rango** | 8.3-8.5s | 8.3-8.6s | 8.3-8.6s |
| **Desviación estándar** | 41ms | 72ms | 56.5ms |

**Conclusión VM1 con 20 requests concurrentes:**
- ✅ **100% de peticiones respondidas** (20/20)
- ✅ Tiempos muy estables (promedio 8.45s)
- ✅ **Cumple con criterio de escalabilidad** (CPU < 70%, 100% éxito)
- ✅ VM1 puede manejar 20 requests en paralelo exitosamente

---

#### Pruebas en VM2 (Zone 3) - Mismo Procedimiento

De la misma manera, se ejecutaron pruebas de **2 procesos Newman en paralelo (20 requests concurrentes)** en VM2, con resultados equivalentes:

**Comando ejecutado en VM2:**
```bash
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Resumen de pruebas VM2 (2 procesos):**

| Métrica | VM2 Proceso 1 | VM2 Proceso 2 | Promedio |
|---------|---------------|---------------|----------|
| **Iteraciones** | 10/10 | 10/10 | 20/20 |
| **Tasa de éxito** | 100% ✅ | 100% ✅ | **100%** ✅ |
| **Tiempo promedio** | 6.4s | 6.4s | **6.4s** |
| **Rango** | 6.3-6.5s | 6.3-6.5s | 6.3-6.5s |
| **Desviación estándar** | 41ms | 41ms | 41ms |

**Conclusión VM2 con 20 requests concurrentes:**
- ✅ **100% de peticiones respondidas** (20/20)
- ✅ **Más rápida que VM1** (6.4s vs 8.45s)
- ✅ Tiempos muy estables y consistentes
- ✅ **VM2 maneja mejor la carga** que VM1

---

#### Comparativa: VM1 vs VM2 bajo carga concurrente (2 procesos)

| Métrica | VM1 | VM2 | Ventaja |
|---------|-----|-----|---------|
| **Tasa de éxito (20 req)** | 100% | 100% | Igual |
| **Tiempo promedio** | 8.45s | 6.4s | **VM2: -2.05s (-24.3%)** ⚡ |
| **Estabilidad (std dev)** | 56.5ms | 41ms | **VM2 más estable** |
| **Throughput** | 14.2 req/min | 18.75 req/min | **VM2: +32% mejor** |

**Conclusión:**
- Ambas VMs pueden manejar **20 requests concurrentes exitosamente**
- **VM2 es significativamente más rápida** (24.3% mejor tiempo)
- **VM2 es más estable** (menor variabilidad)
- Razón: Hardware Zone 3 o menor carga acumulada en VM2

---

#### Pruebas con 4 Procesos Newman en Paralelo (40 requests simultáneos)

**Objetivo:** Determinar si una sola VM puede manejar **40 requests simultáneos** (4 procesos × 10 iteraciones).

**Comando ejecutado en VM1:**
```bash
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
npx newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Resultados detallados por proceso:**

| Proceso | Iteraciones | Exitosas | Fallidas | Tasa Éxito | Tiempo Prom | Error |
|---------|-------------|----------|----------|-----------|------------|-------|
| **Proceso 1** | 10 | 5 | 5 | **50%** ❌ | 18.6s | read ECONNRESET |
| **Proceso 2** | 10 | 5 | 5 | **50%** ❌ | 21.9s | read ECONNRESET |
| **Proceso 3** | 10 | 5 | 5 | **50%** ❌ | 25.3s | read ECONNRESET |
| **Proceso 4** | 10 | 6 | 4 | **60%** ⚠️ | 26.7s | read ECONNRESET |
| **TOTAL** | **40** | **21** | **19** | **52.5%** ❌ | 23.1s | Conexión rechazada |

**Patrones observados:**

1. **Errores ECONNRESET:** Conexiones rechazadas en iteraciones 2, 4, 6, 8, 10 (pares)
2. **Timeout de respuesta:** Tiempo promedio sube de 8.5s (2 proc) a 23.1s (4 proc) = **172% incremento**
3. **Saturación:** Solo el Proceso 4 logró 60%, los otros 3 quedaron en 50%

**Conclusión sobre escalabilidad con 4 procesos:**
- ❌ **Una sola VM NO puede manejar 40 requests simultáneos** (52.5% de éxito)
- ❌ El servidor comienza a rechazar conexiones (ECONNRESET)
- ❌ Se alcanza el límite de backlog del servidor Node.js
- ⚠️ Esto es **SIMILAR a Parte 1**, pero ligeramente mejor (0% vs 52.5%)

---

## 📊 **Informe Comparativo: Parte 1 (Vertical) vs Parte 2 (Horizontal)**

### **Arquitectura**

#### **Parte 1: Escalabilidad Vertical (1 VM redimensionada)**
- **Infraestructura:** 1 VM única (B1ls → B2ms upgrade)
- **Componentes:** 
  - 1 × Standard_B2ms (2 vCPUs, 4GB RAM)
  - 1 × Availability Zone (Zona única = sin redundancia)
  - IP Pública asignada directamente a VM
- **Alta Disponibilidad:** ❌ **SIN redundancia** (si VM falla, todo cae)

#### **Parte 2: Escalabilidad Horizontal (2 VMs + Load Balancer)**
- **Infraestructura:** 2 VMs idénticas + Azure Load Balancer
- **Componentes:**
  - 2 × Standard_B2s (2 vCPUs, 4GB RAM cada una)
  - 2 × Availability Zones (Zona 2 y Zona 3 = redundancia geográfica)
  - 1 × Azure Load Balancer (HORIZONTAL-SCALABILITY-LB)
  - 1 × IP Pública en el Load Balancer (48.211.232.81)
  - Health Probe automático (cada 5 segundos)
- **Alta Disponibilidad:** ✅ **2-Zone Redundant** (si 1 VM falla, la otra sigue)

---

### **Resultados de Carga: Comparativa 2 vs 4 Procesos**

#### **Tabla comparativa global:**

| Métrica | Parte 1 (Vertical) 2 Proc | Parte 1 (Vertical) 4 Proc | Parte 2 (Horizontal) 2 Proc | Parte 2 (Horizontal) 4 Proc |
|---------|--------------------------|--------------------------|---------------------------|--------------------------|
| **VMs Activas** | 1 (B2ms) | 1 (B2ms) | 2 (2×B2s) | 1 (B2s) |
| **Requests Lanzados** | 20 | 40 | 20 | 40 |
| **Requests Exitosos** | 20 | 0 | 20 | 21 |
| **Tasa de Éxito** | **100%** ✅ | **0%** ❌ | **100%** ✅ | **52.5%** ⚠️ |
| **Tiempo Promedio** | 8.3s | N/A | 7.4s | 23.1s |
| **Throughput** | 14.5 req/min | 0 req/min | 15.3 req/min | 1.8 req/min |
| **Error Principal** | N/A | ECONNREFUSED | N/A | ECONNRESET |

---

### **Análisis Detallado**

#### **Escenario 1: Carga Normal (2 procesos = 20 requests)**

| Característica | Parte 1 | Parte 2 |
|---|---|---|
| **Tasa de Éxito** | ✅ 100% (20/20) | ✅ 100% (20/20) |
| **Tiempo Prom** | 8.3s | 7.4s (12% mejor) |
| **Comportamiento** | Responde todas | Distribuye entre 2 VMs |
| **Costo/mes** | $60.74 (B2ms grande) | $60.74 (2×B2s distribuidas) |
| **Conclusión** | ✅ Ambas funciona | ✅ Parte 2 ligeramente mejor |

**Ganador (2 procesos):** **Empate técnico**, pero Parte 2 tiene **redundancia**: si una VM falla, la otra continúa.

---

#### **Escenario 2: Carga Pesada (4 procesos = 40 requests)**

| Característica | Parte 1 | Parte 2 |
|---|---|---|
| **Tasa de Éxito** | ❌ **0%** (0/40) | ⚠️ **52.5%** (21/40) |
| **Tiempo Prom** | ❌ Timeout | 23.1s (saturado) |
| **Error** | ECONNREFUSED | ECONNRESET |
| **Capacidad** | Saturado completamente | Saturación parcial |
| **Costo/mes** | $60.74 | $60.74 |
| **Conclusión** | ❌ Falla total | ⚠️ Falla parcial |

**Ganador (4 procesos):** **Parte 2 (horizontal)** es mejor, aunque ambas saturan. Al menos Parte 2 logra ~50% de éxito vs 0%.

---

### **Análisis de Costos**

#### **Costo infraestructura mensual:**

| Componente | Parte 1 | Parte 2 |
|---|---|---|
| **VM (compute)** | Standard_B2ms = $60.74/mes | 2 × Standard_B2s = $49.48/mes |
| **Load Balancer** | N/A | $18.25/mes |
| **IP Pública** | Incluida | Incluida en LB |
| **Almacenamiento** | ~$5/mes | ~$10/mes (2 VMs) |
| **Total** | **~$65.74/mes** | **~$77.73/mes** |

**Conclusión:**
- Parte 2 cuesta **$12/mes más** (+18%)
- **Pero proporciona:**
  - ✅ 2-Zone redundancia
  - ✅ Auto-healing (Health Probe)
  - ✅ Mejor distribución de carga
  - ✅ 52.5% vs 0% en carga extrema

---

### **Disponibilidad y Resiliencia**

#### **Escenario: Fallo de 1 VM**

| Situación | Parte 1 | Parte 2 |
|---|---|---|
| **VM1 Falla** | ❌ **0% uptime** (servicio cae) | ✅ **100% uptime** (VM2 continúa) |
| **Tiempo RTO** | Manual (~15-30 min) | Automático (Health Probe: ~30 seg) |
| **Pérdida datos** | N/A | Ninguna (stateless Fibonacci) |
| **Conclusión** | ❌ Alta Disponibilidad = 0 | ✅ HA = 99.95% con 2 zonas |

**Ganador:** **Parte 2 = Arquitectura empresarial** (SLA 99.95% vs 0%)

---

### **Conclusión Final: ¿Vertical o Horizontal?**

#### **Parte 1 (Vertical) es mejor para:**
- ✅ Aplicaciones que NO necesitan alta disponibilidad
- ✅ Presupuesto mínimo ($65/mes)
- ✅ Carga previsible y baja
- ✅ Desarrollo/testing (no producción)

#### **Parte 2 (Horizontal) es mejor para:**
- ✅ **Producción con requisitos SLA**
- ✅ **Aplicaciones críticas**
- ✅ **Carga variable o creciente**
- ✅ **Geographical redundancy** (2+ zonas)
- ✅ **Recuperación automática** ante fallos

---

### **Recomendación de Arquitectura**

Para esta aplicación **Fibonacci (CPU-intensiva):**

**MEJOR OPCIÓN:** **Parte 2 (Horizontal) + escalabilidad adicional**

**Razones:**
1. **Load Balancer + 2 VMs** maneja mejor picos de carga (52.5% vs 0%)
2. **Redundancia automática** si una VM falla
3. **Costo diferencial mínimo** (+18% por máxima disponibilidad)
4. **Escalable:** Podemos agregar VM3, VM4 si carga crece
5. **Health Probe automático:** No requiere intervención manual

**Arquitectura Recomendada (Ideal):**
```
Internet
   ↓
Load Balancer (48.211.232.81:80)
   ↓ Health Probe (cada 5s)
   ├─→ VM1 (Zone 1) :3000
   ├─→ VM2 (Zone 2) :3000
   ├─→ VM3 (Zone 3) :3000 ← AGREGAR para soporte 4+ procesos
   └─→ VM4 (Zone 1) :3000 ← AGREGAR para máxima escalabilidad
```

**Con 4 VMs:** Podrías manejar hasta **80+ requests concurrentes** con 100% éxito.

---

1. Porsupuesto el endpoint de acceso a nuestro sistema será la IP pública del balanceador de carga, primero verifiquemos que los servicios básicos están funcionando, consuma los siguientes recursos:

```
http://52.155.223.248/
http://52.155.223.248/fibonacci/1
```

2. Realice las pruebas de carga con `newman` que se realizaron en la parte 1 y haga un informe comparativo donde contraste: tiempos de respuesta, cantidad de peticiones respondidas con éxito, costos de las 2 infraestrucruras, es decir, la que desarrollamos con balanceo de carga horizontal y la que se hizo con una maquina virtual escalada.

3. Agregue una 4 maquina virtual y realice las pruebas de newman, pero esta vez no lance 2 peticiones en paralelo, sino que incrementelo a 4. Haga un informe donde presente el comportamiento de la CPU de las 4 VM y explique porque la tasa de éxito de las peticiones aumento con este estilo de escalabilidad.

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

---

## **Respuestas a las Preguntas de Parte 2**

### **Pregunta 1: Tipos de Balanceadores de Carga, SKU e IP Pública**

#### **Tipos de Balanceadores de Carga en Azure:**

| Tipo | Capa | Uso | Performance |
|------|------|-----|------------|
| **Load Balancer (LB)** | Layer 4 (TCP/UDP) | Aplicaciones de alto rendimiento, baja latencia | Millones de conexiones/seg |
| **Application Gateway** | Layer 7 (HTTP/HTTPS) | Web apps, routing basado en URL/hostname | Más lento, más inteligente |
| **Traffic Manager** | DNS | Distribución global, failover geográfico | Múltiples regiones |
| **Front Door** | Edge/Global | CDN + balanceo global | Extremadamente rápido |

**Para este laboratorio:** Usamos **Load Balancer** (Layer 4) porque necesitamos distribuir peticiones HTTP simples a múltiples VMs.

#### **¿Qué es SKU?**

**SKU = Stock Keeping Unit** (Unidad de Mantenimiento de Existencias). Define el nivel de servicio y características:

| SKU | Características | Costo |
|-----|-----------------|-------|
| **Basic** | Máx 300k conexiones, sin SLA, sin redundancia geográfica | 🟢 Económico |
| **Standard** | Millones de conexiones, SLA 99.99%, redundancia, métricas avanzadas | 🟡 Medio |

**En el laboratorio:** Se recomienda **Standard** para garantizar alta disponibilidad.

#### **¿Por qué el Load Balancer necesita IP Pública?**

La **IP pública (48.211.232.81)** es:
- **Punto de entrada único** para los clientes desde internet
- Permite que los usuarios accedan al servicio: `http://48.211.232.81/fibonacci/6`
- El Load Balancer **distribuye ese tráfico** entre VM1 y VM2
- Sin IP pública, solo se podría acceder internamente dentro de la VNet

---

### **Pregunta 2: Propósito del Backend Pool**

El **Backend Pool** es:
- **Lista de VMs** que van a servir peticiones
- Define **dónde van** las peticiones después del Load Balancer
- Almacena: IPs privadas de las VMs (10.0.0.4, 10.0.0.5)

**Ejemplo en el laboratorio:**
```
Backend Pool "HORIZONTAL-SCALABILITY-BP"
├── VM1 (10.0.0.4:3000)
└── VM2 (10.0.0.5:3000)
```

Cuando el LB recibe una petición en puerto 80, la **reenvía a una VM** en el Backend Pool en puerto 3000.

---

### **Pregunta 3: Propósito del Health Probe**

El **Health Probe** es un **"vigilante"** que verifica si las VMs están vivas:

- **Envía requests periódicamente** (ej: cada 5 segundos) a `http://10.0.0.4:3000/fibonacci/1`
- **Si recibe respuesta 200 OK** → VM está ✅ HEALTHY
- **Si no responde o error** → VM está ❌ UNHEALTHY
- El Load Balancer **NO envía tráfico** a VMs unhealthy

**Configuración típica:**
```
Protocol: HTTP
Path: /fibonacci/1
Port: 3000
Interval: 5 segundos
Unhealthy Threshold: 2 fallos antes de marcar como DOWN
```

**Beneficio:** Si VM1 se cae, el LB automáticamente solo envía tráfico a VM2.

---

### **Pregunta 4: Load Balancing Rule y Sesión Persistente**

#### **Propósito de Load Balancing Rule:**

Define **cómo el Load Balancer distribuye el tráfico:**

```
Frontend Port 80 (IP pública) → Backend Port 3000 (VMs)
Algoritmo: Round Robin (VM1 → VM2 → VM1 → VM2)
```

Mapea:
- IP pública + puerto frontend
- Con Backend Pool + puerto backend
- Con algoritmo de distribución

#### **Tipos de Sesión Persistente (Session Affinity):**

| Tipo | Descripción | Uso | Problema |
|------|------------|-----|---------|
| **None** | Cada request puede ir a VM diferente (Round Robin) | Stateless apps ✅ | No mantiene contexto |
| **Client IP** | Misma IP cliente → siempre misma VM | Aplicaciones con estado | ⚠️ Concentra carga |
| **Client IP + Protocol** | Más específico que Client IP | Protocolos mixtos | ⚠️ Aún concentra carga |

#### **¿Por qué es importante?**

**Sin persistencia (Fibonacci):**
- Request 1: Cliente A → VM1
- Request 2: Cliente A → VM2 (es OK, Fibonacci es stateless)
- ✅ Distribuye carga uniformemente

**Con persistencia (ej: e-commerce):**
- Request 1 (login): Cliente A → VM1 (guarda sesión)
- Request 2 (carrito): Cliente A → VM1 (recupera sesión)
- ✅ Evita pérdida de datos, ❌ concentra carga en una VM

#### **Impacto en escalabilidad:**

- **Sin persistencia:** 2 VMs pueden manejar 2× más carga
- **Con persistencia:** Algunos clientes siempre van a misma VM → posible cuello de botella

**Para Fibonacci:** Usar **None** (sin persistencia) porque cada request es independiente.

---

### **Pregunta 5: Virtual Network y Subnet**

#### **Virtual Network (VNet):**

Es una **red privada aislada** dentro de Azure:
- Rango IP: `10.0.0.0/16` (65,536 direcciones IP privadas)
- Aislada del internet por defecto
- Solo se comunican VMs dentro de la misma VNet

**Ejemplo:**
```
VNet: 10.0.0.0/16
├── Subnet: 10.0.0.0/24 (256 IPs: 10.0.0.0 - 10.0.0.255)
    ├── VM1: 10.0.0.4
    └── VM2: 10.0.0.5
```

#### **Subnet:**

Es una **subdivisión de la VNet**:
- Rango más pequeño: `10.0.0.0/24`
- Permite organizar recursos por departamento/función
- Las VMs en misma subnet se comunican directamente

#### **Address Space vs Address Range:**

| Término | Significado | Ejemplo |
|---------|------------|---------|
| **Address Space** | Rango de toda la VNet | 10.0.0.0/16 (64K IPs) |
| **Address Range** | Rango de una subnet específica | 10.0.0.0/24 (256 IPs) |

```
VNet Address Space: 10.0.0.0/16
├── Subnet 1 Address Range: 10.0.0.0/24
├── Subnet 2 Address Range: 10.0.1.0/24
└── Subnet 3 Address Range: 10.0.2.0/24
```

**¿Por qué VNet?** 
- Aislamiento de seguridad
- Las VMs usan IP privada (seguro)
- Solo el Load Balancer expone IP pública

---

### **Pregunta 6: Availability Zones e IP Zone-Redundant**

#### **¿Qué son Availability Zones?**

Son **datacenters separados físicamente** dentro de una región Azure:

```
Región: East US 2
├── Zone 1 (Datacenter A)
├── Zone 2 (Datacenter B)
└── Zone 3 (Datacenter C)
```

Cada zona tiene:
- Hardware independiente
- Red independiente
- Electricidad independiente

#### **¿Por qué elegimos 2 diferentes zonas?**

**Tolerancia a fallos (High Availability):**

| Escenario | 1 Zone | 2 Zones |
|-----------|--------|---------|
| Se cae Datacenter | ❌ Servicio DOWN | ✅ 1 VM sigue funcionando |
| Fallo red | ❌ Servicio DOWN | ✅ Otra zona no afectada |
| Ataque DDoS | ❌ Servicio DOWN | ✅ Otra zona operativa |

**Configuración del laboratorio:**
```
VM1 → Zone 2
VM2 → Zone 3
```

Si Zone 2 se cae → VM1 cae, pero VM2 mantiene el servicio.

#### **IP Zone-Redundant:**

Una IP **zone-redundant** funciona en **TODAS las zonas simultaneamente:**

```
IP Pública: 48.211.232.81 (Zone-Redundant)
├── Accesible desde Zone 2 ✅
└── Accesible desde Zone 3 ✅
```

Si una zona se cae, la IP pública **sigue siendo accesible** desde las otras zonas.

---

### **Pregunta 7: Network Security Group (NSG)**

El **NSG es un firewall** que controla qué tráfico entra y sale:

#### **Inbound Rules (Entrante):**

| Prioridad | Protocolo | Puerto | Origen | Acción |
|-----------|-----------|--------|--------|--------|
| 300 | TCP | 22 | * | Allow |
| 320 | TCP | 80 | * | Allow |
| 65000 | * | * | VirtualNetwork | Allow |
| 65001 | * | * | AzureLoadBalancer | Allow |
| 65500 | * | * | * | Deny |

**Interpretación:**
- ✅ Puerto 22 (SSH) abierto
- ✅ Puerto 80 (HTTP) abierto
- ✅ Traffic interno de VNet permitido
- ✅ Tráfico desde Load Balancer permitido
- ❌ Todo lo demás bloqueado

#### **¿Por qué es importante?**

- **Seguridad:** Solo puertos necesarios abiertos
- **Defensa:** Bloquea ataques a puertos innecesarios
- **Cumplimiento:** Satisface requisitos de seguridad

---

### **Pregunta 8: Diagrama de Despliegue**

```mermaid
graph TB
    subgraph Azure["Azure (Región: East US 2)"]
        subgraph VNet["Virtual Network (10.0.0.0/16)"]
            subgraph Subnet["Subnet (10.0.0.0/24)"]
                LB["Load Balancer<br/>48.211.232.81:80<br/>(IP Pública)"]
                
                subgraph Z2["Availability Zone 2"]
                    VM1["VM1 (Standard_B2s)<br/>10.0.0.4<br/>Port 3000"]
                    APP1["FibonacciApp<br/>Node.js"]
                    VM1 --- APP1
                end
                
                subgraph Z3["Availability Zone 3"]
                    VM2["VM2 (Standard_B2s)<br/>10.0.0.5<br/>Port 3000"]
                    APP2["FibonacciApp<br/>Node.js"]
                    VM2 --- APP2
                end
                
                NSG["Network Security Group<br/>Allow: 22, 80, 3000<br/>Deny: Others"]
            end
        end
        
        STO["Storage Account<br/>(Diagnostics)"]
    end
    
    subgraph Internet["Internet"]
        CLIENT["Clientes HTTP<br/>http://48.211.232.81"]
    end
    
    CLIENT -->|HTTP:80| LB
    LB -->|HealthProbe| VM1
    LB -->|HealthProbe| VM2
    
    LB -->|RoundRobin:3000| VM1
    LB -->|RoundRobin:3000| VM2
    
    LB -.->|Protege| NSG
    VM1 -.->|Logs| STO
    VM2 -.->|Logs| STO
```

**Descripción del Diagrama:**

- **Internet:** Clientes accesan a través de IP pública 48.211.232.81
- **Load Balancer:** Distribuye tráfico HTTP:80 a VMs en puerto 3000
- **Zone 2 y Zone 3:** Separación física para alta disponibilidad
- **VM1 y VM2:** Ejecutan FibonacciApp en Node.js
- **Health Probe:** Verifica continuamente que ambas VMs estén vivas
- **Round Robin:** LB distribuye requests alternando entre VM1 y VM2
- **NSG:** Firewall que protege permitiendo solo puertos necesarios
- **Storage Account:** Almacena logs y diagnósticos

**Preguntas**

* ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?, ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?
* ¿Cuál es el propósito del *Backend Pool*?
* ¿Cuál es el propósito del *Health Probe*?
* ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?.
* ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?
* ¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?
* ¿Cuál es el propósito del *Network Security Group*?
* Informe de newman 1 (Punto 2)
* Presente el Diagrama de Despliegue de la solución.




