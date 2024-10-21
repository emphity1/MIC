
# Guida all'Installazione per l'Ambiente di Sviluppo OpenShift Local

Questa guida fornisce istruzioni dettagliate per configurare un ambiente di sviluppo locale su OpenShift con i seguenti componenti:

- [Opensearch](#opensearch)
- [RabbitMQ](#rabbitmq)
- [PostgreSQL](#postgresql)
- [Keycloak](#keycloak)

Tutti i componenti saranno installati nel namespace `test-requirements`.

---

## Prerequisiti

- **OpenShift Local** installato e in esecuzione.
- **Helm** installato.
- **kubectl** o **oc** installato.
- **Git** installato (opzionale, se si clona da repository).

---

## Indice

- [Creazione del Namespace](#creazione-del-namespace)
- [Opensearch](#opensearch)
  - [Passi per l'Installazione](#passi-per-linstallazione)
- [RabbitMQ](#rabbitmq)
  - [Passi per l'Installazione](#passi-per-linstallazione-1)
- [PostgreSQL](#postgresql)
  - [Passi per l'Installazione](#passi-per-linstallazione-2)
- [Keycloak](#keycloak)
  - [Passi per l'Installazione](#passi-per-linstallazione-3)
- [Verifica delle Installazioni](#verifica-delle-installazioni)
- [Pulizia](#pulizia)
- [Note Aggiuntive](#note-aggiuntive)
- [Contatti](#contatti)

---

## Creazione del Namespace

Per prima cosa, crea il namespace `test-requirements` dove verranno installati tutti i componenti:

```bash
kubectl create namespace test-requirements
```

---

## Opensearch

### Passi per l'Installazione

1. **Aggiungi il Repository Helm di Opensearch:**

   ```bash
   helm repo add opensearch https://opensearch-project.github.io/helm-charts/
   helm repo update
   ```

2. **Crea il File di Configurazione `values.yaml`:**

   Crea un file chiamato `values.yaml` con il seguente contenuto:

   ```yaml
   clusterName: "opensearch-cluster"
   nodeGroup: "master"

   roles:
     master: "true"
     ingest: "true"
     data: "true"
     remote_cluster_client: "true"

   replicas: 3  # Cluster a 3 nodi

   plugins:
     security:
       enabled: false  # Sicurezza disabilitata

   resources:
     requests:
       cpu: "100m"
       memory: "1Gi"
     limits:
       cpu: "1000m"
       memory: "3Gi"

   persistence:
     enabled: true
     size: "5Gi"  # 5Gi di storage per i volumi persistenti

   opensearchJavaOpts: "-Xmx512M -Xms512M"

   networkHost: "0.0.0.0"

   service:
     type: ClusterIP  # Accesso interno

   protocol: http
   httpPort: 9200
   transportPort: 9300
   ```

3. **Installa Opensearch con Helm:**

   ```bash
   helm install opensearch opensearch/opensearch -f values.yaml -n test-requirements
   ```

---

## RabbitMQ

### Passi per l'Installazione

1. **Aggiungi il Repository Helm di Bitnami:**

   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm repo update
   ```

2. **Crea la Secret per le Credenziali:**

   ```bash
   kubectl create secret generic rabbitmq-auth      --from-literal=rabbitmq-username=openk9      --from-literal=rabbitmq-password=openk9      --namespace=test-requirements
   ```

3. **Crea il File di Configurazione `values.yaml`:**

   ```yaml
   replicaCount: 1  # Installazione con singola replica

   auth:
     existingPasswordSecret: "rabbitmq-auth"

   resources:
     requests:
       cpu: "100m"
       memory: "1Gi"
     limits:
       cpu: "1000m"
       memory: "2Gi"

   persistence:
     enabled: true
     size: "5Gi"  # 5Gi di storage per il volume persistente

   service:
     type: ClusterIP  # Accesso interno
   ```

4. **Installa RabbitMQ con Helm:**

   ```bash
   helm install rabbitmq bitnami/rabbitmq -f values.yaml -n test-requirements
   ```

---

## PostgreSQL

### Passi per l'Installazione

1. **Crea la Secret per le Credenziali:**

   ```bash
   kubectl create secret generic postgresql-auth      --from-literal=postgres-password=openk9      --namespace=test-requirements
   ```

2. **Crea il File di Configurazione `values.yaml`:**

   ```yaml
   architecture: standalone  # Installazione con singola replica

   auth:
     existingSecret: "postgresql-auth"
     username: "openk9"
     database: "my_database"  # Sostituisci con il nome del tuo database

   primary:
     persistence:
       enabled: true
       size: "1Gi"  # 1Gi di storage per il volume persistente

   resources:
     requests:
       cpu: "100m"
       memory: "1Gi"
     limits:
       cpu: "1000m"
       memory: "2Gi"
   ```

3. **Installa PostgreSQL con Helm:**

   ```bash
   helm install postgresql bitnami/postgresql -f values.yaml -n test-requirements
   ```

---

## Keycloak

### Passi per l'Installazione

1. **Crea le Secret per le Credenziali:**

   - **Secret per il Database di Keycloak:**

     ```bash
     kubectl create secret generic keycloak-db-secret        --from-literal=password=openk9        --namespace=test-requirements
     ```

   - **Secret per l'Admin di Keycloak:**

     ```bash
     kubectl create secret generic keycloak-admin-secret        --from-literal=admin-password=openk9        --namespace=test-requirements
     ```

2. **Crea il File di Configurazione `values.yaml`:**

   ```yaml
   replicas: 1  # Installazione con singola replica

   postgresql:
     enabled: false  # Utilizza PostgreSQL esterno

   externalDatabase:
     host: "postgresql.test-requirements.svc.cluster.local"
     port: 5432
     user: "openk9"
     database: "keycloak"
     existingSecret: "keycloak-db-secret"

   auth:
     existingSecret: "keycloak-admin-secret"

   ingress:
     enabled: true
     hostname: "keycloak.local"  # Sostituisci con il tuo hostname
     annotations:
       kubernetes.io/ingress.class: "nginx"  # O il tuo Ingress Controller

   resources:
     requests:
       cpu: "100m"
       memory: "1Gi"
     limits:
       cpu: "1000m"
       memory: "2Gi"
   ```

3. **Installa Keycloak con Helm:**

   ```bash
   helm install keycloak bitnami/keycloak -f values.yaml -n test-requirements
   ```

---

## Verifica delle Installazioni

Per verificare che tutti i pod siano in esecuzione correttamente:

```bash
kubectl get pods -n test-requirements
```

Per controllare i servizi:

```bash
kubectl get svc -n test-requirements
```

Per visualizzare gli ingress:

```bash
kubectl get ingress -n test-requirements
```

---

## Pulizia

Per rimuovere tutte le risorse create:

1. **Disinstalla le release Helm:**

   ```bash
   helm uninstall opensearch -n test-requirements
   helm uninstall rabbitmq -n test-requirements
   helm uninstall postgresql -n test-requirements
   helm uninstall keycloak -n test-requirements
   ```

2. **Elimina il namespace:**

   ```bash
   kubectl delete namespace test-requirements
   ```

---

## Note Aggiuntive

- **Ingress Controller:**
  Assicurati di avere un Ingress Controller (come NGINX) installato nel cluster per utilizzare gli ingress.

- **Hostname e DNS:**
  Aggiorna il file `/etc/hosts` o il tuo DNS locale per risolvere `keycloak.local` all'IP del tuo cluster, se necessario.



---




