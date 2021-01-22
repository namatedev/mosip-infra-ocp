"# mosip-infra-ocp" 

ssh-keygen # Generate ssh key in root host
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install ansible
sudo yum install -y git
sudo yum install libguestfs-tools # Install virt-sysprep
git clone https://github.com/namatedev/mosip-infra-ocp.git
cd mosip-infra-ocp/
ansible-playbook ./ansible/setup.yml