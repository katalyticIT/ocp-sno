# OpenShift 4: Single-Node Installation on AWS (UPI)

This guide describes the installation of an OpenShift Single-Node Cluster (SNO)
on AWS using the User-Provisioned Infrastructure (UPI) method. Because AWS
cloud instances (AMIs) do not support "Bootstrap-in-Place" installations, a
temporary bootstrap node is used for the initial cluster creation.

## Architecture & Prerequisites

You'll need three systems to setup the OpenShift Single Node Cluster; at least one of them is temporary:
* **Bootstrap Node (Temporary):** AWS EC2 `m6i.xlarge` (4 vCPUs, 16 GB RAM), 100 GB gp3 storage.
* **Master Node (SNO):** AWS EC2 `m6i.2xlarge` (8 vCPUs, 32 GB RAM), 120 GB gp3 storage, Elastic IP.
* **Management Station & Web Server:** An external server running the CLI tools, hosting the large Ignition files via HTTP(S) using Nginx and being the one IP that's able to access the two AWS instances by ssh/22 (should that be needed).

During the setup process you need to know or have the following information:
* **DNS Domain:** Your own domain (e.g., `mydoma.in`), managed by any DNS provider.
* **Cluster name:** The namem of your cluster; e.g. `ocp` which later results in `api.opc.mydoma.in` for the access point.
* **RedHat pull secret:** Download the pull secret from your RedHat account.
* **Ssh Key:** A ssh key which will enable you to access the underlying CoreOS in case of problems. Ideally the public ssh key of your user on the mgmt. station/web server.

## Phase 1: Preparation on the Management Station

### 1. Fresh Directory & Tools

It is strictly required to work in a completely new/empty directory.
The installer creates _hidden state files_ that can lead to certificate
errors (SNI mismatches) if you retry an installation in an old directory.

```bash
mkdir ~/ocp-install && cd ~/ocp-install
# Download and extract oc and openshift-install (Linux)...
```

### 2. Create install-config.yaml

Create the configuration file (`platform: none` triggers the UPI mode):

```yaml
apiVersion: v1
baseDomain: x68k.de
metadata:
  name: ocp
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 1
  platform: {}
platform:
  none: {}
fips: false
pullSecret: '{"auths":{...}}'
sshKey: 'ssh-ed25519 AAAAC3...'
```

Place your RedHat pullsecret and your public ssh key in the last two lines.

*Important:* Make a backup of your `install-config.yaml` because the OpenShift
installer removes it during its progress. If you want or need to try again,
make sure you've got a copy.


### 3. Generate Manifests and Ignition Files

The master node must be made schedulable so that the ingress router and image registry can run on it.

```bash
openshift-install create manifests --dir=.
sed -i 's/mastersSchedulable: false/mastersSchedulable: true/' manifests/cluster-scheduler-02-config.yml
openshift-install create ignition-configs --dir=.
```

### 4. Host the Ignition Files

AWS EC2 User-Data is strictly limited to 16 KB. The generated `.ign` files exceed this limit.
Copy `bootstrap.ign` and `master.ign` into the webroot of your external Nginx server
(e.g., `https://mgmt.mydoma.in/ignition/`).


### 5. Create Pointer Ignitions for AWS

Create two small JSON files that will be pasted into the AWS User-Data field later.

**pointer-bootstrap.ign:**
```json
{"ignition": {"version": "3.2.0","config": {"replace": {"source": "https://mgmt.mydoma.in/ignition/bootstrap.ign"}}}}
```

**pointer-master.ign:**
```json
{"ignition": {"version": "3.2.0","config": {"replace": {"source": "https://mgmt.mydoma.in/ignition/master.ign"}}}}
```

## Phase 2: Configure AWS Networking and Security

### 1. Security Group (`ocp-sno-sg`)

This is a critical part. Missing rules here will cause etcd split-brain scenarios or API timeouts.

