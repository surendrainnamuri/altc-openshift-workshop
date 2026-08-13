# OpenShift Hotel Website: CI/CD, Webhooks, Autoscaling, and Service Binding Lab

## Lab objectives

By the end of this lab, you will be able to:

1. Deploy a static website to OpenShift across `dev`, `pre-prod`, and `prod` projects.
2. Use Tekton Pipelines with GitHub push webhooks.
3. Build and deploy using OpenShift BuildConfigs and image triggers.
4. Create and run standalone Tekton Tasks and TaskRuns.
5. Configure CPU-based Horizontal Pod Autoscaling (HPA).
6. Demonstrate the OpenShift Service Binding Operator.

## Prerequisites

- Access to an OpenShift cluster with administrator or project-admin access.
- OpenShift Pipelines and Tekton Triggers installed.
- GitHub repository: `https://github.com/surendrainnamuri/hotel-website`.
- OpenShift projects: `dev`, `pre-prod`, and `prod`.
- OpenShift CLI logged in:

```powershell
oc whoami
oc get project dev pre-prod prod
```

The branch-to-environment mapping is:

| OpenShift project | Git branch |
| --- | --- |
| `dev` | `dev` |
| `pre-prod` | `pre-prod` |
| `prod` | `main` |

---

## Exercise 1: Website deployment resources

For each project, create these resources:

- `ImageStream`: `hotel-website`
- `BuildConfig`: `hotel-website`
- `Deployment`: `hotel-website`
- `Service`: `hotel-website`
- Website `Route`
- Tekton `Pipeline`: `hotel-website-ci-cd`
- Tekton `EventListener`: `hotel-website-github`

The public website routes follow this pattern:

```text
https://hotel-website-<environment>-<project>.<apps-domain>
```

Examples:

```text
https://hotel-website-dev-dev.apps.sno.ocp.skillonclick.com
https://hotel-website-pre-prod-pre-prod.apps.sno.ocp.skillonclick.com
https://hotel-website-prod-prod.apps.sno.ocp.skillonclick.com
```

The Service selects website pods and forwards port `8080`. The website Route points to the `hotel-website` Service.

---

## Exercise 2: Tekton CI/CD pipeline

The pipeline uses the following stages:

```text
clone → validate-html → build-image → deploy → report-route
```

| Stage | Responsibility |
| --- | --- |
| `clone` | Clones the environment’s Git branch. |
| `validate-html` | Verifies that core static-site files and folders exist. |
| `build-image` | Starts and waits for the OpenShift BuildConfig build. |
| `deploy` | Restarts the Deployment and waits for a successful rollout. |
| `report-route` | Prints the public route URL after the pipeline finishes. |

### Branch filtering

The EventListener filters GitHub push events before a PipelineRun is created:

```yaml
# Dev
body.ref == 'refs/heads/dev'

# Pre-production
body.ref == 'refs/heads/pre-prod'

# Production
body.ref == 'refs/heads/main'
```

This means a push to `dev` creates a run only in the `dev` project. GitHub itself does not provide a branch selector in the webhook form; the branch filter belongs in the OpenShift EventListener.

### Webhook routes

There is a second route type for CI/CD. It does not serve the website. It sends GitHub POST requests to the Tekton EventListener service.

| Environment | Webhook route |
| --- | --- |
| Dev | `https://hotel-website-webhook-dev-dev.apps.sno.ocp.skillonclick.com` |
| Pre-prod | `https://hotel-website-webhook-pre-prod-pre-prod.apps.sno.ocp.skillonclick.com` |
| Prod | `https://hotel-website-webhook-prod-prod.apps.sno.ocp.skillonclick.com` |

Retrieve a webhook route at any time:

```powershell
oc get route hotel-website-webhook-dev -n dev -o jsonpath='https://{.spec.host}'
```

### Configure GitHub webhooks

For each environment, create a GitHub webhook in the repository:

