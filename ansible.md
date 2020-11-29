# Ansible 2.9.9 [DRAFT]

## Notes

- Ansible is only required on the server side, there's no need to install a specific ansible package on a client (host) to get this working.

## Main settings

### Install facter on clients (you could do this when Ansible is already set up, with a playbook)

```bash
sudo apt update
sudo apt install -y facter
```

### Set up Ansible (server) on the Ubuntu > 20.04

Run the following:

```bash
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

### My Ansible.cfg

My settings:

```bash
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

- Start with `---`
- Jinja variables should be between

{% raw %}

```yaml
{{ }}
```

{% endraw %}

example:

{% raw %}

```yaml
notify: "{{ foo.msg }}"
```

{% endraw %}

- Use `hosts` to define them

```yaml
 - hosts: <name/ip/range>
 ```

- Use `become` when the playbook has to be executed as superuser, you can also use this at task level (so only that task is run as superuser) or vice versa when it's set to false!

```yaml
  become: <true/false>
```

- Use `vars_files` to add a yaml configuration containing all variables for this playbook

```yaml
  vars_files:
    - vars.yml
```

- Use `pre_tasks` to run any tasks before the main tasks

for example, updating apt if it hasn't been updated for longer than 3600 seconds (1h):

```yaml
  pre_tasks:
    - name: Update apt cache.
      apt:
        update_cache: true
        cache_valid_time: 3600
```

- Use `handlers` if you need to run a task, these only run when a change has been made on a client.
   Handlers are tasks that only run at the end of the playbook when they're notified & when the client response is "changed".

for example, restart a service if a task updates the configuration of that service, but not if the configuration is unchanged

```yaml
  handlers:
    - name: reload sshd
      service:
        name: sshd
        state: reloaded
```

example how to notify a handler after a task has been executed under `tasks` (for tasks, see point 8):

```yaml
    - name: Change "/etc/ssh/sshd_config" to disallow PasswordAuthentication.
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#PasswordAuthentication yes'
        insertafter: '^#PasswordAuthentication yes'
        line: PasswordAuthentication no
      notify: reload sshd
```

> ### ⚠️ Exception
>
> You can use `flush handlers` to execute the notified handlers at a certain point, before the end of the playbook!
> <br>
> example:
>
> ```yaml
>    - name: Flush handlers immediately.
>      meta: flush_handlers
>```

- Use `tasks` to execute commands.
   Each task executes a module with specific arguments, when the task is executed, the next task will be executed.

   Tasks exist of a `name`, `module` & `arguments of the module` also adding `states` is considered best practice.

example:

```yaml
  tasks:
  - name: <name of the task>
    <module>:
      state: <present/absent>
      <arg1>: argument 1
      <arg2>: argument 2
```
