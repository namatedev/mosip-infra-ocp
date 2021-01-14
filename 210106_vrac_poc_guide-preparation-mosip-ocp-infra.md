# commits
https://github.com/namatedev/mosip-infra-ocp/commits/main

---

# Guide to add a machine in OCP

```
cd cd mosip-infra-ocp/
git status
git fetch
git pull
# [root@server ~]# ansible-playbook ./ansible/setup_ocp_console.yml
# [root@server ~]# ansible-playbook ./ansible/setup_ocp_console_2.yml
[root@server ~]# ansible-playbook -vvvv ./ansible/setup_ocp_console_2.yml
```
# check if vm created
ls -l /var/lib/libvirt/images/

# check if vm running
virsh list --all

# to access
virsh console ocp4one-console2

# to delete
virsh destroy ocp4one-console
virsh undefine ocp4one-console
rm /var/lib/libvirt/images/ocp4one-console.qcow2 # => y

virsh destroy ocp4one-console2