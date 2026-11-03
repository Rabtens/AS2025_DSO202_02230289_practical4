# GitOps with Argo CD on a Kind Cluster (DSO202 Practical)
---

## 1. Objective

Implement a pull-based GitOps workflow where **GitHub is the single source of truth** and **Argo CD** (running inside a local Kind cluster) continuously reconciles the cluster to match Git.

The practical demonstrates three behaviours:

1. **Git as source of truth** – scaling by editing YAML in Git.
2. **Self-healing** – manual `kubectl` drift is automatically reverted.
3. **Pruning** – deleting a manifest from Git deletes the resource from the cluster.

## 2. Architecture

```
[Developer] --> [GitHub repo: dso202-gitops-demo] <--(polls / pulls)-- [Argo CD in Kind cluster dso202]
                                                                            |
                                                          control-plane + worker + worker
```

Argo CD components used: `argocd-server` (UI/API), `argocd-repo-server` (clones repo, renders YAML), `argocd-application-controller` (compares Git vs live state).

## 3. Repository Structure

```
dso202-gitops-demo/
├── README.md
├── kind-config.yaml        # 3-node Kind cluster definition
├── argocd-app.yaml         # Argo CD Application (declarative registration)
└── manifests/
    ├── deployment.yaml     # webapp-deploy (nginxdemos/hello:plain-text)
    └── service.yaml        # webapp-svc (ClusterIP)
```

## 4. Prerequisites

- Docker running
- `kind`, `kubectl`, `git` installed
- A GitHub account (public repo named `dso202-gitops-demo`)

Check tool versions:

```bash
docker --version
kind --version
kubectl version --client
git --version
```
![alt text](<evidences/Screenshot from 2026-11-03 16-49-10.png>)
---

## 5. Part A – Environment Setup

### Step 1: Create the 3-node Kind cluster (Already Exists)

```bash
kind create cluster --config kind-config.yaml
```
![alt text](<evidences/Screenshot from 2026-11-03 16-51-44.png>)

### Step 2: Verify nodes

```bash
kubectl cluster-info --context kind-dso202
kubectl get nodes
```
![alt text](<evidences/Screenshot from 2026-11-03 16-52-06.png>)

### Step 3: Install Argo CD

```bash
kubectl create namespace argocd
```
![alt text](<evidences/Screenshot from 2026-11-03 16-55-02.png>)

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
![alt text](<evidences/Screenshot from 2026-11-03 16-55-30.png>)

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```
![alt text](<evidences/Screenshot from 2026-11-03 17-01-00.png>)

```bash
kubectl get pods -n argocd
```

Expected: `argocd-server`, `argocd-repo-server`, `argocd-application-controller`, `argocd-redis`, `argocd-dex-server`, etc. all `Running`.

![alt text](<evidences/Screenshot from 2026-11-03 17-01-19.png>)

### Step 4: Access the Argo CD UI

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
![alt text](<evidences/Screenshot from 2026-11-03 17-02-29.png>)

Port-forward (**run in a second terminal and keep it open**):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

![alt text](<evidences/Screenshot from 2026-11-03 17-07-07.png>)

Open **https://localhost:8080** → Advanced → *Proceed to localhost (unsafe)* → login with `admin` / the password above.

![alt text](<evidences/Screenshot from 2026-11-03 17-09-43.png>)

![alt text](<evidences/Screenshot from 2026-11-03 17-11-37.png>)

---

## 6. Part B – Prepare the GitHub Repository

Create a **public** repo `dso202-gitops-demo` on GitHub (tick *Add a README*), then:

```bash
git clone https://github.com/Rabtens/AS2025_DSO202_02230289_practical4
cd AS2025_DSO202_02230289_practical4
mkdir manifests
```

Create `manifests/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deploy
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: nginxdemos/hello:plain-text
          ports:
            - containerPort: 80
```

Create `manifests/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
spec:
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

Check the files exist:

```bash
ls -R manifests
```
![alt text](<evidences/Screenshot from 2026-11-03 17-15-08.png>)

Commit and push:

```bash
git add .
git commit -m "Initial deployment with 2 replicas"
git push origin main
```

📸 **Screenshot 14 – Push to GitHub**





📸 **Screenshot 15 – GitHub repo showing `manifests/` folder**





---

## 7. Part C – Register the Application in Argo CD (Declarative)

