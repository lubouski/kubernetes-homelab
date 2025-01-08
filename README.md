# kubernetes-homelab
LXC and Ansible based installer of Single Master Kubernetes Cluster v1.32

>[!info] Ansible Installation
>Installation steps for Ubuntu  `24.04.1` LTS:
>```bash
>$ sudo apt update && sudo apt upgrade && sudo apt install python3-virtualenv python3-pip
>$ mkdir virtualenvs && cd virtualenvs && virtualenv -p python3 ansible 
>$ source ansible/bin/activate && python3 -m pip install ansible-core==2.18.1
>$ ansible-galaxy collection install community.general
>$ ansible-galaxy collection install ansible.posix

### Verify if cgroupv2 enabled on host machine 
```bash
# https://kubernetes.io/docs/concepts/architecture/cgroups/#cgroup-v2
$ stat -fc %T /sys/fs/cgroup/
cgroup2fs
```

### Set netfilter conntrack_max
```bash
# required by kube-proxy, it will try to set it, and fail
$ sudo sysctl -w net.netfilter.nf_conntrack_max=655360
```
### Create VM's
```bash
lxc launch --vm ubuntu:22.04 main \
  --config limits.memory=3GiB \
  --config limits.cpu=2 \
  --device root,size=30GiB

lxc launch --vm ubuntu:22.04 node1 \
  --config limits.memory=3GiB \
  --config limits.cpu=2 \
  --device root,size=30GiB

lxc launch --vm ubuntu:22.04 node2 \
  --config limits.memory=3GiB \
  --config limits.cpu=2 \
  --device root,size=30GiB
```
### Configure ssh
```bash
$ lxc file push /tmp/id_rsa.pub main/root/.ssh/authorized_keys # user root
$ lxc file push /tmp/id_rsa.pub node1/root/.ssh/authorized_keys # user root
$ lxc file push /tmp/id_rsa.pub node1/root/.ssh/authorized_keys # user root
```

### Checkout git repository
```bash
$ cd virtualenvs # directory with vitrual env activates so ansible could work
$ git clone git@github.com:lubouski/kubernetes-homelab.git
```
### Create Ansible inventory and Verify ansible connection to all nodes
```bash
# inventory change parameters as needed according to ssh keys setup
# main ansible_host=10.64.80.173 ansible_user=root ansible_port=22 ansible_ssh_private_key_file=/home/<username>/.ssh/id_ed25519
# this will help with adding key fingerprint to known_hosts 
$ ansible -i inventory main -m ping # or all insted of main
$ ansible -i inventory node1 -m ping # or all insted of main
$ ansible -i inventory node2 -m ping # or all insted of main
$ ansible-inventory -i inventory --list
```

### Run Ansible playbook and Verify installation
```bash
$ ansible-playbook main.yml -i inventory
...
PLAY RECAP *************************************************************************************************************************
main                       : ok=31   changed=24   unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
node1                      : ok=26   changed=19   unreachable=0    failed=0    skipped=6    rescued=0    ignored=0   
node2                      : ok=26   changed=19   unreachable=0    failed=0    skipped=6    rescued=0    ignored=0   
...
$ lxc exec main bash
root@main:~# kubectl get po -A
NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-76ccd8b6b9-g899s          1/1     Running   0          108s
calico-apiserver   calico-apiserver-76ccd8b6b9-pd8rl          1/1     Running   0          108s
calico-system      calico-kube-controllers-79d4c69d65-8n7hl   1/1     Running   0          108s
calico-system      calico-node-g5zx9                          1/1     Running   0          108s
calico-system      calico-node-w9lpv                          1/1     Running   0          108s
calico-system      calico-node-xd6t6                          1/1     Running   0          108s
calico-system      calico-typha-9df668c86-6pbfb               1/1     Running   0          99s
calico-system      calico-typha-9df668c86-x58s2               1/1     Running   0          108s
calico-system      csi-node-driver-7z25h                      2/2     Running   0          108s
calico-system      csi-node-driver-kqjwd                      2/2     Running   0          108s
calico-system      csi-node-driver-zhr5z                      2/2     Running   0          108s
kube-system        coredns-668d6bf9bc-9gv79                   1/1     Running   0          116s
kube-system        coredns-668d6bf9bc-zgqd5                   1/1     Running   0          116s
kube-system        etcd-main                                  1/1     Running   0          2m1s
kube-system        kube-apiserver-main                        1/1     Running   0          2m2s
kube-system        kube-controller-manager-main               1/1     Running   0          2m1s
kube-system        kube-proxy-9pbfw                           1/1     Running   0          116s
kube-system        kube-proxy-ns4wc                           1/1     Running   0          115s
kube-system        kube-proxy-vlwjm                           1/1     Running   0          115s
kube-system        kube-scheduler-main                        1/1     Running   0          2m1s
tigera-operator    tigera-operator-7d68577dc5-2l6j9           1/1     Running   0          116s
```

### Cleanup
```bash
$ lxc stop main node1 node2
$ lxc delete main node1 node2
# deactivate virtual env
# delete virtualenvs directory if needed
```
