Installing Ansible AWX
======================

### Installation Notes

The installation is different from the developers installation guide.

Instead of their preference of using minikube, this deployment will be using an actual kubernetes environment.

This is going to enable the service to be available upon reboot and other users on the network can access it.

Also this is an easier approach compared to using minikube.

Minikube is great for developers who need to spin something up and change some code and redeploy it, but for this deployment, we need it to be accessible for others in our environment and from external services like Slack and ServiceNow going to need to be able to hit that Ansible AWX.

### Deployment Environment

A virtual machine will be used within VMWare VCenter.

The operating system has to be a Linux distribution to run Kubernetes.

The deployment is based on a Ubuntu 22.04 LTS Linux install.

## Installation

Kubernetes is a requirement for Ansible AWX.

We will be sticking with a single-host, non-clustered Kubernetes that is very lightweight and it does not have high availability or scale out functions.

### Update and Upgrade Packages

Be sure to update and upgrade your package manager before installation:

```bash
sudo apt -y update && sudo apt -y upgrade
```

### Kubernetes Installation with Script

```bash
curl -sfL https://get.k3s.io | sh -
```

Check the version of the Kubernetes install:

```bash
sudo kubectl version
```

### Add Privileges to kubectl

Make the /etc/rancher/k3s/k3s.yaml file accessible by the current user (**replace the username and group**):

```bash
sudo chown username:group /etc/rancher/k3s/k3s.yaml
```

Now the kubectl command can be used without elevated privileges.

### Kustomize Setup

Kustomize is a tool that will help us build a Kubernetes environment based on some parameters we pass.

Ansible AWX is going to need to have an AWX operator and it's going to handle the download, installation and provisioning of Ansible AWX inside of the Kubernetes environment. 

The following script detects your OS and downloads the appropriate kustomize binary to your current working directory.

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
```

Notes: 
  - The user might need to be changed to root.
  - This script doesn’t work for ARM architecture.

Move the binary to somewhere within the Linux install that is within the PATH, so that the operating system can find the executable for a command.

```bash
sudo mv kustomize /usr/local/bin
```

### Create kustomization.yaml 

Create the kustomization.yaml file for the Kustomize architect to read.

```bash
touch ~/kustomization.yaml
```

Add the content to the file:

```bash
cat <<EOT >> ~/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=0.28.0

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 0.28.0

# Specify a custom namespace in which to install AWX
namespace: awx
EOT
```

### Kustomize Build

Kick off the kustomization binary to point to our local directory and then pipe the output to the Kubernetes binary called kubectl.

```bash
kustomize build . | kubectl apply -f -
```

Notes: 
  - The user might need to be changed to root.

### Check Build Progress

```bash
kubectl get pods -n awx
```

Notes:
  - Check the workload and ensure that the STATUS is Running (wait about a minute for the container process to complete).

### Create awx.yaml

Create a file called awx.yaml

```bash
touch ~/awx.yaml
```

Append the following content to the file and tell AWX that we want the web service to be exposed on a specific port (**customize the nodeport_port variable value**).

```bash
cat <<EOT >> ~/awx.yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
  nodeport_port: 80
EOT
```

### Add awx.yaml Reference to the kustomization.yaml File

Add the awx.yaml entry to resources, like so:

```bash
cat <<EOT > ~/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=0.28.0
  - awx.yaml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 0.28.0

# Specify a custom namespace in which to install AWX
namespace: awx
EOT
```

### Run Kustomize Build Again

```bash
kustomize build . | kubectl apply -f -
```

The deployment can be watched using the kubectl logs command.

```bash
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager --namespace awx
```

After the deployment is done, check our workload and ensure that the STATUS is Running (wait about 10 minutes for the container process to complete):

```bash
kubectl get pods -n awx
```

### Retrieve the admin Password

A password was generated during the build process and we need to ask Kubernetes to reveal that password to us.

By default, the admin user is admin and the password is available in the <resourcename>-admin-password secret. To retrieve the admin password, run:

```bash
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" --namespace awx | base64 --decode ; echo
```

### Login to the AWX Web UI

Open a web browser and navigate to the IP:PORT page.

The username to login is: admin
The password to login with admin is the string that was generated by kubectl.