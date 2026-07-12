Production Blueprint: vSphere CPI & CSI Integration for Vanilla Kubernetes
This repository provides an enterprise-ready infrastructure-as-code (IaC) layout and deployment playbook for running upstream, vanilla Kubernetes on VMware vSphere.

Instead of treating vSphere as a dumb hypervisor, this architecture leverages the native Cloud Provider Interface (CPI) and Container Storage Interface (CSI) to transform vSphere into a fully native Kubernetes cloud provider, automating compute lifecycles, node topologies, and persistent storage allocation on shared SAN arrays (such as HPE MSA/VMFS backends).

Repository Architecture
Plaintext
├── README.md
└── manifests/
    ├── cpi/
    │   ├── vsphere-cpi-secret.yaml       # Sanitized credential mapping
    │   ├── vsphere-cpi-config.yaml       # Global configmap & cloud-config
    │   └── vsphere-cpi-daemonset.yaml    # Cloud controller manager specs
    └── csi/
        ├── vsphere-csi-secret.yaml       # CSI vCenter connection secrets
        ├── vsphere-csi-driver.yaml       # Controller, Attacher, Resizer, Syncer & Node DS
        └── storageclass-spbm.yaml        # StorageClass mapping to vSphere SPBM policies
1. Multi-Cluster Architecture & RBAC Isolation
Running multiple environments (e.g., Staging and Production) against a single vCenter instance requires strict cryptographic and namespace isolation to limit the failure blast radius.

The Identity Model
Never use global admin service accounts (administrator@vsphere.local) inside Kubernetes manifests.

Separate user identities must be provisioned in vSphere: csi-stage-user@yourdomain.local and csi-prod-user@yourdomain.local.

Scoped Permissions Matrix
Rather than applying an administrative role globally with broad child inheritance, a single custom role (k8s-cns-csi-role) containing CNS, Datastore, and Virtual Machine configurations should be bound at three distinct structural layers within vCenter:

vCenter Object	Applied Identity	Inheritance Rule	Purpose
vCenter Root Node	csi-prod-user	Propagate: NO	Grants access to global profile-driven storage APIs (StorageProfile.View) and CNS searching without exposing other inventory trees.
Production Compute Cluster	csi-prod-user	Propagate: YES	Allows the controller to perform VirtualMachine.Config.AddRemoveDevice actions strictly on production worker nodes.
Production Datastores / SAN	csi-prod-user	Propagate: YES	Restricts storage allocation permissions (Datastore.AllocateSpace) strictly to the designated production LUNs.
2. Infrastructure Pre-requisites & vCenter Setup
The underlying vSphere environment must be modified before booting the cloud controller containers, or the storage layer will fail silently.

Node Parameters: Hardware Serial Visibility
Linux guest kernels inside your virtual machine worker nodes cannot natively match block devices back to vCenter First Class Disks (FCD) without explicitly exposed serial tags.

For every control-plane and worker VM in the cluster:

Power down the guest OS.

Edit Settings -> VM Options -> Advanced -> Edit Configuration.

Insert the following entry exactly:

```Ini, TOML
disk.EnableUUID = "TRUE"
```
Power on the VM.

Zoning and Topology (vSphere Tagging)
To allow the Kubernetes scheduler to intelligently distribute stateful workloads across physical availability zones (e.g., preventing all MinIO tenant pods from landing on the exact same physical ESXi host or rack), you must define a failure domain topology using vSphere Tags.

Create Tag Categories:

Create category k8s-region (Cardinality: Single Selection, Associable to: Datacenter objects).

Create category k8s-zone (Cardinality: Single Selection, Associable to: Compute Cluster or Host objects).

Assign Tags:

Tag your primary Datacenter as region-west or datacenter-main.

Tag distinct physical ESXi Compute Clusters or Host groups as zone-a, zone-b, etc.

The CPI automatically detects these tags and applies them natively as scheduling metadata directly to your Kubernetes nodes:

topology.kubernetes.io/region

