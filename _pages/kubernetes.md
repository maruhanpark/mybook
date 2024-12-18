---
title: Kubernetes
author: Kirill Kuklin
date: 2024-12-15
category: k8s
layout: post
cover: ../assets/k8s-probes.gif
---

# Readiness Probes

## What Are Readiness Probes and Why Do You Need Them? 🚀

When you deploy an application in Kubernetes, you want it to run smoothly without any hiccups. But sometimes, getting 
an application up and running isn't as simple as "just start it and go." Your app might need time to initialize a 
database, load dependencies, or simply ensure that it's ready to handle requests. That's where *readiness probes* 
come into play — small but incredibly useful helpers that let Kubernetes know: "Hey, is this pod actually ready?"

## Why Are Readiness Probes Important?

Here's the scenario: your application starts, but it's not ready to process requests just yet. Without a readiness 
probe, Kubernetes might immediately begin sending traffic to it. The result? Errors, unhappy users, and you scrambling 
to figure out what went wrong.

A readiness probe, however, checks your app's state and tells Kubernetes: "Hold on, not ready yet. Give me a few 
seconds to settle." Only when your application signals that it's ready does Kubernetes start routing traffic to it.

## A Simple Example 🧪

Let's say you have an application that needs to check its database connection before it can handle requests. In your pod's manifest, you'd configure a readiness probe like this:

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 3
```

### What's Happening:
- Kubernetes sends HTTP requests to http://your_pod:8080/healthz to check if the application is ready.
- `initialDelaySeconds` is the time Kubernetes waits after the pod starts before it begins checking.
- `periodSeconds` is how often Kubernetes will perform the check.

If your app returns a 200 OK response, Kubernetes knows it's ready to receive traffic.

## When Does This Save the Day?

1. **Microservices**: If one service depends on another, it's crucial to ensure traffic only flows to ready instances. For example, your API service won't work if the authentication service hasn't finished initializing.

2. **Heavy Applications**: Some apps (like Java monoliths) take a while to warm up. Readiness probes give them the breathing room they need to start properly.

3. **Deployment Chaos**: During rolling updates, readiness probes prevent a half-initialized pod from taking traffic, which could otherwise break the entire system.

## Beyond HTTP: Other Readiness Probe Types

Readiness probes aren't limited to just HTTP checks. Here are a few other ways they can verify if a pod is ready:

### TCP Socket

Checks if a port is open and accepting connections.

```yaml
readinessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 10
  periodSeconds: 5
```

### Command Execution

Runs a custom script or command.

```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/ready
  initialDelaySeconds: 5
  periodSeconds: 3
```

In this example, the pod is considered ready if the file `/tmp/ready` exists.

# Kubernetes configuration linting tools

## Introduction to Infrastructure as Code Validation

One of the benefits of Infrastructure as Code (IaC) is that the proposed desired state specification can be validated before it is applied to the live system. The same is true for Kubernetes configuration.

Checking the proposed desired state is generally easier with Kubernetes configuration tools than with other Infrastructure as Code tools because the configuration can be fully rendered to the representation that will be applied to the Kubernetes API, which makes specified properties explicit. Additionally, the `kubectl` flag `--dry-run=server` can be used to discover values set by default and values set by mutating admission control.

## Admission Control Validation

Admission control can be used to vet Kubernetes resources with tools like:
- Kyverno (5800 GitHub stars)
- Polaris (3200 stars)
- OPA Gatekeeper (3700 stars)

These tools generally have ways to be applied outside the cluster as well. Kubernetes CEL-based validating admission policies are still relatively new, so it remains to be seen how many users can meet all of their policy needs with that mechanism. However, Kyverno has already adapted by generating validating admission policies.

## Live Cluster State Scanning Tools

Some tools scan the live states of clusters, such as:
- Popeye (5300 GitHub stars)
- Clusterlint (500 stars)

Note: These are not the primary focus of this document.

## Schema Validation

### Kubeconform
The most recommended tool was kubeconform (2300 GitHub stars), which performs resource schema validation using the OpenAPI specs published by Kubernetes. It:
- Checks resource types and attributes
- Catches typos and YAML indentation errors
- Helpful in the age of LLM hallucinations

Example of a simple schema validation process would involve running kubeconform on your Kubernetes configuration files.

### Custom Policies

#### Kyverno Policy Example
Here's an example of a Kyverno policy to prevent workload controllers from running in the default namespace:

```yaml
spec:
  validationFailureAction: audit
  background: true
  rules:
  - name: validate-podcontroller-namespace
    match:
      any:
      - resources:
          kinds:
          - DaemonSet
          - Deployment
          - Job
          - StatefulSet
    validate:
      message: "Using 'default' namespace is not allowed for pod controllers."
      pattern:
        metadata:
          namespace: "!default"
```

## Popular Custom Policy Tools

1. **Kyverno** (Kubernetes-specific)
2. **Polaris** (Kubernetes-specific)
3. **Checkov** (7100 GitHub stars, not Kubernetes-specific but has many K8s checks)

## Additional Policy-as-Code Tools

- Trivy (23700 GitHub stars)
- Conftest (2900 GitHub stars)
- OpenRewrite (2200 GitHub stars)

### Rego Policy Example
An example of a Rego policy for service validation:

```rego
deny[explanation] {
 input.request.kind.kind == "Service"
 input.request.operation == "CREATE"
 loadbalancer.is_external_lb
 namespace := input.request.namespace
 name := input.request.object.metadata.name
 not whitelisted[{"name": name, "namespace": namespace}]
 explanation = sprintf("Service %v/%v is an external load balancer but has not been whitelisted", [namespace, name])
}
```

## Best Practices Tools

- KubeLinter (3000 GitHub stars)
- kube-score (2800 stars)

### Example KubeLinter Output
```
pod.yaml: (object: <no namespace>/security-context-demo /v1, Kind=Pod) container "sec-ctx-demo" does not have a 
read-only root file system (check: no-read-only-root-fs, remediation: Set readOnlyRootFilesystem to true in your 
container's securityContext.)
```

## Dashboards

- Polaris Dashboard
- Kubescape Lens Extension (10200 GitHub stars)
- Monokle (1800 stars)
- Kubevious (1600 stars)

## Special Purpose Tools

- Pluto: Check for deprecated API versions
- istioctl: Validate Istio configuration
- helm lint: Lint Helm charts
- argo lint: Validate Argo workflows
