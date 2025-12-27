# EXAMEN-SUSTITUTORIO-CC3S2-SECCION-B
Examen Sustitutorio (Variante A)

## 1: Infraestructura como Código Declarativa con Políticas Automatizadas

### Por qué es Obligatoria

#### 1. Seguridad Física + Menor Privilegio

Acceso físico directo permite modificaciones imperativas (`kubectl edit`) sin trazabilidad. Un operador escalando control-service de 1 a 3 réplicas genera instancias paralelas tomando decisiones contradictorias sobre válvulas físicas, produciendo oscilación destructiva.

**Mitigación**: Todos los cambios transitan por archivos `.tf` versionados en Git. OPA/Kyverno validan restricciones (ej: `replicas == 1`). `terraform plan` exhibe cambios exactos. State file mantiene registro inmutable para auditoría.

#### 2. Desviación e Idempotencia

**Drift**: Estado real diverge del código. Modificar timeout de BD manualmente funciona temporalmente, pero `terraform apply` posterior lo resetea causando fallos.

**Idempotencia**: Aplicar configuración N veces produce resultado idéntico. Sin esto, `iptables -A` ejecutado 3 veces crea 3 reglas duplicadas.

#### 3. Reconstrucción Forense

Post-incidente (pH fuera de rango 47 min), Git y Terraform responden: ¿Qué NetworkPolicies existían? ¿Qué versión corría? ¿Quién tenía permisos RBAC? Queries simples:

```bash
git log --since="2025-12-26 14:00" --until="2025-12-26 15:00" -- network-policies/
terraform state pull | jq '.resources[] | select(.name=="control_service_deployment")'
```

### Escenario de Fallo: Infraestructura Imperativa

Operador A crea NetworkPolicy `default-deny-all`. Operador B la elimina por fallo de red. Operador A re-ejecuta asumiendo que existe. Resultado: estado inconsistente, reporting-service accede a control-service, inyección de comandos permite abrir válvulas simultáneamente.

**Prevención declarativa**: State lock serializa operaciones. OPA rechaza reglas permisivas violando políticas. `terraform plan` muestra diferencias forzando coordinación.

### Relaciones Clave

Drift + Menor Privilegio: NetworkPolicy desviando a "allow-all" otorga privilegios no autorizados. Políticas automatizadas minimizan blast radius: "control-service en nodo dedicado" asegura aislamiento físico si sensor-service falla.


## 2: Detección y Remediación de Desviaciones

### Situación

Control-service escalado manualmente, NetworkPolicy modificada, parámetros de BD diferentes en producción vs código.

### Estrategia

#### 1. Detección Automatizada

**Terraform drift**: Cron job ejecuta `terraform plan -detailed-exitcode` cada 5 min. Exit code 2 indica drift, guardando diff en `/var/log/drift-$(date +\%s).log`.

**Admission Controller**: ConstraintTemplate valida recursos antes de creación:

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequirelabels
spec:
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirelabels
        violation[{"msg": msg}] {
          not input.review.object.metadata.labels["managed-by"]
          msg := "Recursos deben tener label managed-by=terraform"
        }
```

**BD monitoring**: Script compara configuración esperada vs actual, genera `/var/log/db-drift.json` si detecta diferencias:

```python
expected_config = {"statement_timeout": "5000", "max_connections": "50"}
actual_config = fetch_from_db()
drift = {k: {"expected": expected_config[k], "actual": actual_config[k]}
         for k in expected_config if expected_config[k] != actual_config.get(k)}
```

**Evidencias**: Logs JSON con timestamp y diff, eventos en AlertManager, audit log de Kubernetes.

#### 2. Bloqueo Preventivo

**RBAC restrictivo**: Solo permisos `get, list, watch` para operadores.

**ValidatingAdmissionPolicy**: Bloquea recursos no gestionados por Terraform usando CEL:

```yaml
validations:
  - expression: "object.metadata.labels['managed-by'] == 'terraform'"
    message: "Solo recursos gestionados por Terraform"
