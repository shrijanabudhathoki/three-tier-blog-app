# Kubernetes 3-Tier Blog Application Architecture

## Overview
This document describes the architecture of a 3-tier blog application deployed on a Kubernetes cluster using kubeadm, with NGINX Ingress Controller, MetalLB load balancer, and Helm for deployment management.

## Infrastructure Architecture

```mermaid
graph TB
    subgraph "VMware Infrastructure"
        subgraph "Private Network (192.168.14.0/24)"
            Master["Master Node<br/>192.168.14.50<br/>4GB RAM, 4 CPUs"]
            Worker1["Worker1 Node<br/>192.168.14.54<br/>2GB RAM, 2 CPUs"]
            Worker2["Worker2 Node<br/>192.168.14.55<br/>2GB RAM, 2 CPUs"]
        end
    end
    
    subgraph "Host Machine"
        Vagrant["Vagrant<br/>VM Orchestration"]
        Browser["Web Browser<br/>https://myapp.blog.com"]
    end
    
    Vagrant --> Master
    Vagrant --> Worker1
    Vagrant --> Worker2
    Browser --> Master
    
    style Master fill:#e1f5fe
    style Worker1 fill:#f3e5f5
    style Worker2 fill:#f3e5f5
```

## Kubernetes Cluster Architecture

```mermaid
graph TB
    subgraph "External Access"
        Client["Client Browser<br/>https://myapp.blog.com"]
        DNS["DNS Resolution<br/>/etc/hosts<br/>192.168.14.240 myapp.blog.com"]
    end
    
    subgraph "MetalLB LoadBalancer"
        LB["MetalLB<br/>IP Pool: 192.168.14.240-250<br/>External IP: 192.168.14.240"]
    end
    
    subgraph "Kubernetes Cluster"
        subgraph "ingress-nginx namespace"
            IC["NGINX Ingress Controller<br/>LoadBalancer Service<br/>Port: 80/443"]
        end
        
        subgraph "default namespace"
            subgraph "Application Layer"
                FE["Frontend Pod<br/>shrijanab/blog-frontend:latest<br/>Port: 80<br/>Nginx + React"]
                BE1["Backend1 Pod<br/>shrijanab/blog-backend1:latest<br/>Port: 3000<br/>User Management"]
                BE2["Backend2 Pod<br/>shrijanab/blog-backend2:latest<br/>Port: 3001<br/>Blog & Comments"]
            end
            
            subgraph "Services"
                FE_SVC["frontend-service<br/>ClusterIP:80"]
                BE1_SVC["backend1-service<br/>ClusterIP:3000"]
                BE2_SVC["backend2-service<br/>ClusterIP:3001"]
            end
            
            subgraph "Data Layer"
                PG["PostgreSQL StatefulSet<br/>postgres:15<br/>Port: 5432"]
                PG_SVC["postgres-service<br/>Headless Service"]
                PV["Persistent Volume<br/>5Gi Storage<br/>/mnt/data/postgres"]
            end
            
            subgraph "Configuration"
                TLS_SECRET["TLS Secret<br/>myapp-tls-secret<br/>Self-signed Certificate"]
                APP_SECRET["Application Secret<br/>backend-secret<br/>Clerk API Keys"]
                INGRESS["Ingress Resource<br/>myapp-ingress<br/>Host: myapp.blog.com"]
            end
        end
        
        subgraph "metallb-system namespace"
            METALLB["MetalLB Controller<br/>Speaker DaemonSet<br/>IP Address Pool"]
        end
    end
    
    Client --> DNS
    DNS --> LB
    LB --> IC
    IC --> INGRESS
    
    INGRESS --> FE_SVC
    INGRESS --> BE1_SVC
    INGRESS --> BE2_SVC
    
    FE_SVC --> FE
    BE1_SVC --> BE1
    BE2_SVC --> BE2
    
    BE1 --> PG_SVC
    BE2 --> PG_SVC
    PG_SVC --> PG
    PG --> PV
    
    INGRESS -.-> TLS_SECRET
    BE1 -.-> APP_SECRET
    BE2 -.-> APP_SECRET
    
    IC -.-> METALLB
    
    style Client fill:#e8f5e8
    style LB fill:#fff3e0
    style IC fill:#e3f2fd
    style FE fill:#f3e5f5
    style BE1 fill:#e8f5e8
    style BE2 fill:#e8f5e8
    style PG fill:#fff8e1
    style TLS_SECRET fill:#ffebee
    style APP_SECRET fill:#ffebee
```

