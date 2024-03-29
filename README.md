# PaaS-challenge

## Architecture

### Nodes

- **Gateways:** 2 gateway node with shared IP and HA with keepalived

- **Ansible-master:** A control node to manage all nodes and execute the ansible playbooks.

- **etcd:** 3 nodes for etcd cluster with etcd v3.4.7

- **K8S:** 3 master nodes + 1 worker node

### Diagram

<p align="center">
  <img src="./pictures/diagram-paas.jpg">
</p>

**Note:** The etcd servers are connected to the masters api and there is no loadbalancing via the LBs.

## Gateways

The gateways are the communication way to the nodes. In this architecture, All nodes has private network. So there is no way accessing nodes unless the gateway. In other way, all nodes can access the internet via NAT protocol through the gateway node. So as we see, the gateway could be our point of failures. So we need to HA the gateway node and make it as high available as possible. For this purpose we could use the `keepalived`. 

The other role of gateways are:

- Loadbalancing requests between the kubernetes master nodes.
- Loadbalancing requests from the user to the cluster.

### Implementation

These gateways has two components:

- Keepalived: To ha the gateways I installed the keepalived and configure for ha the public IP.
- Haproxy: To loadbalance the client traffic to the ingress. (In our scenario we had not implement any ingress. We talk about the production.)

You can deploy the gateways via the [`lb`](./ansibles/roles/lb) roles in ansible folder.

## etcd

### etcd Hardware requirement

