"# mosip-infra-ocp" 

sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install ansible
sudo yum install -y git
git clone https://github.com/namatedev/mosip-infra-ocp.git
cd mosip-infra-ocp/
ansible-playbook ./tmpansible/setup.yml