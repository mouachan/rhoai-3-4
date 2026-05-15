# RHOAI 3.4 Installation & Configuration Guide

Complete setup of Red Hat OpenShift AI 3.4 on a fresh OpenShift cluster with all features enabled.

## Cluster Information

| Property | Value |
|----------|-------|
| Cluster | sandbox5465 |
| Region | eu-central-1 |
| OCP Version | 4.20 |
| RHOAI Version | 3.4.0 |
| Ingress Domain | `apps.cluster-ksc6j.ksc6j.sandbox5465.opentlc.com` |

---

## 1. Operators Installed

Install the following operators from OperatorHub before configuring RHOAI:

| Operator | Version | Namespace | Purpose |
|----------|---------|-----------|---------|
| Red Hat OpenShift AI | 3.4.0 | redhat-ods-operator | Core RHOAI platform |
| Red Hat Connectivity Link | 1.3.3 | openshift-operators | Kuadrant + Authorino + Limitador for MaaS auth |
| Tempo Operator | 0.20.0-3 | openshift-operators | Distributed tracing backend |
| Red Hat build of OpenTelemetry | 0.144.0-3 | openshift-operators | Telemetry data collection |
| Cluster Observability Operator | - | openshift-cluster-observability-operator | Prometheus, Alertmanager, Perses |

---

## 2. DataScienceCluster (DSC) Configuration

### 2.1 TrustyAI / LMEval

Enable LMEval with code execution and online evaluation for guardrails:

```bash
oc patch datasciencecluster default-dsc --type='merge' -p '{
  "spec": {
    "components": {
      "trustyai": {
        "managementState": "Managed",
        "mcpGuardrailsMode": true,
        "eval": {
          "lmeval": {
            "permitCodeExecution": "allow",
            "permitOnline": "allow"
          }
        }
      }
    }
  }
}'
```

### 2.2 KServe / Models-as-a-Service

Already configured by default in RHOAI 3.4:

```yaml
kserve:
  managementState: Managed
  modelsAsService:
    managementState: Managed
  nim:
    airGapped: false
    managementState: Managed
  rawDeploymentServiceConfig: Headless
  wva:
    managementState: Removed
```

### 2.3 Model Registry

```yaml
modelregistry:
  managementState: Managed
  registriesNamespace: rhoai-model-registries
```

### 2.4 Kueue

> **Note:** `Managed` is no longer supported as a managementState for Kueue in RHOAI 3.4.
> Kueue is now managed externally via the Red Hat build of Kueue Operator.
> Set to `Unmanaged` or leave as `Removed` and control via the dashboard flag.

---

## 3. OdhDashboardConfig

Enable all RHOAI 3.4 features in the dashboard:

```bash
oc patch odhdashboardconfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type merge \
  -p '{
    "spec": {
      "dashboardConfig": {
        "disableKueue": false,
        "disableLLMd": false,
        "disableLMEval": false,
        "disableTracking": false,
        "genAiStudio": true,
        "maasAuthPolicies": true,
        "modelAsService": true,
        "observabilityDashboard": true,
        "trainingJobs": true
      }
    }
  }'
```

> **Deprecated fields (rejected by 3.4 API):** `disableFineTuning`, `mlflow`

---

## 4. JobSetOperator (Required for Training Jobs)

The DSC Trainer component requires a JobSetOperator CR to be present:

```bash
oc apply -f - <<'EOF'
apiVersion: operator.openshift.io/v1
kind: JobSetOperator
metadata:
  name: cluster
spec:
  managementState: Managed
EOF
```

If the DSC still reports "JobSetOperator not found" after creation, restart the RHOAI operator:

```bash
oc delete pod -n redhat-ods-operator -l name=rhods-operator
```

---

## 5. User Workload Monitoring

Enable Prometheus monitoring for user-defined projects:

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

---

## 6. Models-as-a-Service (MaaS) Infrastructure

### 6.1 Kuadrant CR

```bash
oc apply -f - <<'EOF'
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
  namespace: kuadrant-system
spec: {}
EOF
```

> If Kuadrant reports "Gateway API provider not found" but Istio is running in `openshift-ingress` (not `istio-system`), restart the kuadrant-operator pod in `openshift-operators`.

### 6.2 Authorino TLS Configuration

```bash
# Annotate Authorino service for service-ca TLS cert
oc annotate service authorino-authorino-authorization -n kuadrant-system \
  service.beta.openshift.io/serving-cert-secret-name=authorino-server-cert --overwrite

# Patch Authorino with TLS cert ref
oc patch authorino authorino -n kuadrant-system --type=merge --patch '{
  "spec": {
    "listener": {
      "tls": {
        "enabled": true,
        "certSecretRef": {
          "name": "authorino-server-cert"
        }
      }
    }
  }
}'

# Set CA bundle env vars on Authorino deployment
oc -n kuadrant-system set env deployment/authorino \
  SSL_CERT_FILE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt
```