```

**BD inmutable**: `REVOKE ALTER SYSTEM FROM operator_role;`

#### 3. Remediación Segura

**Deployment**: Verificar recursos actuales, aplicar con `-target=kubernetes_deployment.control_service`, esperar readiness.

**NetworkPolicy**: Backup actual, `terraform plan -out=remediate.tfplan`, aplicar, verificar conectividad.

**BD**: Aplicar con `-target`, cambiar primero parámetros sin reinicio, acumular los que sí para ventana de mantenimiento.

**Evidencias**: JSON con timestamp, tipo de drift, valores esperado/actual, blast radius, estado de remediación. Almacenado en Prometheus, Loki y tabla PostgreSQL inmutable.


## 3: Estándar de Repositorios IaC

### Estructura

```
environments/
  prod/    # main.tf, variables.tf, terraform.tfvars, backend.tf
  staging/
modules/       # k8s-deployment, network-policy, database
policies/      # *.rego validations
tests/         # unit, integration
.github/workflows/  # plan.yml, apply.yml
```

**Justificación**: Separa prod/staging previniendo errores. Módulos reutilizables versionados. Policies centralizadas. Tests aseguran cambios no destructivos.

### Reglas de PR

Branch `feature/TICKET-ID-descripcion` desde main. Validaciones: `terraform fmt -check`, `validate`, OPA. Output de `terraform plan` como comentario. 2 aprobaciones: operador + SRE. Merge si "0 recursos destruidos". Apply automático solo en staging, manual en prod.

**CODEOWNERS**: `/environments/prod/` → @sre-team + @plant-manager, `/modules/database/` → @dba-team + @sre-team, `/policies/` → @security-team.

### Versionado de Módulos

Semantic Versioning: MAJOR (rompe compatibilidad), MINOR (features compatibles), PATCH (bug fixes). Pin explícito:

```hcl
source = "git::https://github.com/org/modules.git//k8s-deployment?ref=v2.1.0"
```

Nunca `ref=main`, siempre tag para reproducibilidad.

### Naming

**Recursos K8s**: `{entorno}-{servicio}-{tipo}-{criticidad}` → `prod-control-deployment-critical`

**Variables Terraform**: `{scope}_{resource}_{attribute}` → `control_service_replicas`

Semántica operativa: `critical` en nombre indica aprobación adicional y ventana de mantenimiento.

### Gestión de Secretos

Vault como source of truth. Terraform obtiene via data sources:

```hcl
data "vault_generic_secret" "db_credentials" {
  path = "secret/agua-inteligente/${var.environment}/database"
}
```

Segregación por rol: `vault/secret/agua-inteligente/{environment}/database/{role}` (admin, app, readonly). Policies granulares de acceso. Nunca en código, outputs con `sensitive = true`.

### Anti-Patrones

**Module Vendoring**: Copiar código de módulo externo al repo causa drift de mantenimiento (patches no se aplican), aumenta blast radius (cambios temporales se propagan), imposibilita rollback selectivo.

**Solución**: Referencias versionadas con hash verification.


## 4: Arquitectura de Módulos con Patrones

### Módulo 1: Infraestructura Base - Immutable Infrastructure

Cluster K8s con node pools por criticidad. Pool critical: n2-standard-4, 100GB, 3 nodos fijos, taints `critical:NoSchedule`. Pool standard: n2-standard-2, 50GB, 2-5 nodos auto-scaling.

**Patrón**: Cada cambio crea nueva versión completa vs modificar en caliente.

**Seguridad**: Nodos desde imágenes versionadas. Parchear vulnerabilidad = nuevo node pool + drenaje del viejo. Evita drift por parches manuales.

**Resiliencia**: Rollback completo a versión anterior sin estado intermedio. Separación física workloads críticos/no-críticos.

**Costo cognitivo**: Alto inicial (versionado + estrategia reemplazo), bajo a largo plazo (elimina snowflake servers).

### Módulo 2: Observabilidad - Builder

Stack componente por componente:

- `with_metrics`: Prometheus, scrape 15s, retention 90d, métricas custom (valve_position, ph_level)
- `with_logs`: Loki, buffer 10MB, level INFO, enmascara db_password/api_key
- `with_alerts`: HighErrorRate (threshold 0.05 en 5min), ServiceDown (threshold 1 en 1min)
- `with_tracing`: OpenTelemetry opcional, sample rate configurable

**Patrón**: Builder permite configuración incremental por servicio.

**Seguridad**: Valida métricas no expongan datos sensibles. Permisos mínimos por componente habilitado.

**Resiliencia**: Degradación granular. Si Loki cae, desactiva `with_logs` manteniendo `with_metrics` y `with_alerts`. Resource limits proporcionales a criticality.

**Costo cognitivo**: Bajo para consumidores (interfaz fluida). Costo en mantener builder se paga una vez.

### Módulo 3: Control Operacional - Facade

Interfaz simple para operaciones complejas:

```hcl
control_plane {
  enable_emergency_stop = true
  emergency_contacts = ["ops@planta.com"]
  maintenance_windows = [{day="Sunday", hour="02:00", duration="4h"}]
  blast_radius_limits = {max_concurrent_service_updates = 1}
}
```

Internamente crea ConfigMaps, Jobs emergencia, políticas RBAC.

**Patrón**: Facade oculta complejidad K8s detrás de interfaz unificada.

**Seguridad**: Abstrae RBAC complexity. Valida `emergency_contacts` sean emails @planta.com.

**Resiliencia**: Asegura `max_concurrent_service_updates = 1` coordinando updates via lock.

**Costo cognitivo**: Bajo para operadores (conceptos alto nivel). Específico para control industrial pero reutilizable entre plantas.


## 5: Resiliencia - DIP, Circuit Breaker, Adapter

### Problema

Control-service depende de sensor-service para lecturas. Si sensor-service falla, control-service debe seguir operando con última información conocida.

### DIP (Dependency Inversion Principle)

Interfaz `SensorDataPort` usando ABC:

```python
class SensorDataPort(ABC):
    @abstractmethod
    def get_current_readings(self) -> Optional[SensorReading]: pass

    @abstractmethod
    def is_healthy(self) -> bool: pass