## Application Flow Architecture

```mermaid
sequenceDiagram
    participant Client
    participant Ingress
    participant Frontend
    participant Backend1
    participant Backend2
    participant Database
    
    Client->>Ingress: HTTPS Request (myapp.blog.com)
    Note over Ingress: TLS Termination<br/>Route based on path
    
    alt Frontend Request (/)
        Ingress->>Frontend: Forward to frontend:80
        Frontend->>Client: Serve React App
    end
    
    alt User API (/api/users)
        Ingress->>Backend1: Forward to backend1:3000
        Backend1->>Database: Query user data
        Database->>Backend1: Return user data
        Backend1->>Client: JSON Response
    end
    
    alt Blog/Comments API (/api/blogs, /api/comments)
        Ingress->>Backend2: Forward to backend2:3001
        Backend2->>Database: Query blog/comment data
        Database->>Backend2: Return blog/comment data
        Backend2->>Client: JSON Response
    end
```

## Helm Chart Structure

```mermaid
graph TB
    subgraph "Helm Chart: three-tier-blog-app"
        CHART["Chart.yaml<br/>Chart Metadata"]
        VALUES["values.yaml<br/>Configuration Values"]
        
        subgraph "templates/"
            FE_DEPLOY["frontend-deployment.yaml<br/>Frontend Deployment"]
            BE1_DEPLOY["backend1-deployment.yaml<br/>Backend1 Deployment"]
            BE2_DEPLOY["backend2-deployment.yaml<br/>Backend2 Deployment"]
            PG_DEPLOY["postgres-statefulset.yaml<br/>Database StatefulSet"]
            
            FE_SVC_T["service-frontend.yaml<br/>Frontend Service"]
            BE1_SVC_T["service-backend1.yaml<br/>Backend1 Service"]
            BE2_SVC_T["service-backend2.yaml<br/>Backend2 Service"]
            PG_SVC_T["service-postgres.yaml<br/>Database Service"]
            
            INGRESS_T["ingress.yaml<br/>Ingress Configuration"]
            PV_T["persistent-volume.yaml<br/>Storage Configuration"]
            SECRET_T["secrets.yaml<br/>Application Secrets"]
        end
    end
    
    subgraph "GitHub Pages Hosting"
        REPO["GitHub Repository<br/>shrijanabudhathoki/three-tier-blog-app"]
        PAGES["GitHub Pages<br/>https://shrijanabudhathoki.github.io/three-tier-blog-app/"]
        INDEX["index.yaml<br/>Helm Repository Index"]
        PACKAGE["Chart Package<br/>three-tier-blog-app-0.1.0.tgz"]
    end
    
    CHART --> REPO
    VALUES --> REPO
    FE_DEPLOY --> REPO
    BE1_DEPLOY --> REPO
    BE2_DEPLOY --> REPO
    PG_DEPLOY --> REPO
    FE_SVC_T --> REPO
    BE1_SVC_T --> REPO
    BE2_SVC_T --> REPO
    PG_SVC_T --> REPO
    INGRESS_T --> REPO
    PV_T --> REPO
    SECRET_T --> REPO
    
    REPO --> PAGES
    PAGES --> INDEX
    PAGES --> PACKAGE
    
    style CHART fill:#e3f2fd
    style VALUES fill:#e8f5e8
    style REPO fill:#fff3e0
    style PAGES fill:#f3e5f5
```

## Network Flow and Routing