### 6.3 MaaS Gateway

```bash
oc apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
  annotations:
    opendatahub.io/managed: "false"
    security.opendatahub.io/authorino-tls-bootstrap: "true"
spec:
  gatewayClassName: data-science-gateway-class
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      certificateRefs:
      - group: ""
        kind: Secret
        name: maas-default-gateway-tls
      mode: Terminate
    allowedRoutes:
      namespaces:
        from: All
EOF
```

Annotate the gateway for Authorino TLS bootstrap:

```bash
oc annotate gateway maas-default-gateway -n openshift-ingress \
  security.opendatahub.io/authorino-tls-bootstrap="true" --overwrite
```

### 6.4 MaaS PostgreSQL

Deploy a PostgreSQL instance in `redhat-ods-applications` shared by MaaS, MLflow, and Model Registry:

```bash
# PVC
oc apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maas-postgres-pvc
  namespace: redhat-ods-applications
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
EOF

# Deployment
oc apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: maas-postgres
  namespace: redhat-ods-applications
spec:
  replicas: 1
  selector:
    matchLabels:
      app: maas-postgres
  template:
    metadata:
      labels:
        app: maas-postgres
    spec:
      containers:
      - name: postgresql
        image: registry.redhat.io/rhel9/postgresql-16:latest
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRESQL_USER
          value: maas
        - name: POSTGRESQL_PASSWORD
          value: "<REPLACE_WITH_PASSWORD>"
        - name: POSTGRESQL_DATABASE
          value: maas_db
        volumeMounts:
        - name: data
          mountPath: /var/lib/pgsql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: maas-postgres-pvc
EOF

# Service
oc apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: maas-postgres
  namespace: redhat-ods-applications
spec:
  selector:
    app: maas-postgres
  ports:
  - port: 5432
    targetPort: 5432
EOF
```

### 6.5 MaaS DB Config Secret

The DSC requires a secret with a `DB_CONNECTION_URL` key (single connection URL, not individual keys):

```bash
oc create secret generic maas-db-config \
  -n redhat-ods-applications \
  --from-literal=DB_CONNECTION_URL="postgresql://maas:<PASSWORD>@maas-postgres.redhat-ods-applications.svc.cluster.local:5432/maas_db"
```

> **Common mistake:** Using individual keys (host, port, database, etc.) instead of the single `DB_CONNECTION_URL` key will cause the DSC to report: `missing required key 'DB_CONNECTION_URL'`.

---

## 7. MLflow

### 7.1 Create MLflow Database

Reuse the MaaS PostgreSQL instance:

```bash
oc exec -n redhat-ods-applications deployment/maas-postgres -- \
  psql -U postgres -c "CREATE DATABASE mlflow_db OWNER maas;"
```

### 7.2 MLflow DB Credentials Secret

```bash
oc create secret generic mlflow-db-credentials \
  -n redhat-ods-applications \
  --from-literal=BACKEND_STORE_URI="postgresql://maas:<PASSWORD>@maas-postgres.redhat-ods-applications.svc.cluster.local:5432/mlflow_db?sslmode=disable"
```

> **Important:** Add `?sslmode=disable` to the connection URL. Without it, the db-migration init container fails with: `server does not support SSL, but SSL was required`.

### 7.3 MLflow CR

```bash
oc apply -f - <<'EOF'
apiVersion: mlflow.opendatahub.io/v1
kind: MLflow
metadata:
  name: mlflow
  namespace: redhat-ods-applications
spec:
  backendStoreUriFrom:
    name: mlflow-db-credentials
    key: BACKEND_STORE_URI
  serveArtifacts: true
  artifactsDestination: "file:///mlflow/artifacts"
  storage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  replicas: 1
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "1Gi"
      cpu: "500m"
EOF
```

> **Note on `backendStoreUriFrom` format:** Use flat structure with `name` and `key` at the same level. A nested `secretKeyRef` structure is rejected by the API.

---

## 8. Observability Stack (Technology Preview)

### 8.1 Prerequisites

Three operators must be installed (see Section 1):
- Cluster Observability Operator
- Tempo Operator
- Red Hat build of OpenTelemetry

### 8.2 Enable in DSCInitialization

```bash
oc patch dsci default-dsci --type='merge' -p '{
  "spec": {
    "monitoring": {
      "managementState": "Managed",
      "namespace": "redhat-ods-monitoring",
      "alerting": {},
      "metrics": {
        "replicas": 1,
        "storage": {
          "size": "5Gi",
          "retention": "90d"
        },
        "exporters": {}
      },
      "traces": {
        "sampleRatio": "0.1",
        "storage": {
          "backend": "pv",
          "retention": "2160h"
        },
        "exporters": {}
      }
    }
  }
}'
```