1. Repository → **Settings** → **Webhooks** → **Add webhook**.
2. Use the matching environment webhook route.
3. Select content type `application/json`.
4. Select **Just the push event**.
5. Add the environment’s webhook secret.

Retrieve a secret locally; never paste it into chat:

```powershell
oc get secret hotel-website-github-webhook -n dev -o jsonpath='{.data.secretToken}' |
  %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

### TLS note for lab clusters

If GitHub reports `x509: certificate signed by unknown authority`, the OpenShift router uses a certificate GitHub does not trust. For a lab-only cluster, disable SSL verification on that GitHub webhook. For a real production platform, install a publicly trusted certificate on the router instead.

### Verify pipeline execution

```powershell
oc get pipeline,pipelinerun,taskrun -n dev
oc get build,deploy,svc,route -n dev
```

---

## Exercise 3: Native OpenShift BuildConfig automation

Create a BuildConfig named:

```text
automated-buildconfig-trigger
```

For each environment, configure it to:

- Clone the matching branch.
- Build the static site using an NGINX source image.
- Publish to the `hotel-website:latest` ImageStreamTag.
- Respond to GitHub and config-change triggers.

Use **one image tag consistently**. In this lab, both the Tekton build and native BuildConfig publish to:

```text
hotel-website:latest
```

The website Deployment has an OpenShift image-trigger annotation that watches that tag. When a successful build updates `latest`, OpenShift updates the Deployment template and rolls out new pods.

Verify:

```powershell
oc get bc automated-buildconfig-trigger -n prod
oc get build -n prod
oc get deployment hotel-website -n prod
oc get is hotel-website -n prod
```

Important: a native BuildConfig GitHub webhook receives pushes without a branch filter. Its Git source reference controls which branch is built. Tekton EventListener filtering is the strict branch-isolation mechanism.

---

## Exercise 4: Tekton Task and TaskRun

### Concepts

A **Task** is a reusable unit of work. It defines:

- parameters;
- steps and container images;
- optional workspaces;
- optional results.

A **TaskRun** is one execution of a Task. It supplies parameter values, creates the execution pod, records logs, and reports success or failure.

```text
Task     = reusable recipe
TaskRun  = one execution of the recipe
```

### Sample Task

Create `hotel-website-demo-task`. It accepts an `environment` parameter and emits a completion message as a result.

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: hotel-website-demo-task
  namespace: dev
spec:
  params:
    - name: environment
      type: string
      default: dev
  results:
    - name: message
  steps:
    - name: show-context
      image: registry.access.redhat.com/ubi9/ubi-micro:latest
      script: |
        #!/bin/sh
        message="Hotel website Tekton task completed successfully for $(params.environment)."
        echo "$message"
        printf '%s' "$message" > "$(results.message.path)"
```

### Trigger the Task with a TaskRun

The OpenShift Console version used in this lab does not have **Actions → Start** for standalone Tasks. Create a TaskRun instead:

```yaml
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  generateName: hotel-website-demo-taskrun-
  namespace: dev
spec:
  taskRef:
    name: hotel-website-demo-task
  params:
    - name: environment
      value: dev
```

Use `oc create`, not `oc apply`, because `generateName` requests a unique TaskRun name each time:

```powershell
oc create -f hotel-website-demo-taskrun.yaml -n dev
```

Use `oc apply` only for the reusable Task definition:

```powershell
oc apply -f hotel-website-demo-task.yaml -n dev
```

Verify:

```powershell
oc get task hotel-website-demo-task -n dev
oc get taskrun -n dev
```

---

## Exercise 5: Service Binding demonstration

Install the Red Hat Service Binding Operator:

```powershell
oc apply -f openshift/service-binding-operator-subscription.yaml
oc get subscription rh-service-binding-operator -n openshift-operators
```

Create these demonstration resources in `dev`:

| Resource | Purpose |
| --- | --- |
| `hotel-booking-api` | A sample application Deployment. |
| `hotel-booking-database-credentials` | A Secret containing database-style connection values. |
| `hotel-booking-api-config-binding` | A ServiceBinding connecting the application to the Secret. |

