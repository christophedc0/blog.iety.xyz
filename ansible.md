# Ansible 2.9.9 [DRAFT]

## Notes
- Ansible is only required on the server side, there's no need to install a specific ansible package on a client (host) to get this working.

## Main settings

### Install facter on clients (you could do this when Ansible is already set up, with a playbook)
```
sudo apt update
sudo apt install -y facter
```

### Set up Ansible (server) on the Ubuntu > 20.04

Run the following:
```
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

### My Ansible.cfg

My settings:
```
[defaults]
# inventory file of your hosts
inventory = /etc/ansible/hosts
# use python3 as interpreter on the clients
interpreter_python = python3
# Gather facts of the clients
gathering = implicit
# Gather facts from other packages for example: "facter"
gather_subset = all
# Disable host key checking (or the host key must already be known on the server
host_key_checking = False
# Convert output to YAML
stdout_callback = yaml
# Specify where to cache the gathered facts (this is a folder)
fact_caching_connection=/etc/ansible/facts

[ssh_connection]
# Use sftp and then try scp for transferring files (default: smart; true (scp only); false (sftp only))
## Old method
scp_if_ssh = smart
## New method
transfer_method = smart
# Number of times to retry if the client is unreachable
retries = 3
```

## Playbook syntax

1. Start with `---`
2. Jinja variables should be between `"{{  }}"`

    example: `notify: "{{ foo.msg }}"
3. Define hosts with `- hosts: <name>`
4. 