### 8.3 Verification

After applying, the following pods should appear in `redhat-ods-monitoring`:

```
alertmanager-data-science-monitoringstack-0          2/2   Running
alertmanager-data-science-monitoringstack-1          2/2   Running
data-science-collector-collector-0                   1/1   Running
data-science-collector-collector-1                   1/1   Running
data-science-collector-targetallocator-*             1/1   Running
data-science-perses-0                                1/1   Running
data-science-prometheus-cluster-proxy-*              1/1   Running
data-science-prometheus-namespace-proxy-*            2/2   Running
prometheus-data-science-monitoringstack-0            3/3   Running
tempo-data-science-tempomonolithic-0                 3/3   Running
thanos-querier-data-science-thanos-querier-*         1/1   Running
```

### 8.4 Enable Dashboard Menu

Already done via `observabilityDashboard: true` in OdhDashboardConfig (Section 3).

> **Do NOT manually create a Perses CR.** The observability stack deploys its own Perses instance (`data-science-perses-0`) automatically when the DSCI monitoring is properly configured.

---

## 9. Model Registry

### 9.1 Component Enablement

Already enabled in DSC (Section 2.3). Verify:

```bash
oc get namespace rhoai-model-registries
oc get pods -n redhat-ods-applications | grep model-registry-operator
```

### 9.2 Create a Database for the Registry

Reuse the MaaS PostgreSQL:

```bash
oc exec -n redhat-ods-applications deployment/maas-postgres -- \
  psql -U postgres -c 'CREATE DATABASE "model-registry" OWNER maas;'
```

> **Note:** The dashboard uses the database name as entered in the form. If you type `model-registry` (with hyphen), the database must be created with that exact name (quoted in PostgreSQL because of the hyphen).

### 9.3 Create from Dashboard

Create the model registry from the RHOAI dashboard UI:

1. Go to **Settings -> Model resources and operations -> Model registry settings**
2. Click **Create model registry**
3. Fill in:
   - **Name:** `model-registry`
   - **Database type:** PostgreSQL
   - **Host:** `maas-postgres.redhat-ods-applications.svc.cluster.local`
   - **Port:** `5432`
   - **Username:** `maas`
   - **Password:** `<PASSWORD>`
   - **Database:** `model-registry`
4. Do NOT check "Add CA certificate"
5. Click **Create**

> **Gotcha:** Make sure there is no leading/trailing space in the password field. A space causes `net/url: invalid userinfo` in the PostgreSQL DSN.

### 9.4 Configure Permissions

After creation, click **Manage permissions** and add:
- **Groups:** `system:authenticated` (grants access to all authenticated users)

### 9.5 Verification

```bash
oc get modelregistries.modelregistry.opendatahub.io -n rhoai-model-registries
# NAME             AVAILABLE   AGE
# model-registry   True        5m
```

---

## 10. Summary of Databases

All databases share the same PostgreSQL instance (`maas-postgres` in `redhat-ods-applications`):

| Database | Owner | Used By |
|----------|-------|---------|
| `maas_db` | maas | MaaS (Models-as-a-Service) |
| `mlflow_db` | maas | MLflow experiment tracking |
| `model-registry` | maas | Model Registry |

---

## 11. DSC Final Status

After all configurations, the DSC should show all components ready:

```bash
oc get datasciencecluster default-dsc -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

Key conditions to verify:
- `ModelsAsServiceReady: True`
- `KServeReady: True`
- `TrustyAIReady: True`
- `TrainerReady: True`
- `ModelRegistryReady: True`

---

## 12. Troubleshooting

### DSC stuck on NotReady
- **MaaS:** Check that `maas-default-gateway`, `maas-db-config` secret, and Kuadrant CR all exist
- **Trainer:** Check that `JobSetOperator` CR exists (`oc get jobsetoperator cluster`)
- **After creating missing resources:** Restart RHOAI operator pods: `oc delete pod -n redhat-ods-operator -l name=rhods-operator`

### Observability dashboard "Service Unavailable"
- Do NOT create a Perses CR manually
- Configure the full monitoring spec in DSCI (Section 8.2)
- Ensure all 3 prerequisite operators are installed

### MLflow db-migration SSL error
- Add `?sslmode=disable` to the PostgreSQL connection URL in the secret

### MaaS "MissingDependency" on Kuadrant
- If Istio runs in `openshift-ingress` (not `istio-system`), restart the kuadrant-operator pod

### Model Registry "Unavailable"
- Check pod logs: `oc logs -n rhoai-model-registries -l app=model-registry -c rest-container`
- Common issues: database does not exist, password with special chars or spaces in DSN
