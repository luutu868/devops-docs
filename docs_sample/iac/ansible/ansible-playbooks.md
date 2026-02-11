# Ansible Playbooks

## Giới thiệu Playbooks

### Playbook là gì?

Playbook là file YAML định nghĩa các tasks cần thực hiện trên managed nodes.

**Cấu trúc cơ bản:**
```yaml
---
- name: Playbook name
  hosts: target_hosts
  become: yes
  vars:
    variable_name: value
  tasks:
    - name: Task description
      module_name:
        parameter: value
```

**Components:**
- **hosts**: Target hosts/groups
- **become**: Run as sudo
- **vars**: Variables
- **tasks**: List of tasks
- **handlers**: Triggered by notify

### First Playbook

**playbook.yml:**
```yaml
---
- name: Install and start Apache
  hosts: webservers
  become: yes
  
  tasks:
    - name: Install Apache
      apt:
        name: apache2
        state: present
        update_cache: yes
    
    - name: Start Apache service
      service:
        name: apache2
        state: started
        enabled: yes
    
    - name: Copy index.html
      copy:
        content: "<h1>Hello from Ansible</h1>"
        dest: /var/www/html/index.html
```

**Run playbook:**
```bash
ansible-playbook playbook.yml

# Check mode (dry run)
ansible-playbook playbook.yml --check

# Verbose output
ansible-playbook playbook.yml -v
ansible-playbook playbook.yml -vvv

# Limit to specific hosts
ansible-playbook playbook.yml --limit web01

# Start at specific task
ansible-playbook playbook.yml --start-at-task="Install Apache"
```

## Variables

### Defining Variables

**In playbook:**
```yaml
---
- name: Using variables
  hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
    server_name: "{{ ansible_hostname }}"
  
  tasks:
    - name: Display variables
      debug:
        msg: "Server {{ server_name }} on port {{ http_port }}"
```

**External vars file** - `vars/main.yml`:
```yaml
http_port: 80
https_port: 443
ssl_certificate: /etc/ssl/certs/server.crt
ssl_key: /etc/ssl/private/server.key
```

**Use vars file:**
```yaml
---
- name: Using external vars
  hosts: webservers
  vars_files:
    - vars/main.yml
  
  tasks:
    - name: Configure Apache
      template:
        src: apache.conf.j2
        dest: /etc/apache2/sites-available/default.conf
```

### Variable Precedence

```
Priority (highest to lowest):
1. Extra vars (-e)
2. Task vars
3. Block vars
4. Role vars
5. Play vars_files
6. Play vars
7. Host facts
8. Inventory vars
9. Role defaults
```

**Example:**
```bash
# Extra vars (highest priority)
ansible-playbook playbook.yml -e "http_port=8080"
ansible-playbook playbook.yml -e "@extra_vars.yml"
```

### Facts

**Gather facts automatically:**
```yaml
---
- name: Display facts
  hosts: all
  gather_facts: yes
  
  tasks:
    - name: Show OS family
      debug:
        msg: "OS: {{ ansible_os_family }}"
    
    - name: Show IP address
      debug:
        msg: "IP: {{ ansible_default_ipv4.address }}"
    
    - name: Show memory
      debug:
        msg: "Memory: {{ ansible_memtotal_mb }} MB"
```

**Useful facts:**
```yaml
ansible_hostname          # Hostname
ansible_fqdn              # FQDN
ansible_os_family         # Debian, RedHat, etc
ansible_distribution      # Ubuntu, CentOS, etc
ansible_distribution_version  # 20.04, 8, etc
ansible_default_ipv4.address  # IP address
ansible_processor_cores   # CPU cores
ansible_memtotal_mb       # Total memory
ansible_devices           # Disk devices
ansible_mounts            # Mount points
```

**Custom facts:**
```bash
# /etc/ansible/facts.d/custom.fact
[general]
environment=production
datacenter=us-east-1
```

```yaml
- name: Display custom fact
  debug:
    msg: "Env: {{ ansible_local.custom.general.environment }}"
```

## Conditionals

### When Condition

```yaml
---
- name: Conditional tasks
  hosts: all
  
  tasks:
    - name: Install Apache on Debian
      apt:
        name: apache2
        state: present
      when: ansible_os_family == "Debian"
    
    - name: Install Apache on RedHat
      yum:
        name: httpd
        state: present
      when: ansible_os_family == "RedHat"
    
    - name: Install on Ubuntu 20.04 only
      apt:
        name: package
        state: present
      when:
        - ansible_distribution == "Ubuntu"
        - ansible_distribution_version == "20.04"
```