**Inbound Rules:**
1. **Management (from your mgmt./web server):** TCP 22 (SSH) and TCP 6443 (API). (You may open 22/ssh publicly if you need to access the instances from more than one PC during setup, but make sure taht you narrow this down again after the Master is up and running.)
2. **Public:** TCP 443 and TCP 80 (Ingress/Apps).
3. **Cluster-Internal (Bootstrap <-> Master):**
   * **CRITICAL:** Allow *All Traffic* with the source set to the Security Group's own ID (e.g., `sg-012...`).
   * **CRITICAL:** Additionally, allow *All Traffic* for the Master's private IPv4 (/32) and the Bootstrap's private IPv4 (/32).

If your Single Node setup doesn't succeed at the first time, you may widen up the last rule temporarily
to the network 172.16.0.0/12. As the IP addresses and their range may change over different setups,
this assures both instances can reach each other properly. When the setup is done, this part of the
network rules should be removed again.

### 2. DNS (Bootstrap Phase)

## Phase 3: Launch Instances (The EBS Volume Trick)

_This is important because of timing:_
It's advised to have the DNS console already open when the Bootstrap Node is running,
so you can assign the IP addresses quickly. Note the IP addresses (private and public)
that the Bootstrap Node received while starting.

### 1. Launch Bootstrap Node

* **AMI:** Search under *EC2 -> AMIs -> Public Images* for the official RHCOS image for your region (e.g., using `openshift-install coreos print-stream-json`).
* **Type:** `m6i.xlarge`
* **Storage:** _Change_ the default 16 GB gp2 root volume directly in the launch wizard to **100 GB gp3**! Do *not* add a second disk; RHCOS automatically expands the root partition during the first boot.
* **User Data:** Paste the contents of `pointer-bootstrap.ign`.

Once it's running, note the IP addresses the Bootstrap Node received and setup the DNS records _with TTL=60s_:

* `api.clustername.mydoma.in` -> Public bootstrap instance IP
* `api-int.clustername.mydoma.in` -> Private bootstrap instance IP (172.x.x.x)

### 2. Launch Master Node (SNO)

* ** AMI:** Use the same as above with the Bootstrap Node.
* **Type:** `m6i.2xlarge`
* **Storage:** _Change_ the default 16 GB gp2 to **120 GB gp3**.
* **User Data:** Paste the contents of `pointer-master.ign`.
* **Network:** _Immediately_ associate your reserved Elastic IP with the Master node in the AWS console.


Once it's running, return to your DNS console and assign the Elastic IP to the Master Node _with TTL=60s_:

* `*.apps.clustername.mydoma.in` -> Master Node (Elastic IP)


## Phase 4: Monitoring and Troubleshooting

Set your Kubeconfig on the management station:

```bash
export KUBECONFIG=~/ocp-install/auth/kubeconfig
```

### Approve Certificates (CSRs)

This is absolutely mandatory to allow the OVN network to start. Sometimes the
certificate requests don't get approved automatically. Without approval,
all pods will remain stuck in a `ContainerCreating` state. 
Periodically check for and approve all pending CSRs:

```bash
oc get csr
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs -r oc adm certificate approve
```

### Diagnostic Commands

* **Monitor Bootstrap:** `ssh core@<Bootstrap-IP> "journalctl -u bootkube.service -f"`
* **Monitor Kubelet (Master):** `ssh core@<Master-IP> "journalctl -u kubelet.service -f"`
* **Check Containers (Master):** `sudo crictl ps -a` (Watch out for crashing `etcd` or `kube-apiserver` containers).

## Phase 5: Handoff and Cutover

Once the Master successfully spins up its own control plane, the Bootstrap node will
shut down its temporary API service (closing port 6443). Your `oc` client will briefly
fail with `Connection refused`. This is the signal for the cutover.

### 1. DNS Cutover

Update your DNS records to point to the Master's Elastic IP:

* `api.clustername.mydoma.in` -> Master Elastic IP
* `api-int.clustername.mydoma.in` -> Master Private IP

