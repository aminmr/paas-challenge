# PaaS-challenge

## Architecture

### Nodes

**Gateways: **2 gateway node with shared IP and HA with keepalived

**Ansible-master:** A control node to manage all nodes and execute the ansible playbooks.

**ETCD:** 3 nodes for etcd cluster with etcd v3.4.7

**K8S:** 3 master nodes + 1 worker node

### Diagram

### Gateways

The gateways are the communication way to the nodes. In this architecture, All nodes has private network. So there is no way accessing nodes unless the gateway. In other way, all nodes can access the internet via NAT protocol through the gateway node. So as we see, the gateway could be our point of failures. So we need to HA the gateway node and make it as high available as possible. For this purpose we could use the `keepalived`. 

The other role of gateways are:

- Loadbalancing requests between the kubernetes master nodes.
- Loadbalancing requests from the user to the cluster.

### ETCD

#### ETCD Hardware requirement

​	Related to the etcd official doc about the [hardware requirement](https://etcd.io/docs/v3.3/op-guide/hardware/) for small cluster we need the following hardware requirement:

![image-20220707185025194](./pictures/etcd-hardware-req.png)



The cluster has been initialized with three nodes and TLS authentication. All certificates has been issued by `OpenSSL`

### ETCD disaster recovery

### K8S cluster

### Challenges

#### Gateways

At first, I wanted to have a `virtual private IP` which can be handle by the openstack. And I wanted to assign a `floating IP` to the virtual ip port to have the internet via this port. But Because of the  [bug](https://bugzilla.redhat.com/show_bug.cgi?id=1707241) in the OVN  module of openstack my scenario has failed. So I had to assing two fixed public IP to each nodes and assing a `virtual public IP` to my gateway nodes to  handle the HA and config the keepalived.

![image-20220714010525364](/home/amin/paas-challenge/pictures/ovn-bug.png)

##### What's wrong?

All the configuration was successfully done but at the last part the floating ip that has been assinged to the virtual ip port had no ping and connection.

### Refreneces

