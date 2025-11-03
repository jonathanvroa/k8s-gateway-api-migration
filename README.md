# NGINX Ingress Controller to Gateway API + Envoy Gateway Migration

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Gateway API](https://img.shields.io/badge/Gateway%20API-v1-00ADD8)](https://gateway-api.sigs.k8s.io/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

A comprehensive guide and reference implementation for migrating from NGINX Ingress Controller to Envoy Gateway using the Kubernetes Gateway API. This project demonstrates production-ready patterns including automated TLS certificate management, multi-tenant architecture, and secure cross-namespace communication.

## 🎯 Overview

This repository provides a complete migration path from traditional Ingress resources to the modern Gateway API, showcasing:

- **Modern Gateway API Implementation**: Native Kubernetes Gateway API v1 resources
- **Automated TLS Management**: Integration with cert-manager and Let's Encrypt
- **Multi-Tenant Architecture**: Namespace isolation with controlled cross-namespace access
- **Production Patterns**: HTTP-to-HTTPS redirects, path-based routing, and backend policies
- **Security Best Practices**: ReferenceGrants for secure resource access across namespaces

## 🏗️ Architecture
Migration Summary
Previous Architecture (NGINX Ingress Controller)

### Request Flow Example
```
1. User Request: tenant-b.example.com/
   ↓
2. NGINX Ingress: SSL/TLS Termination (cert-manager)
   ↓
3. NGINX Server Snippet: 301 Redirect → /tenant-b/home
   ↓
4. User Redirected: tenant-b.example.com/tenant-b/home
   ↓
5. NGINX Location Snippet: Custom proxy settings
   ↓
6. NGINX Ingress: Proxy to shared-backend.backend-services.svc.cluster.local:8000
   ↓
7. Backend: Serve application at /tenant-b/home
```
### Previous Architecture (NGINX Ingress Controller)
Limitations:

- **Vendor Lock-in**: Heavy reliance on NGINX-specific annotations and snippets (nginx.org/server-snippets, nginx.org/location-snippets)
- **Complex Certificate Management**
- **Configuration Complexity**: Routing logic embedded in raw NGINX configuration blocks
- **Limited Portability**: Difficult to migrate between different ingress implementations
- **Monolithic Resource**: All configuration (routing, TLS, policies) centralized in a single Ingress manifest

### New Architecture (Gateway API + Envoy Gateway)

### Request Flow Example

```
1. User Request: tenant-a.example.com/
   ↓
2. HTTPRoute: 301 Redirect → /tenant-a/home
   ↓
3. User Redirected: tenant-a.example.com/tenant-a/home
   ↓
4. HTTPRoute: Route to shared-backend:8000
   ↓
5. Backend: Serve application at /tenant-a/home
```

### Component Overview



```
┌─────────────────────────────────────────────────────────────┐
│                     Main Gateway                            │
│              (envoy-gateway-system)                         │
│                                                             │
│          ┌──────────────┐         ┌──────────────┐          │
│          │ HTTP :80     │         │ HTTPS :443   │          │
│          │ (Global      │         │ (TLS         │          │
│          │  Redirect)   │         │  Termination)│          │
│          └──────────────┘         └──────────────┘          │
└─────────────────────────────────────────────────────────────┘
                              │
                ┌─────────────┴──────────────┐
                │                            │
        ┌───────▼─────────┐         ┌────────▼────────┐
        │   Tenant A      │         │   Tenant B      │
        │   Namespace     │         │   Namespace     │
        │                 │         │                 │
        │ • HTTPRoute     │         │ • HTTPRoute     │
        │ • Certificate   │         │ • Certificate   │
        │ • BackendPolicy │         │ • BackendPolicy │
        └───────┬─────────┘         └────────┬────────┘
                │                            │
                └──────────────┬─────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Backend Services   │
                    │     Namespace       │
                    │                     │
                    │ • shared-backend    │
                    │ • ReferenceGrants   │
                    └─────────────────────┘
```

## 📋 Prerequisites

- Kubernetes cluster (v1.29+)
- kubectl configured
- Envoy Gateway installed
- cert-manager installed (v1.17+)
- DNS records pointing to your cluster's LoadBalancer

## 🚀 Quick Start

### 1. Install Envoy Gateway

```bash
helm install \
  eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.5.0 \
  -n envoy-gateway-system \
  --create-namespace
```

### 2. Configure cert-manager RBAC for Gateway API

```bash
kubectl apply -f manifests/cert-manager-rbac.yaml
```

### 3. Deploy the Gateway

```bash
kubectl apply -f manifests/gateway.yaml
```

### 4. Configure Let's Encrypt ClusterIssuer

Update the email in `manifests/clusterissuer.yaml`, then apply:

```bash
kubectl apply -f manifests/clusterissuer.yaml
```

### 5. Enable Global HTTPS Redirect

```bash
kubectl apply -f manifests/global-https-redirect.yaml
```

## 📁 Project Structure

```
k8s-gateway-api-migration/
├── manifests/
│   ├── gateway.yaml                 # Main Gateway resource (LoadBalancer)
│   ├── cert-manager-rbac.yaml       # RBAC for cert-manager Gateway API support
│   ├── clusterissuer.yaml           # Let's Encrypt ClusterIssuer
│   ├── global-https-redirect.yaml   # HTTP to HTTPS redirect
│   └── multi-tenant/
│       ├── reference-grant.yaml     # Cross-namespace service access
│       ├── tenant-a/
│       │   ├── certificate.yaml     # TLS certificate for tenant-a
│       │   ├── httproute.yaml       # Routing rules for tenant-a
│       │   ├── backend-policy.yaml  # Backend traffic policy
│       │   └── reference-grant.yaml # Certificate access grant
│       └── tenant-b/
│           ├── certificate.yaml     # TLS certificate for tenant-b
│           ├── httproute.yaml       # Routing rules for tenant-b
│           ├── backend-policy.yaml  # Backend traffic policy
│           └── reference-grant.yaml # Certificate access grant
├── README.md
└── LICENSE
```

## 🔧 Multi-Tenant Setup

### Create Namespaces

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
kubectl create namespace backend-services
```

### Multi-tenant Backend Service (Example)
Architecture Overview
```yaml
Service: shared-backend
Namespace: backend-services
Port: 8000
```

Tenant Routing:

```tenant-a.example.com``` → Routes to ```/tenant-a/home```

```tenant-b.example.com``` → Routes to ```/tenant-b/home```



### Deploy Tenant A

```bash
kubectl apply -f manifests/multi-tenant/tenant-a/
```

### Deploy Tenant B

```bash
kubectl apply -f manifests/multi-tenant/tenant-b/
```

### Configure Cross-Namespace Access

```bash
kubectl apply -f manifests/multi-tenant/reference-grant.yaml
```

## 🔐 Security Features

### ReferenceGrants

This project implements secure cross-namespace communication using ReferenceGrants:

- **Service Access**: HTTPRoutes in tenant namespaces can access services in the backend-services namespace
- **Certificate Access**: The Gateway can access TLS certificates stored in tenant namespaces
- **Principle of Least Privilege**: Each grant explicitly defines what resources can be accessed

### TLS Certificate Management

- Automated certificate issuance via cert-manager
- Let's Encrypt ACME HTTP-01 challenge
- Certificates stored in tenant namespaces
- Automatic renewal before expiration

## 📊 Key Features

### Gateway API Resources

| Resource | Purpose | Location |
|----------|---------|----------|
| Gateway | LoadBalancer and listener configuration | envoy-gateway-system |
| HTTPRoute | Layer 7 routing rules | Tenant namespaces |
| ReferenceGrant | Cross-namespace access control | backend-services |
| BackendTrafficPolicy | Timeout and retry configuration | Tenant namespaces |

### Traffic Management

- **HTTP to HTTPS Redirect**: Automatic 301 redirects for all HTTP traffic
- **Path-based Routing**: Root path redirects to tenant-specific home pages
- **Host-based Routing**: Multiple domains on a single Gateway
- **Backend Policies**: Configurable timeouts and connection settings


## 📝 Migration from NGINX Ingress

### Before (NGINX Ingress)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    acme.cert-manager.io/http01-edit-in-place: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod-ingress
    nginx.ingress.kubernetes.io/app-root: /tenant-a/home
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.org/server-snippets: |
      if ($uri = /) {                                                                            
        return 301 $scheme://$http_host/tenant-a/home;       
      }
    nginx.org/location-snippets: |
      location / {
        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
        client_max_body_size 1m;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering on;
        proxy_pass http://shared-backend.backend-services.svc.cluster.local:8000;
      }
  name:  tenant-a-ingress
  namespace:  tenant-a
spec:
  ingressClassName: nginx
  rules:
  - host: tenant-a.example.com
    http:
      paths:
      - backend:
          service:
            name: shared-backend
            port:
              number: 8000
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - tenant-a.example.com
    secretName: tenant-a-tls

```


### 🧪 Test Routing

```bash
curl https://tenant-a.example.com/
# Should redirect to /tenant-a/home

curl https://tenant-a.example.com/tenant-a/home
# Should return application content
```

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🔗 Resources

- [Kubernetes Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway Documentation](https://gateway.envoyproxy.io/)
- [cert-manager Documentation](https://cert-manager.io/docs/)


## 👤 Author

**Jonathan Velasco Roa**

- GitHub: [@jonathanvroa](https://github.com/jonathanvroa)

---

⭐ If you find this project helpful, please consider giving it a star!
