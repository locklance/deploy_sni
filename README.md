# deploy-xray-sni
![Ansible](https://img.shields.io/badge/automation-Ansible-blue?style=flat-square)
![Xray](https://img.shields.io/badge/xray-supported-purple?style=flat-square)

Following guide, you will be able to deploy your own SNI with a DNS-01 certificate, which can be used as self steal SNI for VLESS-REALITY.

Two deploy methods are described below:
1. [Manual deploy](#manual-deploy)
2. [Automatic deploy](#automatic-mode-deployment-ansible)
>[!WARNING]
> You have to obtain Cloudflare domain for accurate SNI setup
---
## Manual deploy
1. [Issue your own API-token](#obtaining-a-cloudflare-token) before
2. Clone SNI repository
```sh
git clone https://github.com/locklance/xray-sni.git
```
3. Fill in `.env` file with obtained API-token: 
```sh
cd xray-sni
vim .env

----- Fill in .env: -----
SNI_DOMAIN="sni.example.com" # SNI dest address
SNI_PORT="9443" # SNI dest port
CF_API_TOKEN="YOUR_CF_API_TOKEN" # Cloudflare API token
```
4. Start SNI
```sh
sudo docker compose up -d && docker compose logs -ft
```
## Automatic mode deployment (Ansible)
Automatic mode deployment may seem complex, but in fact it is very simple and takes much less time if you have two or more servers!
### Preparation
1. Initial setup of remote servers
You can skip this step if your servers are already configured:
- a custom user (not root) is created;
- SSH key-based access is set up.

If the server has not been configured yet, execute the following commands step by step:
```sh
# update packages
sudo apt update && sudo apt upgrade -y

# create a user
sudo adduser USER_NODE_1
sudo usermod -aG sudo USER_NODE_1

# generate an SSH key
sudo ssh-keygen -t ed25519
cat .ssh/id_rsa.pub # copy the contents

# add the public SSH key
vim .ssh/authorized_keys # paste the contents of .ssh/id_rsa.pub

# configure SSH
sudo vim /etc/ssh/sshd_config
# Port <new port>
# PasswordAuthentication no

sudo systemctl restart ssh
```
2. Add ssh keys to ssh-agent
To use ansible you need to own ssh keys for remote servers and add them to ssh-agent, by using:
```sh
ssh-add .ssh/your_key
```
3. Create venv
You also need to create venv:
```sh
python3 -m venv venv
source venv/bin/activate
```
4. Install Ansible
Install Ansible:
```sh
pip install ansible
```
5. Install additional roles
Install role for docker installation:
```sh
ansible-galaxy install geerlingguy.docker
```
6. Configure the inventory file
This is the most important step, where we need to fill in the `ansible/inventory.yml` file with the data required for SSH connections to the servers.
Example:
```yml
vpnnodes:
  hosts:
    DE_NODE_1:
      ansible_host: 127.0.0.1 <---- SERVER IP ADDRESS
      ansible_port: 22 <----SSH PORT
      ansible_user: DE_NODE_1 <---- SSH USER
    DE_NODE_2:
      ansible_host: 127.0.0.2
      ansible_port: 23
      ansible_user: DE_NODE_2
germany:
  hosts:
    DE_NODE_1:
    DE_NODE_2:
  vars:
    domain: de.example.com <---- DOMAIN
```

### Check Ansible configs
Check ansible inventory:
```sh
ansible-inventory -i ansible/inventory.yml --list
```

Ping servers in inventory:
`ansible vpnnodes -m ping -i ansible/inventory.yml`
you can use any of group, hosts etc instead (for example `vpnnodes`, `FL_NODE_1`). 

## SNI deployment
1. Create `secrets.yml` with `CF_API_TOKEN` inside sni role:
```sh
└── roles
    └── sni
        ├── files
        │   └── secrets.yml <---- PUT CF_API_TOKEN HERE
        ├── tasks
        │   └── main.yml
        └── templates
            └── env.j2
```
Example of **secrets.yml** file:
```sh
# secrets.yml example
CF_API_TOKEN: your_api_token
```

2. Start Ansible script:
```sh
ansible-playbook ansible/playbooks/deploy_sni.yml -i ansible/inventory.yml -l finland
```
> \* Use `--ask-become-pass` option to enter sudo if not working

> \*\* To use your own SNI, in the `deploy_sni.yml` file change the `repo_url` variable to the URL of the repository of your own SNI:
> ```yml
> vars:
>     docker_install_compose: true
>     docker_group: docker
>     repo_url: "https://github.com/locklance/xray-sni" <---- SNI URL
>     dest_subdir: "xray-sni"
>     ...
> ```

## Obtaining a Cloudflare token
### Registering a domain with Cloudflare
First of all, we need to register a domain. Create a real-looking unsuspicious domain.
![Register domain](/docs/media/create-domain.png)

### Get API-token
1. Go to your personal profile
![Profile](/docs/media/cf-profile.png)
2. Find the API Tokens section
![API Tokens](/docs/media/get-api-key-2.png)
3. Create a new token
![New token](/docs/media/get-api-key-3.png)
4. Add rules to the token
![Permissions](/docs/media/get-api-key-4.png)
