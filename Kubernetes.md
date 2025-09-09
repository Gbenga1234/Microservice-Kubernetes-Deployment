# Microservices Kubernetes Deployment Guide

## Overview

This setup includes two simple Node.js microservices:
- **User Service**: Manages user data (port 3001)
- **Order Service**: Manages orders and communicates with User Service (port 3002)

## Prerequisites

1. Kubernetes cluster (minikube, kind, or cloud provider)
2. kubectl configured to connect to your cluster
3. Optional: Ingress controller (nginx-ingress) for routing

## Quick Deployment

### 1. Save the YAML Configuration

Save the provided YAML as `microservices.yaml`

### 2. Deploy to Kubernetes

```bash
# Apply all resources
kubectl apply -f microservices.yaml

# Check deployment status
kubectl get pods
kubectl get services
kubectl get deployments
```

### 3. Verify Services are Running

```bash
# Check pod status
kubectl get pods -l app=user-service
kubectl get pods -l app=order-service

# Check logs
kubectl logs -l app=user-service
kubectl logs -l app=order-service
```

## Testing the Services

### Port Forwarding (for local testing)

```bash
# Forward user service
kubectl port-forward service/user-service 3001:3001

# In another terminal, forward order service
kubectl port-forward service/order-service 3002:3002
```

### API Endpoints

#### User Service (localhost:3001)
```bash
# Get all users
curl http://localhost:3001/users

# Get user by ID
curl http://localhost:3001/users/1

# Create new user
curl -X POST http://localhost:3001/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Bob Johnson", "email": "bob@example.com"}'

# Health check
curl http://localhost:3001/health
```

#### Order Service (localhost:3002)
```bash
# Get all orders
curl http://localhost:3002/orders

# Get orders by user (includes user details)
curl http://localhost:3002/orders/user/1

# Create new order
curl -X POST http://localhost:3002/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "product": "Tablet", "amount": 299.99}'

# Health check
curl http://localhost:3002/health
```

## Production Considerations

### 1. Use Custom Docker Images

Instead of using ConfigMaps, create proper Docker images:

```dockerfile
# Dockerfile for user-service
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --only=production
COPY app.js ./
EXPOSE 3001
CMD ["npm", "start"]
```

### 2. Environment Variables

```yaml
env:
- name: NODE_ENV
  value: "production"
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: url
```

### 3. Persistent Storage

Add persistent volumes for databases:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### 4. Service Mesh (Optional)

Consider using Istio for advanced traffic management:

```bash
# Enable Istio injection
kubectl label namespace default istio-injection=enabled
```

## Monitoring and Observability

### Add Prometheus Monitoring

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "3001"
    prometheus.io/path: "/metrics"
```

### Logging

```yaml
# Add to container spec
env:
- name: LOG_LEVEL
  value: "info"
volumeMounts:
- name: logs
  mountPath: /var/log
```

## Scaling and Updates

### Manual Scaling
```bash
# Scale user service
kubectl scale deployment user-service --replicas=5

# Scale order service  
kubectl scale deployment order-service --replicas=3
```

### Rolling Updates
```bash
# Update image
kubectl set image deployment/user-service user-service=your-registry/user-service:v2

# Check rollout status
kubectl rollout status deployment/user-service
```

### Rollback
```bash
# Rollback to previous version
kubectl rollout undo deployment/user-service
```

## Cleanup

```bash
# Remove all resources
kubectl delete -f microservices.yaml

# Or remove specific resources
kubectl delete deployment user-service order-service
kubectl delete service user-service order-service api-gateway
kubectl delete configmap user-service-config order-service-config
kubectl delete hpa user-service-hpa order-service-hpa
kubectl delete ingress microservices-ingress
```

## Next Steps

1. **Add a Database**: Integrate PostgreSQL or MongoDB
2. **Implement Authentication**: Add JWT-based auth service
3. **API Gateway**: Use Kong, Ambassador, or custom gateway
4. **Service Discovery**: Implement proper service registration
5. **Circuit Breaker**: Add resilience patterns with libraries like Hystrix
6. **Distributed Tracing**: Add Jaeger or Zipkin for request tracing

## Common Issues

### Pod Not Starting
```bash
# Check pod events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>
```

### Service Communication Issues
```bash
# Test service connectivity from within cluster
kubectl run debug --image=busybox --rm -it -- sh
# Then: wget -qO- http://user-service:3001/health
```

### Resource Limits
```bash
# Check resource usage
kubectl top pods
kubectl top nodes
```