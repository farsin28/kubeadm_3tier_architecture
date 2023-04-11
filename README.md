Deploying a basic 3 Tier PHP Application consisting on MySQL Database, phpMyAdmin for Database management and our PHP Front-end on Kubernetes then accessing it on our browser by exposing it on NodePort service.



Launch ubuntu EC2 for Master (min 4 GB RAM , 30 GB space)

Lacunch ubuntu EC2 for Node (min 4 GB RAM)

Connect to Master server and node server  then setup kubeadm using below code and setup docker on both server
~~~


 

echo "Step 1: Login with root user and Install Docker ( in Master & Worker Node Both)"
sudo apt-get update -y
sudo apt-get install \
ca-certificates \
curl \
gnupg -y

echo "Add Docker’s official GPG key:"
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg -y

echo "Use the following command to set up the repository:"

echo \
"deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
"$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
echo "To install the latest version, run:"
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

echo "Step 2: Create a file with the name containerd.conf using the command:"
# create the file with root privileges using the vim editor
sudo vim /etc/modules-load.d/containerd.conf <<EOF
i
overlay
br_netfilter
Esc
:wq
EOF

echo "Step 3: Save the file and run the following commands:"
modprobe overlay
modprobe br_netfilter

echo "Step 4: Create a file with the name kubernetes.conf in /etc/sysctl.d folder:"
# create the file with root privileges using the vim editor
sudo vim /etc/sysctl.d/kubernetes.conf <<EOF
i
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
Esc
:wq
EOF

echo "Step 5: Run the commands to verify the changes:"
sudo sysctl --system
sudo sysctl -p

echo "Step 6: Remove the config.toml file from /etc/containerd/ Folder and run reload your system daemon:"
rm -f /etc/containerd/config.toml
systemctl daemon-reload

echo "Step 7: Add Kubernetes Repository:"
apt-get update && apt-get install -y apt-transport-https ca-certificates curl -y
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

echo "Step 8: Disable Swap"
swapoff -a

Step 9: Export the environment variable:
export KUBE_VERSION=1.23.0

echo "Step 10: Install Kubernetes:"
apt-get update -y
apt-get install -y kubelet=${KUBE_VERSION}-00 kubeadm=${KUBE_VERSION}-00 kubectl=${KUBE_VERSION}-00 kubernetes-cni=0.8.7-00
apt-mark hold kubelet kubeadm kubectl
systemctl enable kubelet
systemctl start kubelet
~~~


then goto master server use below code init
kubeadm init --kubernetes-version=${KUBE_VERSION}

after this ini code you will get the command to run in node server to connect node and master



now you we can see the nodes connected via master 

~~~

root@master:/home/ubuntu# kubectl get nodes
NAME     STATUS     ROLES                  AGE     VERSION
master   NotReady   control-plane,master   14m     v1.23.0
node     NotReady   <none>                 4m10s   v1.23.0

~~~


goto master 
kubectl create cm db-config --from-literal=MYSQL_DATABASE=sqldb

kubectl create secret generic db-secret --from-literal=MYSQL_ROOT_PASSWORD=rootpassword

run mysql-pod --image=mysql --dry-run=client -o yaml > mysql-pod.yml

vi mysql-pod.yml

~~~
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: mysql-pod
  name: mysql-pod
spec:
  containers:
  - image: mysql
    name: mysql-pod
    envFrom:
    - configMapRef:
       name: db-config
    - secretRef:
       name: db-secret
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
~           

~~~

kubectl apply -f mysql-pod.yml 


same for phpmyadmin


kubectl create secret generic phpmyadmin-secret --from-literal=PMA_USER=myuser --from-literal=PMA_PASSWORD=mypassword


kubectl create configmap phpmyadmin-config --from-literal=PMA_USER=myuser --from-literal=PMA_PASSWORD=mypassword
~~~
apiVersion: v1
kind: Pod
metadata:
  name: phpmyadmin-pod
  labels:
    run: phpmyadmin-pod
spec:
  containers:
  - name: phpmyadmin
    image: phpmyadmin/phpmyadmin
    ports:
    - containerPort: 80
    envFrom:
    - configMapRef:
        name: phpmyadmin-config
    - secretRef:
        name: phpmyadmin-secret
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always

~~~
upload the db via phpmyadmin
then create pods for php-app
same for php app

refer  https://www.youtube.com/watch?v=-lHKvZ2qYMM&t=394s
