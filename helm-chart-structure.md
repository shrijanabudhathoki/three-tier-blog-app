# Helm Chart Structure - Three Tier Blog App

## Directory Structure

```
three-tier-blog-app/
├── Chart.yaml                    # Chart metadata
├── values.yaml                   # Default configuration values
├── charts/                       # Chart dependencies (empty for this project)
├── templates/                    # Kubernetes resource templates
│   ├── frontend-deployment.yaml  # Frontend deployment
│   ├── backend1-deployment.yaml  # Backend1 (users) deployment
│   ├── backend2-deployment.yaml  # Backend2 (blogs/comments) deployment
│   ├── postgres-statefulset.yaml # PostgreSQL database
│   ├── service-frontend.yaml     # Frontend service
│   ├── service-backend1.yaml     # Backend1 service
│   ├── service-backend2.yaml     # Backend2 service
│   ├── service-postgres.yaml     # PostgreSQL service
│   ├── persistent-volume.yaml    # Storage configuration
│   ├── secrets.yaml              # Application secrets
│   ├── ingress.yaml              # Ingress configuration
│   └── _helpers.tpl              # Template helpers
└── README.md                     # Chart documentation
```

## Chart.yaml
```yaml
apiVersion: v2
name: three-tier-blog-app
description: A Helm chart for 3-tier blog application with React frontend, Node.js backends, and PostgreSQL database
type: application
version: 0.1.0
appVersion: "1.0.0"
keywords:
  - blog
  - three-tier
  - react
  - nodejs
  - postgresql
home: https://github.com/shrijanabudhathoki/three-tier-blog-app
sources:
  - https://github.com/shrijanabudhathoki/three-tier-blog-app
maintainers:
  - name: Shrijana Budhathoki
    email: shrijana@example.com
```

## values.yaml
```yaml
# Global configuration
global:
  domain: myapp.blog.com
  namespace: default

# Frontend configuration
frontend:
  enabled: true
  replicaCount: 1
  image:
    repository: shrijanab/blog-frontend
    tag: latest
    pullPolicy: Always
  service:
    type: ClusterIP
    port: 80
    targetPort: 80
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "200m"

# Backend1 (Users service) configuration
backend1:
  enabled: true
  replicaCount: 1
  image:
    repository: shrijanab/blog-backend1
    tag: latest
    pullPolicy: Always
  service:
    type: ClusterIP
    port: 3000
    targetPort: 3000
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "512Mi"
      cpu: "400m"

# Backend2 (Blogs/Comments service) configuration
backend2:
  enabled: true
  replicaCount: 1
  image:
    repository: shrijanab/blog-backend2
    tag: latest
    pullPolicy: Always
  service:
    type: ClusterIP
    port: 3001
    targetPort: 3001
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "512Mi"
      cpu: "400m"

# PostgreSQL database configuration
postgresql:
  enabled: true
  replicaCount: 1
  image:
    repository: postgres
    tag: "15"
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 5432
    targetPort: 5432
  auth:
    username: fellowship
    password: fellowship
    database: fellowship
  persistence:
    enabled: true
    size: 5Gi
    storageClass: ""
    hostPath: /mnt/data/postgres
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "1Gi"
      cpu: "500m"

# Ingress configuration
ingress:
  enabled: true
  className: nginx
  host: myapp.blog.com
  tls:
    enabled: true
    secretName: myapp-tls-secret
  paths:
    - path: /api/users
      pathType: Prefix
      service: backend1
      port: 3000
    - path: /api/blogs
      pathType: Prefix
      service: backend2
      port: 3001
    - path: /api/comments
      pathType: Prefix
      service: backend2
      port: 3001
    - path: /
      pathType: Prefix
      service: frontend
      port: 80

# Secrets configuration
secrets:
  clerk:
    publishableKey: "cGtfdGVzdF9jR3hsWVhOcGJtY3RjM1IxY21kbGIyNHRNVEV1WTJ4bGNtc3VZV05qYjNWdWRITXVaR1YwSkE="
    secretKey: "c2tfdGVzdF9OM2ZockdJTXZJaXRvdmJIdFduTXBvdmQ0ZmdBcGZzNVNYbGh2czVrekc="

# TLS configuration
tls:
  enabled: true
  secretName: myapp-tls-secret
  certificate: |
    -----BEGIN CERTIFICATE-----
    # Self-signed certificate content would go here
    -----END CERTIFICATE-----
  privateKey: |
    -----BEGIN PRIVATE KEY-----
    # Private key content would go here
    -----END PRIVATE KEY-----
```