Create `argocd-app.yaml` (replace `<YOUR-GITHUB-USERNAME>`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-webapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR-GITHUB-USERNAME>/dso202-gitops-demo.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply it:

```bash
kubectl apply -f argocd-app.yaml
```

📸 **Screenshot 16 – Application created**





```bash
kubectl get applications -n argocd
```

Expected: `my-webapp   Synced   Healthy`

📸 **Screenshot 17 – Application Synced/Healthy**





Verify the pods are spread across the worker nodes:

```bash
kubectl get pods -n demo-app -o wide
```

📸 **Screenshot 18 – 2 pods on worker nodes**





📸 **Screenshot 19 – Argo CD UI tile `my-webapp` (Synced + Healthy)**





📸 **Screenshot 20 – Argo CD resource tree (Application → Deployment → ReplicaSet → 2 Pods + Service)**





---

## 8. Part D – Experiment 1: Git as the Single Source of Truth

Scale from 2 → 4 replicas **through Git**:

```bash
sed -i 's/replicas: 2/replicas: 4/' manifests/deployment.yaml
git diff
```

📸 **Screenshot 21 – git diff showing replicas 2 → 4**





```bash
git add manifests/deployment.yaml
git commit -m "Scale webapp to 4 replicas"
git push origin main
```

📸 **Screenshot 22 – Push of scaling commit**





Click **Refresh** in the Argo CD UI (or wait ≤3 minutes for polling), then:

```bash
kubectl get pods -n demo-app
```

Expected: 4 pods `Running`.

📸 **Screenshot 23 – 4 pods in terminal**





📸 **Screenshot 24 – Argo CD UI showing 4 pods**





**Result:** The cluster changed only because Git changed – no `kubectl apply` was used.

---

## 9. Part E – Experiment 2: Configuration Drift & Self-Healing

Simulate a "rogue admin" bypassing Git:

```bash
kubectl scale deployment webapp-deploy -n demo-app --replicas=1
```

Immediately watch the pods:

```bash
kubectl get pods -n demo-app -w
```

(Press `Ctrl+C` after pods return to 4.)

📸 **Screenshot 25 – Scale-down command and pods terminating/recreating**





📸 **Screenshot 26 – Argo CD UI flashing OutOfSync / pods restored**





Confirm final state:

```bash
kubectl get deployment webapp-deploy -n demo-app
```

Expected: `READY 4/4`

📸 **Screenshot 27 – Deployment back to 4/4**





**Result:** Because `selfHeal: true`, Argo CD reverted the manual change to match Git.

---

## 10. Part F – Experiment 3: Automatic Pruning

Delete the Service from Git:

```bash
git rm manifests/service.yaml
git commit -m "Remove webapp service"
git push origin main
```

📸 **Screenshot 28 – Deleting service.yaml and pushing**





Click **Refresh** in the Argo CD UI, then:

```bash
kubectl get svc -n demo-app
```

Expected: `No resources found in demo-app namespace.`

📸 **Screenshot 29 – Service pruned (terminal)**





📸 **Screenshot 30 – Argo CD UI tree without the Service**





### Rollback with `git revert`

```bash
git revert HEAD --no-edit
git push origin main
```

📸 **Screenshot 31 – git revert and push**





After Refresh:

```bash
kubectl get svc -n demo-app
```

Expected: `webapp-svc` is back.

📸 **Screenshot 32 – Service restored**





```bash
git log --oneline
```

📸 **Screenshot 33 – Git history (audit trail)**





---

## 11. Troubleshooting

| Symptom | Meaning | Fix |
|---|---|---|
| `ComparisonError` | Argo CD can't read repo/path | Repo must be public; `repoURL` ends in `.git`; `path: manifests` spelled correctly |
| `OutOfSync` (stuck) | Change seen but not applied | Auto-sync off, or YAML syntax error – read error under *Sync Status* |
| `Synced` + `Degraded` | Git matches but pods crash | Check pod Events/Logs in the UI |
| `Progressing` forever | Rollout not finishing | Low CPU/RAM on Kind nodes or `ImagePullBackOff` |

## 12. Clean Up

```bash
kind delete cluster --name dso202
```

📸 **Screenshot 34 – Cluster deleted**





---

## 13. Observations & Conclusion

| Experiment | Action | Outcome |
|---|---|---|
| 1 – Source of truth | Changed `replicas: 2 → 4` in Git | Argo CD created 2 extra pods automatically |
| 2 – Self-heal | `kubectl scale ... --replicas=1` | Argo CD reverted to 4 replicas within seconds |
| 3 – Prune | Deleted `service.yaml` from Git | Argo CD deleted `webapp-svc` from the cluster |

GitOps removes the need for CI to hold cluster credentials, keeps every change auditable in Git history, and eliminates configuration drift through continuous reconciliation.
