# Orange Kuma Platform

## Overzicht

Orange Kuma Platform is een volledig GitOps-gestuurd deploymentplatform voor multi-tenant Uptime Kuma omgevingen op Kubernetes.

Het project bestaat uit meerdere componenten die samenwerken:

* Orange Kuma Manager (Flask webapplicatie)
* Gitea container registry
* Semaphore CI/CD orchestration
* Ansible automation playbooks
* ArgoCD GitOps deployment management
* Kubernetes workloads
* Longhorn persistent storage
* Grafana monitoring

Het doel van het platform is het automatisch uitrollen, beheren, monitoren en verwijderen van klant-specifieke Uptime Kuma instanties.

Elke klant krijgt:

* een eigen Kubernetes namespace
* een eigen deployment
* een eigen persistent storage volume
* een eigen NodePort service
* een eigen ArgoCD Application
* een eigen GitOps manifeststructuur

---

# Architectuur

## Componentoverzicht

```text
Orange Kuma Manager
        |
        v
Semaphore API
        |
        v
Ansible Playbooks
        |
        +-------------------+
        |                   |
        v                   v
Gitea Repo            Kubernetes Cluster
(kuma-deployments)         |
        |                  |
        v                  v
      ArgoCD ----------> Deployments
```

---

# Hoofdcomponenten

## 1. Orange Kuma Manager

### Technologie

* Python Flask
* Bootstrap frontend
* REST API integraties
* Server-side rendering

### Functie

De manager is de centrale webinterface voor:

* deployments aanmaken
* deployments verwijderen
* image versies selecteren
* deployment status bekijken
* monitoring openen
* interactie met Semaphore

### Belangrijkste functionaliteiten

#### Nieuwe deployment aanmaken

Gebruiker voert in:

* klantnummer
* gewenste image versie

De manager:

1. haalt beschikbare tags op uit Gitea Registry
2. toont deze in een dropdown
3. start een Semaphore task
4. Semaphore voert Ansible playbook uit
5. Playbook schrijft manifests naar Git
6. ArgoCD deployed automatisch

#### Deployment verwijderen

De manager:

1. start Semaphore remove playbook
2. verwijdert GitOps manifests
3. wacht tot manifests verdwenen zijn
4. verwijdert ArgoCD Application
5. verwijdert namespace volledig
6. Kubernetes verwijdert alle resources

#### Monitoring

Navbar link:

```text
http://grafana.local/login
```

---

# 2. Gitea

## Doel

Gitea wordt gebruikt voor:

* GitOps manifests
* container registry
* versiebeheer
* deployment source-of-truth

## Repository structuur

### kuma-deployments

```text
customers/
  KN-652/
    orange-kuma/
      namespace.yaml
      deployment.yaml
      pvc.yaml
      service.yaml
      kustomization.yaml

argocd/
  orange-kuma-kn-652-application.yaml
```

---

# 3. Semaphore

## Functie

Semaphore wordt gebruikt als automation orchestrator.

### Taken

* deployment playbooks uitvoeren
* remove playbooks uitvoeren
* variabelen injecteren
* logging tonen
* taakstatus beheren

### Voorbeelden

#### Deployment task

```text
Deploy OK, geef klantnummer mee
```

#### Remove task

```text
Remove Orange Kuma Deployment
```

---

# 4. Ansible

## Doel

Ansible verzorgt:

* genereren van manifests
* GitOps repository updates
* Kubernetes cleanup
* ArgoCD management
* deployment orchestration

---

# Deployment Flow

## Stap 1 - Input validatie

Validatie van:

* customer_number
* image_tag

Voorbeeld:

```yaml
customer_number_input: "KN-652"
image_tag_input: "1.23.16"
```

---

## Stap 2 - Manifest generatie

De playbook genereert:

* namespace
* pvc
* deployment
* service
* kustomization
* ArgoCD application

---

## Stap 3 - ConfigMap generatie

Alle bestanden worden in een Kubernetes ConfigMap geplaatst.

---

## Stap 4 - Upload Job

Een tijdelijke Kubernetes Job:

* mount de manifests
* runt Python upload script
* schrijft bestanden naar Gitea API

---

## Stap 5 - GitOps Sync

ArgoCD detecteert:

* nieuwe manifests
* nieuwe Application
* nieuwe namespace

Daarna start automatische deployment.

---

# Remove Flow

## Belangrijk

De remove flow is expliciet GitOps-safe gemaakt.

Een eerdere implementatie verwijderde eerst de live ArgoCD Application.
Daardoor kon ArgoCD de application opnieuw aanmaken zolang de Git manifests nog bestonden.

Dit is opgelost.

## Correcte remove volgorde

### 1. Git manifests verwijderen

Bestanden worden verwijderd uit:

```text
customers/KN-xxx/orange-kuma/
argocd/orange-kuma-kn-xxx-application.yaml
```

### 2. Controleren dat Git cleanup voltooid is

Playbook wacht totdat Gitea bevestigt dat bestanden verdwenen zijn.

### 3. ArgoCD Application verwijderen

Pas nadat Git source verdwenen is.

### 4. Namespace verwijderen

Volledige namespace cleanup:

