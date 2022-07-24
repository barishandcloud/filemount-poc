
# Playbook to create file share mount on vm

Terraform script to create 4 vms, with one of them having Public IP for ssh Access


## Deployment

Prereq for running this script is an azure account & we are signed into azure cli. Ensure Terraform is installed on the machines. 
Ensure that the keypair bearing the name: swarmvm & swarmvm.pub is created in the folder that will run the terraform script
To deploy this project run

```bash
  git clone https://github.com/barishandcloud/mongodb.git  
  cd mongodb
  ssh-keygen -b 2048 -t rsa -f swarmvm
  chmod 400 swarmvm
  terraform init
  terraform plan
  terraform apply 
```

This provisions:
one resource group
4 nic - one with public ip
1 security group with rules allowing ssh 
nic to security group association
4 Ubuntu 18.04-LTS servers with Standard_B1s sizing in eastasia Region. The region can be modified in the variables.tf file. 
1 Storage accound and a file share within it

To change other parameters please alter main.tf accordingly.
terraform output yields the public ip of the machine to ssh. This machine is to be used as a ansible control node.

In order to ssh into vm1
```bash
  ssh -i swarmvm azureuser@<public_ip>
```
Although there are better/more secure alternatives for logging into the other vms, I use a rudimentary method in which I scp the private key into vm1 and from there I ssh into other nodes. This is not production grade and I desist others from using it for anything other than transient vms.
```bash
  scp -i swarmvm swarmvm azureuser@<public_ip>:~  
```

1) note the ip of all vms by issuing the following command on the machine from where terraform command was run : ```echo "azurerm_network_interface.nic{x}.private_ip_address" | terraform console ``` or ```echo "azurerm_linux_virtual_machine.vm{x}.private_ip_address" | terraform console ``` where x would be the machine number.
2) Edit the ```/etc/hosts ``` on swarmvm1
```bash
sudo nano /etc/hosts #add the below entries
10.0.1.X swarmvm2 #ip of vm2
10.0.1.X swarmvm3 #ip of vm3
10.0.1.X swarmvm4 #ip of vm4
```
3) Install ansible on swarmvm1 : ```sudo apt update && sudo apt install ansible -y ```
4) clone the playbook and inventory : ```git clone https://github.com/barishandcloud/filemount-poc.git ```
5) change to dir: ```cd filemount-poc ```
6) check if hosts.ini are resolving: ```ansible cluster -m shell -a "cat /etc/hosts" -i hosts.txt ``` outcome of this command should resemble this:
```bash
swarmvm2 | SUCCESS | rc=0 >>
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

swarmvm3 | SUCCESS | rc=0 >>
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

swarmvm4 | SUCCESS | rc=0 >>
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```
7) Finally run the playbook: ```ansible-playbook fshare.yml -i hosts.ini --extra-vars "smb_user=userid smb_pass=password"  ``` Ensure to fetch the user id and password from either gui console or through cli and populate the ```--extra-vars``` values prior to running the command. Outcome of the command is like this:
```bash
PLAY [cluster] *****************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************************************************************
ok: [swarmvm3]
ok: [swarmvm2]
ok: [swarmvm4]

TASK [create file share] *******************************************************************************************************************************************************************************************
changed: [swarmvm3]
changed: [swarmvm4]
changed: [swarmvm2]

TASK [Check if smb dir exists] *************************************************************************************************************************************************************************************
ok: [swarmvm3]
ok: [swarmvm4]
ok: [swarmvm2]

TASK [smb dir directory already existed] ***************************************************************************************************************************************************************************
skipping: [swarmvm2]
skipping: [swarmvm3]
skipping: [swarmvm4]

TASK [create smb directory when not present] ***********************************************************************************************************************************************************************
changed: [swarmvm2]
changed: [swarmvm3]
changed: [swarmvm4]

TASK [check if smb_creds file exists] ******************************************************************************************************************************************************************************
ok: [swarmvm2]
ok: [swarmvm3]
ok: [swarmvm4]

TASK [create and populate smb_cred file if it doesnt exists] *******************************************************************************************************************************************************
changed: [swarmvm3]
changed: [swarmvm4]
changed: [swarmvm2]

TASK [Add fstab entry] *********************************************************************************************************************************************************************************************
changed: [swarmvm2]
changed: [swarmvm3]
changed: [swarmvm4]

TASK [mount share] *************************************************************************************************************************************************************************************************
changed: [swarmvm2]
changed: [swarmvm3]
changed: [swarmvm4]

PLAY RECAP *********************************************************************************************************************************************************************************************************
swarmvm2                   : ok=8    changed=5    unreachable=0    failed=0   
swarmvm3                   : ok=8    changed=5    unreachable=0    failed=0   
swarmvm4                   : ok=8    changed=5    unreachable=0    failed=0   

```