### Multiple Conditions

```yaml
tasks:
  # AND conditions
  - name: Install on production Ubuntu
    apt:
      name: package
      state: present
    when:
      - ansible_distribution == "Ubuntu"
      - environment == "production"
  
  # OR conditions
  - name: Install on Debian or Ubuntu
    apt:
      name: package
      state: present
    when: ansible_os_family == "Debian" or ansible_distribution == "Ubuntu"
  
  # Complex conditions
  - name: Complex condition
    debug:
      msg: "Match"
    when: (ansible_distribution == "Ubuntu" and ansible_distribution_version >= "20.04") or
          (ansible_distribution == "CentOS" and ansible_distribution_major_version == "8")
```

### Checking Variables

```yaml
tasks:
  # Variable is defined
  - name: Task if var defined
    debug:
      msg: "Variable set"
    when: my_var is defined
  
  # Variable is not defined
  - name: Task if var not defined
    set_fact:
      my_var: "default_value"
    when: my_var is not defined
  
  # Variable is empty
  - name: Check if empty
    debug:
      msg: "Empty"
    when: my_var | length == 0
  
  # Boolean check
  - name: If enabled
    debug:
      msg: "Enabled"
    when: feature_enabled | bool
```

## Loops

### Basic Loops

```yaml
---
- name: Loop examples
  hosts: webservers
  
  tasks:
    - name: Install multiple packages
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - nginx
        - postgresql
        - redis
        - git
    
    - name: Create multiple users
      user:
        name: "{{ item }}"
        state: present
        shell: /bin/bash
      loop:
        - alice
        - bob
        - charlie
    
    - name: Create directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /opt/app
        - /opt/logs
        - /opt/config
```

### Loop with Dictionary

```yaml
tasks:
  - name: Create users with details
    user:
      name: "{{ item.name }}"
      uid: "{{ item.uid }}"
      groups: "{{ item.groups }}"
      state: present
    loop:
      - { name: 'alice', uid: 1001, groups: 'admin,developers' }
      - { name: 'bob', uid: 1002, groups: 'developers' }
      - { name: 'charlie', uid: 1003, groups: 'operators' }
  
  - name: Configure virtual hosts
    template:
      src: vhost.conf.j2
      dest: "/etc/nginx/sites-available/{{ item.name }}.conf"
    loop:
      - { name: 'site1', port: 8001, root: '/var/www/site1' }
      - { name: 'site2', port: 8002, root: '/var/www/site2' }
      - { name: 'site3', port: 8003, root: '/var/www/site3' }
```

### Loop Control

```yaml
tasks:
  - name: Loop with index
    debug:
      msg: "Item {{ ansible_loop.index }}: {{ item }}"
    loop:
      - apple
      - banana
      - cherry
    loop_control:
      index_var: idx
  
  - name: Loop with pause
    command: "process {{ item }}"
    loop:
      - file1.txt
      - file2.txt
      - file3.txt
    loop_control:
      pause: 2  # Pause 2 seconds between items
  
  - name: Loop with label
    user:
      name: "{{ item.name }}"
      uid: "{{ item.uid }}"
    loop:
      - { name: 'user1', uid: 1001, other_data: 'xxx' }
      - { name: 'user2', uid: 1002, other_data: 'yyy' }
    loop_control:
      label: "{{ item.name }}"  # Only show name in output
```

### Loop Filters

```yaml
tasks:
  # Loop until condition
  - name: Wait for service
    uri:
      url: "http://localhost:8080/health"
    register: result
    until: result.status == 200
    retries: 10
    delay: 5
  
  # Loop from range
  - name: Create numbered files
    file:
      path: "/tmp/file{{ item }}.txt"
      state: touch
    loop: "{{ range(1, 6) | list }}"  # 1, 2, 3, 4, 5
  
  # Loop with filtering
  - name: Install only on web servers
    debug:
      msg: "Installing on {{ item }}"
    loop: "{{ groups['all'] }}"
    when: "'web' in group_names"
```

## Handlers

### Basic Handlers

