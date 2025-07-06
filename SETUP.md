# Setup
## Jenkins Installation
Official Docs:
- https://www.jenkins.io/doc/book/installing/linux/
- https://www.jenkins.io/doc/tutorials/tutorial-for-installing-jenkins-on-AWS/

```bash
sudo yum update -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade
sudo amazon-linux-extras install epel
sudo amazon-linux-extras install java-openjdk17 -y
sudo yum install java-17-amazon-corretto -y
sudo yum install jenkins -y

sudo systemctl enable jenkins
sudo systemctl start jenkins
systemctl status jenkins
java -version
javac -version
```

## Install and Configure Maven
Official Docs:
- https://maven.apache.org/install.html
- https://maven.apache.org/download.cgi

```bash
cd /opt
sudo wget https://dlcdn.apache.org/maven/maven-3/3.9.10/binaries/apache-maven-3.9.10-bin.tar.gz
sudo tar -xzvf apache-maven-3.9.10-bin.tar.gz
sudo mv apache-maven-3.9.10 maven
```

Update environment variables (`~/.bash_profile`):
```bash
M2_HOME=/opt/maven
M2=/opt/maven/bin
JAVA_HOME=/usr/lib/jvm/java-17-openjdk-<your-version>
PATH=$PATH:$HOME/bin:$JAVA_HOME:$M2_HOME:$M2
```

```bash
source ~/.bash_profile
mvn -v
```

## Setup Ansible Server and Install Ansible
```bash
sudo nano /etc/hostname      # Optional: change hostname
sudo useradd ansadmin
sudo passwd ansadmin
sudo visudo                  # Add: ansadmin ALL=(ALL) NOPASSWD: ALL

sudo nano /etc/ssh/sshd_config
# Enable: PasswordAuthentication yes
sudo service sshd reload

# Generate SSH key for ansadmin:
ssh-keygen

# Install Ansible:
sudo amazon-linux-extras install ansible2
ansible --version
```

## Integrate Ansible with Jenkins
```bash
cd /opt
sudo mkdir docker
sudo chown ansadmin:ansadmin docker
```

In Jenkins:
- Install Publish Over SSH plugin.
- Configure SSH connection to Ansible server.
- Set:
  - Source files: webapp/target/*.war
  - Remove prefix: webapp/target
  - Remote directory: /opt/docker

## Install & Configure Docker on Ansible Server
```bash
sudo yum install docker
sudo usermod -aG docker ansadmin
sudo service docker start
```

Dockerfile Example (placed in `/opt/docker`):
```Dockerfile
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
COPY ./*.war /usr/local/tomcat/webapps
```

## Create Ansible Playbook to Build & Push Docker Image
Update `/etc/ansible/hosts`:
```ini
[ansible]
<ansible-host-private-ip>
```
```bash
ssh-copy-id <ansible-host-ip>
```

Create `create_image_regapp.yml`:
```yaml
---
- hosts: ansible
  tasks:
    - name: Create Docker image
      command: docker build -t regapp:latest .
      args:
        chdir: /opt/docker

    - name: Tag image
      command: docker tag regapp:latest your-dockerhub-user/regapp:latest

    - name: Push image
      command: docker push your-dockerhub-user/regapp:latest
```

Run:
```bash
ansible-playbook create_image_regapp.yml
```

## Setup Bootstrap Server (for Kubernetes Tools)
Install AWS CLI:
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Install `kubectl`:
```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.27.1/2023-04-19/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

Install `eksctl`:
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

## Create EKS Cluster
```bash
eksctl create cluster --name your-cluster-name \
# Change region and node-type
--region eu-central-1 \
--node-type t2.micro
kubectl get nodes
```

## Kubernetes Manifests
Deployment (`regapp-deployment.yml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: regapp-cluster-regapp
  labels:
    app: regapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: regapp
  template:
    metadata:
      labels:
        app: regapp
    spec:
      containers:
        - name: regapp
          image: your-dockerhub-user/regapp
          ports:
            - containerPort: 8080
```

Service (`regapp-service.yml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: regapp-cluster-service
spec:
  selector:
    app: regapp
  ports:
    - port: 8080
      targetPort: 8080
  type: LoadBalancer
```

## Connect Bootstrap Server with Ansible
```bash
passwd root
nano /etc/ansible/hosts
```

```ini
[ansible]
<ansible-host-ip>

[kubernetes]
<bootstrap-server-ip>
```

```bash
ssh-copy-id root@<bootstrap-server-ip>
```

## Ansible Deployment Playbook for Kubernetes
```yaml
---
- hosts: kubernetes
  user: root
  tasks:
    - name: Deploy app
      command: kubectl apply -f regapp-deployment.yml

    - name: Create service
      command: kubectl apply -f regapp-service.yml

    - name: Restart deployment
      command: kubectl rollout restart deployment.apps/regapp-cluster-regapp
```

Run:
```bash
ansible-playbook /opt/docker/kube_deploy.yml
```

## Cleanup
```
kubectl delete deployment.apps/regapp-cluster-regapp
kubectl delete service/regapp-cluster-service
eksctl delete cluster --region eu-central-1 --name regapp-cluster
```
