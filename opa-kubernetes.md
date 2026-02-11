# OPA Gatekeeper on Kubernetes
## Use Case: Enforcing Trusted Container Image Registries

---

## Overview

This Proof of Concept (POC) demonstrates how to use **OPA Gatekeeper** to enforce **policy-as-code** in Kubernetes by restricting Pods to pull container images **only from approved registries**.

The policy is enforced **at admission time**, meaning insecure workloads are blocked **before they are persisted to etcd**.

---

## Objective

- Prevent Kubernetes Pods from using images from **untrusted container registries**
- Enable platform and DevOps teams to **centrally control allowed images**
- Eliminate reliance on manual reviews or developer discipline

---

## From a Developer’s Perspective

- Immediate and clear feedback when deploying unapproved images
- Human-readable policy violation messages
- Focus on coding while security is enforced automatically

---

## From a Security Team’s Perspective

- Ensure only verified and approved images enter the cluster
- Reduce attack surface from public or unknown images
- Standardize image usage across teams and environments

---

## From a DevOps / Platform Engineer’s Perspective

- Enforce policies centrally without modifying application code
- Avoid repetitive manual checks during deployments
- Maintain consistent security enforcement across namespaces

---

## From an Organization’s Perspective

- Align workloads with compliance and governance policies
- Reduce supply-chain security risks
- Enable scalable governance as teams and services grow

---

## Why This Is Important

- Prevents deployment of insecure or unknown images
- Automatically enforces organizational security standards
- Shifts security **left** (before runtime)
- Demonstrates real-world **DevSecOps practices**

---

## Architecture & Flow

1. Kubernetes API Server sends an AdmissionReview request
2. Gatekeeper intercepts the request
3. Rego policies are evaluated
4. Request is **allowed or denied** based on policy outcome

---

## Step-by-Step Implementation

---

## STEP 1: Start Kubernetes Cluster

```bash
minikube start
```
### Why?

- OPA Gatekeeper works as a Kubernetes admission controller and requires a running cluster.

Verify
```
kubectl get nodes
```

## STEP 2: Install OPA Gatekeeper
```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml
```

Verify
```
kubectl get pods -n gatekeeper-system
kubectl get validatingwebhookconfigurations
```
### Why?
- Gatekeeper integrates OPA with Kubernetes by registering a Validating Admission Webhook.


## STEP 3: Create ConstraintTemplate (Policy Logic)
```
trusted-registry-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8strustedregistry
spec:
  crd:
    spec:
      names:
        kind: K8sTrustedRegistry
      validation:
        openAPIV3Schema:
          type: object
          properties:
            registry:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8strustedregistry

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not startswith(container.image, input.parameters.registry)
          msg := sprintf("Untrusted image used: %s", [container.image])
        }
```
Apply
```
kubectl apply -f trusted-registry-template.yaml
```
Verify
```
kubectl get constrainttemplates
```
### Why?
- Defines the Rego policy logic
- Dynamically creates a custom CRD
- OpenAPI schema ensures strict parameter validation

## STEP 4: Create Constraint (Policy Enforcement)
```
trusted-registry-constraint.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sTrustedRegistry
metadata:
  name: allow-only-approved-registry
spec:
  parameters:
    registry: "gcr.io/approved/"
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```
Apply
```
kubectl apply -f trusted-registry-constraint.yaml
```
Verify
```
kubectl get constraints
```
### Why?
- Instantiates the ConstraintTemplate
- Defines enforcement scope (Pods)
- Supplies registry as a dynamic parameter

## STEP 5: Test - DENY Case
```
invalid-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-invalid
spec:
  containers:
    - name: nginx
      image: nginx:latest
```
Apply
```
kubectl apply -f invalid-pod.yaml
```
Expected Output
```
admission webhook "validation.gatekeeper.sh" denied the request:
Untrusted image used: nginx:latest
```
### Why?
- Confirms that untrusted images are blocked at admission time.



## STEP 6: Test - ALLOW Case
```
valid-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-valid
spec:
  containers:
    - name: nginx
      image: gcr.io/approved/nginx:1.25
```
Apply
```
kubectl apply -f valid-pod.yaml
```
Expected Output
```
pod/nginx-valid created
```
### Why?
- Ensures approved images are deployed successfully without policy violations.
