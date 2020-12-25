# Guide to prepare the servers of mosip-infra in kubernetes

In order to install an kubernetes virtual network it's necessary to have the following hardware capacity:

- 12 processors.
- 250Gb of memory.
- 864Gb of space in disk
- Ubuntu 18.04.4 LTS (Bionic Beaver) installed OS

After that, initiate session in the server in order to install the VirtualBox software and configure it, as it follows:

    sudo apt-get update 
    sudo yum install wget
    wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | sudo apt-key add -
    sudo apt-get install software-properties-common
    sudo add-apt-repository "deb [arch=amd64] http://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib"
    sudo apt update && sudo apt install virtualbox-6.1

Check the version of VirtualBox as follows:

    vboxmanage --version
You have to get this result: `6.1.8r137981`

After that, it's necessary to install the extension pack for VirtualBox:

    wget http://download.virtualbox.org/virtualbox/6.1.8/Oracle_VM_VirtualBox_Extension_Pack-6.1.8-137981.vbox-extpack
    sudo VBoxManage extpack install Oracle_VM_VirtualBox_Extension_Pack-6.1.8-137981.vbox-extpack

After that, it's necessary to download the base virtual OS in order to use it as a template for all the virtual servers for Virtualbox. CentOS is the selected OS for our setup.

    wget https://vorboss.dl.sourceforge.net/project/linuxvmimages/VirtualBox/C/7/CentOS_7.7.1908_VBM.zip
    sudo apt-get install zip
    unzip CentOS_7.7.1908_VBM.zip

As a result you have to view this file `CentOS_7.7.1908_VirtualBox_Minimal_Installation_Image_LinuxVMImages.com.ova` in your current folder.

After that, it's necessary to prepare or setup the network configuration for VirtualBox, as follows:

    sudo VBoxManage dhcpserver remove --network=HostInterfaceNetworking-vboxnet0
    sudo VBoxManage hostonlyif remove vboxnet0
    sudo VBoxManage hostonlyif create
    sudo VBoxManage hostonlyif ipconfig vboxnet0 --ip 10.20.30.1 --netmask 255.255.255.0
    sudo VBoxManage dhcpserver add --ifname vboxnet0 --ip 10.20.30.2 --netmask 255.255.255.0 --lowerip 10.20.30.10 --upperip 10.20.30.254
    sudo VBoxManage dhcpserver modify --ifname vboxnet0 --enable

You have to see the following results:

    sudo vboxmanage list dhcpservers

=>

    NetworkName:    HostInterfaceNetworking-vboxnet0
    Dhcpd IP:       10.20.30.2
    LowerIPAddress: 10.20.30.10
    UpperIPAddress: 10.20.30.254
    NetworkMask:    255.255.255.0
    Enabled:        Yes
    Global Configuration:
    minLeaseTime:     default
    defaultLeaseTime: default
    maxLeaseTime:     default
    Forced options:   None
    Suppressed opts.: None
        1/legacy: 255.255.255.0
    Groups:               None
    Individual Configs:   None

And:

    sudo VBoxManage list -l hostonlyifs

=>

    Name:            vboxnet0
    GUID:            786f6276-656e-4074-8000-0a0027000000
    DHCP:            Disabled
    IPAddress:       10.20.30.1
    NetworkMask:     255.255.255.0
    IPV6Address:     fe80::800:27ff:fe00:0
    IPV6NetworkMaskPrefixLength: 64
    HardwareAddress: 0a:00:27:00:00:00
    MediumType:      Ethernet
    Wireless:        No
    Status:          Up
    VBoxNetworkName: HostInterfaceNetworking-vboxnet0

Make the following directory for the VMs:

    sudo mkdir /opt/virtualbox/vms

