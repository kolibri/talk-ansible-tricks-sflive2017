# Ansible Tricks 'n' Tips

## To keep you rocking!

(Be warned, this is not for beginners)

---
# Who is this guy?

Lukas Sadzik, Software Developer @ Sensiolabs

Automatization fan (aka "lazy guy")

[@ko_libri](https://twitter.com/ko_libri)

lukas.sadzik@sensiolabs.de

------
# Setup ansible

------
# Install from git

```bash
# install requirements
$ sudo easy_install pip
$ sudo pip install paramiko PyYAML Jinja2 httplib2 six
```

```bash
# clone & source ansible
$ git clone git://github.com/ansible/ansible.git --recursive

# you may add this to your ~/.bashrc
$ source ./ansible/hacking/env-setup
```

```bash
# update
$ git pull --rebase
```

------
# Name your steps

Just do it!

*Info*:
- allows you to use the `--start-at-task` option of the `ansible-playbook` command which may save a lot of time.
- it is documentation of your scripts.
- will lead to better output and a possibility to search for a (failing) step name in your scripts.

------
# Directory layout

```
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   ├── all.yml    <- Vars for all hosts
│   │   ├── aerith.yml <- Vars for all hosts on aerith group
│   │   ├── cid.yml    <- and so on
│   │   └── cloud.yml  <- ...
│   ├── host_vars
│   │   └── aerith.ko.yml <- vars only for host this host
│   ├── prod           <- inventory for "prod"
│   ├── dev            <- and for "dev"
│   └── test           <- you know what this is..
├── roles
│   └── my_role
└── site.yml
```

```bash
$ ansible-playbook -i inventory/dev site.yml
```

*Info*: 
- Combines seperate inventory files and grouped vars files
- May be unhandy, with all the group files in one directory. So, it does not scale to very large environments.

---
## Inventory: 

### Use IPs and group your hosts

```ini
[cid]
cid.ko ansible_host=192.168.178.23
cid.dev ansible_host=192.168.3.23

[aerith]
aerith.ko ansible_host=192.168.178.24

[cloud]
cloud.ko ansible_host=192.168.178.25

[arch:children]
cid
cloud

[arch:vars]
ansible_python_interpreter=/usr/bin/python2
```

*Info*:
- with this you can use `ansible_nodename` you have IP and hostname at one place. You can use `ansible_nodename` to set the hostname and `ansible_default_ipv4.address` for the ip. Think about using this in you local `/etc/hosts`.

------
# Use "real" yaml syntax instead of inline style

---
```yaml
- name: ensure root privileges for user
  lineinfile:
    dest: /etc/sudoers
    state: present
    regexp: "^{{ user_name }} ALL"
    line: "{{ user_name }} ALL=(ALL) NOPASSWD:ALL"
    validate: "visudo -cf %s" # almost all file modules can validate  
```

*Info*:
- Is more readable ;)
- Allows you to add/remove options by lines (git will thank you)
- Drawback: You have to add `"` around `{{ variables }}`
- Hint: Did you see the `validate`? ;)

------
# Getting started/Initialize a new host

---
- Manage your users with ansible.
- Describe the end-state of your system. 
- Pass inital related config (setup-user credentials) via commandline on first run
- Tag "inital" tasks, to run them on the inital playbook run

*Info*:
- Do not have a "init playbook"
- end-state: includes inventory file, etc.
- Have the script, that ensures you can run ansible as a part of the usual scripts.

---
Example:
```yaml
- name: ensure user
  user:
    name: aerith
    generate_ssh_key: yes
  tags: [init]

- name: add authorized key of controll machine 
  authorized_key:
    user: aerith
    key: "{{ lookup('file', '/home[...]id_rsa.pub') }}"
  tags: [init]
```

```bash
# first run
$ ansible-playbook -u setup_user -k -t init site.yml
# all other runs
$ ansible-playbook site.yml
```

*Info*:
- `lookup` searches on the controll machine! 
- `-k` (lower "k" is shortcut for `ask-pass`, that will ask for connection password)

------
# Ansible Galaxy

---
```yaml
# requirements.yml
- geerlingguy.jenkins # short name for roles from ansible galaxy

- src: https://github.com/bennojoy/nginx
  version: master
  name: nginx
```

```ini
# ansible.cfg
[defaults]
roles_path = ./galaxy_roles:./roles
```

```bash
# install the roles
$ ansible-galaxy install \
  --role-file=requirements.yml \
  --roles-path=galaxy_roles

# does the same, but with shortcuts for options
$ ansible-galaxy install -r=requirements.yml -p=galaxy_roles
```

