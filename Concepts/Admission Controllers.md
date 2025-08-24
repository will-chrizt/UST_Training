# Admission Controllers: The Security and Policy Gatekeepers of Kubernetes

## Understanding Admission Controllers: The Final Quality Gate

Picture a high-security government building where every visitor must pass through multiple checkpoints. First, security verifies your identity (authentication). Then, they check if you're authorized to enter specific areas (authorization). But even after these checks pass, there's one final gate: a comprehensive security screening that can modify your visit request (remove prohibited items) or deny entry altogether based on current security policies.

**Admission controllers serve this exact role in Kubernetes**—they're the final gatekeepers that examine, modify, or reject requests to the API server after authentication and authorization have already succeeded.

### The Critical Position in Request Processing

```
Client Request → Authentication → Authorization → Admission Controllers → etcd
                     ✓               ✓              📋 Final Review
```

This positioning is crucial because admission controllers operate with **complete cluster context**—they can examine the full request, current cluster state, and apply sophisticated business logic that simple RBAC cannot handle.

### Two Fundamental Types of Admission Controllers

#### **Mutating Admission Controllers**
These controllers can **modify** the incoming request before it's stored:
- Add default values to resource specifications
- Inject sidecar containers into pods
- Set resource limits automatically
- Apply standardized labels and annotations

#### **Validating Admission Controllers** 
These controllers **validate** the request and can only accept or reject it:
- Enforce organizational policies
- Validate complex business rules
- Ensure compliance requirements
- Block dangerous configurations

**The Critical Sequence**: Mutating controllers always run **before** validating controllers, ensuring that validations happen against the final, modified resource.

## Built-in Admission Controllers: The Essential Arsenal

### Core Controllers Every Cluster Needs

#### **NamespaceLifecycle: Protecting Namespace Integrity**
```yaml
# This controller prevents operations like:
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: terminating-namespace  # ❌ REJECTED - Namespace is terminating
spec:
  containers:
  - name: app
    image: nginx
```

**What it prevents:**
- Creating resources in non-existent namespaces
- Creating resources in namespaces being deleted
- Deleting required system namespaces (kube-system, kube-public)

#### **ServiceAccount: Ensuring Pod Identity**
```yaml
# Without this controller, pods might run without proper identity
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
spec:
  # If no serviceAccount specified, controller injects:
  # serviceAccountName: default
  # And mounts the service account token automatically
  containers:
  - name: app
    image: nginx
    # Controller adds volume mount:
    # volumeMounts:
    # - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
    #   name: default-token-xxxxx
```

**Critical security functions:**
- Assigns default service account if none specified
- Mounts service account tokens for API access
- Validates service account exists in the target namespace

#### **ResourceQuota: Enforcing Resource Boundaries**
```yaml
# ResourceQuota definition
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20" 
    limits.memory: 40Gi
    persistentvolumeclaims: "4"

---
# This pod would be REJECTED if quota exceeded
apiVersion: v1
kind: Pod
metadata:
  name: resource-hungry-pod
  namespace: development
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "15"      # ❌ Exceeds quota limit
        memory: 30Gi
```

**Quota enforcement scope:**
- CPU and memory requests/limits
- Storage consumption
- Object counts (pods, services, etc.)
- Extended resources (GPUs, custom resources)

#### **LimitRanger: Applying Default Resource Controls**
```yaml
# LimitRange configuration
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-cpu-limits
  namespace: production
spec:
  limits:
  - default:          # Default limits
      memory: 512Mi
      cpu: 500m
    defaultRequest:   # Default requests
      memory: 256Mi
      cpu: 250m
    max:              # Maximum allowed
      memory: 2Gi
      cpu: 2
    min:              # Minimum required
      memory: 100Mi
      cpu: 100m
    type: Container

---
# Pod without resource specifications gets defaults applied
apiVersion: v1
kind: Pod
metadata:
  name: auto-limited-pod
  namespace: production
spec:
  containers:
  - name: app
    image: nginx
    # LimitRanger automatically adds:
    # resources:
    #   requests:
    #     memory: 256Mi
    #     cpu: 250m
    #   limits:
    #     memory: 512Mi  
    #     cpu: 500m
```

### Security-Focused Admission Controllers

