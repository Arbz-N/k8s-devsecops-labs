# AppArmor Security Profiles on Kubernetes

    This is a hands-on project that demonstrates AppArmor security profiles on Kubernetes. 
    It covers writing custom profiles, loading them into the kernel, applying them to pods via annotations (legacy) and securityContext (v1.30+),
    and validating enforcement with live tests inside containers.
    Key highlights:

    AppArmor deny-write profile blocks all file writes inside a pod
    deny-network profile blocks all outbound network access
    readonly profile allows only reads — no writes, no execution, no network
    Complain mode used for safe profile development before enforcing
    Before/after test validation with touch and echo inside pods

Project Structure
 
    AppArmor-Security-Profiles-on-Kubernetes/
    │
    ├── apparmor-profiles/
    │   ├── k8s-apparmor-example-deny-write   # Deny all file writes
    │   ├── k8s-deny-network                  # Deny all network access
    │   ├── k8s-readonly                      # Read-only + no network
    │   └── k8s-complain-test                 # Complain mode (dev/test)
    │
    ├── k8s/
    │   ├── pod.yaml                          # Pod without AppArmor (baseline)
    │   ├── pod2.yaml                         # Pod with deny-write (annotation method)
    │   ├── pod-network-deny.yaml             # Pod with deny-network
    │   ├── pod-complain.yaml                 # Pod with complain mode profile
    │   └── pod-new-api.yaml                  # Pod with securityContext (v1.30+)
    │
    └── README.md

Prerequisites
 
Requirement                      Detail
Ubuntu/Debian                    AppArmor is enabled by default on Ubuntu
kubectl                          Connected to cluster

