# kubernetes_with_mongoDB_ReplicaSet

# 0. Goal

|  | CPU | Memory | GPU | GPU Driver |
| --- | --- | --- | --- | --- |
| Master | 2 | 4,096 MB | no | N/A |
| Worker1 | 2 | 4,096 MB | no | N/A |
| Worker2 | 2 | 4,096 MB | no | N/A |
| Worker3 | 2 | 4,096 MB | no | N/A |
| haproxy | 2 | 4,096 MB | no | N/A |
| nfsserver | 2 | 4,096 MB | no | N/A |

# 1. Install Vagrant on your Ubuntu
```
$ sudo apt install --yes vagrant vagrant-libvirt
```

# 2. Make dir and Initization vagrant
You can select the OS images which is called as "box" in https://app.vagrantup.com/boxes/search.
```
$ mkdir -p /mnt/vagrant/k8s
$ cd /mnt/vagrant/k8s
$ vagrant init generic/ubuntu2010
```

# 3. Edit Vagrantfile
You might use the file of Vagrantfile located in the /mnt/vagrant/ubuntu below:
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2010"
  config.vm.provider :libvirt do |kvm|
    kvm.memory = 4096
    kvm.cpus = 2
  end
#-------------------- master --------------------#
  config.vm.define "master_192.168.33.100" do |server|
    server.vm.network "private_network", ip: "192.168.33.100"
    server.vm.hostname = "master"
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo apt-get install -y sshpass
      ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa <<< y
      cat <<EOF > ~/.ssh/config
host 192.168.33.*
   StrictHostKeyChecking no
EOF
      chmod 600 ~/.ssh/config
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.101
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.102
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.103
      sudo cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
      sudo systemctl enable docker
      sudo systemctl daemon-reload
      sudo systemctl restart docker
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
      sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.33.100
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      sudo kubectl taint nodes --all node-role.kubernetes.io/master-
      sudo kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
      token=$(sudo kubeadm token list |tail -n 1 |awk '{print $1}')
      hashkey=$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //')
      ssh vagrant@192.168.33.101 sudo kubeadm join 192.168.33.100:6443 --token $token --discovery-token-ca-cert-hash sha256:$hashkey
      ssh vagrant@192.168.33.102 sudo kubeadm join 192.168.33.100:6443 --token $token --discovery-token-ca-cert-hash sha256:$hashkey
      sudo kubectl label node worker1 node-role.kubernetes.io/node=worker1
      sudo kubectl label node worker2 node-role.kubernetes.io/node=worker2
      #curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
      #chmod 700 get_helm.sh
      #./get_helm.sh
      #helm repo add stable https://charts.helm.sh/stable
      sudo apt-get -y install nfs-client
      sudo mount -v 192.168.33.11:/ /mnt
    SHELL
  end
#-------------------- worker1 --------------------#
  config.vm.define "worker1_192.168.33.101" do |server|
    server.vm.network "private_network", ip: "192.168.33.101"
    server.vm.hostname = "worker1"
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
      sudo systemctl enable docker
      sudo systemctl daemon-reload
      sudo systemctl restart docker
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
      sudo apt-get -y install nfs-client
      sudo mount -v 192.168.33.11:/ /mnt
    SHELL
  end
#-------------------- worker2 --------------------#
  config.vm.define "worker2_192.168.33.102" do |server|
    server.vm.network "private_network", ip: "192.168.33.102"
    server.vm.hostname = "worker2"
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
      sudo systemctl enable docker
      sudo systemctl daemon-reload
      sudo systemctl restart docker
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
      sudo apt-get -y install nfs-client
      sudo mount -v 192.168.33.11:/ /mnt
    SHELL
  end
#-------------------- worker3 --------------------#
  config.vm.define "worker2_192.168.33.103" do |server|
    server.vm.network "private_network", ip: "192.168.33.103"
    server.vm.hostname = "worker3"
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
      sudo systemctl enable docker
      sudo systemctl daemon-reload
      sudo systemctl restart docker
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
      sudo apt-get -y install nfs-client
      sudo mount -v 192.168.33.11:/ /mnt
    SHELL
  end
#-------------------- haproxy --------------------#
  config.vm.define "haproxy_192.168.133.10" do |server|
    server.vm.network "private_network", ip: "192.168.33.10"
    server.vm.network "private_network", ip: "192.168.133.10"
    server.vm.hostname = "haproxy"
    server.vm.provision "shell", privileged: false, inline: <<-SHELL
      sudo apt-get update
      sudo apt-get install -y docker.io
      sudo apt-get install -y sshpass
      ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa <<< y
      cat <<EOF > ~/.ssh/config
host 192.168.33.*
   StrictHostKeyChecking no
EOF
      chmod 600 ~/.ssh/config
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.100
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.101
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.102
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.103
      sudo apt-get -y install nfs-client
      sudo mount -v 192.168.33.11:/ /mnt
    SHELL
  end
#-------------------- nfsserver --------------------#
  config.vm.define "nfsserver_192.168.133.11" do |server|
    server.vm.network "private_network", ip: "192.168.33.11"
    server.vm.network "private_network", ip: "192.168.133.11"
    server.vm.hostname = "nfsserver"
    server.vm.provision "shell", privileged: false, inline: <<-SHELL
      sudo apt-get update
      sudo apt-get install -y docker.io
      sudo docker pull itsthenetwork/nfs-server-alpine
      mkdir -p /home/vagrant/shared
      sudo docker run -itd --name nfs --rm --privileged -p 2049:2049 -v /home/vagrant/shared:/data -e SHARED_DIRECTORY=/data itsthenetwork/nfs-server-alpine:latest
    SHELL
  end
#-------------------------------------------------#
end
```

# 4. Vagrant Up for all VMs
```
$ vagrant up --provider=libvirt
```
You can login to the master-node (192.168.33.100), so you can check if master and workers are ready.
```
vagrant@master:~$ sudo kubectl get nodes
NAME      STATUS   ROLES                  AGE    VERSION
master    Ready    control-plane,master   2m8s   v1.22.2
worker1   Ready    node                   96s    v1.22.2
worker2   Ready    node                   94s    v1.22.2
```



```
$ cat headless-service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: mongo-srv
  labels:
    run: mongo-srv
spec:
  ports:
  - name: mongo
    port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    run: mongo-test
    
$ cat mongodb-statefulset.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo-test
spec:
  selector:
    matchLabels:
      run: mongo-test
  serviceName: "mongo-srv"
  replicas: 3
  template:
    metadata:
      labels:
        run: mongo-test
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: mongodb
        image: docker.io/mongo
        command: 
        - mongod 
        - "--bind_ip_all"
        - "--replSet"
        - rs0
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db
        ports:
        - containerPort: 27017
      volumes:
        - name: mongo-data
          persistentVolumeClaim:
            claimName: mongo-pvc



```