#### **PodSecurityPolicy (Deprecated but Educational)**
While deprecated in favor of Pod Security Standards, PSP demonstrates advanced admission control concepts:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
spec:
  privileged: false                    # No privileged containers
  allowPrivilegeEscalation: false     # Prevent privilege escalation
  requiredDropCapabilities:           # Must drop these capabilities
    - ALL
  volumes:                            # Allowed volume types
    - 'configMap'
    - 'emptyDir' 
    - 'projected'
    - 'secret'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'         # Cannot run as root
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

#### **NodeRestriction: Limiting kubelet Permissions**
This controller ensures that kubelet can only modify resources related to its own node:

```yaml
# kubelet on node1 can only update:
apiVersion: v1
kind: Node
metadata:
  name: node1  # ✅ Same node - ALLOWED
spec:
  # kubelet can update node status, labels, etc.

---
# kubelet on node1 CANNOT update:
apiVersion: v1
kind: Node  
metadata:
  name: node2  # ❌ Different node - REJECTED
```

## Custom Admission Controllers: Extending Kubernetes Policy

### Admission Webhooks: Your Custom Policy Engine

#### **ValidatingAdmissionWebhook: Custom Validation Logic**
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionWebhook
metadata:
  name: security-policy-webhook
webhooks:
- name: container-security.company.com
  clientConfig:
    service:
      name: security-webhook-service
      namespace: webhook-system
      path: "/validate"
    # Custom CA for webhook TLS
    caBundle: LS0tLS1CRUdJTi0tLS0t...
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  # Webhook behavior configuration
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  failurePolicy: Fail  # Reject requests if webhook fails
  timeoutSeconds: 10   # Maximum webhook response time
```

**Example webhook server implementation:**
```python
# webhook-server.py - Custom security validation
from flask import Flask, request, jsonify
import json
import base64

app = Flask(__name__)

@app.route('/validate', methods=['POST'])
def validate_admission():
    admission_review = request.get_json()
    
    # Extract the pod specification
    pod = admission_review["request"]["object"]
    allowed = True
    message = ""
    
    # Custom validation logic
    containers = pod.get("spec", {}).get("containers", [])
    
    for container in containers:
        # Reject containers running as root
        security_context = container.get("securityContext", {})
        if security_context.get("runAsUser") == 0:
            allowed = False
            message = f"Container {container['name']} cannot run as root"
            break
            
        # Require resource limits
        if not container.get("resources", {}).get("limits"):
            allowed = False
            message = f"Container {container['name']} must specify resource limits"
            break
            
        # Block privileged containers
        if security_context.get("privileged", False):
            allowed = False
            message = f"Container {container['name']} cannot run in privileged mode"
            break
    
    # Return admission response
    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview", 
        "response": {
            "uid": admission_review["request"]["uid"],
            "allowed": allowed,
            "status": {"message": message} if not allowed else {}
        }
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8443, ssl_context='adhoc')
```

#### **MutatingAdmissionWebhook: Automatic Resource Modification**
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionWebhook
metadata:
  name: pod-defaults-webhook
webhooks:
- name: pod-defaults.company.com
  clientConfig:
    service:
      name: defaults-webhook-service
      namespace: webhook-system
      path: "/mutate"
    caBundle: LS0tLS1CRUdJTi0tLS0t...
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"] 
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
```

**Mutation webhook example - Automatic sidecar injection:**
```python
# mutating-webhook.py - Automatic sidecar injection
import json
import jsonpatch
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/mutate', methods=['POST'])
def mutate_admission():
    admission_review = request.get_json()
    pod = admission_review["request"]["object"]
    
    # Create modifications using JSON Patch
    patches = []
    
    # Add logging sidecar to every pod
    sidecar_container = {
        "name": "log-collector",
        "image": "fluent/fluentd:latest",
        "resources": {
            "requests": {"memory": "64Mi", "cpu": "50m"},
            "limits": {"memory": "128Mi", "cpu": "100m"}
        },
        "volumeMounts": [{
            "name": "shared-logs",
            "mountPath": "/var/log/app"
        }]
    }
    
    # Add sidecar container
    containers = pod.get("spec", {}).get("containers", [])
    containers.append(sidecar_container)
    
    patches.append({
        "op": "replace",
        "path": "/spec/containers", 
        "value": containers
    })
    
    # Add shared volume for logs
    volumes = pod.get("spec", {}).get("volumes", [])
    volumes.append({
        "name": "shared-logs",
        "emptyDir": {}
    })
    
    patches.append({
        "op": "add" if not volumes else "replace",
        "path": "/spec/volumes",
        "value": volumes
    })
    
    # Return mutation response
    patch_bytes = json.dumps(patches).encode()
    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": admission_review["request"]["uid"],
            "allowed": True,
            "patch": base64.b64encode(patch_bytes).decode(),
            "patchType": "JSONPatch"
        }
    })
```