​	Related to the etcd official doc about the [hardware requirement](https://etcd.io/docs/v3.3/op-guide/hardware/) for small cluster we need the following hardware requirement:

![image-20220707185025194](./pictures/etcd-hardware-req.png)



The cluster has been initialized with three nodes and TLS authentication. All certificates has been issued by `OpenSSL`.

### Implementation

The etcd has been implemented via the systemd service and etcd binary files from offical repo.

This process has been automated with the TLS authentication and you can use it by the [`etcd-implementation`](./ansibles/roles/etcd-implementation)

### etcd disaster recovery

Related to the etcd official [documentation](https://etcd.io/docs/v3.5/op-guide/recovery/):

> etcd is designed to withstand machine failures.

So we can have some plans and scenarios to escape from any etcd disaster. For all the scenarios the most important thing is to have the etcd backup periodically. 

#### How to backup the etcd cluster?

select one of the etcd nodes and execute the following command:

```shell
etcdctl --endpoints https://192.168.0.10:2379 --cert=/etc/etcd/ssl/peer.crt  --key=/etc/etcd/ssl/peer.key  --cacert=/etc/etcd/ssl/ca.crt snapshot save /home/ubuntu/snapshot-22-07-14.db
```

**Note:** It's hardly recommended to save the etcd snapshot on another storage such as the object storage for keep it safe.

#### 1. Minor follower failure 

In this scenario the cluster is functional and can write and read the requests. So before any more failure happens and the quorum fails, we need to repair the failed node or replace it with new node. For this implementation follow the steps:

1. stop the etcd.service on the node:

   ```shell
   systemctl stop etcd.service
   ```

2. remove the node from the cluster:

   ```shell
   etcdctl  --endpoints=https://192.168.0.10:2379,https://192.168.0.11:2379,https://192.168.0.12:2379 --cert=/etc/etcd/ssl/peer.crt --key=/etc/etcd/ssl/peer.key --cacert=/etc/etcd/ssl/ca.crt member remove 6f03c26636586d04
   ```

3. add the new node (or the currently node) to the cluster:

   ```shell
   etcdctl  --endpoints=https://192.168.0.10:2379,https://192.168.0.11:2379,https://192.168.0.12:2379 --cert=/etc/etcd/ssl/peer.crt --key=/etc/etcd/ssl/peer.key --cacert=/etc/etcd/ssl/ca.crt member add etcd-3 --peer-urls=https://192.168.0.12:2380
   ```

4. If you are doing this process on a failure node which only lost the data dir you need only change the `/etc/etcd/etcd.conf.yaml` and set the `initial-cluster-state:exisiting`. But if you add replace node, after installing the etcd and etcdctl you need to add the following conf to `/etc/etcd/etcd.conf.yaml`:

   ```yaml
   data-dir: /var/lib/etcd/etcd-1.etcd
   name: etcd-1
   initial-advertise-peer-urls: https://192.168.0.10:2380
   listen-peer-urls: https://192.168.0.10:2380,https://127.0.0.1:2380
   advertise-client-urls: https://192.168.0.10:2379
   listen-client-urls: https://192.168.0.10:2379,https://127.0.0.1:2379
   initial-cluster-state: existing
   initial-cluster: etcd-1=https://192.168.0.10:2380,etcd-2=https://192.168.0.11:2380,etcd-3=https://192.168.0.12:2380
   client-transport-security:
     cert-file: /etc/etcd/ssl/server.crt
     key-file: /etc/etcd/ssl/server.key
     trusted-ca-file: /etc/etcd/ssl/ca.crt
   peer-transport-security:
     cert-file: /etc/etcd/ssl/peer.crt
     key-file: /etc/etcd/ssl/peer.key
     trusted-ca-file: /etc/etcd/ssl/ca.crt
   ```

5. start the etcd.service:

   ```shell
   systemctl start etcd.service
   ```

#### 2. Leader failure

In this scenario when the leader fails, the etcd cluster automatically start a new election to choose the new leader. This process takes the `election timeout` to replace the new leader.

According to the etcd official documentation:

> During the leader election the cluster cannot process any writes. Write requests sent during the election are queued for processing until a new leader is elected.

So after new leader elected, this scenario is similar to [Minor follower failure](####minor-follower-failure).

#### 3. Majority failure

When the majority of the nodes has failed, the cluster has collapsed. So there are two options:

1. Try to transform the scenario to the [Minor follower failure](####minor-follower-failure) by get some nodes back online!
2. Initialize the new-cluster on the healty node and join new members to it!

Let's see the second option:

1. First you need to choose one of the nodes to work on it.

2. Stop the `etcd.service` and delete the data-direcotyr on all nodes.

   ```shell
   systemctl stop etcd.service
   rm -rf /etc/etcd
   ```

3. Restore the snapshot.

   ```shell
    etcdctl  --endpoints https://192.168.0.10:2379  --cert=/etc/etcd/ssl/peer.crt  --key=/etc/etcd/ssl/peer.key  --cacert=/etc/etcd/ssl/ca.crt  --initial-cluster=etcd-1=https://192.168.0.10:2380,etcd-2=https://192.168.0.11:2380,etcd-3=https://192.168.0.12:2380  --initial-cluster-token=etcd-cluster-1  --initial-advertise-peer-urls=https://192.168.0.10:2380  --name=etcd-1  --skip-hash-check=true  --data-dir /var/lib/etcd snapshot restore /home/ubuntu/snapshot-22-07-14.db
   ```

4. edit the `/etc/etcd/etcd.conf.yaml` configuration file and add the `force-new-cluster` flag. Be sure that the `initial-cluster-state` set to new.

```yaml
name: etcd-1
initial-advertise-peer-urls: https://192.168.0.10:2380
listen-peer-urls: https://192.168.0.10:2380,https://127.0.0.1:2380
advertise-client-urls: https://192.168.0.10:2379
listen-client-urls: https://192.168.0.10:2379,https://127.0.0.1:2379
initial-cluster-state: new
force-new-cluster: true
initial-cluster: etcd-1=https://192.168.0.10:2380,etcd-2=https://192.168.0.11:2380,etcd-3=https://192.168.0.12:2380
client-transport-security:
  cert-file: /etc/etcd/ssl/server.crt
  key-file: /etc/etcd/ssl/server.key
  trusted-ca-file: /etc/etcd/ssl/ca.crt
peer-transport-security:
  cert-file: /etc/etcd/ssl/peer.crt
  key-file: /etc/etcd/ssl/peer.key
  trusted-ca-file: /etc/etcd/ssl/ca.crt
```

5. Start the etcd.service to initialize a new cluster:

```shell
systemctl start etcd.service
```

6. Check that if the etcd cluster has been initialized secussessfuly:

```shell
etcdctl  --endpoints=https://192.168.0.10:2379 --cert=/etc/etcd/ssl/peer.crt --key=/etc/etcd/ssl/peer.key --cacert=/etc/etcd/ssl/ca.crt endpoint health
```

7. Now it's time to add new members:

```shell
etcdctl  --endpoints=https://192.168.0.10:2379 --cert=/etc/etcd/ssl/peer.crt --key=/etc/etcd/ssl/peer.key --cacert=/etc/etcd/ssl/ca.crt member add etcd-2 --peer-urls=https://192.168.0.11:2380
```

The new member added to cluster. We need to start the etcd on the etcd-2 node but remember that we only have two initial_cluster nodes and if we start the etcd with another initial member we got the [`member count is unequal`](https://etcd.io/docs/v3.2/op-guide/runtime-configuration/#error-cases-when-adding-members)

8. Edit the `/etc/etcd/etcd.conf.yaml` file:

```yaml
data-dir: /var/lib/etcd
name: etcd-2
initial-advertise-peer-urls: https://192.168.0.11:2380
listen-peer-urls: https://192.168.0.11:2380,https://127.0.0.1:2380
advertise-client-urls: https://192.168.0.11:2379
listen-client-urls: https://192.168.0.11:2379,https://127.0.0.1:2379
initial-cluster-state: existing
initial-cluster: etcd-1=https://192.168.0.10:2380,etcd-2=https://192.168.0.11:2380
client-transport-security:
  cert-file: /etc/etcd/ssl/server.crt
  key-file: /etc/etcd/ssl/server.key
  trusted-ca-file: /etc/etcd/ssl/ca.crt
peer-transport-security:
  cert-file: /etc/etcd/ssl/peer.crt
  key-file: /etc/etcd/ssl/peer.key
  trusted-ca-file: /etc/etcd/ssl/ca.crt
```

It's time to start the service:

```shell
systemctl start etcd.service
```

9. Repeat this process for other members.

**Note:** If your nodes are damaged and you have to use new nodes, you have to install etcd and etcdctl before the above proccess.

#### Automation

Unfrotunately, This process hadn't been automated.

## K8S cluster

### Implementation

This cluster has 3 masters and 1 workers. For loadbalancing masters, I used the haproxy. The cluster has been initialized via `kubeadm`.

This process has been automated via the following roles:

- [kubernetes-prerequsites](./ansibles/roles/kubernetes-prerequsites/)
- [kubernetes-initialize](./ansibles/roles/kubernetes-initialize)
- [kubernetes-master-join](./ansibles/roles/master-join)
- [kubernetes-worker-join](./ansibles/roles/worker-join)

### Storage

My first choice to implement the storageClaas and the local-path provisioner was the [kubernetes local-static-provisioner](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) which due to the errors I got from the persistentVolume in Mariadb-replication I changed the component to [rancher local-path-provisioner](https://github.com/rancher/local-path-provisioner).

#### Implementation

```shell
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.22/deploy/local-path-storage.yaml
```

And testing:

```shell
kubectl create -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/examples/pvc/pvc.yaml
kubectl create -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/examples/pod/pod.yaml
```

For specific configuration you can edit the Configmap in `local-path-storage` namespace ‍‍‍‍ ‍called `local-path-config` and change the config.json parameters:

```yaml
data:
  config.json: |-
    {
            "nodePathMap":[
            {
                    "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                    "paths":["/opt/local-path-provisioner"]
            },
            {
                    "node":"k8s-worker-1",
                    "paths":["/disks"]
            }
            ]
    }
```

### Helm

Installing helm chart can be done by the helm scripts:

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Mariadb-replication

For mariadb-replication I prefer to use the [`bitnami mariadb Helm chart`](https://github.com/bitnami/charts/tree/master/bitnami/mariadb).

#### Implementation

1. Add the helm chart to your repo:

   ```shell
   helm repo add bitnami https://charts.bitnami.com/bitnami
   ```

2. You need to first extract the `vaules.yml` file from the helm chart

   ```
   helm show values bitnami/mariadb > values.yml
   ```

3. Edit the `values.yml` and install the helm:

   Things need to set:

   - StorageClass
   - PVC capacity
   - ...

   ```shell
    helm install RELEASE_NAME bitnami/mariadb -f values.yml --set rootUser.password=<password> --set replication.password=<password>
   ```

After that check the pods ans statefulset

There is an example [values.yml](./values.yml) in this repo.

### Upgrade the Cluster

The upgrade process is one the most tricky process in k8s. You have to pay caution and follow the best practices and steps.

#### First Master

1. Install the kubeadm on the first master: (The selected version)
2. check the `kubeadm upgrade plan` and if there is no manual installation needed continue the steps.
3. Check the `kubeadm upgrade` in dry-run command and check the result.
4. run the `kubeadm upgrade apply -f {{kubernetes_version}}
5. Install the new kubelet package and restart the kubelet
6. Install the new kubectl package

#### Other masters

1. Install the kubeadm on the first master: (The selected version)
2. run the `kubeadm upgrade node`
3. Install the new kubelet package and restart the kubelet
4. Install the new kubectl package

#### Workers

1. Drain the worker node
2. Install the kubeadm on the first master: (The selected version)
3. run the `kubeadm upgrade node`
4. Install the new kubelet package and restart the kubelet
5. Install the new kubectl package
6. Uncordon the node

This process has been automated and you can use the following roles:

- [kubernetes-upgrade](./ansibles/roles/kubernetes-upgrade)
- [kubernetes-upgrade-masters](./ansibles/roles/kubernetes-upgrade-masters)
- [kubernetes-upgrade-workers](./ansibles/roles/kubernetes-upgrade-workers)

## Challenges

### Gateways

At first, I wanted to have a `virtual private IP` which can be handle by the openstack. And I wanted to assign a `floating IP` to the virtual ip port to have the internet via this port. But Because of the  [bug](https://bugzilla.redhat.com/show_bug.cgi?id=1707241) in the OVN  module of openstack my scenario has failed. So I had to assing two fixed public IP to each nodes and assing a `virtual public IP` to my gateway nodes to  handle the HA and config the keepalived.

![image-20220714010525364](./pictures/ovn-bug.png)

#### What's wrong?

All the configuration was successfully done but at the last part the floating ip that has been assinged to the virtual ip port had no ping and connection.

### etcd

One of the most hard challenges I have was the etcd TLS authentication. I tried to issued the certs with `openSSL` but the certs had not worked. Related to the [github issue](https://github.com/etcd-io/etcd/issues/8603) these problems caused because of the lack of good etcd documentation.

After many research and tests I figured out the problems:

- **Key_usage:** I need to add the `keyEncipherment` key_usage to the server csr. Related to the serveruser website the keyEncipherment is:

  > Certificate may be used to encrypt a symmetric key which is then transferred to the target.
  >
  > Target decrypts key, subsequently using it to encrypt & decrypt data between the entities

- **Extended_key_usage:** This flag is additional restrictions need to add to the certs. For server cert should be `serverAuth` , client certs should be `clientAuth` and for peer certs should be both

- **Hostname is SAN:** There is a section in csr which you can specify the hostname and IPs. I just used the IP variable.

Thanks to [this](https://medium.com/nirman-tech-blog/setting-up-etcd-cluster-with-tls-authentication-enabled-49c44e4151bb) article which helps me a lot to troubleshoot.

### Containerd

For pulling images, because of sanctions we need to use proxy. It can be used proxy in the containerd service as an environment:

```shell
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999
Environment="HTTP_PROXY=http://<proxy_ip:<port>/"
Environment="HTTPS_PROXY=http://<proxy_ip:<port>/"
```

But this environment set proxy for the whole cluster! So I got errors while I deployed the calico and the calico node can't communicate to the other components.

So I solved this problem by adding `NO_PROXY`:

```SHELL
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999
Environment="HTTP_PROXY=http://<proxy_ip:<port>/"
Environment="HTTPS_PROXY=http://<proxy_ip:<port>/"
Environment="NO_PROXY=localhost,127.0.0.1,10.244.0.0/16,192.168.0.0/24,10.96.0.1/32"
```

**Note:** The `10.96.0.1` was the IP calico couldn't connect to it.

### Storage

I deployed the kubernetes local-storage provisioner and everything was fine. I added 10 GB disk to the worker node and the provisioner has detected the disk automatically and create a new 10 GB PV.

I tried to deploy very simple nginx with PVC but the the pod stuck on the `pending` state because there is no persistent volume availabe on every node.

At last I understood that the problem was about the capacity of PV which was 1110Mi and not 10Gi!

### Upgrade k8s

After upgrading the first master and everything goes fine, some pod didn't deleted and cluster takes some minutes to delete these pods and pulling the new images.

## Questions

- Is TLS needed for the etcd? Is it best practice?
- Is needed to drain master nodes for upgrading?

## To do list

- Monitoring

- Auto etcd backup

- Auto etcd disaster recovery

- Install StorageClaas / provisioner / Helm via ansible

- Password Clear-text ansible files(Using vault)

- Comments in playbooks and doc for roles

  
