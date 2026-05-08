# Detalles de la caída de AWS

## Contexto

El **lunes 20 de octubre de 2025**, la región **N. Virginia (us-east-1)** de **AWS** sufrió una interrupción mayor que degradó o dejó inoperativos a cientos de servicios y aplicaciones en todo el mundo.  
El origen fue un **problema de resolución DNS** en **Amazon DynamoDB**, que provocó una **cadena de fallas internas** afectando componentes críticos como **EC2** y los **health checks de Network Load Balancer (NLB)**.

AWS publicó un **Post-Event Summary (PES)** técnico y una **cronología completa** en su Health Dashboard con los hitos del incidente y las acciones de recuperación.

---

## ¿Qué pasó?

### La automatización de DNS en DynamoDB (Planner / Enactors)

Para operar a gran escala, **DynamoDB** utiliza muchos **balanceadores de carga (LB)** y mantiene **cientos de miles de registros DNS por región** (endpoint público, FIPS, IPv6, variantes por cuenta, etc.).  
Esa “foto” DNS no se gestiona manualmente: la mantiene un **sistema automatizado** con dos componentes principales:

- **DNS Planner (planificador):**  
  Supervisa la capacidad y salud de los balanceadores y genera *planes DNS* (listas de balanceadores con pesos) por endpoint.  
  Usualmente existe un **plan regional único** que se replica entre endpoints (p. ej., IPv6 y público).

- **DNS Enactor (aplicador):**  
  Aplica los planes generados en **Amazon Route 53**.  
  Para garantizar resiliencia, existen **instancias independientes** del Enactor en **cada Zona de Disponibilidad (AZ)**.  
  Cada una busca nuevos planes y ejecuta **transacciones atómicas en Route 53**, asegurando consistencia incluso si varios Enactors actualizan en paralelo.

#### ¿Dónde falló?

Existía un **defecto latente** (race condition) en el proceso:

1. Un **Enactor A** quedó muy lento aplicando un **plan antiguo** (reintentos por contención).  
2. Mientras tanto, el **Planner** siguió generando nuevos planes, y otro **Enactor B** aplicó uno reciente sin problemas.  
3. El **Enactor B**, al finalizar, ejecutó una **limpieza de planes antiguos**.  
4. Justo en ese momento, el **Enactor A** —aún con un plan viejo en caché— **sobrescribió** el endpoint regional con ese plan obsoleto.  
5. La limpieza posterior de B eliminó ese plan “por demasiado antiguo”, dejando el **endpoint sin IPs**.  

Resultado:  
El endpoint regional de DynamoDB quedó **vacío y bloqueado**, sin posibilidad de actualización automática. Fue necesaria **intervención manual** para restaurar los registros DNS.

---

## Cronología técnica (PDT)

- **11:48 PM (Oct 19):** comienzan errores de API en DynamoDB (us-east-1) por fallas de resolución DNS (`dynamodb.us-east-1.amazonaws.com`).  
- **12:38 AM:** AWS identifica el problema en el DNS.  
- **1:15 AM:** se aplican mitigaciones iniciales para restaurar herramientas internas.  
- **2:25 AM:** se restaura el DNS; los clientes recuperan conectividad conforme **expiran cachés locales** (~2:40 AM).  
  - Las **Global Tables** completan sincronización a las **2:32 AM**.  
- **2:25–10:36 AM:**  
  - **EC2** continúa afectado: fallan los lanzamientos de nuevas instancias debido a fallas en los subsistemas **DWFM** (gestiona leases de hosts físicos) y **Network Manager** (propaga estado de red).  
  - **DWFM** entra en **colapso por congestión**, se aplican **throttles** y **reinicios selectivos (~4:14 AM)**.  
  - **5:28 AM:** DWFM restablece leases.  
  - **10:36 AM:** se normaliza la propagación de red.  
  - **1:50 PM:** la API de EC2 vuelve a operar con normalidad.  
- **5:30 AM–2:09 PM:**  
  - **NLB** sufre errores de conexión por health checks en nodos cuya red aún no estaba lista → *flapping* y *failovers* DNS por AZ.  
  - **9:36 AM:** AWS deshabilita temporalmente el failover automático → caída de errores.  
  - **2:09 PM:** se reactiva el failover.  
- **3:01 PM:** AWS declara recuperación completa; algunas colas (Config, Redshift, Connect) tardan más en vaciarse.

---

## Cadena causal

