# High-Availability-Kubernetes-Cluster-

Kubernetes Deployment on-premises for production environment it is recommended to deploy with High Availability:

OS and hardware requirements:

**Step 1: Nodes:**
master-node-1 minimal 2GB RAM, 2vCPU with 40GB Disk.  
master-node-2 minimal 2GB RAM, 2vCPU with 40GB Disk.  
master-node-3 minimal 2GB RAM, 2vCPU with 40GB Disk.  
worker-node-1 minimal 2GB RAM, 2vCPU with 40GB Disk.    
worker-node-2 minimal 2GB RAM, 2vCPU with 40GB Disk.  
worker-node-3 minimal 2GB RAM, 2vCPU with 40GB Disk.

**Step 2:** Firewall setup   
**Step 3:** Installation of Docker (CRI)  
**Step 4:** Installation of kubeadm, kubectl, kubelet on all nodes.  
**Step 5:** Initialize the k8s cluster from a master node  
**Step 6:** Networking between the nodes (Master & worker)  
**Step 7:** High Availability k8 cluster.  

### Step 1:

1.	Setup ` nodes` with appropriate names in file `/etc/hosts` file with `example: master-node-1` or other way with command line `hostnamectl set-hostname “master-node-1` if not working shift to root to change the name.  
a.	master-node-1  
b.	master-node-2  
c.	master-node-3  
d.	worker-node-1  
e.	worker-node-2  
f.	worker-node-3  

2.	Configure the ip address for these nodes.  
3.	Install Keepalive and HAProxy on all master-nodes. run `sudo yum install haproxy keepalived -y` and follow steps to configure the script files.   
o	run `sudo vi /etc/keepalived/checl_apiserver.sh` update the file.  
o	allow permissions `sudo chmod +x /etc/keepalived/checl_apiserver.sh`  
o	run `sudo cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf-org` to take the backup of the file. Run`sudo sh -c '> /etc/keepalived/keepalived.conf`  
o	run `vi /etc/keepalived/keepalived.conf ` update the file save & close it.  

o	Make sure you to the same on master nodes. other changes needed is  

   **priority** number to be decreased with one number (255 to 254) respectively.  
   **State** parameter to be changed from `Master` to `Slave` for other master nodes.  
4.	Run `sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg-org`  
5.	Run `sudo vi /etc/haproxy/haproxy.cfg`  

#---------------------------------------------------------------------  
# apiserver frontend which proxys to the masters  
#---------------------------------------------------------------------  
frontend apiserver  
    bind *:8443    
    mode tcp  
    option tcplog  
    default_backend apiserver  
#---------------------------------------------------------------------  
# round robin balancing for apiserver  
#---------------------------------------------------------------------  
backend apiserver  
    option httpchk GET /healthz  
    http-check expect status 200  
    mode tcp  
    option ssl-hello-chk  
    balance     roundrobin  
        server k8s-master-1 192.168.1.40:6443 check  
        server k8s-master-2 192.168.1.41:6443 check  
        server k8s-master-3 192.168.1.42:6443 check  
  

6.	copy these files same as to master-node-2 and master-node-3. Steps 2 to 5.  
  
for f in k8s-master-2 k8s-master-3; do scp /etc/keepalived/check_apiserver.sh /etc/keepalived/keepalived.conf root@$f:/etc/keepalived; scp /etc/haproxy/haproxy.cfg root@$f:/etc/haproxy; done  

#### **Step 2**  firewall:  
  
o	Setup firewall on all master-nodes:  
`sudo firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' –permanent`  
`sudo firewall-cmd --permanent --add-port=8443/tcp`  
`sudo firewall-cmd --reload`  

o	Start and enable keepalived and haproxy services on all master nodes.  
`sudo systemctl enable keepalived –now`  
`sudo systemctl enable haproxy –now`  
o	Verify the ip address of the master nodes run `ip a s`  


**Disable Swap and set SEliux permissions**  

o	Run the below command on all nodes to disable swap.  
`sudo swapoff -a `  
`sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab`  
o	Permission to SElinux on all nodes.  
`sudo setenforce 0`  
`sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config`  

#### setting up the firewall rules on master-nodes:  

