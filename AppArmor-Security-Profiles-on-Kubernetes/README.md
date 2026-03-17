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


    # Verify AppArmor is active
    sudo systemctl status apparmor
    # Active: active (running) 
    
    cat /sys/module/apparmor/parameters/enabled
    # Y 
    
    sudo aa-status
    # apparmor module is loaded. 

Architecture

        Kubernetes Node (Ubuntu)
          ┌──────────────────────────────────────────────────────────┐
          │                                                          │
          │  AppArmor Kernel Module                                  │
          │  /etc/apparmor.d/                                        │
          │  ├── k8s-apparmor-example-deny-write  (enforce)          │
          │  ├── k8s-deny-network                (enforce)           │
          │  ├── k8s-readonly                    (enforce)           │
          │  └── k8s-complain-test               (complain)          │
          │                          │                               │
          │                          │ apparmor_parser (load)        │
          │                          ▼                               │
          │  Pod: hello-apparmor                                     │
          │  ┌──────────────────────────────────────────────────┐    │
          │  │  annotation: localhost/k8s-apparmor-deny-write   │    │
          │  │  container: busybox                              │    │
          │  │                                                  │    │
          │  │  touch test    → Permission denied               │    │
          │  │  echo x > f    → Permission denied               │    │
          │  │  cat /etc/hostname → hello-apparmor              │    │
          │  └──────────────────────────────────────────────────┘    │
          └──────────────────────────────────────────────────────────┘

Setup — Install AppArmor Tools

    sudo apt-get update
    sudo apt-get install -y \
      apparmor \
      apparmor-utils \
      apparmor-profiles
    
    aa-status
    # apparmor module is loaded 
    
    ls /etc/apparmor.d/
    # abstractions/  tunables/  usr.bin.man  usr.sbin.rsyslogd ...

Task 1 — Load AppArmor Profiles

    sudo apparmor_parser \
      /etc/apparmor.d/k8s-apparmor-example-deny-write
    sudo aa-status | grep "deny-write"
    # k8s-apparmor-example-deny-write  ← enforce mode 

    #deny-network Profile
    sudo apparmor_parser /etc/apparmor.d/k8s-deny-network
    sudo aa-status | grep "deny-network"

    # k8s-deny-network
    readonly Profile
    sudo apparmor_parser /etc/apparmor.d/k8s-readonly

Task 2 — Pod Without AppArmor (Baseline)

    kubectl apply -f k8s/pod.yaml
    kubectl get pod
    # hello-apparmor   1/1   Running 
    
    kubectl exec -it hello-apparmor -- sh
    
    echo '12345' > test.txt
    cat test.txt
    # 12345   ← write allowed (no AppArmor)
    
    exit

Task 3 — Pod With deny-write Profile

    kubectl delete pod hello-apparmor
    
    kubectl apply -f k8s/pod2.yaml
    kubectl get pod
    # hello-apparmor   1/1   Running 
    
    kubectl describe pod hello-apparmor | grep -A5 "Annotations"
    # container.apparmor.security.beta.kubernetes.io/hello:
    # localhost/k8s-apparmor-example-deny-write 



 


    

