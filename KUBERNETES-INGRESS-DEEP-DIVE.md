# How Kubernetes Ingress Actually Works

> *A production engineer's honest breakdown — from the pain that created Ingress, to the two-layer AWS architecture your team will eventually land on anyway. Better to learn it now.*

---

Let me paint you a picture.

You've just finished the migration. The monolith is dead. You've got 10 shiny microservices — frontend, auth, billing, notifications, user profile, the works — all running beautifully in Kubernetes. You expose each one with a `Service` of type `LoadBalancer` because that's the obvious move. Traffic flows. You demo it to the team. Everyone's happy.

Then the AWS bill arrives.

Ten load balancers. Ten public IPs. Ten separate line items. Your manager forwards the invoice with zero additional commentary, which somehow says everything.

This is not a hypothetical. This exact scenario — repeated across hundreds of engineering teams — is what pushed the Kubernetes community to design Ingress and the Ingress Controller model. Once you understand the *why*, the *how* stops feeling like magic and starts feeling like common sense.

Let's walk through this properly.

---

## Before Ingress: How Traffic Actually Flowed (And Why It Hurt)

To appreciate the solution, you need to sit with the problem for a moment.

In a standard Kubernetes cluster on EC2, your infrastructure splits into two planes:

```
+----------------------------------------------------------+
|                      PUBLIC SUBNET                       |
|                                                          |
|   [ Control Plane / Master Node ]                        |
|     - API Server                                         |
|     - etcd  (cluster state store)                        |
|     - Scheduler, Controller Manager                      |
|     - Security Group: ONLY ports 80, 443 open            |
+----------------------------+-----------------------------+
                             |
                      internal proxy
                      127.0.0.1:NodePort
                             |
+----------------------------v-----------------------------+
|                     PRIVATE SUBNET                       |
|                                                          |
|   [ Data Plane / Worker Nodes ]                          |
|     - Your application Pods                              |
|     - NodePort Services                                  |
|     - NOT directly reachable from the internet           |
+----------------------------------------------------------+
```

The master node lives in a public subnet with a security group locked to ports 80 and 443. Traffic can be internally proxied from master to worker via `127.0.0.1:<NodePort>`, but you cannot redirect requests straight from the internet to a worker node IP and port combo. In dev environments, this is fine. In production with multiple services, you hit three walls fast.

---

## The Three Pain Points That Made Ingress Inevitable

### Pain Point #1 — The Cloud Bill Problem

Every `Service` of type `LoadBalancer` sends a provisioning request to your cloud provider. AWS is delighted to spin one up. AWS is equally delighted to charge you for it, every single month, per service.

```
WITHOUT INGRESS
===============

  Internet
    |
    +---> [ALB #1  $$$] ---> frontend-service
    +---> [ALB #2  $$$] ---> auth-service
    +---> [ALB #3  $$$] ---> billing-service
    +---> [ALB #4  $$$] ---> notifications-service
    +---> [ALB #5  $$$] ---> user-profile-service

  5 services = 5 Load Balancers = 5 bills


WITH INGRESS
============

  Internet
    |
    +--> [ALB #1] --> Ingress Controller
                           |
                           +---> /          --> frontend-service
                           +---> /auth      --> auth-service
                           +---> /billing   --> billing-service
                           +---> /notify    --> notifications-service
                           +---> /profile   --> user-profile-service

  5 services = 1 Load Balancer = 1 bill
```

At scale, this goes from a minor annoyance to a budget conversation you really don't want to have.

---

### Pain Point #2 — Layer 4 vs Layer 7: Why Standard Load Balancers Are Blind

Standard Kubernetes `LoadBalancer` services operate at **Layer 4** of the OSI model — the transport layer. They understand IP addresses and TCP ports. That's the full extent of their world.

```
OSI LAYER COMPARISON
====================

  Layer 7  (Application)   <-- Ingress sees THIS
  ----------------------------------------
    HTTP method:   GET, POST, PUT
    Domain:        api.company.com
    URL path:      /v2/users/profile
    Headers:       Authorization, Content-Type
    Cookies:       session_id=abc123


  Layer 4  (Transport)     <-- Classic LoadBalancer sees this
  ----------------------------------------
    IP address:    192.168.1.100
    Port:          80, 443, 8080
    Protocol:      TCP, UDP
```

A Layer 4 load balancer cannot tell the difference between a request for `example.com/api/users` and `example.com/dashboard`. To it, both are just TCP connections on port 80. You'd need a separate listener — and a separate load balancer — for each.