After that, it's necessary to prepare the following virtual OS servers:
 
    | Component       | Number of VMs | Configuration     | Persistence |
    |---              |---            |---                |---          |
    |Console          | 1             | 4 VCPU*, 8 GB RAM | 128 GB SSD  |
    |K8s MZ master    | 1             | 4 VCPU, 8 GB RAM  | -           |
    |K8s MZ workers   | 9             | 4 VCPU, 16 GB RAM | -           |
    |K8s DMZ master   | 1             | 4 VCPU, 8 GB RAM  | -           |
    |K8s DMZ workers  | 1             | 4 VCPU, 16 GB RAM | -           |

### Virtual servers: console, mzmaster, dmzmaster ###
Let's see the creation of the console virtual SO in VirtualBox, it's as follows:

    sudo vboxmanage import CentOS_7.7.1908_VirtualBox_Minimal_Installation_Image_LinuxVMImages.com.ova           --vsys 0 --group "/mosipgroup" --vsys 0 --description "mosip" --vsys 0 --eula accept --vsys 0 --unit 14 --ignore --vsys 0 --unit 15 --ignore --vsys 0 --unit 17 --ignore --vsys 0 --memory 8192 --vsys 0 --basefolder "/opt/virtualbox/vms" --vsys 0 --cpus 4 --vmname console
    sudo vboxmanage modifyvm console --nic1 nat
    sudo VBoxManage modifyvm console --nic2 hostonly --hostonlyadapter2 vboxnet0

This step is only specific for the console virtual server, it is not needed for the other servers:

    sudo vboxmanage modifyvm console --natpf1 "rulename,tcp,,80,,80"
    sudo vboxmanage modifyvm console --natpf1 "rulename2,tcp,,443,,443"
    sudo vboxmanage modifyvm console --natpf1 "rulename3,tcp,,30090,,30090"

Power on the virtual server:

    sudo vboxmanage startvm console --type headless

After that, it's necessary to monitor the properties of this server in order to locate the assigned IP for this virtual server by the following command:

    sudo VBoxManage guestproperty enumerate console

You have to view this result among the lines:

    Name: /VirtualBox/GuestInfo/Net/1/V4/IP, value: 10.20.30.10, timestamp: 1596637864670847000, flags:

After that, it's necessary to persist this IP in an internal file of the console virtual server, it's as follows:

- Initiate session on ssh `ssh centos@10.20.30.10`
- After that do the following:

      sudo touch /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo tee -a /etc/sysconfig/network-scripts/ifcfg-enp0s8 > /dev/null <<EOT
      TYPE=Ethernet
      PROXY_METHOD=none
      BROWSER_ONLY=no
      BOOTPROTO=none
      IPADDR=10.20.30.10
      PREFIX=24
      GATEWAY=10.20.30.1
      DEFROUTE=yes
      IPV4_FAILURE_FATAL=no
      IPV6INIT=no
      NAME=enp0s8
      ONBOOT=yes
      AUTOCONNECT_PRIORITY=-999
      EOT

      sudo hostnamectl set-hostname console
      sudo hostname console
      sudo systemctl restart network
      sudo systemctl stop firewalld
      sudo systemctl disable firewalld

- After that, it's necessary to modify the sudo config file in order to make every user an sudoer password-less.

      sudo vi /etc/sudoers
      #uncomment the line deleting the character # and then save the file
      #%wheel  ALL=(ALL)       NOPASSWD: ALL

- After that, it's necessary to create the user mosipuser and reboot the virtual server
 
      sudo adduser -s /bin/bash -m -g 1000 -G wheel mosipuser
      #the pass could be k3WUFKLMLR5UPb84
      sudo passwd mosipuser
      sudo su - mosipuser -c "mkdir /home/mosipuser/.ssh"
      reboot

You can repeat the steps described above in order to create the virtual severs mzmaster and dmzmaster.

### Virtual servers: mzworkers dmzworkers ###

Let's see the creation of the mzworker0 virtual SO in VirtualBox, it's as follows:

    sudo vboxmanage import CentOS_7.7.1908_VirtualBox_Minimal_Installation_Image_LinuxVMImages.com.ova           --vsys 0 --group "/mosipgroup" --vsys 0 --description "mosip" --vsys 0 --eula accept --vsys 0 --unit 14 --ignore --vsys 0 --unit 15 --ignore --vsys 0 --unit 17 --ignore --vsys 0 --memory 16384 --vsys 0 --basefolder "/opt/virtualbox/vms" --vsys 0 --cpus 4 --vmname mzworker0
    sudo vboxmanage modifyvm mzworker0 --nic1 nat
    sudo VBoxManage modifyvm mzworker0 --nic2 hostonly --hostonlyadapter2 vboxnet0

