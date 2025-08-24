# Rolling Updates in Kubernetes: A Complete Guide

## Understanding Rolling Updates

**Rolling Updates** in Kubernetes are the default deployment strategy that gradually replaces old pods with new ones, ensuring **zero-downtime deployments**. Think of it like renovating a busy restaurant one table at a time while keeping the establishment open and serving customers.

### The Core Philosophy
```
Old Version → Gradual Transition → New Version
    ↓              ↓                  ↓
  Still Running   Mixed State    Fully Updated
```

Unlike blue-green deployments that require double the resources, rolling updates work within your existing resource constraints by strategically managing the replacement process.

## How Rolling Updates Work

### The Update Process Flow

1. **Initiation**: New ReplicaSet created with updated pod template
2. **Scale Up**: New pods created according to strategy parameters
3. **Health Verification**: New pods must pass readiness checks
4. **Scale Down**: Old pods terminated once new ones are ready
5. **Iteration**: Process repeats until all pods are updated
6. **Cleanup**: Old ReplicaSet retained for rollback capability

### Visual Representation
```
Initial State:    [Pod1-v1] [Pod2-v1] [Pod3-v1]
Step 1:          [Pod1-v1] [Pod2-v1] [Pod3-v1] [Pod4-v2]
Step 2:          [Pod1-v1] [Pod2-v1] [Pod4-v2] [Pod5-v2]
Step 3:          [Pod1-v1] [Pod4-v2] [Pod5-v2] [Pod6-v2]
Final State:      [Pod4-v2] [Pod5-v2] [Pod6-v2]
```

## Rolling Update Strategy Configuration

### Basic Strategy Parameters
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%    # Max pods that can be unavailable
      maxSurge: 25%          # Max extra pods during update
  template:
    metadata:
      labels:
        app: webapp
        version: v2.1.0
    spec:
      containers:
      - name: webapp
        image: myapp:v2.1.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

### Strategy Parameters Explained

#### maxUnavailable
- **Purpose**: Controls service availability during updates
- **Values**: Absolute number (e.g., `2`) or percentage (e.g., `25%`)
- **Impact**: Lower values = more conservative updates, higher availability

```yaml
maxUnavailable: 1      # Only 1 pod down at a time (safest)
maxUnavailable: 25%    # 25% of pods can be unavailable (balanced)
maxUnavailable: 50%    # Faster updates, less availability guarantee
```

#### maxSurge
- **Purpose**: Controls resource usage during updates
- **Values**: Absolute number or percentage
- **Impact**: Higher values = faster updates but more resources needed

```yaml
maxSurge: 0        # No extra pods (resource-constrained)
maxSurge: 25%      # 25% extra pods allowed (balanced)
maxSurge: 100%     # Double the pods temporarily (fastest)
```

## Real-World Strategy Examples

### Conservative Update (High Availability)
```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 0      # Never reduce capacity
    maxSurge: 1           # Add one pod at a time
```
**Use Case**: Critical production services, databases
**Trade-offs**: Slower updates, requires extra resources

### Balanced Update (Default)
```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```
**Use Case**: Most web applications
**Trade-offs**: Good balance of speed and resource usage

### Fast Update (Development)
```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 50%
    maxSurge: 50%
```
**Use Case**: Development environments, non-critical services
**Trade-offs**: Faster updates, temporary service degradation possible

### Resource-Constrained Update
```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 0          # No extra resources needed
```
**Use Case**: Limited cluster resources, cost optimization
**Trade-offs**: Slower updates, temporary capacity reduction

## Health Checks and Readiness

### Critical Configuration Elements

#### Readiness Probes
```yaml
readinessProbe:
  httpGet:
    path: /api/health
    port: 8080
  initialDelaySeconds: 15    # Wait before first check
  periodSeconds: 5           # Check every 5 seconds
  timeoutSeconds: 3          # Timeout after 3 seconds
  successThreshold: 1        # 1 success = ready
  failureThreshold: 3        # 3 failures = not ready
```

#### Liveness Probes
```yaml
livenessProbe:
  httpGet:
    path: /api/health
    port: 8080
  initialDelaySeconds: 30    # Longer delay for startup
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3        # 3 failures = restart pod
```

### Health Check Best Practices

1. **Separate Endpoints**: Use different endpoints for readiness vs liveness
2. **Lightweight Checks**: Avoid database queries in health checks
3. **Proper Timeouts**: Set realistic timeout values
4. **Dependency Awareness**: Readiness should check downstream dependencies

## Executing Rolling Updates

### Triggering an Update
```bash
# Method 1: Update image directly
kubectl set image deployment/webapp-deployment webapp=myapp:v2.1.0

# Method 2: Edit deployment
kubectl edit deployment webapp-deployment

# Method 3: Apply updated YAML
kubectl apply -f deployment.yaml

# Method 4: Patch deployment
kubectl patch deployment webapp-deployment -p '{"spec":{"template":{"spec":{"containers":[{"name":"webapp","image":"myapp:v2.1.0"}]}}}}'
```

### Monitoring Update Progress
```bash
# Watch rollout status
kubectl rollout status deployment/webapp-deployment

# Monitor pods during update
kubectl get pods -l app=webapp -w

# Detailed rollout history
kubectl rollout history deployment/webapp-deployment

# Check ReplicaSets
kubectl get rs -l app=webapp
```