Ingress gives you Layer 7 awareness. That means proper routing rules:

**Host-based routing:**

| Incoming Host          | Routes To               |
|------------------------|-------------------------|
| `app.company.com`      | `frontend-service:80`   |
| `api.company.com`      | `api-service:8080`      |
| `admin.company.com`    | `admin-service:3000`    |

**Path-based routing:**

| Incoming Path   | Routes To               |
|-----------------|-------------------------|
| `/`             | `frontend-service:80`   |
| `/api/v1/`      | `api-service:8080`      |
| `/auth/`        | `auth-service:5000`     |
| `/uploads/`     | `storage-service:9000`  |

This is the kind of routing intelligence a plain load balancer simply cannot provide — not without a lot of pain and money.

---

### Pain Point #3 — Security Was a Distributed Nightmare

Without a centralized ingress point, every service team owned their own TLS certificates, rate limiting configuration, and IP allowlists. When a wildcard cert expired — or a new org-wide security policy came in — you were hunting through dozens of service configs to patch everything.

Ingress gives you one front door:

```
INGRESS AS A UNIFIED SECURITY CHECKPOINT
=========================================

  Incoming HTTPS Request
         |
         v
  [ TLS Termination  ] <-- One certificate, managed here
         |
         v
  [ Rate Limiting    ] <-- One rule, applies to all services
         |
         v
  [ IP Allowlist     ] <-- Block bad actors at the cluster edge
         |
         v
  [ Route to Service ] -- Plain HTTP internally (already inside the cluster)
```

One certificate. One place to rotate it. One security perimeter for the entire cluster.

---

## Ingress vs. Ingress Controller: Let's Clear This Up

This is the part most tutorials breeze past, and it causes real confusion.

**The analogy that actually works:** think of a restaurant.

- **The Ingress resource** is the *seating chart and menu*. It's a YAML file you apply to Kubernetes that declares routing intent: *"requests to `/api` go to the `api-service`."* It's pure configuration. It sits in etcd. By itself, it does absolutely nothing to live traffic.

- **The Ingress Controller** is the *maitre d' who's actually standing at the door*. It's a running pod (or set of pods) that continuously watches the Kubernetes API for Ingress resources and actively enforces those rules against real, live HTTP traffic.

```
HOW INGRESS AND INGRESS CONTROLLER RELATE
==========================================

  You write and apply:
  +----------------------------------------------+
  | kind: Ingress                                |  <-- YAML config only
  | rules:                                       |      stored in etcd
  |   - path: /api  --> api-service:8080         |
  |   - path: /     --> frontend-service:80      |
  +---------------------+------------------------+
                        |
                  watches for changes
                        |
                        v
  +----------------------------------------------+
  | Ingress Controller Pod(s)                    |  <-- Actually handles
  | (NGINX / AWS ALB / Traefik / Istio / etc.)   |      live traffic
  |                                              |
  | - Reads Ingress resources from API Server    |
  | - Reconfigures its internal proxy engine     |
  | - Accepts and routes real HTTP/HTTPS traffic |
  +----------------------------------------------+
```

**The part nobody tells you upfront:** Kubernetes does not ship with an Ingress Controller. It never has. Kubernetes defined the Ingress API spec — the contract — and told the ecosystem: *build your own controller that honors it.* That's why you have NGINX, AWS ALB Controller, Traefik, HAProxy, Istio, and a dozen others. Same spec. Different engines.

---

## Pattern 1: One Ingress Class, One ALB (EKS + Fargate)

Let's get into a real AWS production scenario.

When you're running **EKS with Fargate**, both your control plane and data plane live inside a private VPC. Your pods have no public IP. Nothing is directly reachable from the internet. You need a cloud-native entry point.

The solution is the **AWS Load Balancer Controller** (previously called the ALB Ingress Controller). You install it into your cluster. It watches for any `Ingress` resources annotated with `ingressClassName: alb` and automatically provisions an **Application Load Balancer** in AWS on your behalf — no manual ALB configuration needed.

Here's the traffic path:

```
PATTERN 1 - AWS ALB + ALB CONTROLLER
======================================

  Internet
     |
     v
  [ AWS Application Load Balancer ]
     |
     |   AWS LB Controller reads Ingress rules
     |   from etcd via Kubernetes API Server
     |
     v
  [ AWS Load Balancer Controller Pod ]   (kube-system namespace)
     |
     |   target-type: ip = routes directly to pod IPs
     |
     v
  [ Your Application Pods ]
  (frontend-pod, api-pod, auth-pod...)
```