From here you can follow the remaining steps of the previous section called **Virtual servers: console, mzmaster, dmzmaster**.

In order to create the virtual servers mzworker1, mzworker2, mzworker3, mzworker4, mzworker5, mzworker6, mzworker7, mzworker8 and dmzworker0 you have to repeat the previous steps in this section.

### Preparations in the host server ###

It's necessary to persist all the IPs of the virtual servers in the `/etc/hosts` file of the host server, as it follows:

    sudo nano /etc/hosts
    ### Hetzner Online GmbH installimage
    # nameserver config
    # IPv4
    127.0.0.1 localhost.localdomain localhost

    10.20.30.10  console
    10.20.30.11  mzmaster
    10.20.30.12  dmzmaster
    10.20.30.13  mzworker0
    10.20.30.14  mzworker1
    10.20.30.15  mzworker2
    10.20.30.17  mzworker3
    10.20.30.18  mzworker4
    10.20.30.19  mzworker5
    10.20.30.20  mzworker6
    10.20.30.21  mzworker7
    10.20.30.22  mzworker8
    10.20.30.16  dmzworker0

Initiate session in the console virtual server with the user mosipuser, the password is: `k3WUFKLMLR5UPb84`

    ssh mosipuser@console

Genenate ssh keys:

    ssh-keygen -t rsa

Copy the public key to the authorized_keys file

    cat /home/mosipuser/.ssh/id_rsa.pub >> /home/mosipuser/.ssh/authorized_keys

After that, logout from the console virtual server and execute theses command from your current user in the host server:

    scp mosipuser@console:~/.ssh/id_rsa ~/.ssh/
    scp mosipuser@console:~/.ssh/id_rsa.pub ~/.ssh/

The final result is that you can login to the console virtual server by user mosipuser without issuing the password.

### Installing MOSIP infra in console virtual server ###

Initiate session in the console virtual server with the user mosipuser

    ssh mosipuser@console

Install the ansible package:

    sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
    sudo yum install ansible

Install git

    sudo yum install -y git

Git clone this repo in user home directory.
    cd ~/
    git clone https://github.com/mosip/mosip-infra

The specific commit that worked for me was the hash: `4f42506`, you can reset to this commit by using:

    git reset --hard 4f42506

After that:

    cd mosip-infra/deployment/sandbox-v2

From here you have to modify the following files:

    modified:   deployment/sandbox-v2/group_vars/all.yml
    modified:   deployment/sandbox-v2/group_vars/dmzcluster.yml
    modified:   deployment/sandbox-v2/group_vars/mzcluster.yml
    modified:   deployment/sandbox-v2/hosts.ini
    modified:   deployment/sandbox-v2/keys.sh
    modified:   deployment/sandbox-v2/playbooks/dmzcluster.yml
    modified:   deployment/sandbox-v2/playbooks/mzcluster.yml
    modified:   deployment/sandbox-v2/roles/config-repo/files/properties/kernel-mz.properties
    modified:   deployment/sandbox-v2/roles/config-repo/files/properties/pre-registration-mz.properties
 

