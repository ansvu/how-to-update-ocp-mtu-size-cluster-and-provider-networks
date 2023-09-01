# how-to-update-ocp-mtu-size-cluster-and-provider-networks
This Repo will show how to update or migrate the OpenShift MTU Size for cluster and provider networks to Jumbo Frame.

For Provider Network MTU Size update to Jumbo Frame, it will come later. 

## Prerequisites
- You installed the OpenShift CLI (oc).
- Butane installed  
  [butane](https://coreos.github.io/butane/getting-started/)
- You are logged in to the cluster with a user with cluster-admin privileges.
- You identified the target MTU for your cluster. The correct MTU varies depending on the network plugin that your cluster uses
- OVN-Kubernetes: The cluster MTU must be set to 100 less than the lowest hardware MTU value in your cluster.
- current cluster-netwok-mtu=1400, overlay_to=9000, machine_to=9100, ethernet.mtu=<machine_to>

**Note**: Tested with OVN network type only but it should work with SDN also with slightly different configuration.

## To obtain the current MTU for the cluster network, enter the following command
- Get Current Network Config Cluster
```yaml
$ oc describe network.config cluster
Name:         cluster
API Version:  config.openshift.io/v1
Kind:         Network
Spec:
  Cluster Network:
    Cidr:         10.132.0.0/14
    Host Prefix:  23
  External IP:
    Policy:
  Network Type:  OVNKubernetes
  Service Network:
    172.30.0.0/16
Status:
  Cluster Network:
    Cidr:               10.132.0.0/14
    Host Prefix:        23
  Cluster Network MTU:  1400
  Network Type:         OVNKubernetes
  Service Network:
    172.30.0.0/16
```
## Prepare your configuration for the hardware MTU
- Find the primary network interface for OVN Network Type
```shellSession
$ oc debug node/nokiavf.hubcluster-1.lab.eng.cert.redhat.com -- chroot /host nmcli -g connection.interface-name c show ovs-if-phys0
Temporary namespace openshift-debug-wkxp8 is created for debugging node...
Starting pod/nokiavfhubcluster-1labengcertredhatcom-debug ...
To use host binaries, run `chroot /host`
eno5
```
- Create the following NetworkManager configuration in the <interface>-mtu.conf file
```shellSession
$ cat eno5-mtu.conf 
[connection-eno5-mtu]
match-device=interface-name:eno5
ethernet.mtu=9100
```
- Create the following Butane config in the control-plane-interface.bu file:
```yaml
$ cat control-plane-interface.bu 
variant: openshift
version: 4.11.0
metadata:
  name: 01-control-plane-interface
  labels:
    machineconfiguration.openshift.io/role: master
storage:
  files:
    - path: /etc/NetworkManager/conf.d/99-eno5-mtu.conf
      contents:
        local: eno5-mtu.conf
      mode: 0600
```
- Create the following Butane config in the worker-interface.bu file
```shellSession
$ cat worker-interface.bu 
variant: openshift
version: 4.11.0
metadata:
  name: 01-worker-interface
  labels:
    machineconfiguration.openshift.io/role: worker
storage:
  files:
    - path: /etc/NetworkManager/conf.d/99-eno5-mtu.conf
      contents:
        local: eno5-mtu.conf
      mode: 0600
```
- Create MachineConfig objects from the Butane configs by running the following command
```bash
for manifest in control-plane-interface worker-interface; do
    butane --files-dir . $manifest.bu > $manifest.yaml
done
```
## To begin the MTU migration, specify the migration configuration by entering the following command. 
The Machine Config Operator performs a rolling reboot of the nodes in the cluster in preparation for the MTU change.
```shellSession
$ oc patch Network.operator.openshift.io cluster --type=merge --patch \
  '{"spec": { "migration": { "mtu": { "network": { "from": 1400, "to": 9000 } , "machine": { "to" : 9100} } } } }'
network.operator.openshift.io/cluster patched

$ oc describe node | egrep "hostname|machineconfig"
                    kubernetes.io/hostname=nokiavf.hubcluster-1.lab.eng.cert.redhat.com
                    machineconfiguration.openshift.io/controlPlaneTopology: SingleReplica
                    machineconfiguration.openshift.io/currentConfig: rendered-master-81e581dccfe0d43cb193387606d1abf4
                    machineconfiguration.openshift.io/desiredConfig: rendered-master-81e581dccfe0d43cb193387606d1abf4
                    machineconfiguration.openshift.io/desiredDrain: uncordon-rendered-master-81e581dccfe0d43cb193387606d1abf4
                    machineconfiguration.openshift.io/lastAppliedDrain: uncordon-rendered-master-81e581dccfe0d43cb193387606d1abf4
                    machineconfiguration.openshift.io/reason: 
                    machineconfiguration.openshift.io/state: Done
```
- To confirm that the machine config is correct, enter the following command
```shellSession
$ oc get machineconfig rendered-master-81e581dccfe0d43cb193387606d1abf4 -o yaml | grep ExecStart
          ExecStart=/usr/local/bin/mtu-migration.sh
```
- Check network config cluster status
```yaml
$ oc describe network.config cluster
Name:         cluster
API Version:  config.openshift.io/v1
Kind:         Network
Spec:
  Cluster Network:
    Cidr:         10.132.0.0/14
    Host Prefix:  23
  External IP:
    Policy:
  Network Type:  OVNKubernetes
  Service Network:
    172.30.0.0/16
Status:
  Cluster Network:
    Cidr:               10.132.0.0/14
    Host Prefix:        23
  Cluster Network MTU:  9000
  Migration:
    Mtu:
      Machine:
        To:  9100
      Network:
        From:    1400
        To:      9000
  Network Type:  OVNKubernetes
  Service Network:
    172.30.0.0/16
```

## Update the underlying network interface MTU value:
If you are specifying the new MTU with a NetworkManager connection configuration, enter the following command. The MachineConfig Operator automatically performs a rolling reboot of the nodes in your cluster.  

- Start apply controlplane and worker interface MCP Update  
```bash
for manifest in control-plane-interface worker-interface; do
    oc create -f $manifest.yaml
done
machineconfig.machineconfiguration.openshift.io/01-control-plane-interface created
machineconfig.machineconfiguration.openshift.io/01-worker-interface created
```
- CHeck MCP Update State
```shellSession
$ oc describe node | egrep "hostname|machineconfig"
                    kubernetes.io/hostname=nokiavf.hubcluster-1.lab.eng.cert.redhat.com
                    machineconfiguration.openshift.io/controlPlaneTopology: SingleReplica
                    machineconfiguration.openshift.io/currentConfig: rendered-master-f56880b935a0e6798b7064f128ddd6bb
                    machineconfiguration.openshift.io/desiredConfig: rendered-master-f56880b935a0e6798b7064f128ddd6bb
                    machineconfiguration.openshift.io/desiredDrain: uncordon-rendered-master-f56880b935a0e6798b7064f128ddd6bb
                    machineconfiguration.openshift.io/lastAppliedDrain: uncordon-rendered-master-f56880b935a0e6798b7064f128ddd6bb
                    machineconfiguration.openshift.io/reason: 
                    machineconfiguration.openshift.io/ssh: accessed
                    machineconfiguration.openshift.io/state: Done

$ oc get machineconfig rendered-master-f56880b935a0e6798b7064f128ddd6bb -o yaml | grep path:
$ oc get machineconfig rendered-master-f56880b935a0e6798b7064f128ddd6bb -o yaml | grep path:|grep mtu
        path: /usr/local/bin/mtu-migration.sh
        path: /etc/NetworkManager/conf.d/99-eno5-mtu.conf
```

## To finalize the MTU migration, enter one of the following commands
If you are using the OVN-Kubernetes cluster network provider:
`overlay_to: mtu=9000` 
```shellSession
$ oc patch Network.operator.openshift.io cluster --type=merge --patch \
  '{"spec": { "migration": null, "defaultNetwork":{ "ovnKubernetesConfig": { "mtu": 9000 }}}}'
network.operator.openshift.io/cluster patched
```
**Note**: It will reboot the node again

- Check MCP Update state
```shellSession
$ oc describe node | egrep "hostname|machineconfig"
                    kubernetes.io/hostname=nokiavf.hubcluster-1.lab.eng.cert.redhat.com
                    machineconfiguration.openshift.io/controlPlaneTopology: SingleReplica
                    machineconfiguration.openshift.io/currentConfig: rendered-master-357789291e119dbaf01416a1711d0d10
                    machineconfiguration.openshift.io/desiredConfig: rendered-master-357789291e119dbaf01416a1711d0d10
                    machineconfiguration.openshift.io/desiredDrain: uncordon-rendered-master-357789291e119dbaf01416a1711d0d10
                    machineconfiguration.openshift.io/lastAppliedDrain: uncordon-rendered-master-357789291e119dbaf01416a1711d0d10
                    machineconfiguration.openshift.io/reason: 
                    machineconfiguration.openshift.io/ssh: accessed
                    machineconfiguration.openshift.io/state: Done

$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-357789291e119dbaf01416a1711d0d10   True      False      False      1              1                   1                     0                      85m
worker   rendered-worker-1938883ce27495135634ffe4ab0cbb59   True      False      False      0              0                   0                     0                      85m

$ oc get po -A|grep -vE '(Completed|Running)'
NAMESPACE      NAME        READY   STATUS      RESTARTS         AGE

$ oc get no
NAME                                           STATUS   ROLES                         AGE   VERSION
nokiavf.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   90m   v1.25.7+eab9cc9

$ ip a sh |egrep 'k8s-mp|eno5|br-ex'|grep mtu
9: eno5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9100 qdisc mq master ovs-system state UP group default qlen 1000
22: ovn-k8s-mp0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UNKNOWN group default qlen 1000
23: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9100 qdisc noqueue state UNKNOWN group default qlen 1000

$ oc get po -A|grep ovn
openshift-ovn-kubernetes    ovnkube-node-nwm8p                                                      5/5     Running     15               84m

$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.12.8    True        False         False      22m     
baremetal                                  4.12.8    True        False         False      69m     
cloud-controller-manager                   4.12.8    True        False         False      69m     
cloud-credential                           4.12.8    True        False         False      74m     
cluster-autoscaler                         4.12.8    True        False         False      69m     
config-operator                            4.12.8    True        False         False      83m     
console                                    4.12.8    True        False         False      59m     
control-plane-machine-set                  4.12.8    True        False         False      69m     
csi-snapshot-controller                    4.12.8    True        False         False      74m     
dns                                        4.12.8    True        False         False      42m     
etcd                                       4.12.8    True        False         False      80m     
image-registry                             4.12.8    True        False         False      64m     
ingress                                    4.12.8    True        False         False      68m     
insights                                   4.12.8    True        False         False      69m     
kube-apiserver                             4.12.8    True        False         False      65m     
kube-controller-manager                    4.12.8    True        False         False      66m     
kube-scheduler                             4.12.8    True        False         False      67m     
kube-storage-version-migrator              4.12.8    True        False         False      76m     
machine-api                                4.12.8    True        False         False      69m     
machine-approver                           4.12.8    True        False         False      69m     
machine-config                             4.12.8    True        False         False      74m     
marketplace                                4.12.8    True        False         False      74m     
monitoring                                 4.12.8    True        False         False      63m     
network                                    4.12.8    True        False         False      83m     
node-tuning                                4.12.8    True        False         False      69m     
openshift-apiserver                        4.12.8    True        False         False      3m11s   
openshift-controller-manager               4.12.8    True        False         False      65m     
openshift-samples                          4.12.8    True        False         False      65m     
operator-lifecycle-manager                 4.12.8    True        False         False      69m     
operator-lifecycle-manager-catalog         4.12.8    True        False         False      69m     
operator-lifecycle-manager-packageserver   4.12.8    True        False         False      65m     
service-ca                                 4.12.8    True        False         False      83m     
storage                                    4.12.8    True        False         False      69m     

$ oc describe network.config cluster
Name:         cluster
API Version:  config.openshift.io/v1
Kind:         Network
Spec:
  Cluster Network:
    Cidr:         10.132.0.0/14
    Host Prefix:  23
  External IP:
    Policy:
  Network Type:  OVNKubernetes
  Service Network:
    172.30.0.0/16
Status:
  Cluster Network:
    Cidr:               10.132.0.0/14
    Host Prefix:        23
  Cluster Network MTU:  9000
  Network Type:         OVNKubernetes
  Service Network:
    172.30.0.0/16
```

## Update MTU Size For Provider Networks With NMState
## Deploy NMState Operator 
- create NMState CR 
```yaml
create-nmstate-cr.yaml:
Version: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
```
```shellSession
$ oc apply -f create-nmstate-cr.yaml
$ oc -n openshift-nmstate get po
NAME                                   READY   STATUS    RESTARTS      AGE
nmstate-cert-manager-66c975c97-v8pml   1/1     Running   2             111m
nmstate-handler-h9mng                  1/1     Running   4 (17m ago)   111m
nmstate-operator-fccffbb5-whvp4        1/1     Running   2             119m
nmstate-webhook-56d99bcfbb-6r8hl       1/1     Running   2             111m
```

## Start Update Provider networks interface 
### Deploy NMState Operator 
Deploy NMState from Console GUI  
- create NMState CR
```yaml
create-nmstate-cr.yaml:
Version: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
```
```shellSession
$ oc apply -f create-nmstate-cr.yaml
$ oc -n openshift-nmstate get po
NAME                                   READY   STATUS    RESTARTS      AGE
nmstate-cert-manager-66c975c97-v8pml   1/1     Running   2             111m
nmstate-handler-h9mng                  1/1     Running   4 (17m ago)   111m
nmstate-operator-fccffbb5-whvp4        1/1     Running   2             119m
nmstate-webhook-56d99bcfbb-6r8hl       1/1     Running   2             111m
```

## Start Update Provider networks interface 
Get a list of provider network interfaces from worker or master nodes you want to update the MTU size to Jumbo Frame.

- Prepare NodeNetworkConfigurationPolicy Config 
```yaml
$ cat test-two-nmstate.yaml 
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: mtu-network-nncp 
spec:
  nodeSelector: 
    node-role.kubernetes.io/master: "" 
  desiredState:
    interfaces:
    - name: ens5f0
      type: ethernet
      state: up 
      ipv4:
        enabled: false
      mtu: 9000
    - name: ens5f1
      type: ethernet
      state: up 
      ipv4:
        enabled: false
      mtu: 9000
```
```shellSession
$ oc apply -f test-two-nmstate.yaml
```
- Check NNCE Status 
```shellSession
$ oc get nnce
NAME                                                            STATUS      REASON
nokiavf.hubcluster-1.lab.eng.cert.redhat.com.mtu-network-nncp   Progressing   ConfigurationProgressing
nokiavf.hubcluster-1.lab.eng.cert.redhat.com.mtu-network-nncp   Available   SuccessfullyConfigured
```
- Verify MTU Size from the node  
```shellSession
ssh -i ~/.ssh/id_rsa core@192.168.24.111 sudo ip a sh|grep ens5
18: ens5f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
19: ens5f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000

$ nmcli c s
NAME                 UUID                                  TYPE           DEVICE 
ens5f0               dc94ad27-48f4-42b9-8cca-a70d82f2de94  ethernet       ens5f0 
ens5f1               89721b93-4f6e-449a-b5aa-4e91a9fc5fef  ethernet       ens5f1 
```
## Update Provider Networks MTU Jumbo Frame with OCP MachineConfig
```yaml
99-master-mtu-multiple-interfaces.yaml:
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-worker-mtu
spec:
  config:
    ignition:
      version: 3.2.0
    storage: {}
    systemd:
      units:
        - contents: |
            [Unit]
            Description=Set MTU To Jumbo Frame
            After=network-online.target
            [Service]
            Type=oneshot
            ExecStart=ip link set ens2f0 mtu 9000 ; ip link set ens2f1 mtu 9000 ; ip link set ens4f0 mtu 9000 ; ip link set ens4f1 mtu 9000
            [Install]
            WantedBy=multi-user.target
          enabled: true
          name: one-shot-mtu.service
  osImageURL: ""
```
```shellSession
$ 99-master-mtu-multiple-interfaces.yaml
$ oc describe node | egrep "hostname|machineconfig"|grep state:
                    machineconfiguration.openshift.io/state: Done

$ ssh -i ~/.ssh/id_rsa core@192.168.24.111 sudo ip a sh|egrep 'ens4f[0-1]|ens2f[0-1]'
4: ens2f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
6: ens2f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
14: ens4f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
15: ens4f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
```

## Ping Jumbo Frame Testing 
- Create two test PODs  
```shellSession
$ ocj get po
NAME             READY   STATUS    RESTARTS   AGE
test-ping-pod1   1/1     Running   0          6m45s
test-ping-pod2   1/1     Running   0          6m2s
```
- Use crictl to container PID ID  
```shellSession
$ crictl inspect 399d9960917e7|jq -r '.info.pid'
131734
```
- Check Secondary interface of Multus
```shellSession 
$ nsenter -t 131734 -n -- ip a
2: eth0@if106: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP group default 
    link/ether 0a:58:0a:84:00:4f brd ff:ff:ff:ff:ff:ff link-netns be0913f0-ef01-4989-b5b3-3aabfe633297
    inet 10.132.0.79/23 brd 10.132.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::858:aff:fe84:4f/64 scope link 
       valid_lft forever preferred_lft forever
3: net1@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP group default 
    link/ether 4e:65:cc:75:f6:63 brd ff:ff:ff:ff:ff:ff link-netns be0913f0-ef01-4989-b5b3-3aabfe633297
    inet 192.168.150.1/24 brd 192.168.150.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::4c65:ccff:fe75:f663/64 scope link 
       valid_lft forever preferred_lft forever
```
- Ping Jumbo Frame Packet Size  
```shellSession
$ nsenter -t 131734 -n -- ping -s 8000 192.168.150.2
PING 192.168.150.2 (192.168.150.2) 8000(8028) bytes of data.
8008 bytes from 192.168.150.2: icmp_seq=1 ttl=64 time=0.069 ms
8008 bytes from 192.168.150.2: icmp_seq=2 ttl=64 time=0.066 ms
8008 bytes from 192.168.150.2: icmp_seq=3 ttl=64 time=0.054 ms
8008 bytes from 192.168.150.2: icmp_seq=4 ttl=64 time=0.060 ms
8008 bytes from 192.168.150.2: icmp_seq=5 ttl=64 time=0.063 ms
8008 bytes from 192.168.150.2: icmp_seq=6 ttl=64 time=0.062 ms
8008 bytes from 192.168.150.2: icmp_seq=7 ttl=64 time=0.059 ms
8008 bytes from 192.168.150.2: icmp_seq=8 ttl=64 time=0.062 ms
8008 bytes from 192.168.150.2: icmp_seq=9 ttl=64 time=0.050 ms
8008 bytes from 192.168.150.2: icmp_seq=10 ttl=64 time=0.060 ms
--- 192.168.150.2 ping statistics ---
10 packets transmitted, 10 received, 0% packet loss, time 9196ms
rtt min/avg/max/mdev = 0.050/0.060/0.069/0.009 ms

$ nsenter -t 131734 -n -- ping -s 9000 192.168.150.2
PING 192.168.150.2 (192.168.150.2) 9000(9028) bytes of data.
9008 bytes from 192.168.150.2: icmp_seq=1 ttl=64 time=0.053 ms
9008 bytes from 192.168.150.2: icmp_seq=2 ttl=64 time=0.075 ms
9008 bytes from 192.168.150.2: icmp_seq=3 ttl=64 time=0.083 ms
--- 192.168.150.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2037ms
rtt min/avg/max/mdev = 0.053/0.070/0.083/0.014 ms
```

## Troubleshooting
If something goes wrong, then check this pod logs
```shellSession
$ oc -n openshift-ovn-kubernetes logs ovnkube-node-nl8wr -c ovnkube-node -f
$ oc -n openshift-machine-config-operator logs machine-config-daemon-tkxvx -f
```

## Links
- Official OpenShift Document--> Click [Migrate Cluster Network MTU Size](https://docs.openshift.com/container-platform/4.12/networking/changing-cluster-network-mtu.html)  

- MTU Size issue article helper [MTU issues](https://access.redhat.com/solutions/5736191)  
- Optimize MTU [optimizing-networking](https://docs.openshift.com/container-platform/4.12/scalability_and_performance/optimization/optimizing-networking.html)