```

Implementaciones: `RemoteSensorSource` (HTTP), `CachedSensorSource` (memoria), `MockSensorSource` (tests).

**Beneficio**: Control-service depende de abstracción, no implementación concreta. Hot-swap de implementación via feature flag sin redespliegue.

### Circuit Breaker

Estados: CLOSED (normal), OPEN (5 fallos, rechaza llamadas), HALF_OPEN (después 60s, intenta reconectar).

```python
CircuitBreaker(failure_threshold=5, timeout=60)
```

`RemoteSensorSource` integra wrapeando fetch en `circuit_breaker.call()`. Si falla, retorna None. `is_healthy()` verifica circuito no OPEN.

**Beneficio**: Evita bombardear servicio caído. Detección rápida de fallo permite recuperación.

### Adapter

`SensorServiceAdapter` implementa `SensorDataPort` adaptando API REST compleja:

```python
def get_current_readings(self):
    try:
        data = sensor_client.fetch()
        self.last_valid_reading = data
        return data
    except:
        if self.last_valid_reading.age < 300:  # 5 min
            return self.last_valid_reading
        return SensorReading(ph=7.0, turbidity=5.0, flow=100.0)  # defaults seguros
```

**Beneficio**: Control-service siempre obtiene `SensorReading`, nunca None/excepción. Usa cache o defaults seguros manteniendo operación.

### Contratos e Invariantes

**Contrato**: `get_current_readings()` retorna SensorReading|None, NO lanza excepciones, completa <2s. `is_healthy()` retorna True si lecturas confiables.

**Invariantes**: Si `is_healthy()==False`, datos son cache/defaults. `SensorReading.ph` entre 0-14, `turbidity >= 0`. Circuit OPEN transiciona a HALF_OPEN tras exactamente `timeout` segundos.

### Métricas

```python
sensor_requests_total = Counter('sensor_requests_total', ['status'])
circuit_breaker_state = Gauge('circuit_breaker_state')  # 0=CLOSED, 1=HALF_OPEN, 2=OPEN
sensor_reading_age_seconds = Gauge('sensor_reading_age_seconds')
```

**Flujo fallo**: Sensor-service cae → 5 requests fallan → Circuit abre → Adapter retorna cache → control-service continúa → 60s después reintenta → vuelve CLOSED si éxito.


## 6: Estrategia de Pruebas

### NO Automatizables

Hardware físico requiere observación visual y desgasta mecanismos. Emergencias extremas (incendio) requieren coordinación física. Calibración de sensores es proceso manual con soluciones buffer.

### En Cada Commit

**Validación estática**: `terraform fmt -check`, `validate`, `tflint`

**Validaciones OPA**: Carga política Rego, ejecuta `policy.evaluate()` sobre terraform plan, verifica `allowed` y `replicas == 1`.

**Contract tests**: Mock server sensor-service, verifica status 200, campos ph/turbidity, pH entre 0-14.

**Smoke tests**: Terraform crea infraestructura en namespace temporal K8s, verifica pods arrancan y healthchecks pasan, destruye en <3 min.

### Manejo de Flakiness

BD tarda inicializar: `@retry(max_attempts=3, backoff=2)`. NetworkPolicy tarda aplicar: `wait_until(check_blocked, timeout=60, interval=5)`. Prometheus sin scrape: fixture con warmup.

### Rollback y Caos

**Rollback**: `terraform state pull > backup.json`. Si falla, `state push backup.json` + re-apply. Test: capturar estado inicial, aplicar config mala, rollback, verificar estado final == inicial. Rollback granular con `-target`.

**Caos controlado**: Inyecta latencia 5000ms en sensor-service por 120s. Verifica circuit breaker abre (estado 2), control-service healthy, usa cache (<300s old). Limpia inyección, espera 70s, verifica circuit cierra.

**Pipeline**: Cada commit: static validation → OPA → contracts → smoke tests. Solo en main: chaos tests (5-10 min).


## 7: Despliegue con Hardening

### Docker Hardening

Multi-stage build: `python:3.11-slim AS builder` instala deps, copia solo necesario a final. Usuario no-root `appuser` uid 1000 sin shell `/bin/false`. Sin cache apt. Read-only filesystem en K8s. HEALTHCHECK nativo intervalo 30s.

**Resultado**: Imagen 1GB → 200MB. Pipeline ejecuta `trivy image --severity HIGH,CRITICAL --exit-code 1` fallando si detecta vulnerabilidades.

### Kubernetes Hardening

**Pod Security Standards**: Namespace con `pod-security.kubernetes.io/enforce: restricted`

**Security context Pod**: `runAsNonRoot: true`, `runAsUser: 1000`, `fsGroup: 1000`, `seccompProfile: RuntimeDefault`

**Security context Container**: `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities: drop ALL`. EmptyDir para /tmp y /app/cache. Resources requests/limits. NodeSelector `workload: critical` + tolerations.

**NetworkPolicy**: Default-deny-all primero (`podSelector: {}`, sin reglas). Whitelist: ingress solo de api-gateway:8080, egress solo a sensor-service:8080 y postgres:5432.

**RBAC**: ServiceAccount `control-service-sa`. Role permite solo `get` en configmap específico "control-config". RoleBinding conecta SA a Role.

### CI/CD Local-First

**Pipeline GitLab**: Stages: validate, build, test, deploy-staging, deploy-prod. Variable `FREEZE_WINDOWS = "Mon-Fri 06:00-08:00,Mon-Fri 18:00-20:00"`. Deploy-prod requiere manual trigger, verifica freeze window, ejecuta plan/apply.

**Local-first**: Runner GitLab + Registry Docker en servidor local. Backup de imágenes en disco.

### Estrategia: RollingUpdate con Pre-warming

**NO Canary**: 2 versiones simultáneas emitirían comandos contradictorios a válvulas. Canary requiere service mesh agregando latencia 5-20ms inaceptable (<100ms requerido).

**RollingUpdate**: `maxSurge: 1`, `maxUnavailable: 0`, `minReadySeconds: 30`. Pod nuevo hace pre-warming (carga modelo ML, conecta BD), espera readiness, recibe tráfico, pod viejo termina. Downtime: 0 segundos.

### Alertas

**Critical (SMS + llamada + sirena)**:
- `ValveCommandConflict`: rate cambios >10/min → modo seguro
- `SensorReadingStale`: lecturas >5min → verificación hardware
- `PHOutOfSafeRange`: pH fuera 6.5-8.5 → dosificación emergencia

**Warning (Slack + email)**:
- `FlowRateZero`: caudal 0 >5min → inspección bombas
- `NodeTemperatureHigh`: temp >70°C → verificación ventilación
- `DiskUsageDatabase`: disco >85% → limpieza logs

Cada alerta vinculada a runbook con pasos exactos.