Modify the file `deployment/sandbox-v2/group_vars/all.yml` (`-` means the previous value, `+` means the current and modified value):

    -artifactory_url: http://104.211.246.17:8040
    +artifactory_url: http://15.207.222.178

    -sandbox_domain_name: minibox.mosip.net
    +sandbox_domain_name: mospsndbxv2.namategroup.com

    -    get_certificate: true  # get a fresh certificate for the domain using Letsencrypt.
    -    email: info@mosip.io
    +    get_certificate: false  # get a fresh certificate for the domain using Letsencrypt.
    +    email: mosip@namategroup.com

       mz:
         kube_config:  "{{lookup('env', 'HOME') }}/.kube/mzcluster.config"
    -    nodeport_node: mzworker0.sb  # Any node on cluster for nodeport access
    -    any_node_ip: '10.20.20.157' # ip address of above node
    +    nodeport_node: mzworker0  # Any node on cluster for nodeport access
    +    any_node_ip: '10.20.30.13' # ip address of above node

       dmz:
        kube_config:  "{{lookup('env', 'HOME') }}/.kube/dmzcluster.config"
    -    nodeport_node: dmzworker0.sb  # Any node on cluster for nodeport access
    -    any_node_ip: '10.20.20.207' # ip adddress of above node
    +    nodeport_node: dmzworker0  # Any node on cluster for nodeport access
    +    any_node_ip: '10.20.30.16' # ip adddress of above node

    -        host: 'mzworker0.sb:30080/elasticsearch' # Connect to elasticsearch on MZ
    +        host: 'mzworker0:30080/elasticsearch' # Connect to elasticsearch on MZ

    -    node: 'mzworker1.sb' # Hostname. Run only on this node, and nothing else should run on this node
    +    node: 'mzworker1' # Hostname. Run only on this node, and nothing else should run on this node

     nfs:
    -  server: console.sb
    +  server: console

       node_affinity:
         enabled: false # To run all hdfs pods exclusively on a single node.
    -    node: 'mzworker2.sb' # Hostname. Run only on this node, and nothing else should run on this node
    +    node: 'mzworker2' # Hostname. Run only on this node, and nothing else should run on this node

Modify the file `deployment/sandbox-v2/group_vars/dmzcluster.yml` (`-` means the previous value, `+` means the current and modified value):

     cluster_name: 'dmzcluster'  # Same as in hosts.ini
    -cluster_master: 'dmzmaster.sb' # Same as in hosts.ini
    +cluster_master: 'dmzmaster' # Same as in hosts.ini

    -network_interface: "eth0"
    +network_interface: "enp0s8"

Modify the file `deployment/sandbox-v2/group_vars/mzcluster.yml` (`-` means the previous value, `+` means the current and modified value):

     cluster_name: 'mzcluster'  # Same as in hosts.ini
    -cluster_master: 'mzmaster.sb' # Same as in hosts.ini
    +cluster_master: 'mzmaster' # Same as in hosts.ini

    -network_interface: "eth0"
    +network_interface: "enp0s8"

Modify the file `deployment/sandbox-v2/hosts.ini` (`-` means the previous value to be replaced, `+` means that line is the current and modified value without the + character):

     [console]
    -console.sb ansible_user=mosipuser
    +console ansible_user=mosipuser

     [nginxserver]
    -console.sb ansible_user=mosipuser
    +console ansible_user=mosipuser

     [nfsserver]
    -console.sb ansible_user=mosipuser
    +console ansible_user=mosipuser

     [mzmaster]
    -mzmaster.sb ansible_user=root
    +mzmaster ansible_user=root

     [mzworkers]
    -mzworker0.sb ansible_user=root
    -mzworker1.sb ansible_user=root
    -mzworker2.sb ansible_user=root
    -mzworker3.sb ansible_user=root
    -mzworker4.sb ansible_user=root
    -mzworker5.sb ansible_user=root
    -mzworker6.sb ansible_user=root
    -mzworker7.sb ansible_user=root
    -mzworker8.sb ansible_user=root
    +mzworker0 ansible_user=root
    +mzworker1 ansible_user=root
    +mzworker2 ansible_user=root
    +mzworker3 ansible_user=root
    +mzworker4 ansible_user=root
    +mzworker5 ansible_user=root
    +mzworker6 ansible_user=root
    +mzworker7 ansible_user=root
    +mzworker8 ansible_user=root

     [dmzmaster]
    -dmzmaster.sb ansible_user=root
    +dmzmaster ansible_user=root
     [dmzworkers]
    -dmzworker0.sb ansible_user=root
    +dmzworker0 ansible_user=root

