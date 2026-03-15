# NGINX Ingress + Cert-Manager TLS on Kind Cluster

    Overview
    This project is a hands-on project that sets up a Kubernetes Ingress Controller on a local Kind cluster 
    and progressively configures traffic routing — starting with plain HTTP, then upgrading to automated TLS using Cert-Manager.
    The entire lab runs locally without any cloud account.
    Key highlights:
    
    Kind cluster with extraPortMappings forwarding ports 80 and 443 to localhost
    NGINX Ingress Controller installed via the official Kind-specific manifest
    Caddy web server used as a lightweight backend pod
    HTTP Ingress created first, then upgraded to HTTPS
    Cert-Manager automatically generates the TLS Secret and auto-renews 30 days before expiry — no manual openssl commands needed
    selfSigned Issuer used for local testing (swap for Let's Encrypt in production)
    Multi-path routing with nginx.ingress.kubernetes.io/rewrite-target annotation

## Project Structure:

    NGINX-Ingress-Cert-Manager-TLS-on-Kind-Cluster/
    │
    ├── kind-ingress.yaml       # Kind cluster config (ports 80/443 + node labels)
    ├── issuer.yaml             # Cert-Manager Issuer (selfSigned for local)
    ├── certificate.yaml        # Certificate resource (domain + secret + duration)
    ├── ingress-http.yaml       # Step 1: Plain HTTP Ingress (no TLS)
    ├── ingress-https.yaml      # Step 2: HTTPS Ingress with Cert-Manager TLS
    ├── ingress-multi.yaml      # Step 3: Multi-path routing example
    │
    └── README.md               # Project documentation

## Prerequisites:

    Requirement              Check
    Kind                     kind --version
    kubectl                  kubectl version --client
    sudo access              /etc/hosts editing + port 80/443 binding

## Architecture:

        Browser / curl
               │
               ├── http://example.com   → port 80  (ingress-http.yaml)
               └── https://example.com  → port 443 (ingress-https.yaml)
               │
               ▼
          Kind extraPortMappings (host → container)
               │
               ▼
          NGINX Ingress Controller (ingress-nginx namespace)
          ┌──────────────────────────────────────────────┐
          │  Ingress: simple-ingress                     │
          │  Host: example.com                           │
          │  TLS:  sec-example ← cert-manager generated  │
          │  Path: / → caddy-svc:80                      │
          └──────────────────────────────────────────────┘
               │
               ▼
          caddy-svc (ClusterIP:80) → caddy pod
        
          Cert-Manager (cert-manager namespace)
          ┌──────────────────────────────────────────────┐
          │  Issuer: selfsigned-issuer (selfSigned)      │
          │  Certificate: example-tls                    │
          │    duration:    90 days                      │
          │    renewBefore: 30 days  ← auto-renew        │
          │    secretName:  sec-example (auto-created)   │
          └──────────────────────────────────────────────┘


## Cluster Setup:

    Create Kind Cluster
    kind create cluster --config kind-ingress.yaml

    kubectl get nodes
    # ingress-lab-control-plane   Ready   control-plane 
    Install NGINX Ingress Controller
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

    kubectl get pods -n ingress-nginx
    # ingress-nginx-controller-xxx   1/1   Running 

### Task 1 — Deploy Backend:
    
    kubectl run caddy --image caddy
    kubectl expose pod caddy --name caddy-svc --port 80
    
    kubectl get pod
    # caddy   1/1   Running 
    
    kubectl describe svc caddy-svc
    # Endpoints: 10.x.x.x:80 

### Task 2 — HTTP Ingress (ingress-http.yaml):

    No TLS — plain HTTP routing only.
    kubectl apply -f ingress-http.yaml
    # ingress.networking.k8s.io/simple-ingress created 
    
    kubectl describe ingress simple-ingress
    # Host: example.com
    # Path: /  →  caddy-svc:80 
    
    # Add to /etc/hosts for local resolution
    echo "127.0.0.1 example.com" | sudo tee -a /etc/hosts
    
    # Test
    curl http://example.com
    # Caddy default page 

### Task 3 — Install Cert-Manager:

    Why Cert-Manager?
    Without Cert-Manager you manually run openssl, create the Secret yourself, and repeat when the cert expires. 
    Cert-Manager handles generation, Secret storage, and auto-renewal — fully automated.

    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
    
    kubectl get pods -n cert-manager
    # cert-manager-xxx            1/1   Running 
    # cert-manager-cainjector-xxx 1/1   Running 
    # cert-manager-webhook-xxx    1/1   Running 

    Create Issuer

    kubectl apply -f issuer.yaml

    kubectl get issuer
    # selfsigned-issuer   True 

    Issuer is namespace-scoped. Use ClusterIssuer to issue certificates across all namespaces.
    For production with a real domain, replace selfSigned: {} with an ACME Let's Encrypt config.

    Create Certificate

    kubectl apply -f certificate.yaml
    
    kubectl get certificate
    # example-tls   True   sec-example 
    
    kubectl get secret sec-example
    # sec-example   kubernetes.io/tls   2   (auto-created by cert-manager)
    
    kubectl describe certificate example-tls
    # Not After:    2026-06-15T...  (90 days)
    # Renewal Time: 2026-05-16T...  (auto-renew 30 days before) 

    
    Task 4 — HTTPS Ingress (ingress-https.yaml)
    Upgrades the Ingress to use TLS. 
    previous ingress-http.yaml is replaced by ingress-https.yaml — same resource name (simple-ingress) so kubectl apply updates it in place.

    kubectl apply -f ingress-https.yaml
    # ingress.networking.k8s.io/simple-ingress configured 
    
    kubectl describe ingress simple-ingress
    # TLS:  sec-example terminates example.com 
    # Path: /  →  caddy-svc:80 
    
    # Test HTTPS (-k because self-signed cert is not browser-trusted)
    curl -k https://example.com
    # Caddy response 
    
    # View certificate details
    curl -kv https://example.com 2>&1 | grep -E "subject|issuer|SSL"
    # subject: CN=example.com 
    # issuer:  CN=example.com (self-signed)
    
    # Check auto-renew status
    kubectl get certificaterequest
    # example-tls-xxxx   True   True 

### Task 5 — Multi-Path Routing (ingress-multi.yaml):

    Routes different URL paths to different backend services

    kubectl apply -f ingress-multi.yaml

    curl -k https://example.com/
    curl -k https://example.com/api

### Cleanup:

    kubectl delete ingress simple-ingress multi-ingress 2>/dev/null
    kubectl delete certificate example-tls 2>/dev/null
    kubectl delete issuer selfsigned-issuer 2>/dev/null
    kubectl delete secret sec-example 2>/dev/null
    kubectl delete svc caddy-svc 2>/dev/null
    kubectl delete pod caddy 2>/dev/null
    
    kind delete cluster --name ingress-lab
    
    sudo sed -i '/example.com/d' /etc/hosts
    
    cd .. && rm -rf ingress-lab/

### License:

    This project is licensed under the MIT License.