## Advanced Admission Controller Patterns

### Policy as Code with Open Policy Agent (OPA)

#### **OPA Gatekeeper: Declarative Policy Management**
```yaml
# Constraint Template - defines policy structure
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        properties:
          labels:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        
        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("Missing required label: %v", [missing])
        }

---
# Constraint - applies the policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: must-have-environment-label
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces: ["production", "staging"]
  parameters:
    labels: ["environment", "team", "cost-center"]
```

### Multi-Stage Admission Control

#### **Coordinated Admission Pipeline**
```yaml
# Stage 1: Mutating webhook adds defaults
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionWebhook
metadata:
  name: 01-defaults-webhook
webhooks:
- name: defaults.company.com
  # Adds resource limits, labels, annotations

---
# Stage 2: Mutating webhook injects sidecars  
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionWebhook
metadata:
  name: 02-sidecar-webhook
webhooks:
- name: sidecars.company.com
  # Injects monitoring, logging, security sidecars

---
# Stage 3: Validating webhook enforces policies
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionWebhook
metadata:
  name: 03-policy-webhook
webhooks:
- name: policies.company.com
  # Validates final configuration against policies
```

## Troubleshooting Admission Controllers: Common Issues and Solutions

### Diagnostic Methodology

#### **Step 1: Identify the Failing Controller**
```bash
# Check API server logs for admission failures
kubectl logs -n kube-system kube-apiserver-master-node

# Look for patterns like:
# "admission webhook denied the request"
# "failed calling webhook"
# "admission controller rejected"

# Test specific operations with verbose logging
kubectl create -f problem-resource.yaml --dry-run=server -v=8
```

#### **Step 2: Webhook-Specific Debugging**
```bash
# Check webhook configuration
kubectl get validatingadmissionwebhooks
kubectl get mutatingadmissionwebhooks

# Examine webhook details
kubectl describe validatingadmissionwebhook security-webhook

# Test webhook service connectivity
kubectl get svc -n webhook-system security-webhook-service
kubectl get endpoints -n webhook-system security-webhook-service

# Check webhook server logs
kubectl logs -n webhook-system deployment/security-webhook
```

### Common Issues and Resolutions

#### **Issue 1: Webhook Timeout/Connectivity**
```yaml
# Problem: Webhook service unreachable
# Symptoms: "context deadline exceeded" errors

# Solution 1: Verify service and endpoint configuration
kubectl get svc security-webhook-service -o yaml
kubectl get endpoints security-webhook-service

# Solution 2: Check network policies
kubectl get networkpolicies -n webhook-system

# Solution 3: Validate TLS certificate
openssl x509 -in webhook-cert.pem -text -noout
```

#### **Issue 2: Certificate Validation Failures**
```bash
# Problem: TLS certificate issues
# Symptoms: "x509: certificate signed by unknown authority"

# Solution: Update webhook caBundle
WEBHOOK_CA=$(kubectl get secret webhook-certs -o jsonpath='{.data.ca\.crt}')
kubectl patch validatingadmissionwebhook security-webhook \
  --type='json' -p="[{'op': 'replace', 'path': '/webhooks/0/clientConfig/caBundle', 'value': '$WEBHOOK_CA'}]"
```

#### **Issue 3: Resource Creation Blocked by Policies**
```yaml
# Problem: Legitimate resources being rejected
# Symptoms: Pods stuck in Pending with admission errors

# Solution: Check resource against policies
kubectl get constraintviolations
kubectl describe k8srequiredlabels must-have-environment-label

# Temporary bypass for debugging (USE CAREFULLY)
kubectl label namespace test-ns admission.gatekeeper.sh/ignore=true
```

