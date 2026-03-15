# 🚦 Kubernetes Ingress Routing Lab

> A hands-on lab demonstrating production-grade traffic routing strategies using the NGINX Ingress Controller on Kubernetes.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Lab Environment](#lab-environment)
- [Architecture](#architecture)
- [Setup](#setup)
- [Tasks](#tasks)
  - [Task 1 — Path-based Routing](#task-1--path-based-routing)
  - [Task 2 — Add Third Route (/admin)](#task-2--add-third-route-admin)
  - [Task 3 — Host-based Routing](#task-3--host-based-routing)
  - [Task 4 — PathType (Prefix vs Exact)](#task-4--pathtype-prefix-vs-exact)
  - [Task 5 — Default Backend](#task-5--default-backend)
- [Key Concepts](#key-concepts)
- [Verification Commands](#verification-commands)
- [Conclusion](#conclusion)

---

## Overview

This lab demonstrates how to configure and test **Kubernetes Ingress** using the **NGINX Ingress Controller**.

It covers multiple routing strategies commonly used in production environments:

| Strategy | Description |
|---|---|
| **Path-based Routing** | Routes traffic based on the URL path |
| **Host-based Routing** | Routes traffic based on the request hostname |
| **PathType** | Controls matching behavior (Prefix vs Exact) |
| **Default Backend** | Catch-all fallback for unmatched requests |

---

## Lab Environment

| Component | Value |
|---|---|
| **Kubernetes Cluster** | Minikube |
| **Ingress Controller** | NGINX Ingress |
| **Service Type** | ClusterIP |
| **Test Domain** | `myapp.local` |

---

## Architecture

The lab simulates a simple application composed of **three backend services**, all traffic routed through a single Ingress Controller:

```
                        ┌─────────────────────────┐
   Internet Traffic ──► │   NGINX Ingress Controller │
                        └────────────┬────────────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  │                  │                   │
            ┌─────▼─────┐     ┌──────▼─────┐     ┌──────▼──────┐
            │ webapp-svc │     │  api-svc   │     │  admin-svc  │
            └───────────┘     └────────────┘     └─────────────┘
```

---

## Setup

### 1. Enable Ingress Controller

```bash
minikube addons enable ingress
```

Verify the controller is running:

```bash
kubectl get pods -n ingress-nginx
```

**Expected output:**

```
ingress-nginx-controller-xxxxx   Running
```

### 2. Verify IngressClass

```bash
kubectl get ingressclass
```

**Expected output:**

```
NAME    CONTROLLER
nginx   k8s.io/ingress-nginx
```

### 3. Deploy Backend Applications

```bash
kubectl apply -f 01-deployments-and-services.yaml
```

Verify deployments and services:

```bash
kubectl get deploy,svc
```

**Expected services:** `webapp-svc` · `api-svc` · `admin-svc`

---

## Tasks

### Task 1 — Path-based Routing

**File:** `02-ingress-basic-path.yaml`

| Path | Service |
|---|---|
| `/` | `webapp-svc` |
| `/api` | `api-svc` |

**Apply:**

```bash
kubectl apply -f 02-ingress-basic-path.yaml
```

**Get Ingress address:**

```bash
kubectl get ingress
```

**Test:**

```bash
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/api
```

---

### Task 2 — Add Third Route (/admin)

**File:** `06-ingress-full-lab.yaml`

**Delete previous ingress:**

```bash
kubectl delete ingress basic-ingress
```

**Apply new ingress:**

```bash
kubectl apply -f 06-ingress-full-lab.yaml
```

| Path | Service |
|---|---|
| `/` | `webapp-svc` |
| `/api` | `api-svc` |
| `/admin` | `admin-svc` |

**Test all routes:**

```bash
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/api
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/admin
```

**Test unknown path:**

```bash
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/random
```

> ✅ **Expected:** `404 Not Found`

---

### Task 3 — Host-based Routing

**File:** `03-ingress-host-based.yaml`

**Delete previous ingress:**

```bash
kubectl delete ingress lab-ingress
```

**Apply:**

```bash
kubectl apply -f 03-ingress-host-based.yaml
```

| Host | Service |
|---|---|
| `myapp.local` | `webapp-svc` |
| `api.myapp.local` | `api-svc` |
| `admin.myapp.local` | `admin-svc` |

**Test all hosts:**

```bash
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local
curl --resolve "api.myapp.local:80:<ADDRESS>" http://api.myapp.local
curl --resolve "admin.myapp.local:80:<ADDRESS>" http://admin.myapp.local
```

**Test unknown host:**

```bash
curl --resolve "other.myapp.local:80:<ADDRESS>" http://other.myapp.local
```

> ✅ **Expected:** `404 Not Found`

---

### Task 4 — PathType (Prefix vs Exact)

**File:** `05-ingress-path-types.yaml`

**Delete previous ingress:**

```bash
kubectl delete ingress host-ingress
```

**Apply:**

```bash
kubectl apply -f 05-ingress-path-types.yaml
```

| Path | PathType | Service |
|---|---|---|
| `/api` | `Prefix` | `api-svc` |
| `/admin` | `Exact` | `admin-svc` |
| `/` | `Prefix` | `webapp-svc` |

**Prefix test** — matches all subpaths:

```bash
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/api/users
```

> ✅ **Expected:** Works — `/api/users` matches the `/api` Prefix rule

**Exact test** — matches only the exact path:

```bash
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/admin
```

> ✅ **Expected:** Works

**Exact fail test** — subpath does NOT match Exact:

```bash
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/admin/settings
```

> ❌ **Expected:** `404 Not Found` — `/admin/settings` does NOT match the Exact `/admin` rule

---

### Task 5 — Default Backend

**File:** `04-ingress-default-backend.yaml`

**Delete previous ingress:**

```bash
kubectl delete ingress pathtype-demo
```

**Apply:**

```bash
kubectl apply -f 04-ingress-default-backend.yaml
```

| Path | Service |
|---|---|
| `/api` | `api-svc` |
| `/admin` | `admin-svc` |
| *(default)* | `webapp-svc` |

**Test:**

```bash
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/api
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/admin
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/anything
```

| Request | Result |
|---|---|
| `/api` | → `api-svc` |
| `/admin` | → `admin-svc` |
| `/anything` | → `webapp-svc` *(default backend)* |

---

## Key Concepts

### Path-based Routing
Routes traffic based on the **URL path** within a single domain.

```
myapp.local/api    →  api-svc
myapp.local/admin  →  admin-svc
```

### Host-based Routing
Routes traffic based on the **request hostname**, allowing multiple virtual hosts on one IP.

```
api.myapp.local    →  api-svc
admin.myapp.local  →  admin-svc
```

### PathType

| Type | Behavior |
|---|---|
| `Prefix` | Matches the path and all its subpaths (e.g., `/api` matches `/api/users`) |
| `Exact` | Matches only the exact path (e.g., `/admin` does NOT match `/admin/settings`) |

### Default Backend
A **fallback service** that handles all requests that don't match any defined rule.

Common real-world uses:
- Custom `404` pages
- Landing / catch-all pages
- Traffic logging & auditing

---

## Verification Commands

```bash
# List all Ingress resources
kubectl get ingress

# Describe Ingress for detailed routing rules
kubectl describe ingress

# Check Ingress Controller pod status
kubectl get pods -n ingress-nginx
```

---

## Conclusion

This lab demonstrates how **Kubernetes Ingress** can replace multiple external load balancers and simplify traffic management using a single controller with:

- ✅ **Path-based routing** — route by URL path
- ✅ **Host-based routing** — route by hostname
- ✅ **Flexible path matching** — Prefix vs Exact control
- ✅ **Default fallback routing** — catch-all backend

These techniques are widely used in **production Kubernetes environments** to manage application traffic efficiently and reduce infrastructure complexity.

---

<div align="center">

**Lab completed successfully ✅**

*Kubernetes · NGINX Ingress · Minikube*

</div>
