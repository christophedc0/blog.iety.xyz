# Ansible 2.9.9 [DRAFT]

## Notes

This page is based on own research, [Ansible Latest Docs](https://docs.ansible.com/ansible/latest/), [r/homelab](http://reddit.com/r/homelab/) and especially Jeff Geerling's [Ansible for DevOps book](https://www.ansiblefordevops.com) & [YouTube livestream](http://jeffgeerling.com/blog/2020/ansible-101-jeff-geerling-youtube-streaming-series).

- Ansible is only required on the server side, there's no need to install a specific ansible package on a client (host) to get this working.

## Main settings

### Install facter on clients (you could do this when Ansible is already set up, with a playbook)

This allows us to gather more facts from the clients.

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

- Use `become` when the playbook has to be executed as superuser,
  you can also use this at task level (so only that task is run as superuser) or vice versa when it's set to false!

  ```yaml
    become: <true/false>
  ```

- Use `vars_files` to add a yaml configuration containing all variables for this playbook

  ```yaml
    vars_files:
      - vars.yml
  ```

- Use `vars` to add variables directly in the playbook.

  ```yaml
    vars:
      - key: foo
  ```

### Tasks

- Use `pre_tasks` to run any tasks before the main tasks

  for example, updating the package manager if it hasn't been updated for longer than 3600 seconds (1h):

  ```yaml
    pre_tasks:
      - name: Update package manager cache.
        package:
          update_cache: true
          cache_valid_time: 3600
  ```

> We use package so it works on different package managers, apt, yum, ...

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

- Use `register`to make something a variable.
  
  In this example we use the shell output in a variable named "path":

```yaml
    - name: Get the value of an environment variable.
      shell: 'echo $PATH'
      register: path
```

- Use `debug` to print out the variable.

```yaml
    - debug: msg="The variable is {{ path.stdout }}."
```

- Use `when` to decide when to run this task depending on other tasks/variables.

- Use `changed_when` or `failed_when` to decide when to run this task depending on changes or fails.

- Use `ignore_errors` to true if the task could give an error, but you don't care about it and want to continue.

- Use `tags` to be able to only execute that specific task from cli. Use this sparingly.

- Use `import_tasks` to import another yaml with "fixed" tasks, use this as a default (if it breaks, try `include_tasks`).

- Use `include_tasks` to import another yaml with dynamically changing tasks

- Use `import_playbook` to import another playbook.

### Handlers

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

  > #### ⚠️ Exception
  >
  > You can use `flush handlers` to execute the notified handlers at a certain point, before the end of the playbook!
  > <br>
  > example:
  >
  > ```yaml
  >    - name: Flush handlers immediately.
  >      meta: flush_handlers
  >```

### Use variables for specific tasks

I will use the variable "{{ ansible_os_family }}" to determine weither the package to install Apache is "httpd" (centos, redhat) or "apache2" (debian,ubuntu).

In this example I also use other `vars` and `register` to be more dynamic.

Playbook.yml

```yaml
---
  - hosts: all
    become: true

    handlers:
      - name: restart apache
        service:
          - name: "{{ apache_service }}"
            state: restarted

    pre_tasks:
# I add debug to verify if the os family is correct. This is optional.
      - debug: var=ansible_os_family
      - name: Load variable files.
        state: present
        include_vars: "{{ item }}"
        with_first_found:
          - configs/apache_{{ ansible_os_family }}.yml
          - configs/apache_default.yml

    tasks:
      - name: Install apache
        package:
          state: present
          name: "{{ apache_service }}"
          state: present
        register: install_status_apache

# I add debug to check what happens with the installation status message
      - debug: var=install_status_apache['message']

      - name: Copy config file
        copy:
          state: present
          src: files/apache.conf
          dest: "{{ apache_config_dir }}/apache.conf"
        notify: restart apache
```

Apache_RedHat.yml

```yaml
---
apache_package: httpd
apache_service: httpd
apache_config_dir: /etc/httpd/conf.d/
```

Apache_Ubuntu.yml

```yaml
---
---
apache_package: apache2
apache_service: apache2
apache_config_dir: /etc/apache2/sites-enabled/
```

### Roles

Roles are used to package/group tasks, to be reused.

A roles folder must contain at minimum a `meta` and `tasks` directory.

1. Create /etc/ansible/roles/role1

    Create a main.yml playbook and add a role in it.

2. Create a `meta` directory inside of /etc/ansible/roles/role1

    Create a main.yml file that contains a list of dependencies that are needed.

3. Create a `tasks` directory inside of /etc/ansible/roles/role1

    Create the tasks in main.yml that the role needs to execute.

#### Example

/etc/ansible/roles/role1/*`meta`*/main.yml

(This example role doesn't have any dependencies)

```yaml
---
dependencies: []
```

/etc/ansible/roles/role1/*`tasks`*/main.yml

```yaml
---
- name: Copy config file
  copy:
    state: present
    src: files/apache.conf
    dest: "{{ apache_config_dir }}/apache.conf"
  notify: restart apache
```

/etc/ansible/roles/*`role1`*/main.yml

```yaml
---
  - hosts: all
    become: true

    handlers:
      - name: restart apache
        service:
          - name: "{{ apache_service }}"
            state: restarted

    pre_tasks:
# I add debug to verify if the os family is correct. This is optional.
      - debug: var=ansible_os_family
      - name: Load variable files.
        state: present
        include_vars: "{{ item }}"
        with_first_found:
          - configs/apache_{{ ansible_os_family }}.yml
          - configs/apache_default.yml

    roles:
      - role1

    tasks:
      - name: Install apache
        package:
          state: present
          name: "{{ apache_service }}"
          state: present
        register: install_status_apache


```

## Ansible Vault

example playbook /etc/ansible/playbooks/main.yml

```yaml
---
- hosts: localhost
  connection: local

  vars_files:
    - /etc/ansible/vars/api_key.yml

  tasks:
    - name: Echo the API key which is injected in the environment.
      shell: echo $API_KEY
      environment:
        API_KEY = "{{ api_key }}"
      register: echo_result

    - name: Show the result.
      debug: var = echo_result.stdout
```

### How to encrypt keys, passwords & sensitive data

We have an API key in /etc/ansible/vars/api_key.yml
which contains the following:

```yaml
---
api_key: "AFGG2piXh0ht6dmXUxqv4nA1PU120r0yMAQhuc13i8"
```

To encrypt this:

```bash
ansible-vault encrypt /etc/ansible/vars/api_key.yml
```

This will ask for a password to encrypt it.
Now the file is encrypted using AES256 from ansible vault.

But how do we use this file now in our playbook?

```bash
ansible-playbook main.yml --ask-vault-pass
```

Now it will ask for the password that you used for encrypting the api key file.

But we can supply a password file for this.

```bash
touch ~/.ansible/api_key_pass.txt
echo "YOURPASSWORD" > ~/.ansible/api_key_pass.txt
```

Now to run the playbook:

```bash
ansible-playbook main.yml --vault-password-file ~/.ansible/api_key_pass.txt
```

### How to decrypt files

```bash
ansible-vault decrypt /etc/ansible/vars/api_key.yml
```

Now it will ask for the password that you used for encrypting the api key file.

### How to edit encrypted files without decrypting it

```bash
ansible-vault edit /etc/ansible/vars/api_key.yml
```

Now it will ask for the password that you used for encrypting the api key file.
Now you can edit the file without having to decrypt and re-encrypt it.

### How to change the ecryption password

```bash
ansible-vault rekey /etc/ansible/vars/api_key.yml
```

Now it will ask for the password that you used for encrypting the api key file, and requires to type in a new password.