*Info*:
- [galaxy cli docs](http://docs.ansible.com/ansible/galaxy.html#installing-multiple-roles-from-a-file)

---
## Create a new role with `ansible-galaxy`

```bash
$ ansible-galaxy init --init-path=roles/ --offline my_new_role
```

---
```
.
└── my_new_role
    ├── README.md
    ├── defaults      <- Role configuration interface
    │   └── main.yml
    ├── files         <- static files
    ├── handlers      <- post run commands, notified by tasks
    │   └── main.yml
    ├── meta          <- informations about role. Author, dependencies, etc.
    │   └── main.yml
    ├── tasks         <- Main part of the role
    │   └── main.yml
    ├── templates     <- Jinja2 file templates
    ├── tests         <- Tests for role
    │   ├── inventory
    │   └── test.yml
    └── vars          <- internal configuration
        └── main.yml
```

*Info*:
[`ansible-galaxy init` in the docs](http://docs.ansible.com/ansible/galaxy.html#create-roles)

------
# Variable naming schema

---

## This looks good, but...

```yaml
user:
  name: my_user
  home: /home/my_user

```
---

## ... this does the job better

```yaml
user_name: my_user
user_home: "/home/{{ user_name }}"
```

*Info*:
- Prefix your variable names with the role name.
- Do NOT use `hash_behavior=merge`, and "nested" variables, even if it looks sexy on the first time.
    + this allows you to reuse variables.

------
# Using variables from another role

---
```yaml
# roles/user/defaults/main.yml
user_home: /home/my_user
```

```yaml
# roles/zsh/defaults/main.yml
zsh_user_home: "{{ user_home }}"
```

```yaml
# roles/zsh/tasks/main.yml
- name: set .zshrc
  template:
    src: zshrc.j2
    dest: "{{ zsh_user_home }}/.zshrc"
```

*Info*:
- In your tasks, do not rely on variables from another role.
- "Import" the variable in you `defaults/main.yml`.

------
# `defaults` vs `vars`

---
## defaults

- Configuration, you expose to the user
- Some kind of "arguments", you can use for the role
- "public"
---

## vars

- Configuration inside the role
- Typically environment specific & loaded by facts
- "private"

---
## Vars example
```yaml
# roles/ssh/vars/Debian.yml
ssh_service_name: ssh
```

```yaml
# roles/ssh/vars/Archlinux.yml
ssh_service_name: sshd
```

```yaml
# roles/ssh/tasks/main.yml
- name: Include OS-Specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: enable ssh service
  service:
    name: "{{ ssh_service_name }}"
    enabled: yes
```

------
# `become: true`

*Info*:
There is a lot of hazzle about the `become` directive, most targeting its, in first case, uncommon naming.

And yes, it may look strange to say `become: true` to execute something with root privileges. 

---
## become another user? Yes!

---
## Configure `become`

- `become`: `true`/`false` (default: `false`), toggles `become`.

- `become_user`: Username to become (default: `root`).

- `become_method`: One of: `sudo`, `su`, `pbrun`, `pfexec`, `doas`, `dzdo`, `ksu` (default: `sudo`).

- `become_flags`: Additional flags, like `-s /bin/sh`.

------
# Places to configure `become`

- At task level
- At includes
- At Role includes in playbook
- At the command line

------
# A word about `block`

Don't use them. Use `include`s instead.

*Info*:
- the idea of block is, to group tasks and apply some directives at a single point.
- Blocks can't be looped, what's sad, really sad.
- Blocks introduce strange, not intuitive conventions, intendention, etc.
- [Block in the docs](http://docs.ansible.com/ansible/playbooks_blocks.html)

------
# `register`

---
```yaml
- name: ensure user 
  user:
    name: aerith
    generate_ssh_key: true
  register: user_info

- debug: var=user_info
```

```json
{
    "user_info": {
        "append": false,
        "changed": false,
        "comment": "",
        "group": 100,
        "home": "/home/aerith",
        "move_home": false,
        "name": "aerith",
        "shell": "/bin/zsh",
        "ssh_fingerprint": "2048 SHA256:G[...]A ansible-generated (RSA)",
        "ssh_key_file": "/home/aerith/.ssh/id_rsa",
        "ssh_public_key": "ssh-rsa AA[...]bF ansible-generated",
        "state": "present",
        "uid": 1000
    }
}
```

*Info*:
- A LOT modules support the `register` directive. Just try it!
- Output may depend on what you've set up in the registering task
- [`register` in the docs](http://docs.ansible.com/ansible/playbooks_variables.html#registered-variables)

------
# Read input from JSON/YAML

```yaml
- shell: cat file.json
  register: result

- set_fact: myvar="{{ result.stdout | from_json }}"
```

```yaml
- shell: cat file.yaml
  register: result

- set_fact: myvar="{{ result.stdout | from_yaml }}"
```

*Info*: 
[`from_yaml` & `from_json` in the docs](http://docs.ansible.com/ansible/playbooks_filters.html#filters-for-formatting-data)

------
# Tip about services and handlers

- In tasks: Configure service, enable them, etc., but do not restart them!
- In Handlers: just only restart services.

*Info*:
- This makes your configuration more flexible
- This delegates responsibilites to where there belong

---
```yaml
# role/tasks/main.yml
- name: enable some service
  service:
    name: someservice
    enabled: true
  notify: restart someservice
```

```yaml
# role/handlers/main.yml
- name: restart someservice
  service: someservice
    state: restarted
```

*Info*: 
- Do not forget the `notify`
- [handlers in the docs](http://docs.ansible.com/ansible/playbooks_intro.html#handlers-running-operations-on-change)
