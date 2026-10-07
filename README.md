# 🚀 Bubu & Aqua CI/CD Lab

Lokale CI/CD- und GitOps-Plattform auf Basis von:

- GitHub
- Tekton Pipelines
- Tekton Triggers
- Kaniko
- Minikube Registry
- Argo CD
- Kubernetes / Minikube
- Cloudflare Tunnel

## Architektur

Developer
│
│ git push
▼
GitHub
│
│ Webhook
▼
Cloudflare Tunnel
│
▼
Tekton EventListener
│
├── TriggerBinding
└── TriggerTemplate
│
▼
PipelineRun
│
├── clone
│     └── checkout Git SHA
│
├── hello
│
├── verify
│
├── build-image
│     └── Kaniko
│           ↓
│       Minikube Registry
│
└── update-gitops
│
▼
gitops/deployment.yaml
│
│ git push [skip ci]
▼
GitHub
│
▼
Argo CD
│
Auto-Sync
│
▼
Kubernetes
│
▼
Application

## Repository

cicd-bubu-aqua/
├── app/
│   ├── Dockerfile
│   └── index.html
│
├── gitops/
│   ├── deployment.yaml
│   └── service.yaml
│
├── tekton/
│   ├── pipeline.yaml
│   ├── task-git-clone.yaml
│   ├── task-update-gitops.yaml
│   ├── trigger-binding.yaml
│   ├── trigger-template.yaml
│   ├── event-listener.yaml
│   └── trigger-rbac.yaml
│
└── README.md

## Pipeline

Die Pipeline `bubu-aqua-ci` besteht aus:

clone
↓
hello
↓
verify
↓
build-image
↓
update-gitops

### clone

Klont das GitHub Repository und checkt exakt den Commit aus,
der den GitHub Webhook ausgelöst hat.

Der Commit SHA wird als `git-revision` durch die komplette
Pipeline transportiert.

### build-image

Kaniko baut das Docker Image.

Image innerhalb des Kubernetes Clusters:

registry.kube-system.svc:80/bubu-aqua:<GIT-SHA>

### GitOps Deployment

Für Kubernetes wird das Image verwendet als:

localhost:5000/bubu-aqua:<GIT-SHA>

Dieser Unterschied ist wichtig:

Tekton Pod
→ registry.kube-system.svc:80

Kubernetes Node / kubelet
→ localhost:5000

## GitOps Update

Nach erfolgreichem Build aktualisiert Tekton automatisch:

gitops/deployment.yaml

Beispiel:

image: localhost:5000/bubu-aqua:67e0ab05b40d19abf5aff6d48fd32398602c49fb

Danach erzeugt Tekton einen Commit:

Deploy <SHA> [skip ci]

und pusht diesen zurück nach GitHub.

## Loop Protection

Der Tekton EventListener besitzt einen CEL Filter.

Normale Commits:

Developer Commit
→ Pipeline startet

Automatische GitOps Commits:

Deploy <SHA> [skip ci]
→ Webhook
→ EventListener
→ STOP

Dadurch entsteht keine Endlosschleife.

## Argo CD

Argo CD überwacht:

gitops/

Auto-Sync:

Enabled

Self Heal:

Enabled

Prune:

Enabled

Eine Änderung des Image-Tags führt automatisch zu einem
Kubernetes Rollout.

## GitHub Credentials

Das GitHub Token befindet sich NICHT im Repository.

Es wird als Kubernetes Secret gespeichert:

github-credentials

Namespace:

cicd-bubu-aqua

Kontrolle:

kubectl get secret github-credentials -n cicd-bubu-aqua

Tekton verwendet das Secret über:

secretKeyRef:
name: github-credentials
key: token

## Vollständiger Deployment Flow

git push
↓
GitHub Webhook
↓
Tekton
↓
Checkout Git SHA
↓
Verify
↓
Kaniko Build
↓
Image:<SHA>
↓
Registry
↓
GitOps Update
↓
Git Push [skip ci]
↓
Argo CD Auto-Sync
↓
Kubernetes Rollout
↓
Application Running

## Aktueller Meilenstein

Version:

v1.0.0-cicd

Der Stand wurde als Git Tag gespeichert.

Der vollständige CI/CD-Durchlauf wurde erfolgreich mit einem
SHA-getaggten Image getestet.

## Nächste Schritte

- GitHub Webhook Secret
- permanenter Cloudflare Tunnel
- Cleanup alter PipelineRuns
- Cleanup alter Images
- Rollback
- zusätzliche Tests
- Pipeline Notifications
- Codex Integration