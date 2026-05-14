# Lab 13 — GitOps with ArgoCD

## Task 1 — ArgoCD Installation & Setup

### 1.1 Install ArgoCD via Helm

1. Add the ArgoCD Helm repository:
   ![Screenshot: Add Helm repo](/docs_lab13/screenshots/helm_repo_add_argo.png)

2. Create dedicated namespace and install ArgoCD:
   ![Screenshot: Create namespace](/docs_lab13/screenshots/kubectl_create_namespace_argocd.png)

3. Verify all pods are ready:
   All components should show `Running` status.

### 1.2 Access ArgoCD UI

1. Set up port forwarding to access the web interface:
   ![Screenshot: Port forwarding](/docs_lab13/screenshots/task2_port_forward_1.png)

2. Retrieve the initial admin password:
   ![Screenshot: Get password](/docs_lab13/screenshots/task2_get_pswrd.png)

3. Access ArgoCD at `https://localhost:8080` in your browser:
   ![Screenshot: UI login page](/docs_lab13/screenshots/task2_app_test_ui.png)

4. Log in with credentials:
   - Username: `admin`
   - Password: `pfZ3jJ3qkuw5l5qo`
   ![Screenshot: After login](/docs_lab13/screenshots/task2_after_login.png)

### 1.3 Install ArgoCD CLI

1. Download the CLI for your platform (Windows):
   ```PowerShell
   Invoke-WebRequest -Uri "https://github.com/argoproj/argo-cd/releases/latest/download/argocd-windows-amd64.exe" -OutFile "argocd.exe"
   ```

2. Log in via CLI:
   ![0](/docs_lab13/screenshots/login_via_cl.png)

3. Verify CLI connection:
   ![1](/docs_lab13/screenshots/agrocd_version.png)
   ![2](/docs_lab13/screenshots/agrocd_user.png)


## Task 2 — Application Deployment

### 2.1 ArgoCD Application Manifests

All application manifests are stored in `k8s/argocd/`:

- **`application.yaml`** - Basic application in default namespace (for testing)
- **`application-dev.yaml`** - Dev environment with auto-sync enabled
- **`application-prod.yaml`** - Prod environment with manual sync only
- **`applicationset.yaml`** - ApplicationSet for automated generation of dev/prod apps (bonus)

### 2.2 Application Manifest Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: python-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Daniil20xx/DevOps-Core-Course.git
    targetRevision: lab12
    path: k8s/mychart
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

### 2.3 Deploy the Application

Apply the application manifest:
![1](/docs_lab13/screenshots/task2_k8s_apply.png)

Monitor the deployment:
![2](/docs_lab13/screenshots/task2_lifecheck.png)

### 2.5 Perform Initial Sync

Trigger manual sync via CLI:
![sync_after_change_2](/docs_lab13/screenshots/to_1-sync_after_change_2.png)

Verify resources are created:
![task2_heakth_check](/docs_lab13/screenshots/task2_heakth_check.png)

Verify that 1 replica:
![after_change_2_to_1](/docs_lab13/screenshots/after_change_2_to_1.png)

### GitOps Workflow Explanation

1. A change is made in the Git repository (e.g., replica count updated)
2. ArgoCD detects that the cluster state differs from Git (OutOfSync)
3. Manual sync is triggered
4. Cluster state is reconciled to match Git

This demonstrates the GitOps principle where Git is the single source of truth.


## Task 3 — Multi-environment deployment (dev/prod)

### 3.1 Create namespaces

![apply_namespace](/docs_lab13/screenshots/apply_namespace.png)

### 3.2 Create ArgoCD Applications for dev/prod

```bash
kubectl apply -f k8s/argocd/applicationset.yaml
kubectl apply -f k8s/argocd/application-dev.yaml
kubectl apply -f k8s/argocd/application-prod.yaml
```

App list:
![app_list](/docs_lab13/screenshots/app_list.png)

Sync prod:
```bash
argocd app sync devops-info-service-prod
```

![task3_verify_deplay](/docs_lab13/screenshots/task3_verify_deplay.png)
![task3_verify_prod](/docs_lab13/screenshots/task3_verify_prod.png)

*If apps show `OutOfSync` + `Missing`, it only means resources are not created yet.*

### 3.3 Dev vs Prod differences

The development (dev) and production (prod) environments are configured differently to reflect their distinct purposes.

**Replica count:**

* Dev uses `replicaCount: 1` to minimize resource usage and allow quick iterations.
* Prod uses `replicaCount: 3` to ensure high availability and fault tolerance.

**Image configuration:**

* Dev uses the `latest` tag with `Always` pull policy to quickly test new changes.
* Prod uses a fixed version (`1.0`) with `IfNotPresent` to ensure stability and reproducibility.

**Resources:**

* Dev has lower CPU and memory limits (`200m / 256Mi`) to conserve resources.
* Prod allocates higher resources (`500m / 512Mi`) to handle real workloads reliably.

**Health checks:**

* Dev uses faster probes (shorter delays and intervals) for quicker feedback during development.
* Prod uses more conservative probe settings to avoid false positives and unnecessary restarts.