topology.kubernetes.io/zone

3. Deployment Sequence
Phase 1: Compute Routing & Node Initialization (CPI)
The CPI must be healthy before installing the storage layer. Until the CPI runs its first sweep, your nodes will sit with an uninitialized taint and lack cloud provider identities.

Edit manifests/cpi/vsphere-cpi-secret.yaml and enter your sanitized credentials.

Configure the infrastructure scope inside manifests/cpi/vsphere-cpi-config.yaml:

```YAML
[Global]
secret-name = "vsphere-cpi-secret"
secret-namespace = "kube-system"

[VirtualCenter "<YOUR_VCENTER_IP_OR_FQDN>"]
port = "443"
datacenters = "<YOUR_PRODUCTION_DATACENTER_NAME>"
```
Apply the manifests:

```Bash
kubectl apply -f manifests/cpi/vsphere-cpi-secret.yaml
kubectl apply -f manifests/cpi/vsphere-cpi-config.yaml
kubectl apply -f manifests/cpi/vsphere-cpi-daemonset.yaml
```
Operational Guardrail: The CPI runs as a multi-master DaemonSet with active leader election. If standard logs look idle during verification, find the true active controller instance holding the leasing lock before parsing outputs:

```Bash
kubectl get lease -n kube-system cloud-controller-manager -o jsonpath='{.spec.holderIdentity}'
```
Verify that your node specifications now natively present their underlying virtual hardware IDs:

```Bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.providerID}{"\n"}{end}'
```
# Expected Output: node-name   vsphere://4236b2cb-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Phase 2: Dynamic Cloud Native Storage (CSI)
Once spec.providerID strings match cleanly across your node endpoints, apply the storage platform components.

Update your unique cluster identifier in your CSI configurations:

```Ini, TOML
[Global]
cluster-id = "hekmat-prod-cluster" # Must be unique across all environments sharing this vCenter
```
Deploy the core multi-container controller pods, sidecars (Attacher, Resizer, Provisioner), and node daemons:

```Bash
kubectl apply -f manifests/csi/
```
Establish dynamic storage provisioning by mapping a Kubernetes StorageClass to a vSphere Storage Policy Based Management (SPBM) rule that filters datastores by your custom SAN tags:

```YAML
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: msa2040-production-sc
provisioner: csi.vsphere.vmware.com
parameters:
  StoragePolicyName: "Production-SAN-Storage-Policy" # Matches your vSphere SPBM Policy Name
  DatastoreURL: "ds:///vmfs/volumes/xxxxx/"           # Optional: Pin directly to specific SAN LUNs
```


4. Operational Playbook & Common Traps
Symptom: Core Sidecars Loop-Failing on /csi/csi.sock
Root Cause: When running kubectl logs, the command defaults alphabetically to the csi-attacher or csi-provisioner sidecar containers. If these containers throw a no such file or directory socket error, it means the primary driver container (vsphere-csi-controller) inside that same pod hasn't initialized yet.

Remediation: Query the core driver container specifically to find the configuration or API authentication blocker:

```Bash
kubectl logs deployment/vsphere-csi-controller -c vsphere-csi-controller -n kube-system
```

Symptom: Applications (e.g., MinIO Console) Stuck in Pending via Ingress
Root Cause Cascade: If an internal application stack fails to launch and your web browser blocks connection attempts via HSTS warnings showing a Kubernetes Ingress Controller Fake Certificate, the root error isn't networking—it's down at the storage layer.

Diagnostic Order:

The Ingress presents a generic fake fallback cert because the backend target app isn't routing traffic.

The application is routing zero traffic because its pods are in a Pending state.

The pods are Pending because their persistent volume claims (PVC) are awaiting structural provisioning from a block layer missing node providerID mappings.

Isolation Workflow: Bypass the entire cluster networking layer to test internal components locally using port forwarding straight to the operational namespaces once hardware attributes are verified:

```Bash
kubectl port-forward service/minio-console-service 9001:9001 -n minio-system
```