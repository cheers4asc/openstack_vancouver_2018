### <br> 

**Unified networking using Calico **Labs 

1.  *Openstack networking Lab *
1.  *Openstack FloatingIP in Calico*
1.  *Install k8s and Calico for k8s*
1.  *Deploy App on K8s using Calico network *
1.  *Network policies *
1.  *calicoctl cli *
1.  *BGP Peering *

*****

### Lab Prereqs 

Need Laptop with 6 GB free RAM, 2 cores allocated to virtualbox<br> Disk
required is 8 GB ( vbox has capacity to grow to 40 GB)<br> Virtualbox release
5.2.10 + required

vbox image provided is installed with RDO Openstack and Calico v3.1 for
openstack . 

![](https://cdn-images-1.medium.com/max/800/1*TCjWj9tvNXFi831NVgri-g.png)

You will have to run setup.sh scripts after booting this VM . ( root/centos )
Make sure that status of following command shows felix agent is alive .

    source keystonerc_admin
    openstack network agent list

    +--------------------------------------+----------------------------

    | ID                                   | Agent Type                    | Host   | Availability Zone | Alive | State | Binary       |

    ---+--------+-------------------+-------+-------+--------------+

    | 45b72ee4-800b-4ba0-afa2-2ca1ee4169a7 | Calico per-host agent (felix) | myhost | None              | 
       | UP    | calico-felix |

    +--------------------------------------+----------------------------

<br> 

<br> 

<br> 

### Lab 1 :- Openstack networking Lab

a. Login as root user and source in keystonerc_admin file . 

b. make sure you can probe the env and nake sure that you see valid working
output 

    openstack network agent list

c. Create two networks using neutron API

    openstack network create --share --provider-network-type local internal 
    openstack network create --share --provider-network-type local external

d. create subnets 

    openstack subnet create --gateway 10.0.0.1 --ip-version 4 --subnet-range 10.0.0.0/24 --network internal calico-internal
    openstack subnet create --gateway 172.24.4.1 --ip-version 4 --subnet-range 172.24.4.0/24 --network external  calico-external

e. create new VM on those network 

    openstack server create --flavor m1.tiny --image cirros --nic net-id=<UUID of internal network > int-vm
    openstack server create --flavor m1.tiny --image cirros --nic net-id=<UUID of external network >  ext-vm

f.you should see output such as below when you get list of VM 

    # openstack server list

    + — — — — — — — — — — — — — — — — — — — + — — — — + — — — — + — — — 

    | ID | Name | Status | Networks | Image | Flavor |

    + — — — — — — — — — — — — — — — — — — — + — — — — + — — — — + — — — 

    | 5932feb1–9238–43d3–81ee-c90bb6fd10a6 | int-vm | ACTIVE | internal=10.0.0.5 | cirros | m1.tiny |

    | 345bd377–685c-42a6-abe5-c7123fd8dc79 | ext-vm | ACTIVE | external=172.24.4.4 | cirros | m1.tiny |

<br> 

openstack security groups ‘defaults’ is currently open for all tcp communication
. 

if you ssh to one of VM deployed , you should be able to ping /ssh to another VM
( in example below assumption is internal network VM IP is 10.0.0.4 and external
network IP is 172.24.4.4 

    ssh cirros@10.0.0.4 ( password is “
    ) 

    >ping 172.24.4.4
    PING 172.24.4.4 (172.24.4.4): 56 data bytes
    64 bytes from 172.24.4.4: seq=0 ttl=63 time=3.669 ms
    64 bytes from 172.24.4.4: seq=1 ttl=63 time=1.607 ms

*****

### Lab2. openstack floatingip setup and communication .  

networking-calico includes beta support for floating IPs. Currently this
requires running Calico as a Neutron core plugin (i.e. `core_plugin = calico`)
instead of as an ML2 mechanism driver.

lets look at routing table , you should see tap device with your VM IP . 


lets create a external network and subnet from where we will allocate floating
IP 


now lets create a route connecting the tenant ( internal network that we created
in Lab) and external network . 



lets now create floating IP and associate with target VM , here we are creating
a floating IP and associating with IP address of int-vm ( you need to do neutron
port-list to get UUID of that port ) 



now if you look at ipnat rules it will show you how Calico agents arrange
floating IP is routed to the instance’s compute host, and then DNAT’d to the
instance’s fixed

    ......
    ......
    Chain cali-fip-dnat (2 references)
    pkts bytes target prot opt in out source destination
    0 0 DNAT all — * * 0.0.0.0/0 
     /* cali:2MtBEAz79X4sNqk5 */ to:

    Chain cali-fip-snat (1 references)
    pkts bytes target prot opt in out source destination
    0 0 SNAT all — * * 10.0.0.5 10.0.0.5 /* cali:lCv2pcqmysqMU1r4 */ 
    to:172.16.1.7

now you can try to ssh to floating IP and see its connected to int-vm

    # ssh cirros@172.16.1.7
    cirros@172.16.1.7’s password:

    #ip r
    default via 10.0.0.1 dev eth0
    10.0.0.0/24 dev eth0  src 10.0.0.5

*****

### Lab3. Install K8s and calico profile

a. restart the kubelet process using systemctl cli 

    systemctl restart kubelet

b . go to /root/labs/k8s folder 

we have a config file where we are telling k8s to use existing etcd that we used
for Openstack . 

    kubeadm init --config ./config 

Once installation is successful , run the following command which will setup
envs for kubectl cli . 

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

Taint the node so that we can use this node of k8s as a node to deploy pods 

    kubectl taint nodes --all node-role.kubernetes.io/master-

Run kubectl cli to check status of node , you will notice the status of
“NotReady” 

    NAME STATUS ROLES AGE VERSION
    myhost NotReady master 2m v1.10.2

if . you run describe command it will show you that node is in not ready status
bbecuase network plugin is not setup . 



we will go ahead and install Calico network . <br> pause for a second and review
calico.yaml 

    kubectl apply -f ./calico.yaml

if you check status of node now it should show all of the containers in
kube-system running . ( This step will take appx 3 mins or so to complete) <br>
<br> 

    calico-etcd-9dc9  
    2

*****

**Lab4 :- Deploy App on Calico **

a. Deploy a Sample nginx app on K8s . <br> You have been provided with sample
file name run-my-nginx.yaml in /root/labs/k8s folder

Deploy app using kubectl .

    # 

b.get deployment status 


c. get status of pods 


d . Get detailed output to see the ip address of the pod . ip address should
match with the subnet that has been provided while installing Calico . 


## show that Openstack VM can reach out to this 

*****

**LAB5 :- Network policies **

This Lab is Star policies demo as documented by[ calico website
](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/stars-policy/).
This Lab will help you understand how the network policies works . 

a)Create the frontend, backend, client, and management-ui apps<br>  go to
/root/labs/k8s/net-policy folder 

Run following command (** This steps pull image from docker.io/calico/star-probe
, external connectivity is required and 300 secs time out is there ) 

    kubectl create -f 00-namespace.yaml

    kubectl create -f 01-management-ui.yaml

    kubectl create -f 02-backend.yaml

    kubectl create -f 03-frontend.yaml

    kubectl create -f 04-client.yaml

Wait for all the pods to enter `Running` state.

You can monitor using following command , This step will take a while .

    kubectl get pods -n stars --watch

Management U I which runs as Node Port can be viewed by going to browser
[http://localhost:30002](http://localhost:30002/) 

b) following commands will prevent all access to the frontend, backend, and
client Services

    default-deny.yaml
    default-deny.yaml

refresh GUI on your browser to confirm isolation ( browser will not show correct
results as its cannot connect to Pod) 

c) allow UI to access services 

    allow-ui-client.yaml

After a few seconds, refresh the UI — it should now show the Services, but they
should not be able to access each other any more.

d)Run following command to allow traffic from the frontend to the backend<br>
<br> 


Refresh the UI. You should see the following:

* The frontend can now access the backend (on TCP port 80 only).
* The backend cannot access the frontend at all.
* The client cannot access the frontend, nor can it access the backend.

e) we will expose the frontend service to the `client` namespace


The client can now access the frontend, but not the backend. Neither the
frontend nor the backend can initiate connections to the client. The frontend
can still access the backend.

*****

**Lab6:- Route Reflector and BGP peers **

This one is study lab as you need multiple node setup for this lab exercise . 

The purpose of this lab exercise is to understand how you will connect multiple
compute node using Router Reflector and also what type of BGP peering is
supported by Calico . 

<br> 