```yaml
---
- name: Configure web server
  hosts: webservers
  become: yes
  
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
      notify: restart nginx
    
    - name: Copy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
        - restart php-fpm
    
    - name: Copy site config
      copy:
        src: site.conf
        dest: /etc/nginx/sites-available/default
      notify: reload nginx
  
  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
    
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
    
    - name: restart php-fpm
      service:
        name: php-fpm
        state: restarted
```

**Handler features:**
- Run only once at the end
- Run only if notified
- Run in order defined
- Can be notified by multiple tasks

### Handler Listen

```yaml
tasks:
  - name: Update config 1
    copy:
      src: config1.conf
      dest: /etc/app/config1.conf
    notify: restart app
  
  - name: Update config 2
    copy:
      src: config2.conf
      dest: /etc/app/config2.conf
    notify: restart app

handlers:
  - name: restart app
    service:
      name: myapp
      state: restarted
    listen: restart app
  
  - name: clear cache
    command: rm -rf /var/cache/myapp
    listen: restart app
```

### Force Handlers

```yaml
---
- name: Playbook with forced handlers
  hosts: webservers
  force_handlers: yes  # Run handlers even if task fails
  
  tasks:
    - name: Update config
      copy:
        src: config.conf
        dest: /etc/app/config.conf
      notify: restart app
    
    - name: This might fail
      command: /bin/false
  
  handlers:
    - name: restart app
      service:
        name: myapp
        state: restarted
```

## Templates (Jinja2)

### Basic Template

**templates/nginx.conf.j2:**
```jinja2
user www-data;
worker_processes {{ ansible_processor_cores }};

events {
    worker_connections {{ worker_connections | default(1024) }};
}

http {
    server {
        listen {{ http_port }};
        server_name {{ server_name }};
        
        root {{ document_root }};
        index index.html index.php;
        
        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

**Playbook:**
```yaml
---
- name: Configure Nginx
  hosts: webservers
  vars:
    http_port: 80
    server_name: "example.com"
    document_root: "/var/www/html"
    worker_connections: 2048
  
  tasks:
    - name: Deploy Nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
        backup: yes
      notify: reload nginx
  
  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
```

### Template with Conditionals

**templates/apache.conf.j2:**
```jinja2
<VirtualHost *:{{ http_port }}>
    ServerName {{ server_name }}
    DocumentRoot {{ document_root }}
    
    {% if ssl_enabled %}
    SSLEngine on
    SSLCertificateFile {{ ssl_cert }}
    SSLCertificateKeyFile {{ ssl_key }}
    {% endif %}
    
    {% if environment == "production" %}
    CustomLog /var/log/apache2/access.log combined
    ErrorLog /var/log/apache2/error.log
    {% else %}
    CustomLog /var/log/apache2/dev-access.log combined
    ErrorLog /var/log/apache2/dev-error.log
    {% endif %}
    
    <Directory {{ document_root }}>
        {% if allow_override %}
        AllowOverride All
        {% else %}
        AllowOverride None
        {% endif %}
        Require all granted
    </Directory>
</VirtualHost>
```

### Template with Loops

**templates/hosts.j2:**
```jinja2
127.0.0.1   localhost

# Application servers
{% for host in groups['app'] %}
{{ hostvars[host]['ansible_default_ipv4']['address'] }}   {{ hostvars[host]['ansible_hostname'] }}
{% endfor %}