The operator injects the binding as files into the application pod:

```text
/bindings/hotel-booking-database/DATABASE_HOST
/bindings/hotel-booking-database/DATABASE_PORT
/bindings/hotel-booking-database/DATABASE_NAME
/bindings/hotel-booking-database/DATABASE_USER
/bindings/hotel-booking-database/DATABASE_PASSWORD
```

Verify the binding:

```powershell
oc get servicebinding hotel-booking-api-config-binding -n dev
oc get pods -n dev -l app.kubernetes.io/name=hotel-booking-api
```

The expected status is `Ready=True`. The lesson is that the application Deployment does not directly declare the credential Secret; the `ServiceBinding` manages the relationship declaratively.

---

## Exercise 6: CPU-based HPA

Target workload:

```text
Project: surendrainnamuri
DeploymentConfig: hotel-booking
```

Create an HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hotel-booking-cpu-autoscaler
  namespace: surendrainnamuri
spec:
  scaleTargetRef:
    apiVersion: apps.openshift.io/v1
    kind: DeploymentConfig
    name: hotel-booking
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
```

The application container has a 20m CPU request. Therefore 80% corresponds to approximately 16m average CPU usage per pod.

### Demonstrate scale-up

Create a short-lived in-cluster load Job that sends concurrent requests to the internal Service:

```text
http://hotel-booking.surendrainnamuri.svc.cluster.local:8080/healthz
```

Monitor in separate terminals:

```powershell
oc get hpa hotel-booking-cpu-autoscaler -n surendrainnamuri -w
oc get pods -n surendrainnamuri -l app=hotel-booking -w
```

### Demonstrate scale-down

Delete the traffic generator:

```powershell
oc delete job hotel-booking-autoscale-load -n surendrainnamuri
```

Scale-down is not immediate. The HPA has a stabilization window to prevent rapid up/down oscillation. In this lab it is five minutes:

```yaml
scaleDown:
  stabilizationWindowSeconds: 300
```

The HPA can report `ScaleDownStabilized` even when CPU returns to a low value. That is expected behavior.

---

## Troubleshooting reference

| Symptom | Cause | Resolution |
| --- | --- | --- |
| GitHub webhook fails with unknown CA | Lab router certificate is not trusted by GitHub. | Disable SSL verification only for a lab webhook, or use a trusted certificate in production. |
| Pipeline clone works but validation fails | `emptyDir` workspace is isolated per Tekton task pod. | Use a shared PVC workspace, or make each Task independently clone what it needs. |
| Git clone fails because destination exists | PVC workspace root already exists. | Clone into a subfolder such as `/workspace/source/repo`. |
| `oc apply` fails with `cannot use generate name` | `generateName` requires creation of a new resource. | Use `oc create` for TaskRuns and `oc apply` for Tasks. |
| Pipeline deploy stage fails after a build | Deployment watches a different ImageStream tag from the tag that build publishes. | Align both paths to `hotel-website:latest`. |
| Website is not available after a successful build | Deployment replica count is zero. | Scale the Deployment to its desired count, for example `oc scale deployment/hotel-website -n prod --replicas=2`. |
| HPA does not scale down immediately | HPA stabilization window holds the maximum recent recommendation. | Wait for the configured scale-down window to elapse. |
| Tekton task pod logs are unavailable | Tekton cleanup removed the pod. | Inspect `oc get taskrun <name> -o yaml` and its terminated-step status. |

## Cleanup (optional)

```powershell
oc delete taskrun -n dev -l app.kubernetes.io/component=tekton-demo
oc delete task hotel-website-demo-task -n dev
oc delete servicebinding hotel-booking-api-config-binding -n dev
oc delete deployment hotel-booking-api -n dev
oc delete secret hotel-booking-database-credentials -n dev
oc delete hpa hotel-booking-cpu-autoscaler -n surendrainnamuri
oc delete job hotel-booking-autoscale-load -n surendrainnamuri --ignore-not-found
```