```mermaid
graph LR
    subgraph "External"
        USER["User Browser"]
        DOMAIN["myapp.blog.com"]
    end
    
    subgraph "Load Balancing"
        METALLB_IP["MetalLB<br/>192.168.14.240"]
    end
    
    subgraph "Ingress Layer"
        NGINX_IC["NGINX Ingress<br/>Controller"]
        
        subgraph "Routing Rules"
            ROUTE1["/api/users → backend1:3000"]
            ROUTE2["/api/blogs → backend2:3001"]
            ROUTE3["/api/comments → backend2:3001"]
            ROUTE4["/ → frontend:80"]
        end
    end
    
    subgraph "Application Services"
        FE_APP["Frontend Service<br/>ClusterIP"]
        BE1_APP["Backend1 Service<br/>ClusterIP"]
        BE2_APP["Backend2 Service<br/>ClusterIP"]
    end
    
    subgraph "Database Layer"
        DB_SVC["PostgreSQL Service<br/>Headless Service"]
        DB_POD["PostgreSQL Pod<br/>StatefulSet"]
        STORAGE["Persistent Volume<br/>5Gi hostPath"]
    end
    
    USER --> DOMAIN
    DOMAIN --> METALLB_IP
    METALLB_IP --> NGINX_IC
    
    NGINX_IC --> ROUTE1
    NGINX_IC --> ROUTE2
    NGINX_IC --> ROUTE3
    NGINX_IC --> ROUTE4
    
    ROUTE1 --> BE1_APP
    ROUTE2 --> BE2_APP
    ROUTE3 --> BE2_APP
    ROUTE4 --> FE_APP
    
    BE1_APP --> DB_SVC
    BE2_APP --> DB_SVC
    DB_SVC --> DB_POD
    DB_POD --> STORAGE
    
    style USER fill:#e8f5e8
    style METALLB_IP fill:#fff3e0
    style NGINX_IC fill:#e3f2fd
    style DB_POD fill:#fff8e1
    style STORAGE fill:#ffebee
```

## Security Architecture

```mermaid
graph TB
    subgraph "TLS/SSL Security"
        CERT["Self-Signed Certificate<br/>CN: myapp.blog.com<br/>Validity: 365 days"]
        TLS_SECRET_SEC["Kubernetes TLS Secret<br/>myapp-tls-secret"]
        INGRESS_TLS["Ingress TLS Configuration<br/>HTTPS Termination"]
    end
    
    subgraph "Application Security"
        CLERK_KEYS["Clerk Authentication<br/>API Keys"]
        K8S_SECRET["Kubernetes Secret<br/>backend-secret<br/>Base64 Encoded"]
        ENV_VARS["Environment Variables<br/>in Backend Pods"]
    end
    
    subgraph "Network Security"
        CLUSTER_IP["ClusterIP Services<br/>Internal Communication"]
        NETWORK_POLICIES["Network Isolation<br/>Pod-to-Pod Communication"]
        SELINUX_OFF["SELinux: Permissive<br/>Firewall: Disabled"]
    end
    
    CERT --> TLS_SECRET_SEC
    TLS_SECRET_SEC --> INGRESS_TLS
    
    CLERK_KEYS --> K8S_SECRET
    K8S_SECRET --> ENV_VARS
    
    style CERT fill:#ffebee
    style TLS_SECRET_SEC fill:#ffebee
    style K8S_SECRET fill:#ffebee
    style CLUSTER_IP fill:#e8f5e8
```

## Component Details

### Infrastructure Components
- **VMware VMs**: 3 VMs (1 master, 2 workers) running CentOS 9 Stream
- **Container Runtime**: containerd with systemd cgroup driver
- **CNI Plugin**: Cilium for pod networking
- **Load Balancer**: MetalLB for bare-metal LoadBalancer services

### Kubernetes Components
- **Ingress Controller**: NGINX Ingress Controller with TLS termination
- **Storage**: Local hostPath persistent volumes
- **Secrets Management**: Kubernetes secrets for TLS certificates and API keys
- **Service Discovery**: ClusterIP and Headless services

### Application Components
- **Frontend**: React application served by Nginx (port 80)
- **Backend1**: User management service (port 3000)
- **Backend2**: Blog and comments service (port 3001)
- **Database**: PostgreSQL 15 with persistent storage

### Deployment Management
- **Package Manager**: Helm 3 for application deployment
- **Chart Repository**: GitHub Pages hosting Helm charts
- **CI/CD**: Manual deployment with Helm upgrade capabilities

## Access Information
- **Application URL**: https://myapp.blog.com
- **Ingress IP**: 192.168.14.240 (assigned by MetalLB)
- **Helm Repository**: https://shrijanabudhathoki.github.io/three-tier-blog-app/

## Key Features
- **High Availability**: Multi-node cluster with load balancing
- **Security**: HTTPS with self-signed certificates, secret management
- **Scalability**: Kubernetes deployments with configurable replicas
- **Persistence**: StatefulSet for database with persistent storage
- **Package Management**: Helm charts for easy deployment and updates