Here's the complete deployment for the classic 2048 demo app — the example you'll find in the AWS docs, and a clean pattern to learn from:

```yaml
---
# Service: Exposes the pods within the cluster
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: app-2048

---
# Ingress: Tells the AWS LB Controller to create an ALB
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-2048
                port:
                  number: 80
```

Two annotations carry most of the weight here:

| Annotation                                      | What It Does                                              |
|-------------------------------------------------|-----------------------------------------------------------|
| `alb.ingress.kubernetes.io/scheme: internet-facing` | Provisions a public-facing ALB (not internal VPC-only)  |
| `alb.ingress.kubernetes.io/target-type: ip`     | Routes ALB traffic directly to pod IPs, skipping NodePort |

That second annotation is worth pausing on. Without `target-type: ip`, your traffic path is `ALB → NodePort → Pod` — an extra hop with extra latency. With it, the ALB forwards directly to the pod's IP address. Fewer hops, lower latency, cleaner architecture.

### Installing the AWS Load Balancer Controller (Full Steps)

```bash
# Add the EKS Helm chart repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install cert-manager (the controller needs it for webhook TLS)
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.13.2/cert-manager.yaml

# Wait for cert-manager to be fully up before proceeding
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=cert-manager \
  -n cert-manager \
  --timeout=300s

# Install the AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster-2 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=vpc-05ef07ee5bbb4c6ec

# Create a Fargate profile for the application namespace
eksctl create fargateprofile \
  --cluster demo-cluster-2 \
  --region ap-south-1 \
  --name fp-game-2048 \
  --namespace game-2048

# Deploy the 2048 app
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

Pattern 1 works well for single-team setups and prototypes. But it has a structural limitation that bites you the moment your organization starts growing.

---

### The Catch That Pattern 1 Eventually Hits

Here's the issue: the AWS Load Balancer Controller operates on a **one ALB per Ingress class** model. The moment you have a second team, a second namespace, or a second `ingressClassName`, you're back to provisioning multiple ALBs.

```
MULTIPLE INGRESS CLASSES = MULTIPLE LOAD BALANCERS
====================================================

  Ingress A  (class: alb)    --> ALB Controller #1 --> ALB #1  ($$$)
  Ingress B  (class: alb-2)  --> ALB Controller #2 --> ALB #2  ($$$)
  Ingress C  (class: alb-3)  --> ALB Controller #3 --> ALB #3  ($$$)

  You're back where you started. 😬
```

This is exactly why mature engineering teams don't stop at Pattern 1.

---

## Pattern 2: The Two-Layer Architecture (The Production Standard)

This is where most experienced Kubernetes teams land, and once you see the logic, it clicks immediately and stays clicked.

**The core idea:** use the AWS Load Balancer Controller to create exactly *one* ALB at the cluster edge. That ALB's sole job is to forward all traffic to a single **NGINX Ingress Controller** pod running inside the cluster. NGINX then takes over all the intelligent routing decisions from that point inward.

You get cloud-native load balancing at the edge *and* the full feature set of NGINX for internal routing — at the cost of exactly one ALB.

Let's walk through the setup step by step, because the controller hand-off is the part that confuses people most.

---

### Step 1 — Deploy Ingress #1 (class: alb)

You create an Ingress resource with `ingressClassName: alb`. The AWS Load Balancer Controller picks this up, provisions one ALB, and configures its target to be the NGINX controller pod's IP — not your application pods. The ALB's only job is to hand traffic off to NGINX.

```yaml
# Ingress #1: Tells AWS "create one ALB, forward everything to NGINX"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alb-to-nginx
  namespace: ingress-nginx
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ingress-nginx-controller   # NGINX's own Kubernetes Service
                port:
                  number: 80
```

At this point, one ALB exists. It knows one thing: send all traffic to the NGINX pod.

---

### Step 2 — Deploy Ingress #2 (class: nginx)

You create a second Ingress resource with `ingressClassName: nginx`. The NGINX controller picks this one up — and only this one. The AWS Load Balancer Controller ignores it entirely, because it's not `class: alb`. NGINX reads the routing rules and loads them into its internal proxy configuration.

```yaml
# Ingress #2: Read by NGINX only — AWS LB Controller ignores this completely
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-routing-rules
  namespace: production
spec:
  ingressClassName: nginx
  rules:
    - host: api.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
    - host: app.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