## Template Files Overview

### frontend-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "three-tier-blog-app.fullname" . }}-frontend
  labels:
    {{- include "three-tier-blog-app.labels" . | nindent 4 }}
    component: frontend
spec:
  {{- if not .Values.frontend.autoscaling.enabled }}
  replicas: {{ .Values.frontend.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "three-tier-blog-app.selectorLabels" . | nindent 6 }}
      component: frontend
  template:
    metadata:
      labels:
        {{- include "three-tier-blog-app.selectorLabels" . | nindent 8 }}
        component: frontend
    spec:
      containers:
        - name: frontend
          image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
          imagePullPolicy: {{ .Values.frontend.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.frontend.service.targetPort }}
              protocol: TCP
          resources:
            {{- toYaml .Values.frontend.resources | nindent 12 }}
```

### ingress.yaml
```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "three-tier-blog-app.fullname" . }}-ingress
  labels:
    {{- include "three-tier-blog-app.labels" . | nindent 4 }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls.enabled }}
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Values.ingress.tls.secretName }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
        {{- range .Values.ingress.paths }}
        - path: {{ .path }}
          pathType: {{ .pathType }}
          backend:
            service:
              name: {{ include "three-tier-blog-app.fullname" $ }}-{{ .service }}
              port:
                number: {{ .port }}
        {{- end }}
{{- end }}
```

### postgres-statefulset.yaml
```yaml
{{- if .Values.postgresql.enabled }}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "three-tier-blog-app.fullname" . }}-postgres
  labels:
    {{- include "three-tier-blog-app.labels" . | nindent 4 }}
    component: database
spec:
  serviceName: {{ include "three-tier-blog-app.fullname" . }}-postgres
  replicas: {{ .Values.postgresql.replicaCount }}
  selector:
    matchLabels:
      {{- include "three-tier-blog-app.selectorLabels" . | nindent 6 }}
      component: database
  template:
    metadata:
      labels:
        {{- include "three-tier-blog-app.selectorLabels" . | nindent 8 }}
        component: database
    spec:
      containers:
      - name: postgres
        image: "{{ .Values.postgresql.image.repository }}:{{ .Values.postgresql.image.tag }}"
        imagePullPolicy: {{ .Values.postgresql.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.postgresql.service.targetPort }}
        env:
        - name: POSTGRES_USER
          value: {{ .Values.postgresql.auth.username }}
        - name: POSTGRES_PASSWORD
          value: {{ .Values.postgresql.auth.password }}
        - name: POSTGRES_DB
          value: {{ .Values.postgresql.auth.database }}
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          {{- toYaml .Values.postgresql.resources | nindent 10 }}
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: {{ .Values.postgresql.persistence.size }}
{{- end }}
```

## _helpers.tpl
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "three-tier-blog-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "three-tier-blog-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "three-tier-blog-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "three-tier-blog-app.labels" -}}
helm.sh/chart: {{ include "three-tier-blog-app.chart" . }}
{{ include "three-tier-blog-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "three-tier-blog-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "three-tier-blog-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

## GitHub Pages Repository Setup

### Package and Index Commands
```bash
# Package the Helm chart
helm package .

# Create repository index
helm repo index . --url https://shrijanabudhathoki.github.io/three-tier-blog-app

# Add and install from repository
helm repo add myapp https://shrijanabudhathoki.github.io/three-tier-blog-app/
helm repo update
helm install myapp myapp/three-tier-blog-app
```

### Repository Structure
```
shrijanabudhathoki.github.io/three-tier-blog-app/
├── index.yaml                          # Helm repository index
├── three-tier-blog-app-0.1.0.tgz      # Packaged chart
├── Chart.yaml                          # Chart metadata
├── values.yaml                         # Default values
├── templates/                          # All template files
└── README.md                           # Installation instructions
```

This Helm chart provides a complete deployment solution for the 3-tier blog application with configurable values, proper templating, and GitHub Pages hosting for easy distribution and installation.