1. **Bug de carrera** en la automatización DNS de DynamoDB → publicación de un **registro vacío** y bloqueo del sistema.  
2. **Servicios internos dependientes** (EC2/DWFM, catálogos, colas, leases) fallan por **timeouts** al no resolver el endpoint.  
3. **Restauración del DNS** genera una **avalancha de reintentos** → **colapso por congestión** en DWFM; sin leases válidos, EC2 no puede lanzar nuevas instancias.  
4. **Network Manager** acumula retrasos en propagación de red → nuevas instancias sin conectividad.  
5. **NLB** interpreta erróneamente la salud por datos atrasados → *flapping* de health checks y *failovers* erróneos → **pérdida adicional de capacidad**.

---

## Impacto en servicios

- **DynamoDB (principal):**  
  Fallas de conexión y API. Las **Global Tables** continuaron en otras regiones, pero con **lag** hacia/desde us-east-1.  

- **EC2:**  
  Errores al lanzar nuevas instancias (las activas no se afectaron).  
  Respuestas `InsufficientCapacity` o `RequestLimitExceeded` por throttling y pérdida de leases.  
  Conectividad degradada hasta normalizar propagación de red.  

- **NLB:**  
  Errores de conexión por health checks con información de red desactualizada.  
  Mitigación temporal: **desactivar el failover automático**.  

- **Lambda:**  
  Fallas en creación y actualización de funciones, **ralentización en SQS/Kinesis**, y errores en invocaciones asíncronas.  
  AWS aplicó **throttles temporales** y drenó backlogs durante la recuperación.  

- **ECS/EKS/Fargate:**  
  Fallas en el lanzamiento y escalado de contenedores.  

- **Amazon Connect:**  
  Problemas de voz/chat y ruteo (busy, dead air) durante el pico del incidente.  

- **Impacto externo (“medio Internet”):**  
  Grandes aplicaciones globales (Amazon.com, Alexa, Snapchat, Fortnite, bancos, etc.) reportaron caídas o degradación significativa.

---

## Por qué fue tan severo

1. **Dependencias internas en DynamoDB:**  
   Muchos sistemas de control (p. ej., DWFM, colas internas, catálogos) usan DynamoDB.  
   Si el endpoint regional no responde, **se detiene el control-plane completo**, aunque los data planes sigan funcionando.

2. **Cachés DNS en clientes y resolvers:**  
   Aun tras reparar Route 53, la **recuperación depende de TTLs** → procesos con respuestas viejas tardan en reconectarse.

3. **Backlogs masivos y colapso por congestión:**  
   Cuando todo se restablece, los servicios reintentan al mismo tiempo.  
   Sin control de concurrencia, el sistema **no progresa** sin throttling o reinicios.

4. **Health checks muy sensibles:**  
   **NLB** evaluó nodos nuevos con redes aún no propagadas → *flapping* y *failovers* que retiraron capacidad sana.  
   AWS tuvo que **desactivar temporalmente el failover automático**.

5. **Concentración en us-east-1:**  
   Muchas cargas críticas y servicios globales (p. ej., partes de **IAM**) dependen de esa región.  
   El impacto se propagó más allá de sus fronteras físicas.

---

## Qué hacer sabiendo que esto puede volver a pasar

### 1. DNS y endpoints

- **Evita dependencias de control-plane en una sola región.**  
  Si us-east-1 aloja endpoints críticos (IAM, DDB Global Tables primarias), distribuye la carga entre regiones activas-activas o configura failover automático.  

- **Cuida los TTLs de DNS y la caché.**  
  Ajusta tiempos de vida y usa resolvers locales robustos. Implementa reintentos con **backoff exponencial + jitter** para evitar tormentas de peticiones.

- **Aprovecha los VPC Endpoints.**  
  Cuando sea posible, usa endpoints privados para reducir dependencia de Internet y DNS externos.

---

### 2. EC2 y lanzamientos

- **Evita depender de lanzamientos “en caliente” durante crisis.**  
  Usa **warm pools**, instancias en *standby* o *pre-scaled capacity* para picos esperados.

- **Configura Auto Scaling multi-AZ y “Any AZ”.**  
  Permite que EC2 elija la zona disponible (fue parte de la mitigación oficial de AWS).

- **Prueba tus estrategias multi-región.**  
  Ensaya conmutaciones manuales o automáticas para bases de datos y control planes.

---

### 3. Red y balanceo

- **Revisa la configuración de NLB/ALB:**  
  - Activa **cross-zone load balancing** y ten **over-provisioning** por AZ.  
  - Ajusta **thresholds y timeouts** de health checks para tolerar estados transitorios.  
  - Evita que el DNS retire capacidad por señales temporales: usa health checks multi-signal o canarios.

---

### 4. Serverless y colas

- Define **back-pressure** explícita para Lambda + SQS/Kinesis:  
  controla `concurrency`, `batchSize`, `maxReceiveCount`, `DLQ`, y ten un **plan de ramp-up gradual** tras incidentes.

---

### 5. Procedimientos de recuperación

Ten **runbooks** actualizados para:

