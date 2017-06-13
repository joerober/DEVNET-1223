# __Cisco Live 2017 - DEVNET-1223__
Session: Automating IOS-XR with Ansible

## Setting up IOS-XR to allow Ansible

Generate crypto keys on the target IOS-XR device:

```commandline
crypto key generate rsa
```

Enable SSH version 2 and set a reasonable timeout:

```commandline
config t
  ssh version 2
  ssh timeout 120
  commit
  exit
```

This should be enough to try a simple command from your Ansible host. Make sure the target
device is already defined in your inventory:

```commandline
ansible <host> -u <username> -k -m raw -a "show version"
```


### Example playbooks
Set an SNMP community string:
```commandline
ansible-playbook playbooks/set-snmpv2.yaml --extra-vars="community=cisco123”
```
Create a new user account:
```commandline
ansible-playbook playbooks/create_user.yaml --extra-vars="newuser=bob password=cisco"
```
Delete an existing user:
```commandline
ansible-playbook playbooks/delete_user.yaml --extra-vars="user=bob"
```

## Using YDK-Gen to create custom APIs

YDK's *generate* tool can be used to create custom/targeted APIs for specific use cases. Or it
might also be needed if a particular model isn't included in what comes with YDK (like the
IETF or OpenConfig models). I ended up needing to do my own because the OpenConfig interfaces
model wasn't included.

First, you need some tools on a build server. I used Ubuntu 16.04:

```commandline
sudo apt-get install python-pip zlib1g-dev python-lxml libxml2-dev libxslt1-dev python-dev libboost-dev libboost-python-dev \
  libssh-dev libcurl4-openssl-dev libtool-bin libboost-all-dev libboost-log-dev libpcre3-dev libpcre++-dev libtool pkg-config \
  python3-dev python3-lxml cmake
```

Then you need to clone the YDK-Gen repository to a location on that server:

```commandline
mkdir -p ~/projects/ydk
cd ~/projects/ydk
git clone https://github.com/CiscoDevNet/ydk-gen
cd ydk-gen
```

Doing this in a virtual environment is not a bad idea. If you want to use the generated API
for an Ansible module, then it will need to be present in the global package list. We'll get
to more on that later.

```commandline
virtualenv -p python2.7 venv
source venv/bin/activate
```
Install YDK-Gen's module dependencies into the virtual environment (this will take a bit):

```commandline
pip install -r requirements.txt
```
Now *generate* should work. As a test:

```commandline
./generate.py --help
```
Generate the core API and install it:

```commandline
./generate.py --python --core
pip install gen-api/python/ydk/dist/ydk*.tar.gz
```
Create an API bundle description. You can use the OSPF or MACsec ones included in this repository as examples:

```commandline
vi profiles/bundles/ciscolive-ansible-ospf.json
```
Now we're ready to generate an API from the bundle description:

```commandline
./generate.py --python --bundle profiles/bundles/ciscolive-ansible-ospf.json
```
You can install the generated API into the virtual environment as a test or if you're planning to use the
API for something other than Ansible:

```commandline
pip install gen-api/python/ciscolive_ansible_ospf-bundle/dist/ydk*.tar.gz
```
If you're ready to try Ansible + YDK, you'll need to back out of the virtual environment and install YDK
and your custom API (if any) into the global package list. Ansible doesn't pick up the "hooks" needed to
be aware of the virtual environment, therefore you have to use the global list:

```commandline
deactivate
sudo pip install -r requirements.txt
sudo pip install gen-api/python/ydk/dist/ydk*.tar.gz
sudo pip install gen-api/python/ciscolive_ansible_ospf-bundle/dist/ydk*.tar.gz
```
