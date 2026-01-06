
# Synthlane DevOps Assessment: K3s & OIDC Configuration

## Overview

This repository demonstrates the setup and validation of a single-node Kubernetes cluster using k3s, deployment of Open WebUI via Helm, and configuration/debugging of OIDC authentication. It is designed to showcase practical DevOps skills in cluster setup, application deployment, secure authentication, and troubleshooting.

---

## Part 1: Docker Installation

Install Docker to provide a container runtime for Kubernetes and other tools.

### 1. Update system and install dependencies
```sh
sudo apt update && sudo apt install -y ca-certificates curl gnupg lsb-release
```

### 2. Add Docker’s GPG key and repository
```sh
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
```

### 3. Install Docker Engine
```sh
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

### 4. Start and enable Docker
```sh
sudo systemctl start docker
sudo systemctl enable docker
```


### 5. (Optional) Check Docker status
```sh
sudo systemctl status docker
```

### 6. Allow your user to run Docker (VERY IMPORTANT)
By default, Docker requires sudo. To allow your user to run Docker commands without sudo:

Add your user to the Docker group:
```sh
sudo usermod -aG docker $USER
```

Apply the group change :
- Start a new shell session:
	```sh
	newgrp docker
	```


---


## Part 2: K3s Cluster Setup (Single-Node)

### Step 1: Install k3s

Install with one command:
```sh
curl -sfL https://get.k3s.io | sh -
```


### Step 2: Verify k3s service
```sh
sudo systemctl status k3s
```

If not running:
```sh
sudo systemctl start k3s
```

### Step 3: Configure kubectl for your user

```sh
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
```

### Step 4: Verify kubectl works (no sudo)
```sh
kubectl version --client
kubectl cluster-info
```

### Step 5: Validate cluster (Assessment validation)
Check nodes:
```sh
kubectl get nodes
```
Check all pods:
```sh
kubectl get pods -A
```
You should see system pods (coredns, local-path-provisioner, metrics-server, traefik) in the kube-system namespace. This confirms control plane, networking, and DNS are working.



#### Cluster Status Summary
- Single-node Kubernetes cluster
- Fully operational
- User-level kubectl access
- Suitable for development & early production workloads

#### Why this setup is suitable for an early-stage startup
- Low operational overhead, cost-effective, fast to deploy, production-grade API, easy scaling path
- Limitations: Single-node (no HA), limited fault tolerance, not for heavy production traffic
- Verdict: Excellent for MVPs, internal tools, CI/CD, and cost-sensitive workloads. Should evolve to multi-node or managed K8s as business scales.

---


## Part 3: Application Deployment using Helm


### Step 1: Install Helm (v3+)

```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
Verify Helm:
```sh
helm version
```


### Step 2: Add Open WebUI Helm repository
```sh
helm repo add open-webui https://helm.openwebui.com/
helm repo update
helm repo list
```

### Step 3: Create namespace
```sh
kubectl create namespace openwebui
kubectl get ns
```

### Step 4: Perform a Dry Run (MANDATORY)
Dry-run install:
```sh
helm install webui open-webui/open-webui \
	--namespace openwebui \
	--set service.type=ClusterIP \
	--dry-run
```
This proves chart validity and manifest rendering. No changes applied yet.

### Step 5: Install Open WebUI
Actual deployment:
```sh
helm install webui open-webui/open-webui \
	--namespace openwebui \
	--set service.type=ClusterIP
```
Expected output: `NAME: webui STATUS: deployed`

### Step 6: Validate deployment
Get all resources in namespace:
```sh
kubectl get all -n openwebui
```

Optional: Check logs
```sh
kubectl logs -n openwebui deployment/webui
```

#### Accessing the service
Service type is ClusterIP (internal only). For access:
```sh
kubectl port-forward -n openwebui svc/webui 8080:80
```


## Part 4: Authentication Configuration & Debugging (OIDC)