### Update Output Example
```bash
$ kubectl rollout status deployment/webapp-deployment
Waiting for deployment "webapp-deployment" rollout to finish: 2 out of 6 new replicas have been updated...
Waiting for deployment "webapp-deployment" rollout to finish: 3 out of 6 new replicas have been updated...
Waiting for deployment "webapp-deployment" rollout to finish: 4 out of 6 new replicas have been updated...
Waiting for deployment "webapp-deployment" rollout to finish: 5 out of 6 new replicas have been updated...
Waiting for deployment "webapp-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "webapp-deployment" successfully rolled out
```

## Rollback Operations

### When Rollbacks Are Needed
- Application bugs discovered post-deployment
- Performance degradation
- Failed health checks
- Dependency issues

### Rollback Commands
```bash
# Quick rollback to previous version
kubectl rollout undo deployment/webapp-deployment

# Rollback to specific revision
kubectl rollout undo deployment/webapp-deployment --to-revision=3

# Check rollback status
kubectl rollout status deployment/webapp-deployment
```

### Rollback Configuration
```yaml
spec:
  revisionHistoryLimit: 10    # Keep 10 previous ReplicaSets
  progressDeadlineSeconds: 600 # Timeout after 10 minutes
```

## Advanced Rolling Update Patterns

### Canary Deployments with Rolling Updates
```yaml
# Step 1: Deploy canary version to subset
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-canary
spec:
  replicas: 1              # Small canary deployment
  selector:
    matchLabels:
      app: webapp
      version: canary
  template:
    spec:
      containers:
      - name: webapp
        image: myapp:v2.1.0-canary
```

### Blue-Green with Rolling Updates
```bash
# Deploy to green environment
kubectl apply -f green-deployment.yaml

# Gradually shift traffic using service weights
kubectl patch service webapp-service -p '{"spec":{"selector":{"version":"green"}}}'
```

### A/B Testing Integration
```yaml
# Deploy with feature flags
env:
- name: FEATURE_FLAG_NEW_UI
  value: "true"
- name: ROLLOUT_PERCENTAGE
  value: "25"
```

## Troubleshooting Common Issues

### Failed Rolling Updates

#### Symptom: Update Stuck
```bash
# Check pod status
kubectl get pods -l app=webapp

# Look for events
kubectl describe deployment webapp-deployment

# Check ReplicaSet issues
kubectl describe rs -l app=webapp
```

#### Common Causes and Solutions

1. **Failed Readiness Probes**
   ```bash
   # Check probe configuration
   kubectl describe pod <failing-pod>
   
   # Adjust probe settings
   initialDelaySeconds: 30  # Increase startup time
   periodSeconds: 10        # Less frequent checks
   ```

2. **Resource Constraints**
   ```bash
   # Check node resources
   kubectl top nodes
   
   # Adjust resource requests
   resources:
     requests:
       cpu: 100m      # Reduce if necessary
       memory: 128Mi
   ```

3. **Image Pull Issues**
   ```bash
   # Check image pull secrets
   kubectl get secrets
   
   # Verify image exists
   docker pull myapp:v2.1.0
   ```

4. **Network Connectivity**
   ```bash
   # Test service connectivity
   kubectl exec -it <pod> -- curl webapp-service:8080/health
   ```

### Performance Issues During Updates

#### Slow Updates
- **Cause**: Conservative strategy settings
- **Solution**: Increase `maxSurge` and `maxUnavailable`

#### Resource Spikes
- **Cause**: Aggressive `maxSurge` settings
- **Solution**: Use `maxSurge: 0` for resource-constrained environments

#### Service Disruption
- **Cause**: Inadequate readiness probes
- **Solution**: Implement comprehensive health checks

## Production Best Practices

### Pre-Deployment Checklist
- [ ] Test deployment in staging environment
- [ ] Verify health check endpoints
- [ ] Confirm resource requirements
- [ ] Review rollback procedures
- [ ] Set up monitoring and alerts

### Deployment Strategy Guidelines

#### Critical Production Services
```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 20%
spec:
  progressDeadlineSeconds: 900  # 15 minutes timeout
  revisionHistoryLimit: 10
```

#### Standard Web Applications
```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
spec:
  progressDeadlineSeconds: 300  # 5 minutes timeout
```

### Monitoring and Observability
```yaml
# Add deployment annotations for tracking
metadata:
  annotations:
    deployment.kubernetes.io/revision: "3"
    kubernetes.io/change-cause: "Update to v2.1.0 - Bug fixes"
```

### Integration with CI/CD
```bash
#!/bin/bash
# Automated deployment script
kubectl set image deployment/webapp webapp=myapp:${BUILD_NUMBER}
kubectl rollout status deployment/webapp --timeout=300s

if [ $? -ne 0 ]; then
    echo "Deployment failed, rolling back..."
    kubectl rollout undo deployment/webapp
    exit 1
fi
```

Rolling updates represent the gold standard for zero-downtime deployments in Kubernetes. Master these concepts and patterns, and you'll be able to deploy confidently in any production environment while maintaining service availability and system reliability.