- **Limpiar cachés DNS** (aplicaciones, proxies, gateways) cuando AWS indique problemas de resolución.  
- **Desactivar funciones sensibles** (por ejemplo, auto scaling agresivo) durante recuperaciones parciales.  
- **Reiniciar o vaciar colas internas** si detectas síntomas de *congestive collapse* (como en DWFM).

---

## Conclusión

La caída del 20 de octubre de 2025 en **us-east-1** demostró cómo un fallo puntual en un componente aparentemente aislado —el sistema automatizado de DNS de DynamoDB— puede escalar hasta afectar una parte sustancial de Internet.  
Comprender las **dependencias internas de AWS** y diseñar **arquitecturas multi-región tolerantes a fallas de control-plane** es esencial para reducir el impacto de futuros eventos de esta magnitud.

---

---

# Details of the AWS Outage

## Context

On **Monday, October 20, 2025**, the **N. Virginia (us-east-1)** region of **AWS** experienced a major outage that degraded or rendered hundreds of services and applications around the world unavailable.
The root cause was a **DNS resolution problem** in **Amazon DynamoDB**, which triggered a **chain of internal failures** affecting critical components such as **EC2** and **Network Load Balancer (NLB)** health checks.

AWS later published a **technical Post-Event Summary (PES)** and a **full timeline** on its Health Dashboard describing the event’s key moments and recovery actions.

---

## What Happened?

### DynamoDB’s DNS Automation (Planner / Enactors)

To operate at massive scale, **DynamoDB** uses multiple **load balancers (LBs)** and maintains **hundreds of thousands of DNS records per region** (public endpoints, FIPS, IPv6, per-account variants, etc.).
That DNS “snapshot” isn’t manually maintained — it’s managed by an **automated system** composed of two main components:

* **DNS Planner:**
  Monitors the capacity and health of load balancers and generates *DNS plans* (lists of load balancers with weights) per endpoint.
  Usually, there is a **single regional plan** replicated across related endpoints (for example, IPv6 and public).

* **DNS Enactor:**
  Applies those plans in **Amazon Route 53**.
  To ensure resilience, there are **independent Enactor instances** in **each Availability Zone (AZ)**.
  Each Enactor fetches new plans and performs **atomic transactions in Route 53**, ensuring consistency even when multiple Enactors update in parallel.

#### Where It Failed

A **latent defect** (a race condition) existed in this process:

1. One **Enactor A** became very slow while applying an **old plan** (due to contention and retries).
2. Meanwhile, the **Planner** continued producing new plans, and another **Enactor B** successfully applied a newer one.
3. After completing its work, **Enactor B** performed **cleanup of outdated plans**.
4. At that same moment, **Enactor A** —still holding the old plan in memory— **overwrote** the regional endpoint with that obsolete plan.
5. B’s cleanup process then removed that plan as “too old,” leaving the **endpoint with no IP addresses**.

**Result:**
The regional DynamoDB endpoint became **empty and locked**, unable to auto-update. **Manual intervention** was required to restore the DNS records.

---

## Technical Timeline (PDT)

* **11:48 PM (Oct 19):** API errors begin in DynamoDB (us-east-1) due to DNS resolution failures (`dynamodb.us-east-1.amazonaws.com`).
* **12:38 AM:** AWS identifies the DNS issue.
* **1:15 AM:** Initial mitigations applied to restore internal tooling.
* **2:25 AM:** DNS restored; clients regain connectivity as **local caches expire** (~2:40 AM).

  * **Global Tables** finish synchronization by **2:32 AM**.
* **2:25–10:36 AM:**

  * **EC2** remains impacted — new instance launches fail due to internal subsystem issues in **DWFM** (host lease management) and **Network Manager** (network state propagation).
  * **DWFM** enters **congestive collapse**; AWS applies **throttling** and **selective restarts (~4:14 AM)**.
  * **5:28 AM:** DWFM restores leases.
  * **10:36 AM:** Network propagation normalizes.
  * **1:50 PM:** EC2 APIs fully operational again.
* **5:30 AM–2:09 PM:**

  * **NLB** suffers connection errors due to health checks running on nodes whose networks weren’t fully ready → *flapping* and DNS *failovers* per AZ.
  * **9:36 AM:** AWS disables automatic failover → error rates drop.
  * **2:09 PM:** Failover re-enabled.
* **3:01 PM:** AWS declares full recovery; some queues (Config, Redshift, Connect) drain over the next few hours.

---

## Causal Chain

1. **Race-condition bug** in DynamoDB’s DNS automation → a **blank DNS record** is published, locking the update system.
2. **Dependent internal services** (EC2/DWFM, catalogs, queues, leases) time out because they can’t resolve the endpoint.
3. **DNS restoration** triggers a **massive retry storm** → **congestive collapse** within DWFM; without valid leases, EC2 cannot launch new instances.
4. **Network Manager** delays network-state propagation → new instances come up **without connectivity**.
5. **NLB** misinterprets health using outdated data → *flapping* health checks and erroneous *failovers* → **additional capacity loss**.