### Step 1: Create OIDC values file
Create a file named `values-oidc.yaml`:
```sh
nano values-oidc.yaml
```
Paste the following content (as per assessment):
```yaml
oidc:
	clientId: "test"
	clientSecret: ""
	issuer: "https://<YOUR_DOMAIN>/auth/realms/hyperplane/.well-known/openid-configuration"
	scopes:
		- openid
		- profile
		- email
```


### Step 2: Apply OIDC config using Helm upgrade
Update the existing release with the new OIDC config:
```sh
helm upgrade webui open-webui/open-webui \
	--namespace openwebui \
	--values values-oidc.yaml
```
Helm will update the deployment, trigger a pod restart, and inject the OIDC configuration into the application.

---

## Troubleshooting & Validation

- Use `kubectl get pods -A` to check deployments
- Inspect logs for k3s, Docker, and Helm
- Validate OIDC login via your app or kubectl

---

## Part 5: Ownership Questions

### 1. Production Readiness
**Top 5 risks before going live:**
1. Single-node,no high availability (HA); node failure 
2. No automated backups 
3. Secrets/configs may be exposed or not rotated
4. Lack of monitoring/alerting for resource usage and failures
5. OIDC misconfiguration or IdP downtime blocks all logins

**First 2 things you would fix:**
1. Add a second node or move to managed K8s for HA.
2. Implement automated, tested backups for critical data and configs.

---

### 2. Failure Scenario
**Scenario:** Traffic spikes 10x and node goes down at 2 AM

- **What breaks first?**
	- The single node will run out of CPU/memory, causing pods to be evicted or crash. Eventually, the entire cluster becomes unavailable.
- **How do you recover?**
	- Investigate and restart the node (via Hetzner console if needed). Restore from backup if data is lost. Scale up resources if possible.
- **What do you change the next day?**
	- Add node multiple nodes for redundancy, set up autoscaling or move to a managed/high-availability cluster. Implement monitoring and alerting for early detection.

---

### 3. Security & Secrets
- **How do you manage secrets?**
	- Use Kubernetes Secrets, sealed-secrets, or an external secrets manager (e.g., HashiCorp Vault). Never store secrets in plaintext or in Git.
- **What must never be in Git?**
	- Passwords, API keys, certificates, kubeconfig files, and any sensitive credentials.
- **What should be rotated?**
	- All secrets, API tokens, certificates, and service account keys should be rotated regularly and after any suspected compromise.

---

### 4. Backups & Recovery
- **What data must be backed up?**
	- Application data (databases, persistent volumes), configuration files, Helm values, and cluster state (etcd/sqlite DB).
- **Backup frequency?**
	- At least daily for critical data; more frequently for high-change workloads.
- **How do you test recovery?**
	- Regularly perform restore drills in a test environment to ensure backups are valid and recovery steps are documented.

---

### 5. Cost Ownership (Hetzner)
- **How do you keep infra costs low?**
	- Use minimal VM sizes, spot/preemptible instances, and only run essential workloads. Monitor usage and shut down unused resources.
- **What do you avoid early?**
	- Avoid over-provisioning, expensive managed services, and unnecessary third-party tools.
- **When would you move away from k3s?**
	- When you need multi-node HA, advanced networking, or managed support—or as the team and workload scale beyond a single node’s capacity.

---

## Extra Credit: OIDC Provider with Self-Signed Certificate

**How to make the application trust a custom CA in a maintainable way:**

1. Add the custom CA certificate to a Kubernetes Secret or ConfigMap.
2. Mount the CA into the pod at a standard location 
3. Use an initContainer or startup script to update the trust store.
4. Automate this via Helm values and templates for consistency across environments.
5. Document the process and rotate the CA if needed.
---
## References

- [K3s Documentation](https://rancher.com/docs/k3s/latest/en/)
- [Helm Documentation](https://helm.sh/docs/)
- [OIDC Kubernetes Auth](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)