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
      ssh vagrant@192.168.33.103 sudo kubeadm join 192.168.33.100:6443 --token $token --discovery-token-ca-cert-hash sha256:$hashkey
      sudo kubectl label node worker1 node-role.kubernetes.io/node=worker1
      sudo kubectl label node worker2 node-role.kubernetes.io/node=worker2
      sudo kubectl label node worker3 node-role.kubernetes.io/node=worker3
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
  config.vm.define "worker3_192.168.33.103" do |server|
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
$ cat <<EOF > mongo-replica.yaml
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
---
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
EOF
```
```
vagrant@master:~$ sudo kubectl apply -f mongo-replica.yaml
vagrant@master:~$ sudo kubectl exec -it mongo-test-0 -- mongo
MongoDB shell version v5.0.3
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
```
```
> rs.initiate()
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "mongo-test-0:27017",
	"ok" : 1
}
```
```
rs0:SECONDARY> var cfg = rs.conf()
```
```
rs0:PRIMARY> cfg.members[0].host="mongo-test-0.mongo-srv:27017"
mongo-test-0.mongo-srv:27017
```
```
rs0:PRIMARY> rs.reconfig(cfg)
{
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1633847333, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1633847333, 1)
}
```
```
rs0:PRIMARY> rs.status()
{
	"set" : "rs0",
	"date" : ISODate("2021-10-10T06:32:26.087Z"),
	"myState" : 1,
	"term" : NumberLong(1),
	"syncSourceHost" : "",
	"syncSourceId" : -1,
	"heartbeatIntervalMillis" : NumberLong(2000),
	"majorityVoteCount" : 1,
	"writeMajorityCount" : 1,
	"votingMembersCount" : 1,
	"writableVotingMembersCount" : 1,
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1633847542, 1),
			"t" : NumberLong(1)
		},
		"lastCommittedWallTime" : ISODate("2021-10-10T06:32:22.274Z"),
		"readConcernMajorityOpTime" : {
			"ts" : Timestamp(1633847542, 1),
			"t" : NumberLong(1)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(1633847542, 1),
			"t" : NumberLong(1)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1633847542, 1),
			"t" : NumberLong(1)
		},
		"lastAppliedWallTime" : ISODate("2021-10-10T06:32:22.274Z"),
		"lastDurableWallTime" : ISODate("2021-10-10T06:32:22.274Z")
	},
	"lastStableRecoveryTimestamp" : Timestamp(1633847492, 1),
	"electionCandidateMetrics" : {
		"lastElectionReason" : "electionTimeout",
		"lastElectionDate" : ISODate("2021-10-10T06:27:32.245Z"),
		"electionTerm" : NumberLong(1),
		"lastCommittedOpTimeAtElection" : {
			"ts" : Timestamp(0, 0),
			"t" : NumberLong(-1)
		},
		"lastSeenOpTimeAtElection" : {
			"ts" : Timestamp(1633847252, 1),
			"t" : NumberLong(-1)
		},
		"numVotesNeeded" : 1,
		"priorityAtElection" : 1,
		"electionTimeoutMillis" : NumberLong(10000),
		"newTermStartDate" : ISODate("2021-10-10T06:27:32.257Z"),
		"wMajorityWriteAvailabilityDate" : ISODate("2021-10-10T06:27:32.275Z")
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "mongo-test-0.mongo-srv:27017",                               <----- Only 1 mongoDB at this moment.
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 824,
			"optime" : {
				"ts" : Timestamp(1633847542, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2021-10-10T06:32:22Z"),
			"syncSourceHost" : "",
			"syncSourceId" : -1,
			"infoMessage" : "",
			"electionTime" : Timestamp(1633847252, 2),
			"electionDate" : ISODate("2021-10-10T06:27:32Z"),
			"configVersion" : 2,
			"configTerm" : 1,
			"self" : true,
			"lastHeartbeatMessage" : ""
		}
	],
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1633847542, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1633847542, 1)
}
```
```
rs0:PRIMARY> rs.add("mongo-test-1.mongo-srv:27017")
{
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1633847691, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1633847691, 1)
}
```
```
rs0:PRIMARY> rs.add("mongo-test-2.mongo-srv:27017")
{
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1633847700, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1633847700, 1)
}
```

```
rs0:PRIMARY> rs.status()
{
	"set" : "rs0",
	"date" : ISODate("2021-10-10T06:37:40.211Z"),
	"myState" : 1,
	"term" : NumberLong(1),
	"syncSourceHost" : "",
	"syncSourceId" : -1,
	"heartbeatIntervalMillis" : NumberLong(2000),
	"majorityVoteCount" : 2,
	"writeMajorityCount" : 2,
	"votingMembersCount" : 3,
	"writableVotingMembersCount" : 3,
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1633847852, 1),
			"t" : NumberLong(1)
		},
		"lastCommittedWallTime" : ISODate("2021-10-10T06:37:32.283Z"),
		"readConcernMajorityOpTime" : {
			"ts" : Timestamp(1633847852, 1),
			"t" : NumberLong(1)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(1633847852, 1),
			"t" : NumberLong(1)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1633847852, 1),
			"t" : NumberLong(1)
		},
		"lastAppliedWallTime" : ISODate("2021-10-10T06:37:32.283Z"),
		"lastDurableWallTime" : ISODate("2021-10-10T06:37:32.283Z")
	},
	"lastStableRecoveryTimestamp" : Timestamp(1633847852, 1),
	"electionCandidateMetrics" : {
		"lastElectionReason" : "electionTimeout",
		"lastElectionDate" : ISODate("2021-10-10T06:27:32.245Z"),
		"electionTerm" : NumberLong(1),
		"lastCommittedOpTimeAtElection" : {
			"ts" : Timestamp(0, 0),
			"t" : NumberLong(-1)
		},
		"lastSeenOpTimeAtElection" : {
			"ts" : Timestamp(1633847252, 1),
			"t" : NumberLong(-1)
		},
		"numVotesNeeded" : 1,
		"priorityAtElection" : 1,
		"electionTimeoutMillis" : NumberLong(10000),
		"newTermStartDate" : ISODate("2021-10-10T06:27:32.257Z"),
		"wMajorityWriteAvailabilityDate" : ISODate("2021-10-10T06:27:32.275Z")
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "mongo-test-0.mongo-srv:27017",                               <----- Primary mongoDB
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 1138,
			"optime" : {
				"ts" : Timestamp(1633847852, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2021-10-10T06:37:32Z"),
			"syncSourceHost" : "",
			"syncSourceId" : -1,
			"infoMessage" : "",
			"electionTime" : Timestamp(1633847252, 2),
			"electionDate" : ISODate("2021-10-10T06:27:32Z"),
			"configVersion" : 6,
			"configTerm" : 1,
			"self" : true,
			"lastHeartbeatMessage" : ""
		},
		{
			"_id" : 1,
			"name" : "mongo-test-1.mongo-srv:27017",                               <----- Seconday mongoDB Replicaset
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 168,
			"optime" : {
				"ts" : Timestamp(1633847852, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1633847852, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2021-10-10T06:37:32Z"),
			"optimeDurableDate" : ISODate("2021-10-10T06:37:32Z"),
			"lastHeartbeat" : ISODate("2021-10-10T06:37:40.183Z"),
			"lastHeartbeatRecv" : ISODate("2021-10-10T06:37:40.179Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncSourceHost" : "mongo-test-0.mongo-srv:27017",
			"syncSourceId" : 0,
			"infoMessage" : "",
			"configVersion" : 6,
			"configTerm" : 1
		},
		{
			"_id" : 2,
			"name" : "mongo-test-2.mongo-srv:27017",                               <----- Seconday mongoDB Replicaset
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 160,
			"optime" : {
				"ts" : Timestamp(1633847852, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1633847852, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2021-10-10T06:37:32Z"),
			"optimeDurableDate" : ISODate("2021-10-10T06:37:32Z"),
			"lastHeartbeat" : ISODate("2021-10-10T06:37:40.183Z"),
			"lastHeartbeatRecv" : ISODate("2021-10-10T06:37:38.689Z"),
			"pingMs" : NumberLong(1),
			"lastHeartbeatMessage" : "",
			"syncSourceHost" : "mongo-test-1.mongo-srv:27017",
			"syncSourceId" : 1,
			"infoMessage" : "",
			"configVersion" : 6,
			"configTerm" : 1
		}
	],
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1633847852, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1633847852, 1)
}

```