NGINX now knows: requests for `api.company.com` go to `api-service:8080`. Requests for `app.company.com` go to `frontend-service:80`.

---

### Step 3 — What Happens at Runtime (The Hand-Off)

When a real browser request arrives, this is exactly what happens:

```
RUNTIME REQUEST FLOW
=====================

  Browser  -->  AWS ALB  -->  NGINX pod  -->  api-service  -->  api pod
                   ^                ^
                   |                |
          (Ingress #1 told     (Ingress #2 told
          ALB: "your target    NGINX: "api.company.com
          is the NGINX pod")   goes to api-service")
```

Two Ingress resources. Two different controllers. Each one only reads what belongs to it. One ALB in AWS. All routing logic lives inside the cluster with NGINX.

This is the design elegance of the two-layer pattern: the `ingressClassName` field acts as a selector. Each controller watches only for resources that match its class and ignores everything else. The two systems operate independently — but together they form a complete, cost-efficient routing pipeline.

---

### Full Architecture: What You're Actually Running

Here's the complete picture of the two-layer pattern in an EKS environment:

```
PATTERN 2 - FULL TWO-LAYER ARCHITECTURE (EKS)
===============================================

  Internet
     |
     v
+----------------------------+
|   AWS Application LB       |  <-- 1 ALB, provisioned by AWS LB Controller
|   (internet-facing)        |      from Ingress #1 (class: alb)
+----------------------------+
     |
     |  forwards all traffic to NGINX pod IP
     v
+----------------------------+
|   NGINX Ingress Controller |  <-- Running as pod in ingress-nginx namespace
|   Pod                      |      reads Ingress #2 (class: nginx)
+----------------------------+
     |
     +---> api.company.com    --> api-service:8080      --> api pods
     |
     +---> app.company.com    --> frontend-service:80   --> frontend pods
     |
     +---> admin.company.com  --> admin-service:3000    --> admin pods
     |
     +---> /uploads/*         --> storage-service:9000  --> storage pods
```

One ALB at the boundary. All routing intelligence handled by NGINX inside the cluster. Every new service you add only requires a new NGINX-class Ingress rule — not a new AWS load balancer.

---

### Pattern 1 vs Pattern 2: The Honest Comparison

| Dimension              | Pattern 1 (ALB per Ingress)         | Pattern 2 (ALB + NGINX)             |
|------------------------|--------------------------------------|--------------------------------------|
| **Load Balancers**     | One per Ingress class                | One, total, forever                  |
| **Monthly Cloud Cost** | Grows with teams and services        | Fixed at one ALB                     |
| **Routing Logic**      | Handled at AWS ALB                   | Handled by NGINX inside the cluster  |
| **NGINX Features**     | Not available                        | Full access (rewrites, rate limits, mTLS, canary) |
| **TLS Termination**    | At ALB only                          | At NGINX, ALB, or both               |
| **Multi-Team Support** | Gets expensive quickly               | Scales cleanly — add Ingress rules, not ALBs |
| **Operational Complexity** | Lower initially                  | Slightly higher setup, much lower long-term cost |
| **Best For**           | Prototypes, small projects, single-team | Production, multi-team, multi-namespace orgs |

The short version: Pattern 1 is the quick win. Pattern 2 is where you'll end up once the team grows and finance starts asking questions. Save yourself the migration and design for Pattern 2 from the start if you're heading to production.

---

## The Full Mental Model, Consolidated

If you walk away with one diagram in your head from this post, let it be this:

```
COMPLETE KUBERNETES INGRESS MENTAL MODEL
=========================================

  Step 1:
  You write Ingress YAML
    --> Stored in etcd via Kubernetes API Server

  Step 2:
  Ingress Controller watches the API Server
    --> Only picks up Ingress resources matching its ingressClassName
    --> Translates those rules into live proxy configuration

  Step 3 (AWS-specific):
  AWS LB Controller reads class: alb Ingresses
    --> Provisions an ALB with the right target and scheme annotations

  Step 4:
  Live traffic flows:

  [ Pattern 1 ]
  Internet --> AWS ALB --> Pod  (direct, via target-type: ip)

  [ Pattern 2 ]
  Internet --> AWS ALB --> NGINX Controller --> Service --> Pod
                  ^                ^
          (Ingress #1)       (Ingress #2)
```

The Ingress resource is *intent*. The Ingress Controller is *execution*. The `ingressClassName` is the selector that connects the two. Everything else is just which engine you pick and how many layers you stack.