Modify the file `deployment/sandbox-v2/keys.sh` (`-` means the previous value, `+` means the current and modified value):

    -ansible-playbook -i $1 --extra-vars "ansible_user=mosipuser" --ask-pass --ask-become-pass playbooks/ssh.yml
    +ansible-playbook -vvv -i $1 --extra-vars "ansible_user=mosipuser" --ask-pass --ask-become-pass playbooks/ssh.yml

Modify the file `deployment/sandbox-v2/playbooks/dmzcluster.yml` (`-` means the previous value to be replaced, `+` means that line is the current and modified value without the + character):

         cluster: dmzcluster
    -    master: dmzmaster.sb
    +    master: dmzmaster

Modify the file `deployment/sandbox-v2/playbooks/mzcluster.yml` (`-` means the previous value to be replaced, `+` means that line is the current and modified value without the + character):

         cluster: mzcluster
    -    master: mzmaster.sb
    +    master: mzmaster

Modify the file `deployment/sandbox-v2/roles/config-repo/files/properties/kernel-mz.properties` (`-` means the previous value, `+` means the current and modified value):

    -mosip.kernel.notification.email.from=emailfrom
    -spring.mail.host=smtphost
    -spring.mail.username=username
    -spring.mail.password=password
    +mosip.kernel.notification.email.from=mosip@namategroup.com
    +spring.mail.host=smtp.gmail.com
    +spring.mail.username=mosip@namategroup.com
    +spring.mail.password=ukEkz56wK8

Modify the file `deployment/sandbox-v2/roles/config-repo/files/properties/pre-registration-mz.properties` (`-` means the previous value, `+` means the current and modified value):

    -google.recaptcha.site.key=sitekey
    +google.recaptcha.site.key=6LfdN6QZAAAAAFHrLpMzp7dBLnfEFzDqrXTD233c
    google.recaptcha.verify.url=https://www.google.com/recaptcha/api/siteverify
    -google.recaptcha.secret.key=secret
    +google.recaptcha.secret.key=6LfdN6QZAAAAAPKrdNK2JWOr36zh4C_F9jhPpnD9

In this delivery is provided and diff file in order to replicate the modified files in a handy way using the diff command.

Add the following shortcuts in /home/mosipuser/.bashrc:

    alias an='ansible-playbook -i hosts.ini'
    alias kc1='kubectl --kubeconfig $HOME/.kube/mzcluster.config'
    alias kc2='kubectl --kubeconfig $HOME/.kube/dmzcluster.config'
    alias sb='cd $HOME/mosip-infra/deployment/sandbox-v2/'
    alias helm1='helm --kubeconfig $HOME/.kube/mzcluster.config'
    alias helm2='helm --kubeconfig $HOME/.kube/dmzcluster.config'

After adding the above:

    source  ~/.bashrc

Exchange ssh keys with all machines. Provide the password for 'mosipuser'.

    ./key.sh hosts.ini


Intall the certificates in console virtual server by downloading certbot:

    wget https://dl.eff.org/certbot-auto
    chmod a+x certbot-auto
    sudo ./certbot-auto -v --install-only --no-self-upgrade
    sudo ./certbot-auto -v certonly --standalone

After that, follows the instructions and enter these data when certbot asks:

    mosip@namategroup.com
    mospsndbxv2.namategroup.com

Intall all MOSIP modules:

    ansible-playbook -vvv -i hosts.ini site.yml

The links to various dashboards are available at

    https://mospsndbxv2.namategroup.com/index.html

### Refresh the certificates in console virtual server ###

Initiate session in the console virtual server with the user mosipuser

    ssh mosipuser@console

Verify the status of the ngnix server by the following command:

    sudo systemctl status nginx

Proceed to stop the ngnix server:

    sudo service nginx stop

After that execute this command

    sudo ./certbot-auto -v certonly --standalone

After that, follows the instructions and enter these data when certbot asks:

    mospsndbxv2.namategroup.com

Proceed to start the ngnix server:

    sudo service nginx start