*(Tip: To bypass global DNS caching delays, _temporarily_ add the Master's IP to the `/etc/hosts` file of your management station).*

### 2. Complete the Bootstrap Phase

Wait for the installer to confirm the handoff:

```bash
openshift-install wait-for bootstrap-complete --dir=. --log-level=info
```
> Output: `INFO It is now safe to remove the bootstrap resources`

**You can now safely terminate (delete) the Bootstrap EC2 instance in the AWS Console.**


### 3. Complete the Installation

Let the installer wait for the remaining cluster operators to initialize:

```bash
openshift-install wait-for install-complete --dir=. --log-level=info
```

Another way to watch the final stage of cluster setup is to check the 
status of the ClusterOperators regularly. The cluster is ready when the
list of not available Operatores becomes empty:

```
watch -n 5 oc get co -w -o json \
| jq -r '.items[] select(.status.conditions[] | select(.type=="Available" and .status != "True")) | .metadata.name'
```

## Phase 6: Error handling

### 1. Emergency access

If the API server is actually running internally on the master but is not yet accessible from the
outside – for example, because the SDN/network pods are still in the ‘Pending’ state – you can
logon to the master via ssh and run `oc` locally.

There is a local `kubeconfig` file on the master that bypasses the network layer and
accesses `localhost` directly:

```
# Run on the Master (SSH session):
export KUBECONFIG=/etc/kubernetes/static-pod-resources/kube-apiserver-certs/secrets/node-kubeconfigs/localhost.kubeconfig

# Check if the API answers:
oc get nodes
oc get pods -A
oc get csr
```

### 2. etcd: Lost quorum

Sometimes, if during the and-over process the Bootstrap Node isn't cut out off the cluster properly,
the `etcd` daemon may loose its quorum, making it qait forever for the Bootstrap's `etcd` to come back.

If the setup seems stuck, check if the etcd has quorum:

```
# sudo inside the etcdctl container and check cluster status
sudo crictl exec $(sudo crictl ps --name etcdctl -q) etcdctl endpoint health -w table
```

You should reveice a `is_healthy: true`. If not and if the error message is `context deadline exceeded`,
then reset the `etcd` cluster configuration to just one member:

```
sudo -E /usr/local/bin/quorum-restore.sh
```

If the script hangs and cannot reset the `etcd` configuration, then stop the `etcd` container
manually:

```
# Stops all running etcd containers immediately:
for id in $(sudo crictl ps -q --name "etcd"); do 
  echo "Killing $id"
  sudo crictl stop -t 0 $id
done
```

Then rerun the `/usr/local/bin/quorum-restore.sh`script.

## Phase7: Cleaning up

Once the Master Node is up and running, the following tasks must be carried out:

1. Clean up the firewwall rules: Make sure not everybody may access the ssh/22 port and the API at https/6443. Also remove the 172.x.x.x network if you added that during setup.
2. Remove the ignition files from your web server. (They contain sensitive data.)
3. If not already done, set the TTL on the DNS entries to 3600s.


## Phase 8: Post-Installation (Storage)

Since Single-Node OpenShift does not include a dynamic storage provisioner by default,
the final step is to install **Red Hat LVM Storage (lvms)** via the OperatorHub.

This enables the automatic provisioning of Persistent Volume Claims (PVCs) using the
remaining free space on the root volume or any additionally attached disks. Using AWS
it's advised to add a separate (_not formatted_) volume for this.


### 2. Install the LVMS Operator

When connected to your cluster, find out the channel for the LVMS Operator:

```
oc describe packagemanifest lvms-operator -n openshift-marketplace | grep -iE "defaultChannel|channel:"
```

Note the latest stable channel from the generated list (usually something like `stable-4.xx`)
and install the Operator from there:

```
CHANNEL="place the lasted stable channel here" cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: lvms-operator
  namespace: openshift-storage
spec:
  channel: $CHANNEL
  installPlanApproval: Automatic
  name: lvms-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```
Verify the installation:
```
# 1. InstallPlan kontrollieren
oc get installplan -n openshift-storage

# 2. CSV-Status überwachen (sollte auf "Succeeded" wechseln)
oc get csv -n openshift-storage -w
```

Once the CSV has reached the ‘Succeeded’ phase status, the operator is ready for use and
you can proceed to define the LVMCluster custom resource for your sanitised NVMe disks.


