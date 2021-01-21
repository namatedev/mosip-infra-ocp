# List of commits
https://github.com/namatedev/mosip-infra-ocp/commits/main

---

# Guide to add a machine in OCP

* Generate ssh key in root host

```
ssh-keygen
```

* Install virt-sysprep

```
sudo yum install libguestfs-tools
```

* Create a VM

```
[root@server ~]# git clone https://github.com/namatedev/mosip-infra-ocp.git
[root@server ~]# cd mosip-infra-ocp/
[root@server ~]# git status
[root@server ~]# git fetch
[root@server ~]# git pull
[root@server ~]# # ansible-playbook ./ansible/setup_ocp_console.yml
[root@server ~]# # ansible-playbook ./ansible/setup_ocp_console_2.yml
[root@server ~]# ansible-playbook -vvvv ./ansible/setup_ocp_console_2.yml
```
* check if vm created
```
ls -l /var/lib/libvirt/images/
```

* check if vm running
```
virsh list --all
```

* to access:
``` 
virsh console ocp4one-console 
```