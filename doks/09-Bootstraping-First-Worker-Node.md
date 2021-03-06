# Bootstraping First Worker Node

## Prerequisites
The Certificates and Configuration are created on master node and then copied over to workers using scp.  
Once this is done, the commands are to be run on first worker instance: workerone-rhel8-nodeone.

### Provisioning Kubelet Client Certificates

> Generate a certificate and private key for kubernetes-rhel8-nodeone node in master node.  
Important Note: Please use the FQDN name for the worker node while creating certificate.

    cat > openssl-kubernetes-rhel8-nodeone.cnf <<EOF
    [req]
    req_extensions = v3_req
    distinguished_name = req_distinguished_name
    [req_distinguished_name]
    [ v3_req ]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    subjectAltName = @alt_names
    [alt_names]
    DNS.1 = kubernetes-rhel8-nodeone
    IP.1 = 192.168.15.11
    EOF

    openssl genrsa -out kubernetes-rhel8-nodeone.key 2048
    openssl req -new -key kubernetes-rhel8-nodeone.key -subj "/CN=system:node:kubernetes-rhel8-nodeone/O=system:nodes" -out kubernetes-rhel8-nodeone.csr -config openssl-kubernetes-rhel8-nodeone.cnf
    openssl x509 -req -in kubernetes-rhel8-nodeone.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kubernetes-rhel8-nodeone.crt -extensions v3_req -extfile openssl-kubernetes-rhel8-nodeone.cnf -days 365
    
### kubelet Kubernetes Configuration File

    {
      kubectl config set-cluster kubernetes-cluster \
        --certificate-authority=ca.crt \
        --embed-certs=true \
        --server=https://192.168.15.10:6443 \
        --kubeconfig=kubernetes-rhel8-nodeone.kubeconfig

      kubectl config set-credentials system:node:kubernetes-rhel8-nodeone \
        --client-certificate=kubernetes-rhel8-nodeone.crt \
        --client-key=kubernetes-rhel8-nodeone.key \
        --embed-certs=true \
        --kubeconfig=kubernetes-rhel8-nodeone.kubeconfig

      kubectl config set-context default \
        --cluster=kubernetes-cluster \
        --user=system:node:kubernetes-rhel8-nodeone \
        --kubeconfig=kubernetes-rhel8-nodeone.kubeconfig

      kubectl config use-context default --kubeconfig=kubernetes-rhel8-nodeone.kubeconfig
    }
    
> Results:-

    kubernetes-rhel8-nodeone.kubeconfig
    
### Copy certificates, private keys and kubeconfig files to the worker 'workerone-rhel8-nodeone' node:
On Master 'kubernetes-rhel8-master' node:

    scp ca.crt kubernetes-rhel8-nodeone.crt kubernetes-rhel8-nodeone.key \
      kubernetes-rhel8-nodeone.kubeconfig kubernetes-rhel8-nodeone:~/
    
### Download and Install Worker Binaries
Going forward all activities are to be done on the kubernetes-rhel8-nodeone node.

    wget -q --show-progress --https-only --timestamping \
      https://storage.googleapis.com/kubernetes-release/release/v1.18.2/bin/linux/amd64/kubectl \
      https://storage.googleapis.com/kubernetes-release/release/v1.18.2/bin/linux/amd64/kubelet \
      https://storage.googleapis.com/kubernetes-release/release/v1.18.2/bin/linux/amd64/kube-proxy
      
### Create the installation directories:

    sudo mkdir -p \
      /etc/cni/net.d \
      /opt/cni/bin \
      /var/lib/kubelet \
      /var/lib/kube-proxy \
      /var/lib/kubernetes \
      /var/run/kubernetes
      
Install the worker binaries:

     {
      chmod +x kubectl kube-proxy kubelet
      sudo mv kubectl kube-proxy kubelet /usr/local/bin/
    }
    
### Configure the Kubelet

    {
      sudo mv ${HOSTNAME}.key ${HOSTNAME}.crt /var/lib/kubelet/
      sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
      sudo mv ca.crt /var/lib/kubernetes/
    }
    
### Create the kubelet-config.yaml configuration file:

    cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
    kind: KubeletConfiguration
    apiVersion: kubelet.config.k8s.io/v1beta1
    authentication:
      anonymous:
        enabled: false
      webhook:
        enabled: true
      x509:
        clientCAFile: "/var/lib/kubernetes/ca.crt"
    authorization:
      mode: Webhook
    clusterDomain: "cluster.local"
    clusterDNS:
      - "10.96.0.10"
    resolvConf: "/etc/resolv.conf"
    runtimeRequestTimeout: "15m"
    EOF
    