* deployments
* services
* PVCs
* configmaps
* secrets
* pods

Alles verdwijnt automatisch.

---

# Kubernetes Architectuur

## Namespace per klant

Voorbeeld:

```text
orange-kuma-kn-652
```

## Deployment naam

```text
orange-kuma-kn-652
```

## Persistent storage

Longhorn wordt gebruikt.

PVC:

```yaml
storageClassName: longhorn
```

## Service Type

```yaml
type: NodePort
```

## NodePort generatie

NodePort wordt automatisch berekend:

```yaml
31000 + laatste 2 digits klantnummer
```

Voorbeeld:

```text
KN-652 -> 31052
```

---

# Image Management

## Image Source

Beschikbare versies worden opgehaald uit:

```text
gitea.local:30080/superadmin/orange-kuma
```

## Waarom alleen registry tags?

De manager toont alleen images die:

* reeds geconverteerd
* reeds getest
* reeds beschikbaar
* reeds gepusht

zijn.

Daardoor worden deployment failures door ontbrekende images voorkomen.

---

# Monitoring

## Grafana

Monitoring link:

```text
http://grafana.local/login
```

## Aanbevolen dashboards

### Kubernetes Namespace Dashboard

Toont:

* CPU
* memory
* pod status
* restart count
* network traffic

### PVC Dashboard

Toont:

* storage usage
* Longhorn status
* volume health

### ArgoCD Dashboard

Toont:

* sync status
* out-of-sync apps
* deployment health

---

# Veiligheid

## Input validatie

Regex validatie voorkomt:

* command injection
* invalid image tags
* invalid customer nummers

Voorbeeld:

```yaml
- customer_number_input is match("^KN-[0-9]+$")
- image_tag_input is match("^[A-Za-z0-9_.-]+$")
```

---

# Belangrijke Problemen Die Zijn Opgelost

## 1. Recursive Ansible Variable Loop

### Probleem

```yaml
image_tag: "{{ image_tag | default('1.23.16') }}"
```

Dit veroorzaakte:

```text
recursive loop detected in template string
```

### Oplossing

Nieuwe variabelen:

```yaml
image_tag_input
customer_number_input
```

---

## 2. ArgoCD Recreation Loop

### Probleem

Deployment werd opnieuw aangemaakt na delete.

### Oorzaak

ArgoCD Application werd verwijderd voordat Git manifests verwijderd waren.

### Oplossing

Nieuwe volgorde:

1. Git cleanup
2. bevestiging
3. ArgoCD delete
4. namespace delete

---

## 3. Namespace Cleanup

### Probleem

Namespaces bleven bestaan.

### Oplossing

Volledige namespace delete toegevoegd:

```yaml
kind: Namespace
state: absent
```

---

# Omgevingsvariabelen

## Orange Kuma Manager

### Semaphore

```text
SEMAPHORE_URL
SEMAPHORE_API_TOKEN
SEMAPHORE_TEMPLATE_ID
SEMAPHORE_DELETE_TEMPLATE_ID
```

### Gitea

```text
GITEA_URL
GITEA_API_TOKEN
GITEA_USERNAME
GITEA_REPO
```

---

# Kubernetes Vereisten

## Benodigd

* Kubernetes cluster
* ArgoCD
* Longhorn
* Gitea
* Semaphore
* Grafana

---

# Deployment Voorbeeld

## Nieuwe klant

Input:

```text
Customer: KN-652
Version: 1.23.16
```

Resultaat:

```text
Namespace:
orange-kuma-kn-652

Service:
NodePort 31052

Image:
gitea.local:30080/superadmin/orange-kuma:1.23.16
```

---

# Troubleshooting

## Namespace blijft hangen

Controleer:

```bash
kubectl get namespace orange-kuma-kn-652 -o json | jq '.spec.finalizers'
```

Force remove:

```bash
kubectl patch namespace orange-kuma-kn-652 \
  -p '{"spec":{"finalizers":[]}}' \
  --type=merge
```

---

## ArgoCD app komt terug

Controleer:

* Git manifest nog aanwezig?
* app-of-apps configuratie?
* sync delay?

---

## Deployment verschijnt niet

Controleer:

```bash
kubectl get applications -n argocd
```

en:

```bash
kubectl logs -n semaphore deployment/semaphore
```

---

# Toekomstige Verbeteringen

## Mogelijke uitbreidingen

### Authenticatie

* Keycloak
* OAuth2
* SSO

### Multi-cluster support

* meerdere Kubernetes clusters
* regio deployments
* HA management

### Autoscaling

* HPA
* cluster autoscaler

### SSL/TLS

* cert-manager
* Let's Encrypt
* wildcard ingress

### Backup Management

* PVC snapshots
* Longhorn backups
* restore workflows

---

# Samenvatting

Orange Kuma Platform is een volledig geautomatiseerd GitOps deploymentplatform voor multi-tenant Uptime Kuma workloads op Kubernetes.

De architectuur combineert:

* Flask
* Semaphore
* Ansible
* Gitea
* ArgoCD
* Kubernetes
* Longhorn
* Grafana

tot één beheersysteem waarmee deployments veilig uitgerold, beheerd, gemonitord en verwijderd kunnen worden.