$ sudo firewall-cmd --permanent --add-port=6443/tcp  
$ sudo firewall-cmd --permanent --add-port=2379-2380/tcp  
$ sudo firewall-cmd --permanent --add-port=10250/tcp  
$ sudo firewall-cmd --permanent --add-port=10251/tcp  
$ sudo firewall-cmd --permanent --add-port=10252/tcp  
$ sudo firewall-cmd --permanent --add-port=179/tcp  
$ sudo firewall-cmd --permanent --add-port=4789/udp  
$ sudo firewall-cmd --reload  
$ sudo modprobe br_netfilter  
$ sudo sh -c "echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables"  
$ sudo sh -c "echo '1' > /proc/sys/net/ipv4/ip_forward"  

o	On Worker nodes allow following ports:  
  
$ sudo firewall-cmd --permanent --add-port=10250/tcp  
$ sudo firewall-cmd --permanent --add-port=30000-32767/tcp                                                     
$ sudo firewall-cmd --permanent --add-port=179/tcp  
$ sudo firewall-cmd --permanent --add-port=4789/udp  
$ sudo firewall-cmd --reload  
$ sudo modprobe br_netfilter  
$ sudo sh -c "echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables"  
$ sudo sh -c "echo '1' > /proc/sys/net/ipv4/ip_forward"  


####  **Step 3: ** Installation of Docker (CRI)

Install Docker on all nodes, follow bellow commands.  

`sudo yum install -y yum-utils`  
`sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`  
`sudo yum install docker-ce -y`  

####  **Step 4: ** Installation of kubeadm, kubectl, kubelet on all nodes.  

Reference page: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/  

1.	Configure the repository on all master and worker nodes.  

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo  
[kubernetes]  
name=Kubernetes  
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch  
enabled=1  
gpgcheck=1  
repo_gpgcheck=1  
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg  
exclude=kubelet kubeadm kubectl  
EOF  

2.	yum command to install these packages  ` sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes` on all nodes.
enable kublete service on all nodes `sudo systemctl enable kubelet –now`  


#### **Step 5: ** Initialize cluster from one master-node-1:  

Run the command  `sudo kubeadm init --control-plane-endpoint "k8s-master:8443" --upload-certs` initialize successfully, should get command for other nodes to join (kubeadm join. Command), copy it to a new file.  

•	To use kubectl follow the commands:  

`mkdir -p $HOME/.kube`  
`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`  
`sudo chown $(id -u):$(id -g) $HOME/.kube/config`  

Install cni `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/revert-1043-v0.10.0-documentation/Documentation/kube-flannel.yml`  

•	Move to master-nodes (master-node-2 and master-node-3)and paste the following from master-node-1, output should be `node has joined`.  
  
`sudo kubeadm join k8s-master:8443 --token auinou.7bd34rdxgczyl87i \
    --discovery-token-ca-cert-hash sha256:b79dc98b5118d8eadd2eefe8b8becc69a23df6a4f83991f8065c8e2723c657a3 –control-plane –certificate-key ao….`  
  
•	Move to master-node-1 and type `kubectl get nodes` should verify all master nodes are ready status.  

•	Join the worker nodes (worker-node-1, worker-node-2 and worker-node-3) now with following command.  
  
`sudo kubeadm join k8s-master:8443 --token auinou.7bd34rdxgczyl87i \
 --discovery-token-ca-cert-hash sha256:b79dc98b5118d8eadd2eefe8b8becc69a23df6a4f83991f8065c8e2723c657a3 `  
  
•	Move to master-node-1 and type `kubectl get nodes` should verify all master & worker nodes are ready status.  
•	Run `kubectl get pods -n kube-system` to verify all the nodes are in namespace.  


**Step 7: ** High Availability k8 cluster.  


Create the deployment with nginx and expose this deployment with NodePort.  
  
`kubectl create deployment nginx --image=nginx`  
`kubectl get deployments.apps nginx` to view the deployment status.  
`kubectl get pods` to view the pods status.  
`kubectl scale deployment nginx --replicas=3` to scale the deployment 1 to 3.  
`kubectl get deployments.apps nginx` check the status of deployment scaled to 3.  
  
`kubectl expose deployment nginx --name=nginx --type=NodePort --port=80 --target-port=80` expose the deployment as service NodePort.  
` kubectl get svc nginx` to get the full details of port, to use curl.  
`curl http://ipaddress:port`  