### Create the kubelet.service systemd unit file

    cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
    [Unit]
    Description=Kubernetes Kubelet
    Documentation=https://github.com/kubernetes/kubernetes
    After=docker.service
    Requires=docker.service

    [Service]
    ExecStart=/usr/local/bin/kubelet \\
      --config=/var/lib/kubelet/kubelet-config.yaml \\
      --image-pull-progress-deadline=2m \\
      --kubeconfig=/var/lib/kubelet/kubeconfig \\
      --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.crt \\
      --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}.key \\
      --network-plugin=cni \\
      --register-node=true \\
      --v=2
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    
### Configure the Kubernetes Proxy

    sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
    
### Create the kube-proxy-config.yaml configuration file

    cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
    kind: KubeProxyConfiguration
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    clientConnection:
      kubeconfig: "/var/lib/kube-proxy/kubeconfig"
    mode: "iptables"
    clusterCIDR: "192.168.15.0/24"
    EOF
    
### Create the kube-proxy.service systemd unit file

    cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
    [Unit]
    Description=Kubernetes Kube Proxy
    Documentation=https://github.com/kubernetes/kubernetes

    [Service]
    ExecStart=/usr/local/bin/kube-proxy \\
      --config=/var/lib/kube-proxy/kube-proxy-config.yaml
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    
### Start the Worker Services
Important Note before starting kubelet and kube-proxy.  
ADAPT THE BELOW LINE in /etc/sudoers file. [such that SYSTEMD can search binaries from /usr/local/bin]  
Default Entry: secure_path = /sbin:/bin:/usr/sbin:/usr/bin  
After Adaptation: secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin  
Switch off SWAP:-  
sudo swapoff -a  
sudo sed '$s/^/# /' -i /etc/fstab

> The below command should run on master VM.  
sudo firewall-cmd --permanent --add-port=6443/tcp  
sudo firewall-cmd --reload

    {
      sudo systemctl daemon-reload
      sudo systemctl enable kubelet kube-proxy
      sudo systemctl start kubelet kube-proxy
    }
    
### Verification
Execute the below command in master VM.

    kubectl get nodes --kubeconfig admin.kubeconfig
    
> Output:
    
   | NAME | STATUS | ROLES | AGE | VERSION |
   | :---: | :---: | :---: | :---: | :---: |
   | workerone-rhel8-nodeone | NotReady | < none > | 93s |  v1.18.2


### Error Handling.

    Even after the above steps, you may face issue while starting up the kubelet or kube-proxy processes,
    like the error snapshot below.
    
    sudo systemctl status kubelet
    ● kubelet.service - Kubernetes Kubelet
    Loaded: loaded (/etc/systemd/system/kubelet.service; disabled; vendor preset: disabled)
    Active: activating (auto-restart) (Result: exit-code) since Sat 2020-05-09 19:45:30 UTC; 3s ago
     Docs: https://github.com/kubernetes/kubernetes
    Process: 7652 ExecStart=/usr/local/bin/kubelet --config=/var/lib/kubelet/kubelet-config.yaml --image-pull-progress-deadline=2m 
    --kubeconfig=/var/lib/kubelet/kube>
    Main PID: 7652 (code=exited, status=203/EXEC)
    
    Error in messages file under /var/log/messages will show like below:-
    May  9 19:45:40 kubernetes-rhel8-workervm systemd[7660]: kubelet.service: Failed to execute command: Permission denied
    May  9 19:45:40 kubernetes-rhel8-workervm systemd[7660]: kubelet.service: Failed at step EXEC spawning 
    /usr/local/bin/kubelet: Permission denied
    
    For the above error 'Permission denied' we need to adapt /etc/sudoers file, for which workaround solution has been mentioned above.
    
    Actual Solution for the above issue, is to execute the below command and then restrat etcd.
    sudo setenforce 0
    
> Important Note.  
Even after the kubelet is running now, it throws error which can be seen in the messages file 
and also the status of the Node is NotReady, since we need to configure the Network.  
May 09 19:49:34 kubernetes-rhel8-workervm kubelet[7690]: E0509 19:49:34.678324    7690 kubelet.go:2187] Container runtime network not ready: NetworkReady=false rea>  
May 09 19:49:38 kubernetes-rhel8-workervm kubelet[7690]: W0509 19:49:38.159981    7690 cni.go:237] Unable to update cni config: no networks found in /etc/cni/net.d


Next: [Bootstrapping the Kubernetes Second Worker Node](https://github.com/sanjibbehera/ManuallyInstallKubernetesVer1_18InRHEL8/blob/master/doks/10-Bootstraping-Second-Worker-Node.md)
    