## Production Best Practices: Secure and Reliable Admission Control

### **Security Hardening Guidelines**

#### **Webhook Security Checklist**
```yaml
# 1. Always use HTTPS with proper certificates
clientConfig:
  service:
    name: webhook-service
    namespace: webhook-system
    path: "/validate"
  caBundle: <base64-encoded-ca-cert>  # Required for TLS validation

# 2. Set appropriate failure policies
failurePolicy: Fail  # Production: fail closed for security
# failurePolicy: Ignore  # Development: fail open for flexibility

# 3. Configure reasonable timeouts
timeoutSeconds: 10  # Balance between security and performance

# 4. Specify precise matching rules
rules:
- operations: ["CREATE", "UPDATE"]  # Don't intercept DELETE unnecessarily
  apiGroups: [""]
  apiVersions: ["v1"]
  resources: ["pods"]
  # Use namespaceSelector for targeted enforcement
  namespaceSelector:
    matchLabels:
      admission-policy: "enforced"
```

#### **High Availability Webhook Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-webhook
  namespace: webhook-system
spec:
  replicas: 3  # Multiple replicas for availability
  selector:
    matchLabels:
      app: admission-webhook
  template:
    metadata:
      labels:
        app: admission-webhook
    spec:
      # Anti-affinity for distribution across nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: admission-webhook
              topologyKey: kubernetes.io/hostname
      containers:
      - name: webhook
        image: company/admission-webhook:v1.2.3
        ports:
        - containerPort: 8443
          name: webhook-api
        resources:
          requests:
            memory: 64Mi
            cpu: 50m
          limits:
            memory: 128Mi
            cpu: 100m
        # Health checks for reliability
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8443
            scheme: HTTPS
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8443
            scheme: HTTPS
          initialDelaySeconds: 5
          periodSeconds: 5
```

### **Monitoring and Observability**

#### **Admission Controller Metrics**
```yaml
# Prometheus monitoring configuration
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: admission-webhook-metrics
spec:
  selector:
    matchLabels:
      app: admission-webhook
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics

---
# Key metrics to monitor:
# admission_webhook_request_total - Request count by result
# admission_webhook_request_duration_seconds - Request latency
# admission_webhook_certificate_expiry_seconds - Certificate validity
```

#### **Alerting Rules for Admission Controllers**
```yaml
# Critical admission controller alerts
groups:
- name: admission-controllers
  rules:
  - alert: AdmissionWebhookDown
    expr: up{job="admission-webhook"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Admission webhook is down"
      description: "The admission webhook has been down for more than 1 minute"

  - alert: AdmissionWebhookHighLatency
    expr: |
      histogram_quantile(0.99, 
        rate(admission_webhook_request_duration_seconds_bucket[5m])
      ) > 5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Admission webhook high latency"

  - alert: AdmissionWebhookCertificateExpiry
    expr: |
      admission_webhook_certificate_expiry_seconds - time() < 86400 * 7
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Admission webhook certificate expires soon"
```

## The Future of Admission Controllers

### **Emerging Patterns and Technologies**

#### **CEL (Common Expression Language) Validations**
Kubernetes 1.25+ introduces CEL for lightweight validations without webhooks:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cel-validated-pod
spec:
  # CEL validation rules embedded in CRDs
  containers:
  - name: app
    image: nginx
    # Validation: memory requests must be less than limits
    # cel: self.resources.requests.memory <= self.resources.limits.memory
```

#### **Policy Engines Integration**
- **Falco integration**: Runtime security policy enforcement
- **Kustomize policies**: Git-based policy management
- **Crossplane compositions**: Infrastructure admission control

Admission controllers represent one of Kubernetes' most powerful extensibility mechanisms. They transform the API server from a simple CRUD interface into an intelligent policy enforcement engine that can embody your organization's operational wisdom, security requirements, and compliance mandates.

Master these concepts—from built-in controllers through custom webhooks—and you'll possess the tools to create Kubernetes environments that are not just functional, but truly enterprise-ready with automated governance, security, and operational excellence baked directly into the platform itself.

Remember: **great admission controllers are invisible to users when they work correctly**, but provide essential guardrails that prevent entire classes of operational and security issues. Design them thoughtfully, test them thoroughly, and deploy them with proper monitoring—they're your cluster's immune system.
