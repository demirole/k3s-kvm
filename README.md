# k3s-kvm
Create a Kubernetes cluster based on [Rancher's k3s project](https://k3s.io/) with distinct hosts for the control
plane and worker nodes on a bare metal machine running Linux and libvirt + qemu/kvm. 

## Getting started
### Prerequisites
- A bare metal machine with 
  - at least 16GB memory (for 1 master, 2 worker nodes), 32GB recommended)
  - a CPU with at least 4 physical cores
  - with any Linux distro (this guide assumes Ubuntu 18.04)
- Install libvirt, qemu/kvm. E.g., on Ubuntu:
  ```bash
  apt-get install -qy qemu-kvm libvirt-bin virtinst python3 vagrant
  ```
- Start the libvirt daemon. Using either virsh or virt-manager, create a virtual network:
  ```bash
  virsh -c qemu:///system net-create libvirt/default_network.xml
  ```
- Install the `vagrant-libvirt` plugin:
  ```bash
  vagrant plugin install vagrant-libvirt
  ```
- Initialize and install the virtualenv:
  ```bash
  python -m venv venv
  pip install -r requirements.txt
  ```

### Creating the VMs
```bash
vagrant up
```

### Provisioning the VMs
```bash
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory create-k3s-cluster.yml
```

### Configure local kubectl
- Install `kubectl` on the local machine
- Configure it
  ```bash
  mkdir -p ~/.kube && vagrant ssh master -c 'sudo cat .kube/config' > ~/.kube/config
  ```
- Run `kubectl`
  ```bash
  # kubectl cluster-info
  Kubernetes master is running at https://192.168.122.127:6443
  CoreDNS is running at https://192.168.122.127:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
  Metrics-server is running at https://192.168.122.127:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

  To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

  # kubectl get nodes                                                                                 
  NAME       STATUS   ROLES    AGE    VERSION
  master     Ready    master   149m   v1.19.3+k3s2
  worker-3   Ready    <none>   139m   v1.19.3+k3s2
  worker-2   Ready    <none>   139m   v1.19.3+k3s2
  worker-1   Ready    <none>   139m   v1.19.3+k3s2
  ```

### Smoke testing cluster
The git repository monachus/channel.git by [Adrian Goins](https://adrian.goins.tv/) contains kustomization files 
for deploying a demo application that tests cluster ingress and load balancing. Those files have been made available in this repo.

First, apply the demo project: 
```bash
kubectl apply -k ./cluster-demo
```

Get the `EXTERNAL-IP` of the `traefik` ingress controller by running
```bash
kubectl get services traefik -n kube-system
``` 
, and the hostname of the demo ingress
```bash
kubectl get ingresses.v1.networking.k8s.io rancher-demo
```
, field 'HOSTS'

Define a DNS lookup in your `/etc/hosts` where the hostname above is resolved to the `EXTERNAL-IP` address. Open that 
hostname in your local browser. You should see a webpage that pings one of the three replicas of the demo
project service.
 
Remove the test with
```bash
kubectl delete -k ./cluster-demo
```

## Customisation 
### Change number of nodes
The number of nodes provisioned is defined in the Vagrantfile:
```
NUM_NODES = 3
```
Update the number to get more or less worker nodes. 

### Change VM resources
The number of CPUs and memory allotted for each VM can be change in the section `config.vm.provider`
 of the `Vagrantfile`:
```ruby
config.vm.provider :libvirt do |libvirt|
  libvirt.cpus = 2
  libvirt.memory = 4096
  ...
end
```

## Cleanup
### Destroy the current cluster
Run
```bash
vagrant destroy
```

## Built With
* [Ansible](https://www.ansible.com/) - Simple, agentless IT automation that anyone can use
* [k3s](https://k3s.io/) -  Lightweight Kubernetes, the certified Kubernetes distribution built for IoT & Edge computing
* [Vagrant](https://www.vagrantup.com/) - Development Environments Made Easy.

## Acknowledgments
* This project is based on the work of 
  * [talbotfoundry/k8s-kvm](https://github.com/talbotfoundry/k8s-kvm)
  * [rancher/k3s-ansible](https://github.com/rancher/k3s-ansible)
* The cluster demo project is taken from [Adrian Goins' Youtube channel's](https://adrian.goins.tv/) [Gitlab repository](https://gitlab.com/monachus/channel)