# Database servers
{% for host in groups['db'] %}
{{ hostvars[host]['ansible_default_ipv4']['address'] }}   {{ hostvars[host]['ansible_hostname'] }}-db
{% endfor %}
```

### Template Filters

```jinja2
{# String filters #}
{{ server_name | upper }}
{{ server_name | lower }}
{{ server_name | capitalize }}
{{ description | default("No description") }}

{# List filters #}
{{ users | length }}
{{ users | first }}
{{ users | last }}
{{ users | join(', ') }}
{{ numbers | max }}
{{ numbers | min }}
{{ numbers | sum }}

{# Dictionary filters #}
{{ user | to_json }}
{{ user | to_yaml }}
{{ config | combine(overrides) }}

{# Math filters #}
{{ memory_mb | int / 1024 }}
{{ (cpu_percent | float) * 1.2 }}

{# File path filters #}
{{ file_path | basename }}
{{ file_path | dirname }}

{# IP address filters #}
{{ ip_address | ipaddr('address') }}
{{ network | ipaddr('network') }}

{# Date filters #}
{{ ansible_date_time.date }}
{{ ansible_date_time.time }}
{{ ansible_date_time.iso8601 }}
```

## Blocks

### Error Handling

```yaml
---
- name: Block example
  hosts: webservers
  
  tasks:
    - name: Handle errors with block
      block:
        - name: Install package
          apt:
            name: non-existent-package
            state: present
        
        - name: This won't run if previous fails
          debug:
            msg: "Package installed"
      
      rescue:
        - name: Run on error
          debug:
            msg: "Installation failed, using alternative"
        
        - name: Install alternative
          apt:
            name: alternative-package
            state: present
      
      always:
        - name: Always run
          debug:
            msg: "This always runs"
```

### Block with When

```yaml
- name: Block with condition
  block:
    - name: Task 1
      debug:
        msg: "Task 1"
    
    - name: Task 2
      debug:
        msg: "Task 2"
  when: ansible_os_family == "Debian"
  become: yes
```

## Roles Structure

### Create Role

```bash
# Create role structure
ansible-galaxy init myrole

# Directory structure
myrole/
├── defaults/       # Default variables
│   └── main.yml
├── files/          # Static files
├── handlers/       # Handlers
│   └── main.yml
├── meta/           # Role metadata
│   └── main.yml
├── tasks/          # Tasks
│   └── main.yml
├── templates/      # Templates
├── tests/          # Tests
│   ├── inventory
│   └── test.yml
└── vars/           # Role variables
    └── main.yml
```

### Example Role: webserver

**roles/webserver/tasks/main.yml:**
```yaml
---
- name: Install web server
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - php-fpm
  
- name: Copy Nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: reload nginx

- name: Ensure web root exists
  file:
    path: "{{ web_root }}"
    state: directory
    mode: '0755'

- name: Deploy application
  copy:
    src: "{{ app_files }}"
    dest: "{{ web_root }}"
  notify: reload nginx
```

**roles/webserver/defaults/main.yml:**
```yaml
---
web_root: /var/www/html
http_port: 80
worker_processes: auto
worker_connections: 1024
```

**roles/webserver/handlers/main.yml:**
```yaml
---
- name: reload nginx
  service:
    name: nginx
    state: reloaded

- name: restart nginx
  service:
    name: nginx
    state: restarted
```

**Use role:**
```yaml
---
- name: Setup web servers
  hosts: webservers
  roles:
    - webserver
```

**Override role variables:**
```yaml
---
- name: Setup web servers
  hosts: webservers
  roles:
    - role: webserver
      web_root: /opt/app
      http_port: 8080
```

## Complete Examples

### LAMP Stack Deployment

**playbook.yml:**
```yaml
---
- name: Deploy LAMP stack
  hosts: webservers
  become: yes
  vars:
    mysql_root_password: "SecurePassword123!"
    db_name: "myapp"
    db_user: "appuser"
    db_password: "AppPass123!"
  
  tasks:
    # Apache
    - name: Install Apache
      apt:
        name: apache2
        state: present
        update_cache: yes
    
    - name: Start Apache
      service:
        name: apache2
        state: started
        enabled: yes
    
    # MySQL
    - name: Install MySQL
      apt:
        name:
          - mysql-server
          - python3-pymysql
        state: present
    
    - name: Start MySQL
      service:
        name: mysql
        state: started
        enabled: yes
    
    - name: Set MySQL root password
      mysql_user:
        name: root
        password: "{{ mysql_root_password }}"
        login_unix_socket: /var/run/mysqld/mysqld.sock
    
    - name: Create database
      mysql_db:
        name: "{{ db_name }}"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"
    
    - name: Create database user
      mysql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        priv: "{{ db_name }}.*:ALL"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"
    
    # PHP
    - name: Install PHP
      apt:
        name:
          - php
          - php-mysql
          - libapache2-mod-php
        state: present
    
    - name: Deploy PHP info page
      copy:
        content: "<?php phpinfo(); ?>"
        dest: /var/www/html/info.php
    
    - name: Restart Apache
      service:
        name: apache2
        state: restarted
```

### Docker Container Deployment

**playbook.yml:**
```yaml
---
- name: Deploy application with Docker
  hosts: app_servers
  become: yes
  vars:
    app_name: "myapp"
    app_version: "latest"
    container_port: 8080
    host_port: 80
  
  tasks:
    - name: Install Docker dependencies
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: yes
    
    - name: Add Docker GPG key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present
    
    - name: Add Docker repository
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present
    
    - name: Install Docker
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - python3-docker
        state: present
        update_cache: yes
    
    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes
    
    - name: Pull application image
      docker_image:
        name: "{{ app_name }}:{{ app_version }}"
        source: pull
    
    - name: Stop old container
      docker_container:
        name: "{{ app_name }}"
        state: stopped
      ignore_errors: yes
    
    - name: Remove old container
      docker_container:
        name: "{{ app_name }}"
        state: absent
      ignore_errors: yes
    
    - name: Run application container
      docker_container:
        name: "{{ app_name }}"
        image: "{{ app_name }}:{{ app_version }}"
        state: started
        restart_policy: always
        ports:
          - "{{ host_port }}:{{ container_port }}"
        env:
          NODE_ENV: "production"
          DB_HOST: "{{ db_host }}"
          DB_NAME: "{{ db_name }}"
        volumes:
          - /opt/data:/app/data
        networks:
          - name: app_network
```

### Zero-Downtime Deployment

**playbook.yml:**
```yaml
---
- name: Zero-downtime deployment
  hosts: webservers
  serial: 1  # Deploy one server at a time
  max_fail_percentage: 0
  
  tasks:
    - name: Remove from load balancer
      haproxy:
        state: disabled
        host: "{{ inventory_hostname }}"
        backend: web_backend
        socket: /var/run/haproxy/admin.sock
      delegate_to: "{{ item }}"
      loop: "{{ groups['loadbalancers'] }}"
    
    - name: Wait for connections to drain
      wait_for:
        timeout: 10
    
    - name: Stop application
      service:
        name: myapp
        state: stopped
    
    - name: Deploy new version
      copy:
        src: "{{ app_package }}"
        dest: /opt/myapp/
      notify: restart app
    
    - name: Start application
      service:
        name: myapp
        state: started
    
    - name: Wait for application to start
      wait_for:
        port: 8080
        delay: 5
        timeout: 60
    
    - name: Health check
      uri:
        url: "http://{{ inventory_hostname }}:8080/health"
        status_code: 200
      register: health
      until: health.status == 200
      retries: 5
      delay: 2
    
    - name: Add back to load balancer
      haproxy:
        state: enabled
        host: "{{ inventory_hostname }}"
        backend: web_backend
        socket: /var/run/haproxy/admin.sock
      delegate_to: "{{ item }}"
      loop: "{{ groups['loadbalancers'] }}"
  
  handlers:
    - name: restart app
      service:
        name: myapp
        state: restarted
```

## Best Practices

### Playbook Organization

```
ansible-project/
├── inventories/
│   ├── production/
│   │   ├── hosts
│   │   └── group_vars/
│   └── staging/
│       ├── hosts
│       └── group_vars/
├── roles/
│   ├── common/
│   ├── webserver/
│   └── database/
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── databases.yml
├── group_vars/
│   ├── all.yml
│   └── webservers.yml
├── host_vars/
│   └── web01.yml
├── files/
├── templates/
├── ansible.cfg
└── requirements.yml
```

### Security

```yaml
# Use vault for secrets
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault encrypt secrets.yml
ansible-vault decrypt secrets.yml

# Run with vault
ansible-playbook playbook.yml --ask-vault-pass
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# In playbook
vars_files:
  - secrets.yml

# No log for sensitive data
- name: Set password
  user:
    name: john
    password: "{{ user_password }}"
  no_log: true
```

### Performance

```yaml
# Parallel execution
- hosts: webservers
  serial: 5  # 5 hosts at a time
  
  # Or percentage
  serial: "30%"

# Async tasks
- name: Long running task
  command: /usr/bin/long_task
  async: 300  # Timeout
  poll: 10    # Check every 10 seconds

# Gather facts once
- hosts: all
  gather_facts: yes
  
- hosts: webservers
  gather_facts: no  # Reuse facts

# Use delegate_to
- name: Add to monitoring
  uri:
    url: "http://monitoring/api/add"
    body: "{{ inventory_hostname }}"
  delegate_to: localhost
  run_once: true
```

### Testing

```bash
# Syntax check
ansible-playbook playbook.yml --syntax-check

# Check mode (dry run)
ansible-playbook playbook.yml --check

# Diff mode
ansible-playbook playbook.yml --check --diff

# List tasks
ansible-playbook playbook.yml --list-tasks

# List hosts
ansible-playbook playbook.yml --list-hosts

# Step mode
ansible-playbook playbook.yml --step
```

Master Ansible playbooks for powerful automation!