**Logging:**

* Dev uses `logLevel: debug` for detailed troubleshooting.
* Prod uses `logLevel: warn` to reduce noise and improve performance.

**Environment variables:**

* Dev explicitly sets `env: development`
* Prod sets `env: production` to reflect runtime context.

**High availability:**

* Prod includes a PodDisruptionBudget (`minAvailable: 2`) to maintain service availability during disruptions.
* Dev does not include this, as availability is not critical.

These differences demonstrate how the same application can be tuned for different environments using Helm values.

### 3.4 Why prod stays manual

The production environment is configured to use manual synchronization instead of automatic sync, which is a common best practice in GitOps workflows.

**Reasons for manual sync in production:**

* **Controlled deployments:** Changes are only applied when explicitly approved, reducing the risk of accidental updates.
* **Stability:** Prevents unstable or untested changes (e.g., from `latest` images) from being deployed automatically.
* **Review process:** Allows teams to review changes in Git before syncing them to the cluster.
* **Rollback planning:** Manual control enables better handling of failures and rollback strategies.
* **Compliance and auditing:** Many production environments require approval steps before deployment.

**Why dev uses auto-sync:**

* Faster feedback loop for developers
* Immediate reflection of changes from Git
* Useful for testing GitOps workflows and self-healing

In summary, dev prioritizes speed and experimentation, while prod prioritizes stability and control.

## Task 4 — Self-healing & drift tests (dev)

### 4.1 Self-healing test: manual scale

![task4_scale_deployment](/docs_lab13/screenshots/task4_scale_deployment.png)
![task4_diff](/docs_lab13/screenshots/task4_diff.png)

- Manually scaled deployment to new count of replicas.
- ArgoCD detected configuration drift

### 4.2 Pod deletion test (Kubernetes behavior)
![task4_diff](/docs_lab13/screenshots/task4_diff.png)
![task4_pod_deletion](/docs_lab13/screenshots/task4_pod_deletion.png)

- I manually deleted the pod using `kubectl delete pod`
- Kubernetes automatically recreated the pod via the ReplicaSet
- This is Kubernetes built-in self-healing capability, not ArgoCD


### 4.3 Configuration drift test (ArgoCD behavior)

```bash
kubectl annotate deploy -n dev python-app-dev-mychart drift-ts="$(date +%s)" --overwrite
# deployment.apps/python-app-dev-mychart annotated

# Observation: in this cluster/ArgoCD setup, changing top-level Deployment metadata annotations
# did not immediately flip the app to OutOfSync (depends on tracking method), and behavior may vary:

kubectl get deploy -n dev python-app-dev-mychart \
  -o jsonpath='{.metadata.annotations.drift-ts}{"\n"}'
# 1777066438

argocd app get python-app-dev --refresh | grep -E "Sync Status|Health Status" || true
# Sync Status: Synced ...
# Health Status: Healthy

sleep 8

kubectl get deploy -n dev python-app-dev-mychart \
  -o jsonpath='{.metadata.annotations.drift-ts}{"\n"}' || true
# 1777066438


# Reliable drift for evidence (replicas change):
kubectl patch deployment python-app-dev-mychart -n dev \
  --type merge -p '{"spec":{"replicas":5}}'

# Immediately after patch (before self-heal):
kubectl get deploy -n dev python-app-dev-mychart \
  -o jsonpath='{.spec.replicas}{"\n"}'
# 5

# ArgoCD detects drift and (with self-heal enabled OR during next reconciliation loop) reverts it:
argocd app diff python-app-dev || true

sleep 10

argocd app get python-app-dev --refresh | grep -E "Sync Status|Health Status" || true
# Sync Status: Synced ...
# Health Status: Healthy

kubectl get deploy -n dev python-app-dev-mychart \
  -o jsonpath='{.spec.replicas}{"\n"}'
# 1
```

### 4.4 When does ArgoCD sync and how often it checks Git?
ArgoCD synchronization behavior depends on its configuration.

By default, ArgoCD checks the Git repository every **3 minutes** for changes.

Sync can be triggered in several ways:

- **Manual sync** via UI or CLI
- **Automatic sync** (if enabled in syncPolicy)
- **Webhook trigger** for immediate updates

**Key difference between ArgoCD and Kubernetes:**

- Kubernetes ensures runtime state (e.g., pod count via ReplicaSet)
- ArgoCD ensures configuration state matches Git

Thus:
- Kubernetes handles infrastructure self-healing
- ArgoCD handles configuration drift reconciliation

## Bonus — ApplicationSet

ApplicationSet allows generating multiple applications from a single template.

### Benefits:
- Eliminates duplication of manifests
- Scales better for multiple environments
- Centralized configuration

### Why used here:
Instead of maintaining separate `application-dev.yaml` and `application-prod.yaml`,
ApplicationSet dynamically generates them using environment-specific parameters.

### When to use:
- Multi-environment setups
- Multi-cluster deployments
- Monorepos with multiple apps


Additional Screenshots:
![app_web](/docs_lab13/screenshots/app_web.png)