---

## Impact on Services

* **DynamoDB (primary impact):**
  API and connectivity failures. **Global Tables** continued serving other regions but with replication **lag** to and from us-east-1.

* **EC2:**
  Failures launching new instances (running ones unaffected).
  Returned `InsufficientCapacity` or `RequestLimitExceeded` due to throttling and lost leases.
  Degraded connectivity until network propagation stabilized.

* **NLB:**
  Connection errors from health checks relying on outdated network information.
  Temporary mitigation: **disable automatic failover**.

* **Lambda:**
  Errors creating/updating functions, **slower SQS/Kinesis polling**, and **asynchronous invocation issues**.
  AWS applied **temporary throttles** and drained backlogs during recovery.

* **ECS/EKS/Fargate:**
  Failures in container launch and cluster scaling.

* **Amazon Connect:**
  Voice/chat routing issues (busy tones, dead air) during the peak impact.

* **External impact (“half of the Internet”):**
  Major global apps (Amazon.com, Alexa, Snapchat, Fortnite, banking platforms, etc.) reported outages or severe degradation.

---

## Why It Was So Severe

1. **Internal dependencies on DynamoDB:**
   Many control-plane systems (DWFM, orchestration queues, catalogs) use DynamoDB.
   When the regional endpoint fails, the **entire control plane halts**, even though data planes remain healthy.

2. **DNS caching on clients/resolvers:**
   Even after Route 53 was fixed, recovery relied on **TTL expiration**, creating a **non-uniform, delayed** return to service.

3. **Massive backlogs and congestive collapse:**
   Once DNS returned, all systems retried simultaneously.
   Without concurrency control, the system **failed to progress** until throttling and restarts were applied.

4. **Over-sensitive health checks:**
   **NLB** evaluated new nodes before their networks were fully propagated → *flapping* and false *failovers* that removed healthy capacity.
   AWS had to **temporarily disable automatic failover** to stabilize.

5. **Regional concentration in us-east-1:**
   Many critical workloads and “global” services (e.g., parts of **IAM**) depend on this region.
   The effect **spilled beyond** its physical boundaries.

---

## What to Do Knowing This Could Happen Again

### 1. DNS and Endpoints

* **Avoid single-region control-plane dependencies.**
  If us-east-1 hosts critical endpoints (IAM, DynamoDB Global Tables primaries), distribute the load across active-active regions or set up automated failover.

* **Manage DNS TTLs and caches carefully.**
  Tune TTLs and use robust local resolvers. Implement retries with **exponential backoff + jitter** to prevent retry storms.

* **Leverage VPC Endpoints.**
  When possible, use private endpoints to reduce reliance on public Internet DNS.

---

### 2. EC2 and Instance Launches

* **Don’t rely on “hot” launches during crises.**
  Use **warm pools**, *standby* instances, or *pre-scaled capacity* for predictable peaks.

* **Enable multi-AZ and “Any AZ” Auto Scaling.**
  Let EC2 choose available zones (part of AWS’s own mitigation strategy).

* **Test multi-region strategies.**
  Regularly simulate manual or automated failovers for databases and control planes.

---

### 3. Networking and Load Balancing

* **Review NLB/ALB configuration:**

  * Enable **cross-zone load balancing** and maintain **over-provisioning** per AZ.
  * Adjust **health-check thresholds and timeouts** to tolerate transient states.
  * Prevent DNS from dropping capacity based on temporary signals — use **multi-signal or canary health checks**.

---

### 4. Serverless and Queues

* Define **explicit back-pressure** for Lambda + SQS/Kinesis:
  control `concurrency`, `batchSize`, `maxReceiveCount`, and `DLQ`; plan a **gradual ramp-up** after incidents to drain backlogs safely.

---

### 5. Recovery Procedures

Maintain **up-to-date runbooks** for:

* **Flushing DNS caches** (apps, proxies, gateways) when AWS reports resolution issues.
* **Disabling sensitive features** (e.g., aggressive auto scaling) during partial recoveries.
* **Restarting or purging internal queues** when detecting *congestive collapse* symptoms (like DWFM experienced).

---

## Conclusion

The October 20, 2025 outage in **us-east-1** demonstrated how a seemingly isolated flaw —the automated DNS subsystem in DynamoDB— can cascade into a large-scale Internet disruption.
Understanding **AWS’s internal dependencies** and designing **multi-region, control-plane-resilient architectures** are essential steps to reduce the blast radius of future events of this magnitude.

---
