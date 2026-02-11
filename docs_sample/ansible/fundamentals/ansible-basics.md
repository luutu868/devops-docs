# Ansible Cơ Bản

## Giới Thiệu

### Ansible Là Gì?

Ansible là một công cụ tự động hóa mã nguồn mở được phát triển bởi Red Hat, được sử dụng để quản lý cấu hình, triển khai ứng dụng và tự động hóa các tác vụ IT. Ansible cho phép bạn tự động hóa việc cấu hình và quản lý hàng trăm, thậm chí hàng nghìn máy chủ một cách đơn giản và hiệu quả.

### So Sánh Với Phương Pháp Truyền Thống

**Phương pháp truyền thống:**
- Cấu hình thủ công trên từng máy chủ
- Sử dụng script shell phức tạp và khó bảo trì
- Thiếu tính nhất quán giữa các môi trường
- Khó theo dõi và kiểm soát thay đổi
- Tốn thời gian và dễ xảy ra lỗi con người

**Với Ansible:**
- Tự động hóa hoàn toàn với code
- Infrastructure as Code (IaC)
- Đảm bảo tính nhất quán
- Dễ dàng quản lý phiên bản và rollback
- Tiết kiệm thời gian và giảm thiểu lỗi

### Lợi Ích Của Ansible

**1. Agentless (Không Cần Agent)**

Ansible không yêu cầu cài đặt agent trên các máy chủ được quản lý. Điều này giúp:
- Giảm overhead trên managed nodes
- Đơn giản hóa việc triển khai
- Không cần quản lý và cập nhật agent
- Giảm thiểu rủi ro bảo mật

**2. SSH-Based**

Ansible sử dụng SSH để kết nối và thực thi các tác vụ:
- Sử dụng cơ chế bảo mật đã được kiểm chứng
- Không cần mở thêm port
- Hỗ trợ SSH key và password
- Tận dụng các tính năng SSH như jump hosts

**3. Idempotent (Đồng Nhất)**

Ansible đảm bảo tính idempotent - chạy nhiều lần cùng một playbook sẽ cho kết quả giống nhau:
- Chỉ thực hiện thay đổi khi cần thiết
- An toàn khi chạy lại playbook
- Dễ dàng duy trì trạng thái mong muốn
- Giảm thiểu tác động không mong muốn

**4. Simple YAML Syntax**

Ansible sử dụng YAML - một ngôn ngữ dễ đọc, dễ viết:
- Cú pháp đơn giản, dễ học
- Không cần kiến thức lập trình sâu
- Dễ dàng đọc hiểu và bảo trì
- Phù hợp cho cả developer và ops

**5. Powerful Modules**

Ansible cung cấp hàng nghìn modules sẵn có:
- Module cho hầu hết các tác vụ phổ biến
- Hỗ trợ nhiều platform và service
- Community-maintained và enterprise modules
- Dễ dàng mở rộng với custom modules

## Kiến Trúc Ansible

### Các Thành Phần Chính

```
┌─────────────────────────────────────────────────────────────┐
│                      CONTROL NODE                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Ansible    │  │   Inventory  │  │   Playbooks  │      │
│  │    Engine    │  │    Files     │  │     .yml     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                  │                  │              │
│         └──────────────────┴──────────────────┘              │
└─────────────────────────────┬───────────────────────────────┘
                              │ SSH/WinRM
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│ MANAGED NODE  │     │ MANAGED NODE  │     │ MANAGED NODE  │
│               │     │               │     │               │
│  - No Agent   │     │  - No Agent   │     │  - No Agent   │
│  - SSH Access │     │  - SSH Access │     │  - SSH Access │
│  - Python*    │     │  - Python*    │     │  - Python*    │
└───────────────┘     └───────────────┘     └───────────────┘
```

**Control Node (Máy Điều Khiển)**
- Máy chủ cài đặt Ansible
- Nơi chạy playbooks và ad-hoc commands
- Quản lý inventory và cấu hình
- Yêu cầu: Linux/Unix, Python 3.8+

**Inventory (Danh Sách Máy Chủ)**
- Định nghĩa các managed nodes
- Hỗ trợ static (file) và dynamic (script/plugin)
- Tổ chức thành groups
- Chứa variables cho hosts/groups

**Playbooks**
- File YAML định nghĩa các tasks
- Mô tả trạng thái mong muốn
- Có thể tái sử dụng với roles và includes
- Hỗ trợ conditionals, loops, handlers

**Modules**
- Đơn vị công việc nhỏ nhất trong Ansible
- Được thực thi trên managed nodes
- Trả về JSON kết quả
- Hàng nghìn modules có sẵn

**Managed Nodes (Máy Chủ Được Quản Lý)**
- Máy chủ được Ansible quản lý
- Không cần cài đặt agent
- Yêu cầu: SSH access và Python (hoặc PowerShell cho Windows)
- Có thể là servers, network devices, cloud resources

### Quy Trình Hoạt Động

1. **Đọc Playbook**: Ansible đọc và parse playbook YAML
2. **Lấy Inventory**: Xác định danh sách hosts cần thực thi
3. **Gather Facts**: Thu thập thông tin về managed nodes (nếu enabled)
4. **Thực Thi Tasks**: Chạy từng task theo thứ tự
5. **Trả Về Kết Quả**: Hiển thị status (changed, ok, failed, skipped)

## Cài Đặt Ansible

### Ubuntu/Debian (PPA - Recommended)

```bash
# Thêm Ansible PPA repository
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible

# Cài đặt Ansible
sudo apt install ansible

# Kiểm tra phiên bản
ansible --version
```

### CentOS/RHEL (EPEL)

```bash
# Cài đặt EPEL repository
sudo yum install epel-release

# Hoặc cho RHEL 8/9
sudo dnf install epel-release

# Cài đặt Ansible
sudo yum install ansible

# Hoặc với DNF
sudo dnf install ansible

# Kiểm tra phiên bản
ansible --version
```

### Python Pip (Recommended - Version Control)

Đây là phương pháp được khuyến nghị vì cho phép kiểm soát phiên bản tốt hơn:

```bash
# Cài đặt pip nếu chưa có
sudo apt install python3-pip  # Ubuntu/Debian
sudo yum install python3-pip  # CentOS/RHEL

# Tạo virtual environment (khuyến nghị)
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate

# Cài đặt Ansible
pip install ansible

# Cài đặt phiên bản cụ thể
pip install ansible==9.0.0

# Hoặc cài core version
pip install ansible-core==2.16.0

# Kiểm tra cài đặt
ansible --version
pip list | grep ansible
```

### Xác Minh Cài Đặt

```bash
# Kiểm tra phiên bản
ansible --version

# Output mẫu:
# ansible [core 2.16.0]
#   config file = /etc/ansible/ansible.cfg
#   configured module search path = ['/home/user/.ansible/plugins/modules']
#   ansible python module location = /usr/lib/python3/dist-packages/ansible
#   ansible collection location = /home/user/.ansible/collections
#   executable location = /usr/bin/ansible
#   python version = 3.10.12
#   jinja version = 3.0.3

# Test kết nối localhost
ansible localhost -m ping

# Liệt kê modules có sẵn
ansible-doc -l

# Xem document của một module
ansible-doc ping
ansible-doc apt
```

## Cấu Hình Ansible

### File Cấu Hình ansible.cfg

Ansible đọc file cấu hình theo thứ tự ưu tiên:
1. `ANSIBLE_CONFIG` (biến môi trường)
2. `./ansible.cfg` (thư mục hiện tại)
3. `~/.ansible.cfg` (home directory)
4. `/etc/ansible/ansible.cfg` (global)

### Cấu Hình Đầy Đủ

```ini
[defaults]
# Inventory file location
inventory = ./inventory/hosts

# Number of parallel processes
forks = 10

# SSH timeout
timeout = 30

# Gather facts từ managed nodes
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

# Roles path
roles_path = ./roles:/usr/share/ansible/roles

# Collections path
collections_paths = ./collections:~/.ansible/collections:/usr/share/ansible/collections

# Disable SSH host key checking (không khuyến nghị cho production)
host_key_checking = False

# Retry files
retry_files_enabled = True
retry_files_save_path = ./retry

# Log file
log_path = ./ansible.log

# Default remote user
remote_user = ansible

# Private key file
private_key_file = ~/.ssh/ansible_key

# Ansible-vault password file
vault_password_file = ~/.vault_pass

# Callbacks và plugins
callback_whitelist = profile_tasks, timer

# Hiển thị kết quả với màu sắc
force_color = True
nocolor = False

# Strategy - cách thực thi tasks
strategy = linear  # Options: linear, free, debug

# Error handling
any_errors_fatal = False

# Deprecation warnings
deprecation_warnings = True
command_warnings = True

# Module defaults
module_name = command

[privilege_escalation]
# Sudo/become configuration
become = True
become_method = sudo
become_user = root
become_ask_pass = False

# Sudo flags
# become_flags = -H -S -n

[ssh_connection]
# SSH connection configuration
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s

# Pipelining - tăng tốc độ
pipelining = True

# Control path
control_path = /tmp/ansible-ssh-%%h-%%p-%%r

# SCP/SFTP settings
scp_if_ssh = smart
transfer_method = smart

# SSH timeout
timeout = 30

[persistent_connection]
# Persistent connection timeout
connect_timeout = 30
command_timeout = 30

[colors]
# Customize output colors
highlight = white
verbose = blue
warn = bright purple
error = red
debug = dark gray
deprecate = purple
skip = cyan
unreachable = red
ok = green
changed = yellow
diff_add = green
diff_remove = red
diff_lines = cyan

[inventory]
# Inventory cache
enable_plugins = host_list, script, auto, yaml, ini, toml
cache = yes
cache_plugin = jsonfile
cache_timeout = 3600
cache_connection = /tmp/ansible_inventory
```

### File Cấu Hình Tối Thiểu

```ini
[defaults]
inventory = ./inventory
host_key_checking = False
retry_files_enabled = False
gathering = smart

[privilege_escalation]
become = True
become_method = sudo
become_user = root

[ssh_connection]
pipelining = True
```

### Kiểm Tra Cấu Hình

```bash
# Hiển thị cấu hình hiện tại
ansible-config dump

# Hiển thị các thay đổi từ default
ansible-config dump --only-changed

# Liệt kê tất cả các options
ansible-config list

# Xem giá trị của một setting cụ thể
ansible-config dump | grep forks
```

## Inventory

### INI Format

#### Single Hosts

```ini
# inventory/hosts

# Standalone hosts
web1.example.com
web2.example.com
192.168.1.10
```

#### Groups

```ini
[webservers]
web1.example.com
web2.example.com
web3.example.com

[databases]
db1.example.com
db2.example.com

[monitoring]
monitor.example.com

# Group of groups
[production:children]
webservers
databases
monitoring
```

#### Host Variables

```ini
[webservers]
web1.example.com ansible_host=192.168.1.10 ansible_port=2222
web2.example.com ansible_host=192.168.1.11 http_port=8080
web3.example.com ansible_host=192.168.1.12 ansible_user=ubuntu

[databases]
db1.example.com ansible_host=10.0.1.20 mysql_port=3306
db2.example.com ansible_host=10.0.1.21 mysql_port=3307
```

#### Group Variables

```ini
[webservers]
web1.example.com
web2.example.com

[webservers:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/deploy_key
nginx_version=1.20
http_port=80
https_port=443

[databases]
db1.example.com
db2.example.com

[databases:vars]
ansible_user=dbadmin
mysql_root_password=changeme
mysql_port=3306
backup_enabled=true
```

#### Hierarchies và Patterns

```ini
# Multiple groups và hierarchies
[web]
web[1:3].example.com

[app]
app[01:10].example.com

[db_masters]
db-master-01.example.com
db-master-02.example.com

[db_slaves]
db-slave-[01:05].example.com

[databases:children]
db_masters
db_slaves

[staging:children]
web
app

[production:children]
databases

# Global variables
[all:vars]
ansible_connection=ssh
ansible_python_interpreter=/usr/bin/python3
```

### YAML Format

```yaml
# inventory/hosts.yml

all:
  hosts:
    localhost:
      ansible_connection: local
  
  children:
    webservers:
      hosts:
        web1.example.com:
          ansible_host: 192.168.1.10
          ansible_port: 2222
          http_port: 8080
        web2.example.com:
          ansible_host: 192.168.1.11
          http_port: 8080
        web3.example.com:
          ansible_host: 192.168.1.12
          http_port: 8080
      vars:
        ansible_user: deploy
        ansible_ssh_private_key_file: ~/.ssh/deploy_key
        nginx_version: "1.20"
        enable_ssl: true
    
    databases:
      hosts:
        db1.example.com:
          ansible_host: 10.0.1.20
          mysql_port: 3306
          replica_of: none
          backup_schedule: "0 2 * * *"
        db2.example.com:
          ansible_host: 10.0.1.21
          mysql_port: 3306
          replica_of: db1.example.com
          backup_schedule: "0 3 * * *"
      vars:
        ansible_user: dbadmin
        mysql_root_password: "{{ vault_mysql_password }}"
        mysql_max_connections: 500
        backup_enabled: true
    
    monitoring:
      hosts:
        monitor.example.com:
          ansible_host: 192.168.1.50
          grafana_port: 3000
          prometheus_port: 9090
      vars:
        ansible_user: monitor
        alert_email: ops@example.com
    
    production:
      children:
        webservers:
        databases:
        monitoring:
      vars:
        environment: production
        backup_retention_days: 30
        monitoring_enabled: true
    
    staging:
      hosts:
        stage1.example.com:
          ansible_host: 192.168.2.10
        stage2.example.com:
          ansible_host: 192.168.2.11
      vars:
        environment: staging
        backup_retention_days: 7
        monitoring_enabled: false
```

### Dynamic Inventory

#### AWS EC2 Example

```bash
# Cài đặt boto3
pip install boto3

# Cấu hình AWS credentials
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_REGION="ap-southeast-1"
```

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2

regions:
  - ap-southeast-1
  - ap-southeast-2

filters:
  instance-state-name: running
  tag:Environment:
    - production
    - staging

keyed_groups:
  # Group by instance type
  - key: instance_type
    prefix: instance_type
  # Group by tags
  - key: tags.Environment
    prefix: env
  # Group by availability zone
  - key: placement.availability_zone
    prefix: az

hostnames:
  - dns-name
  - private-ip-address

compose:
  ansible_host: public_ip_address
  ansible_user: "'ubuntu' if 'ubuntu' in image.name else 'ec2-user'"
  environment: tags.Environment
  application: tags.Application
```

```bash
# Sử dụng dynamic inventory
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph

# Chạy playbook với dynamic inventory
ansible-playbook -i inventory/aws_ec2.yml playbook.yml
```

#### Custom Dynamic Inventory Script

```python
#!/usr/bin/env python3
# inventory/custom_inventory.py

import json
import sys

def get_inventory():
    inventory = {
        'webservers': {
            'hosts': ['web1.example.com', 'web2.example.com'],
            'vars': {
                'ansible_user': 'deploy',
                'http_port': 80
            }
        },
        'databases': {
            'hosts': ['db1.example.com', 'db2.example.com'],
            'vars': {
                'ansible_user': 'dbadmin',
                'mysql_port': 3306
            }
        },
        '_meta': {
            'hostvars': {
                'web1.example.com': {
                    'ansible_host': '192.168.1.10'
                },
                'web2.example.com': {
                    'ansible_host': '192.168.1.11'
                },
                'db1.example.com': {
                    'ansible_host': '10.0.1.20'
                },
                'db2.example.com': {
                    'ansible_host': '10.0.1.21'
                }
            }
        }
    }
    return inventory

if __name__ == '__main__':
    if len(sys.argv) == 2 and sys.argv[1] == '--list':
        print(json.dumps(get_inventory(), indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == '--host':
        print(json.dumps({}))
    else:
        print("Usage: {} --list or {} --host <hostname>".format(
            sys.argv[0], sys.argv[0]))
        sys.exit(1)
```

### Inventory Commands

```bash
# Liệt kê tất cả hosts
ansible all --list-hosts

# Liệt kê hosts trong một group
ansible webservers --list-hosts

# Liệt kê hosts với pattern
ansible 'web*' --list-hosts

# Hiển thị inventory dạng graph
ansible-inventory --graph

# Hiển thị inventory dạng JSON
ansible-inventory --list

# Hiển thị variables của một host
ansible-inventory --host web1.example.com

# Liệt kê tất cả groups
ansible localhost -m debug -a "var=groups.keys()"

# Test kết nối với hosts
ansible all -m ping

# Test với specific group
ansible webservers -m ping

# Sử dụng nhiều inventory files
ansible-playbook -i inventory/hosts -i inventory/aws_ec2.yml playbook.yml
```

## Ad-hoc Commands

### Basic Commands

```bash
# Ping tất cả hosts
ansible all -m ping

# Ping specific group
ansible webservers -m ping

# Kiểm tra uptime
ansible all -m command -a "uptime"
ansible all -a "uptime"  # -a ngắn gọn hơn

# Kiểm tra disk space
ansible all -m command -a "df -h"

# Kiểm tra memory
ansible all -m command -a "free -m"

# Kiểm tra processes
ansible all -m command -a "ps aux | head -20"

# Kiểm tra hostname
ansible all -m command -a "hostname"

# Kiểm tra network interfaces
ansible all -m command -a "ip addr show"

# Kiểm tra routing table
ansible all -m command -a "ip route"

# Kiểm tra DNS resolution
ansible all -m command -a "nslookup google.com"

# Kiểm tra OS version
ansible all -m command -a "cat /etc/os-release"

# Kiểm tra kernel version
ansible all -m command -a "uname -r"

# Kiểm tra load average
ansible all -m command -a "cat /proc/loadavg"
```

### Package Management

```bash
# Ubuntu/Debian - APT
# Cài đặt package
ansible webservers -m apt -a "name=nginx state=present" -b

# Cài đặt nhiều packages
ansible webservers -m apt -a "name=nginx,git,curl state=present" -b

# Cài đặt package version cụ thể
ansible webservers -m apt -a "name=nginx=1.18.0-0ubuntu1 state=present" -b

# Update apt cache
ansible all -m apt -a "update_cache=yes" -b

# Upgrade tất cả packages
ansible all -m apt -a "upgrade=dist" -b

# Remove package
ansible webservers -m apt -a "name=nginx state=absent" -b

# Remove package và purge config
ansible webservers -m apt -a "name=nginx state=absent purge=yes" -b

# CentOS/RHEL - YUM/DNF
# Cài đặt package
ansible databases -m yum -a "name=mysql-server state=present" -b

# Cài đặt từ URL
ansible all -m yum -a "name=https://example.com/package.rpm state=present" -b

# Update tất cả packages
ansible all -m yum -a "name=* state=latest" -b

# Remove package
ansible databases -m yum -a "name=mysql-server state=absent" -b

# Cài đặt groups
ansible all -m yum -a "name='@Development Tools' state=present" -b
```

### Service Management

```bash
# Start service
ansible webservers -m service -a "name=nginx state=started" -b

# Stop service
ansible webservers -m service -a "name=nginx state=stopped" -b

# Restart service
ansible webservers -m service -a "name=nginx state=restarted" -b

# Reload service (không downtime)
ansible webservers -m service -a "name=nginx state=reloaded" -b

# Enable service at boot
ansible webservers -m service -a "name=nginx enabled=yes" -b

# Disable service at boot
ansible webservers -m service -a "name=nginx enabled=no" -b

# Start và enable service
ansible webservers -m service -a "name=nginx state=started enabled=yes" -b

# Kiểm tra status service (systemd)
ansible all -m systemd -a "name=sshd state=started" -b

# Daemon reload (sau khi sửa unit file)
ansible all -m systemd -a "daemon_reload=yes" -b
```

### File Operations

```bash
# Copy file từ local đến remote
ansible all -m copy -a "src=/tmp/test.txt dest=/tmp/test.txt mode=0644"

# Copy với owner và group
ansible all -m copy -a "src=/tmp/test.txt dest=/tmp/test.txt owner=www-data group=www-data mode=0644" -b

# Copy với backup
ansible all -m copy -a "src=/etc/nginx/nginx.conf dest=/etc/nginx/nginx.conf backup=yes" -b

# Tạo file với content
ansible all -m copy -a "content='Hello World\n' dest=/tmp/hello.txt"

# Tạo directory
ansible all -m file -a "path=/opt/myapp state=directory mode=0755" -b

# Tạo directory với owner
ansible all -m file -a "path=/var/www/html state=directory owner=www-data group=www-data mode=0755" -b

# Xóa file
ansible all -m file -a "path=/tmp/test.txt state=absent"

# Xóa directory và contents
ansible all -m file -a "path=/tmp/testdir state=absent"

# Tạo symbolic link
ansible all -m file -a "src=/usr/bin/python3 dest=/usr/bin/python state=link"

# Thay đổi permissions
ansible all -m file -a "path=/tmp/test.txt mode=0600"

# Thay đổi ownership
ansible all -m file -a "path=/var/www/html owner=www-data group=www-data recurse=yes" -b

# Fetch file từ remote về local
ansible all -m fetch -a "src=/etc/hostname dest=/tmp/hostnames/"

# Synchronize directories
ansible all -m synchronize -a "src=/local/path dest=/remote/path"
```

### User Management

```bash
# Tạo user
ansible all -m user -a "name=deploy state=present" -b

# Tạo user với home directory
ansible all -m user -a "name=deploy state=present createhome=yes home=/home/deploy" -b

# Tạo user với shell
ansible all -m user -a "name=deploy state=present shell=/bin/bash" -b

# Tạo user với password (hash)
ansible all -m user -a "name=deploy password='$6$salt$hash' state=present" -b

# Thêm user vào groups
ansible all -m user -a "name=deploy groups=sudo,docker append=yes" -b

# Set SSH key cho user
ansible all -m authorized_key -a "user=deploy key='{{ lookup('file', '~/.ssh/id_rsa.pub') }}' state=present" -b

# Xóa user
ansible all -m user -a "name=deploy state=absent" -b

# Xóa user và home directory
ansible all -m user -a "name=deploy state=absent remove=yes" -b

# Tạo group
ansible all -m group -a "name=developers state=present" -b

# Xóa group
ansible all -m group -a "name=developers state=absent" -b
```

### System Information

```bash
# Gather all facts
ansible all -m setup

# Filter facts by pattern
ansible all -m setup -a "filter=ansible_eth*"

# Hiển thị memory facts
ansible all -m setup -a "filter=ansible_memory_mb"

# Hiển thị distribution facts
ansible all -m setup -a "filter=ansible_distribution*"

# Hiển thị hostname facts
ansible all -m setup -a "filter=ansible_hostname"

# Hiển thị IP address
ansible all -m setup -a "filter=ansible_default_ipv4"

# Hiển thị all IP addresses
ansible all -m setup -a "filter=ansible_all_ipv4_addresses"

# Hiển thị processor info
ansible all -m setup -a "filter=ansible_processor*"

# Hiển thị disk info
ansible all -m setup -a "filter=ansible_devices"

# Hiển thị mount points
ansible all -m setup -a "filter=ansible_mounts"

# Lưu facts vào file
ansible all -m setup --tree /tmp/facts
```

### Shell Commands

```bash
# Chạy shell command
ansible all -m shell -a "ps aux | grep nginx"

# Chạy với specific shell
ansible all -m shell -a "echo $HOME" 

# Chạy script
ansible all -m shell -a "/tmp/script.sh"

# Chạy với working directory
ansible all -m shell -a "ls -la" -a "chdir=/tmp"

# Command với pipe
ansible all -m shell -a "cat /etc/passwd | grep root"

# Command với redirection
ansible all -m shell -a "echo 'test' > /tmp/test.txt"

# Raw command (không cần Python)
ansible all -m raw -a "uptime"
```

### Rebooting Systems

```bash
# Reboot ngay lập tức
ansible all -m reboot -b

# Reboot với message
ansible all -m reboot -a "msg='Scheduled maintenance reboot'" -b

# Reboot với delay
ansible all -m reboot -a "pre_reboot_delay=60" -b

# Reboot và đợi system up
ansible all -m reboot -a "reboot_timeout=600" -b

# Test reboot
ansible all -m reboot -a "test_command='uptime'" -b
```

## Common Modules

### Module: apt

```yaml
# Cài đặt package
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

# Cài đặt nhiều packages
- name: Install multiple packages
  apt:
    name:
      - nginx
      - git
      - curl
      - vim
    state: present
    update_cache: yes

# Cài đặt version cụ thể
- name: Install specific version
  apt:
    name: nginx=1.18.0-0ubuntu1
    state: present

# Remove package
- name: Remove apache2
  apt:
    name: apache2
    state: absent
    purge: yes

# Upgrade all packages
- name: Upgrade all packages
  apt:
    upgrade: dist
    update_cache: yes
    cache_valid_time: 3600

# Install .deb file
- name: Install deb package
  apt:
    deb: /tmp/package.deb
```

### Module: yum

```yaml
# Cài đặt package
- name: Install httpd
  yum:
    name: httpd
    state: present

# Cài đặt nhiều packages
- name: Install multiple packages
  yum:
    name:
      - httpd
      - mod_ssl
      - mariadb-server
    state: present

# Cài đặt từ URL
- name: Install RPM from URL
  yum:
    name: https://example.com/package.rpm
    state: present

# Update package
- name: Update nginx
  yum:
    name: nginx
    state: latest

# Remove package
- name: Remove httpd
  yum:
    name: httpd
    state: absent

# Install group
- name: Install development tools
  yum:
    name: "@Development Tools"
    state: present
```

### Module: service

```yaml
# Start service
- name: Start nginx
  service:
    name: nginx
    state: started

# Stop service
- name: Stop apache2
  service:
    name: apache2
    state: stopped

# Restart service
- name: Restart nginx
  service:
    name: nginx
    state: restarted

# Reload service
- name: Reload nginx config
  service:
    name: nginx
    state: reloaded

# Enable service
- name: Enable nginx at boot
  service:
    name: nginx
    enabled: yes

# Start và enable
- name: Ensure nginx is running and enabled
  service:
    name: nginx
    state: started
    enabled: yes
```

### Module: copy

```yaml
# Copy file
- name: Copy nginx config
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes

# Copy với validation
- name: Copy nginx config with validation
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    validate: nginx -t -c %s
    backup: yes

# Tạo file với content
- name: Create index.html
  copy:
    content: |
      <html>
      <body>
        <h1>Welcome to Nginx</h1>
      </body>
      </html>
    dest: /var/www/html/index.html
    mode: '0644'

# Copy directory
- name: Copy website files
  copy:
    src: files/website/
    dest: /var/www/html/
    owner: www-data
    group: www-data
    mode: '0755'
```

### Module: template

```yaml
# Render Jinja2 template
- name: Configure nginx virtual host
  template:
    src: templates/vhost.j2
    dest: /etc/nginx/sites-available/{{ domain }}
    owner: root
    group: root
    mode: '0644'
    backup: yes
  notify: reload nginx

# Template với validation
- name: Configure nginx with validation
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: nginx -t -c %s

# Template với variables
- name: Create application config
  template:
    src: templates/app.conf.j2
    dest: /opt/app/config.ini
    owner: appuser
    group: appuser
    mode: '0600'
  vars:
    database_host: db.example.com
    database_port: 3306
```

Template file example (vhost.j2):
```jinja2
server {
    listen 80;
    server_name {{ domain }};
    root /var/www/{{ domain }};
    index index.html index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    {% if enable_ssl %}
    listen 443 ssl;
    ssl_certificate /etc/ssl/certs/{{ domain }}.crt;
    ssl_certificate_key /etc/ssl/private/{{ domain }}.key;
    {% endif %}
}
```

### Module: file

```yaml
# Tạo directory
- name: Create app directory
  file:
    path: /opt/myapp
    state: directory
    owner: appuser
    group: appuser
    mode: '0755'

# Tạo nested directories
- name: Create nested directories
  file:
    path: /var/www/html/assets/images
    state: directory
    owner: www-data
    group: www-data
    mode: '0755'
    recurse: yes

# Xóa file/directory
- name: Remove temporary files
  file:
    path: /tmp/cache
    state: absent

# Tạo symbolic link
- name: Create symlink
  file:
    src: /opt/app/current
    dest: /opt/app/live
    state: link

# Set permissions
- name: Set directory permissions
  file:
    path: /var/www/html
    owner: www-data
    group: www-data
    mode: '0755'
    recurse: yes

# Touch file (update timestamp)
- name: Create empty file
  file:
    path: /tmp/ansible.lock
    state: touch
    mode: '0644'
```

### Module: user

```yaml
# Tạo user
- name: Create deploy user
  user:
    name: deploy
    comment: "Deployment User"
    shell: /bin/bash
    createhome: yes
    home: /home/deploy
    state: present

# Tạo user với groups
- name: Create user with groups
  user:
    name: developer
    groups:
      - sudo
      - docker
      - www-data
    append: yes
    state: present

# Tạo user với SSH key
- name: Create user with SSH key
  user:
    name: deploy
    generate_ssh_key: yes
    ssh_key_bits: 4096
    ssh_key_file: .ssh/id_rsa

# Set password
- name: Set user password
  user:
    name: deploy
    password: "{{ 'mypassword' | password_hash('sha512') }}"
    update_password: always

# Remove user
- name: Remove user
  user:
    name: olduser
    state: absent
    remove: yes
```

### Module: git

```yaml
# Clone repository
- name: Clone application repository
  git:
    repo: https://github.com/example/app.git
    dest: /opt/app
    version: main

# Clone với authentication
- name: Clone private repository
  git:
    repo: git@github.com:example/private-repo.git
    dest: /opt/private-app
    version: develop
    key_file: /home/deploy/.ssh/id_rsa
    accept_hostkey: yes

# Pull latest changes
- name: Update repository
  git:
    repo: https://github.com/example/app.git
    dest: /opt/app
    version: main
    update: yes
    force: yes

# Clone specific branch/tag
- name: Clone specific version
  git:
    repo: https://github.com/example/app.git
    dest: /opt/app
    version: v2.1.0
```

### Module: command vs shell

```yaml
# command - an toàn hơn, không có shell features
- name: Run simple command
  command: ls -la /tmp
  register: result

- name: Execute script
  command: /opt/scripts/backup.sh
  args:
    chdir: /opt/scripts
    creates: /tmp/backup.lock

# shell - có pipe, redirect, variables
- name: Run with pipe
  shell: ps aux | grep nginx | wc -l
  register: nginx_processes

- name: Run with environment variable
  shell: echo $HOME
  register: home_dir

- name: Run with redirection
  shell: cat /etc/passwd | grep root > /tmp/root_users.txt

# Khi nào dùng command vs shell
# - command: khi không cần shell features (an toàn hơn)
# - shell: khi cần pipe, redirect, variables, wildcards
```

### Module: fetch

```yaml
# Fetch file từ remote về local
- name: Fetch log files
  fetch:
    src: /var/log/nginx/access.log
    dest: /tmp/logs/{{ inventory_hostname }}/
    flat: no

# Fetch với flat directory
- name: Fetch config file
  fetch:
    src: /etc/nginx/nginx.conf
    dest: /tmp/configs/
    flat: yes
    validate_checksum: yes

# Fetch multiple files
- name: Fetch application configs
  fetch:
    src: "{{ item }}"
    dest: /tmp/backup/
    flat: no
  loop:
    - /opt/app/config/database.yml
    - /opt/app/config/application.yml
```

### Module: synchronize

```yaml
# Sync directories (uses rsync)
- name: Sync website files
  synchronize:
    src: /local/website/
    dest: /var/www/html/
    delete: yes
    recursive: yes

# Sync với permissions
- name: Sync application files
  synchronize:
    src: /builds/app/
    dest: /opt/app/current/
    archive: yes
    compress: yes
    delete: yes
    owner: yes
    group: yes

# Sync từ remote về local
- name: Backup remote files
  synchronize:
    src: /var/www/uploads/
    dest: /backups/{{ inventory_hostname }}/
    mode: pull

# Sync với excludes
- name: Sync excluding patterns
  synchronize:
    src: /local/app/
    dest: /remote/app/
    delete: yes
    rsync_opts:
      - "--exclude=*.log"
      - "--exclude=.git"
      - "--exclude=node_modules"
```

### Module: lineinfile

```yaml
# Thêm line nếu chưa tồn tại
- name: Add line to file
  lineinfile:
    path: /etc/hosts
    line: "192.168.1.100 app.local"
    state: present

# Replace line matching pattern
- name: Update configuration
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin no'
    state: present
    backup: yes

# Insert after specific line
- name: Add configuration after match
  lineinfile:
    path: /etc/security/limits.conf
    insertafter: '^# End of file'
    line: '* soft nofile 65536'

# Insert before specific line
- name: Add configuration before match
  lineinfile:
    path: /etc/fstab
    insertbefore: '^# Custom mounts'
    line: '/dev/sdb1 /data ext4 defaults 0 0'

# Remove line
- name: Remove configuration line
  lineinfile:
    path: /etc/hosts
    regexp: '^192\.168\.1\.100'
    state: absent
```

### Module: replace

```yaml
# Replace text in file
- name: Replace string in config
  replace:
    path: /etc/nginx/nginx.conf
    regexp: 'worker_processes\s+\d+;'
    replace: 'worker_processes auto;'
    backup: yes

# Replace multiple occurrences
- name: Update all references
  replace:
    path: /opt/app/config.ini
    regexp: 'localhost'
    replace: 'db.example.com'

# Replace with backreferences
- name: Update version in file
  replace:
    path: /opt/app/version.txt
    regexp: 'Version: (\d+\.\d+\.)\d+'
    replace: 'Version: \g<1>{{ new_patch_version }}'
```

### Module: uri

```yaml
# HTTP GET request
- name: Check if service is up
  uri:
    url: http://localhost:8080/health
    method: GET
    status_code: 200

# POST request with JSON
- name: Create resource via API
  uri:
    url: https://api.example.com/resources
    method: POST
    body:
      name: "Test Resource"
      type: "server"
    body_format: json
    headers:
      Authorization: "Bearer {{ api_token }}"
    status_code: [200, 201]
  register: api_response

# Download file
- name: Download file
  uri:
    url: https://example.com/file.tar.gz
    dest: /tmp/file.tar.gz
    creates: /tmp/file.tar.gz

# Check API and validate response
- name: Validate API response
  uri:
    url: https://api.example.com/status
    return_content: yes
  register: api_status
  failed_when: "'healthy' not in api_status.content"
```

### Module: wait_for

```yaml
# Đợi port mở
- name: Wait for SSH to be available
  wait_for:
    port: 22
    host: "{{ inventory_hostname }}"
    delay: 10
    timeout: 300

# Đợi service start
- name: Wait for application to start
  wait_for:
    port: 8080
    delay: 5
    timeout: 120

# Đợi file tồn tại
- name: Wait for file to be created
  wait_for:
    path: /tmp/deployment.complete
    state: present
    timeout: 600

# Đợi string trong file
- name: Wait for application ready
  wait_for:
    path: /var/log/app.log
    search_regex: "Application started"
    timeout: 300

# Đợi port đóng (service stopped)
- name: Wait for service to stop
  wait_for:
    port: 8080
    state: stopped
    timeout: 60
```

## Variables

### Defining Variables

#### In Playbook

```yaml
---
- name: Example playbook with variables
  hosts: webservers
  vars:
    http_port: 80
    https_port: 443
    server_name: www.example.com
    max_clients: 200
    enable_ssl: true
    packages:
      - nginx
      - git
      - curl
  
  tasks:
    - name: Install packages
      apt:
        name: "{{ packages }}"
        state: present
    
    - name: Configure nginx
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      vars:
        worker_processes: 4  # Task-level variable
```

#### In Inventory

```ini
# INI format
[webservers]
web1.example.com http_port=80 max_clients=100
web2.example.com http_port=8080 max_clients=200

[webservers:vars]
ansible_user=deploy
nginx_version=1.20
enable_monitoring=true
```

```yaml
# YAML format
webservers:
  hosts:
    web1.example.com:
      http_port: 80
      max_clients: 100
    web2.example.com:
      http_port: 8080
      max_clients: 200
  vars:
    ansible_user: deploy
    nginx_version: "1.20"
    enable_monitoring: true
```

#### In Separate Files

```yaml
# group_vars/webservers.yml
---
nginx_version: "1.20"
http_port: 80
https_port: 443
max_clients: 200

packages:
  - nginx
  - git
  - curl
  - vim

ssl_config:
  enabled: true
  certificate_path: /etc/ssl/certs
  key_path: /etc/ssl/private
```

```yaml
# host_vars/web1.example.com.yml
---
ansible_host: 192.168.1.10
max_clients: 300
custom_modules:
  - geoip
  - headers
```

### Variable Precedence

Ansible áp dụng variables theo thứ tự ưu tiên (từ thấp đến cao):

1. Command line values (e.g., `-u my_user`, lowest)
2. Role defaults (defined in role/defaults/main.yml)
3. Inventory file or script group vars
4. Inventory group_vars/all
5. Playbook group_vars/all
6. Inventory group_vars/*
7. Playbook group_vars/*
8. Inventory file or script host vars
9. Inventory host_vars/*
10. Playbook host_vars/*
11. Host facts / cached set_facts
12. Play vars
13. Play vars_prompt
14. Play vars_files
15. Role vars (defined in role/vars/main.yml)
16. Block vars (only for tasks in block)
17. Task vars (only for the task)
18. Include_vars
19. Set_facts / registered vars
20. Role (and include_role) params
21. Include params
22. Extra vars (e.g., `-e "user=my_user"`, highest)

```yaml
# Ví dụ về precedence
---
- name: Variable precedence demo
  hosts: localhost
  vars:
    my_var: "playbook level"
  
  tasks:
    - name: Show variable from playbook
      debug:
        msg: "{{ my_var }}"  # Output: "playbook level"
    
    - name: Set variable with set_fact
      set_fact:
        my_var: "set_fact level"
    
    - name: Show variable after set_fact
      debug:
        msg: "{{ my_var }}"  # Output: "set_fact level"
    
    - name: Override with task vars
      debug:
        msg: "{{ my_var }}"
      vars:
        my_var: "task level"  # Output: "task level"
    
    # Extra vars (command line) luôn thắng:
    # ansible-playbook playbook.yml -e "my_var='extra vars level'"
```

### Using Variables with Jinja2

```yaml
---
- name: Jinja2 variables examples
  hosts: localhost
  vars:
    app_name: myapp
    app_version: "2.1.0"
    environment: production
    config:
      database:
        host: db.example.com
        port: 3306
      cache:
        host: redis.example.com
        port: 6379
  
  tasks:
    # Simple variable substitution
    - name: Simple substitution
      debug:
        msg: "Application {{ app_name }} version {{ app_version }}"
    
    # Dictionary access
    - name: Dictionary access
      debug:
        msg: "Database: {{ config.database.host }}:{{ config.database.port }}"
    
    # Alternative dictionary syntax
    - name: Alternative syntax
      debug:
        msg: "Cache: {{ config['cache']['host'] }}"
    
    # String concatenation
    - name: Build string
      debug:
        msg: "{{ app_name }}-{{ environment }}-{{ app_version }}"
    
    # Filters
    - name: Using filters
      debug:
        msg: "{{ app_name | upper }}"  # MYAPP
    
    - name: Default values
      debug:
        msg: "{{ undefined_var | default('default_value') }}"
    
    # Conditionals in templates
    - name: Conditional expression
      debug:
        msg: "{{ 'SSL enabled' if enable_ssl | default(false) else 'SSL disabled' }}"
    
    # Math operations
    - name: Math operations
      debug:
        msg: "{{ (config.database.port | int + 1) }}"
    
    # List operations
    - name: Join list
      debug:
        msg: "{{ ['nginx', 'git', 'curl'] | join(', ') }}"
```

### Facts (Automatic Variables)

```yaml
---
- name: Using Ansible facts
  hosts: all
  gather_facts: yes
  
  tasks:
    # Display all facts
    - name: Show all facts
      debug:
        var: ansible_facts
    
    # Common facts
    - name: Show hostname
      debug:
        msg: "Hostname: {{ ansible_hostname }}"
    
    - name: Show OS information
      debug:
        msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
    
    - name: Show IP address
      debug:
        msg: "IP: {{ ansible_default_ipv4.address }}"
    
    - name: Show memory
      debug:
        msg: "Total memory: {{ ansible_memtotal_mb }}MB"
    
    - name: Show processor info
      debug:
        msg: "CPUs: {{ ansible_processor_vcpus }}"
    
    # Conditional based on facts
    - name: Task only for Ubuntu
      debug:
        msg: "This is Ubuntu system"
      when: ansible_distribution == "Ubuntu"
    
    - name: Task based on memory
      debug:
        msg: "High memory system"
      when: ansible_memtotal_mb >= 8192
```

### Custom Facts with set_fact

```yaml
---
- name: Custom facts with set_fact
  hosts: webservers
  
  tasks:
    # Simple set_fact
    - name: Set deployment timestamp
      set_fact:
        deployment_time: "{{ ansible_date_time.iso8601 }}"
    
    # Complex fact
    - name: Build application config
      set_fact:
        app_config:
          name: "{{ app_name }}"
          version: "{{ app_version }}"
          environment: "{{ environment }}"
          deployed_at: "{{ deployment_time }}"
          deployed_by: "{{ ansible_user }}"
    
    # Conditional fact
    - name: Determine database type
      set_fact:
        db_type: "{{ 'postgresql' if ansible_distribution == 'Ubuntu' else 'mysql' }}"
    
    # Fact from command output
    - name: Get current git branch
      command: git rev-parse --abbrev-ref HEAD
      args:
        chdir: /opt/app
      register: git_output
    
    - name: Set branch fact
      set_fact:
        current_branch: "{{ git_output.stdout }}"
    
    # Use custom facts
    - name: Display custom facts
      debug:
        msg: "Deployed {{ app_config.name }} v{{ app_config.version }} at {{ app_config.deployed_at }}"
    
    # Persist custom facts
    - name: Create custom fact file
      copy:
        content: |
          [deployment]
          last_deployment={{ deployment_time }}
          version={{ app_version }}
        dest: /etc/ansible/facts.d/deployment.fact
        mode: '0644'
```

## Conditionals

### When Clause

```yaml
---
- name: Conditional examples
  hosts: all
  vars:
    create_user: true
    required_packages:
      - nginx
      - git
  
  tasks:
    # Simple boolean
    - name: Create user if enabled
      user:
        name: deploy
        state: present
      when: create_user
    
    # Negation
    - name: Task when disabled
      debug:
        msg: "User creation is disabled"
      when: not create_user
    
    # Undefined check
    - name: Task if variable is defined
      debug:
        msg: "Variable is defined"
      when: my_var is defined
    
    - name: Task if variable is undefined
      debug:
        msg: "Variable is not defined"
      when: my_var is not defined
    
    # Variable comparison
    - name: Task based on value
      debug:
        msg: "Environment is production"
      when: environment == "production"
    
    - name: Task with inequality
      debug:
        msg: "Not production environment"
      when: environment != "production"
```

### Distribution Checks

```yaml
---
- name: Distribution-specific tasks
  hosts: all
  
  tasks:
    # Check specific distribution
    - name: Install package on Ubuntu
      apt:
        name: nginx
        state: present
      when: ansible_distribution == "Ubuntu"
    
    - name: Install package on CentOS
      yum:
        name: nginx
        state: present
      when: ansible_distribution == "CentOS"
    
    # Check distribution family
    - name: Task for Debian family
      debug:
        msg: "This is Debian-based system"
      when: ansible_os_family == "Debian"
    
    - name: Task for RedHat family
      debug:
        msg: "This is RedHat-based system"
      when: ansible_os_family == "RedHat"
    
    # Version check
    - name: Task for Ubuntu 20.04
      debug:
        msg: "This is Ubuntu 20.04"
      when:
        - ansible_distribution == "Ubuntu"
        - ansible_distribution_version == "20.04"
    
    # Major version
    - name: Task for Ubuntu 20.x
      debug:
        msg: "This is Ubuntu 20"
      when:
        - ansible_distribution == "Ubuntu"
        - ansible_distribution_major_version == "20"
```

### AND/OR Conditions

```yaml
---
- name: Complex conditions
  hosts: all
  vars:
    environment: production
    enable_monitoring: true
    memory_threshold: 4096
  
  tasks:
    # AND conditions (list format)
    - name: Task with AND conditions (list)
      debug:
        msg: "Production with monitoring"
      when:
        - environment == "production"
        - enable_monitoring == true
    
    # AND conditions (single line)
    - name: Task with AND conditions (inline)
      debug:
        msg: "Production with monitoring"
      when: environment == "production" and enable_monitoring == true
    
    # OR conditions
    - name: Task with OR conditions
      debug:
        msg: "Development or staging"
      when: environment == "development" or environment == "staging"
    
    # Complex conditions
    - name: Complex condition
      debug:
        msg: "High memory production system"
      when:
        - environment == "production"
        - ansible_memtotal_mb >= memory_threshold
        - ansible_distribution in ["Ubuntu", "Debian"]
    
    # Parentheses for grouping
    - name: Grouped conditions
      debug:
        msg: "Matched complex condition"
      when: >
        (environment == "production" and enable_monitoring) or
        (environment == "staging" and ansible_memtotal_mb >= 2048)
```

### File Existence with stat

```yaml
---
- name: File existence checks
  hosts: all
  
  tasks:
    # Check if file exists
    - name: Check if file exists
      stat:
        path: /etc/nginx/nginx.conf
      register: nginx_config
    
    - name: Task if file exists
      debug:
        msg: "Nginx config exists"
      when: nginx_config.stat.exists
    
    - name: Task if file does not exist
      debug:
        msg: "Nginx config does not exist"
      when: not nginx_config.stat.exists
    
    # Check if directory exists
    - name: Check if directory exists
      stat:
        path: /var/www/html
      register: web_dir
    
    - name: Create directory if not exists
      file:
        path: /var/www/html
        state: directory
      when: not web_dir.stat.exists
    
    # Check file properties
    - name: Check file properties
      stat:
        path: /etc/shadow
      register: shadow_file
    
    - name: Task based on file mode
      debug:
        msg: "File has correct permissions"
      when:
        - shadow_file.stat.exists
        - shadow_file.stat.mode == "0640"
    
    # Check if file is a link
    - name: Check if symbolic link
      stat:
        path: /usr/bin/python
      register: python_link
    
    - name: Task if symbolic link
      debug:
        msg: "Python is a symbolic link to {{ python_link.stat.lnk_source }}"
      when:
        - python_link.stat.exists
        - python_link.stat.islnk
```

### Numeric Comparisons

```yaml
---
- name: Numeric comparison examples
  hosts: all
  vars:
    max_connections: 1000
    min_memory: 4096
    port: 8080
  
  tasks:
    # Greater than
    - name: Task if memory is sufficient
      debug:
        msg: "System has enough memory"
      when: ansible_memtotal_mb > min_memory
    
    # Greater than or equal
    - name: Task if memory meets minimum
      debug:
        msg: "Memory meets minimum requirement"
      when: ansible_memtotal_mb >= min_memory
    
    # Less than
    - name: Warning for low memory
      debug:
        msg: "Warning: Low memory system"
      when: ansible_memtotal_mb < min_memory
    
    # Range check
    - name: Task if port in valid range
      debug:
        msg: "Port is valid"
      when: port >= 1024 and port <= 65535
    
    # Math operations in conditions
    - name: Task based on calculation
      debug:
        msg: "Calculated value is sufficient"
      when: (ansible_memtotal_mb / ansible_processor_vcpus) >= 2048
```

## Loops

### Simple Lists

```yaml
---
- name: Simple loop examples
  hosts: all
  
  tasks:
    # Loop over simple list
    - name: Install multiple packages
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - nginx
        - git
        - curl
        - vim
        - htop
    
    # Loop with variable
    - name: Install packages from variable
      apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"
      vars:
        packages:
          - nginx
          - git
          - curl
    
    # Loop to create multiple files
    - name: Create multiple files
      file:
        path: "/tmp/{{ item }}"
        state: touch
      loop:
        - file1.txt
        - file2.txt
        - file3.txt
    
    # Loop to create users
    - name: Create users
      user:
        name: "{{ item }}"
        state: present
      loop:
        - alice
        - bob
        - charlie
```

### Dictionaries with Item Access

```yaml
---
- name: Loop over dictionaries
  hosts: all
  
  tasks:
    # Loop over list of dictionaries
    - name: Create users with properties
      user:
        name: "{{ item.name }}"
        group: "{{ item.group }}"
        shell: "{{ item.shell }}"
        state: present
      loop:
        - { name: 'alice', group: 'developers', shell: '/bin/bash' }
        - { name: 'bob', group: 'developers', shell: '/bin/bash' }
        - { name: 'charlie', group: 'ops', shell: '/bin/zsh' }
    
    # Loop with complex dictionaries
    - name: Create virtual hosts
      template:
        src: vhost.j2
        dest: "/etc/nginx/sites-available/{{ item.domain }}"
      loop:
        - domain: example.com
          port: 80
          root: /var/www/example
          ssl_enabled: false
        - domain: api.example.com
          port: 443
          root: /var/www/api
          ssl_enabled: true
        - domain: blog.example.com
          port: 80
          root: /var/www/blog
          ssl_enabled: false
    
    # Loop over dictionary (key-value pairs)
    - name: Set multiple environment variables
      lineinfile:
        path: /etc/environment
        line: "{{ item.key }}={{ item.value }}"
        state: present
      loop: "{{ env_vars | dict2items }}"
      vars:
        env_vars:
          JAVA_HOME: /usr/lib/jvm/java-11
          NODE_ENV: production
          APP_PORT: 8080
    
    # Access nested dictionary values
    - name: Configure databases
      debug:
        msg: "Database {{ item.name }} on {{ item.config.host }}:{{ item.config.port }}"
      loop:
        - name: maindb
          config:
            host: db1.example.com
            port: 3306
            user: root
        - name: cachedb
          config:
            host: cache.example.com
            port: 6379
            user: redis
```

### Loop with Index

```yaml
---
- name: Loop with index examples
  hosts: all
  
  tasks:
    # Basic loop with index
    - name: Create numbered files
      file:
        path: "/tmp/file_{{ item }}.txt"
        state: touch
      loop: "{{ range(1, 6) | list }}"  # 1, 2, 3, 4, 5
    
    # Loop with index using loop_control
    - name: Create files with index
      copy:
        content: "This is file number {{ index }}"
        dest: "/tmp/indexed_{{ index }}.txt"
      loop:
        - file1
        - file2
        - file3
      loop_control:
        index_var: index
    
    # Extended loop_control
    - name: Install packages with index and pause
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - nginx
        - git
        - curl
      loop_control:
        index_var: idx
        pause: 2
        label: "{{ item }}"
    
    # Loop with custom loop variable name
    - name: Create users with custom loop var
      user:
        name: "{{ user.name }}"
        state: present
      loop:
        - name: alice
        - name: bob
      loop_control:
        loop_var: user
        index_var: user_index
```

### Range Loops

```yaml
---
- name: Range loop examples
  hosts: all
  
  tasks:
    # Simple range (0 to 4)
    - name: Create numbered directories (0-4)
      file:
        path: "/tmp/dir{{ item }}"
        state: directory
      loop: "{{ range(5) | list }}"
    
    # Range with start and stop (1 to 10)
    - name: Create numbered directories (1-10)
      file:
        path: "/tmp/dir{{ item }}"
        state: directory
      loop: "{{ range(1, 11) | list }}"
    
    # Range with step
    - name: Create even numbered directories
      file:
        path: "/tmp/dir{{ item }}"
        state: directory
      loop: "{{ range(2, 21, 2) | list }}"  # 2, 4, 6, ..., 20
    
    # Zero-padded numbers
    - name: Create zero-padded directories
      file:
        path: "/tmp/dir{{ '%03d' | format(item) }}"
        state: directory
      loop: "{{ range(1, 11) | list }}"  # dir001, dir002, ..., dir010
    
    # Descending range
    - name: Countdown loop
      debug:
        msg: "{{ item }}"
      loop: "{{ range(10, 0, -1) | list }}"  # 10, 9, 8, ..., 1
```

### Advanced Loop Patterns

```yaml
---
- name: Advanced loop patterns
  hosts: all
  
  tasks:
    # Loop with when condition
    - name: Install packages conditionally
      apt:
        name: "{{ item.name }}"
        state: present
      loop:
        - { name: 'nginx', install: true }
        - { name: 'apache2', install: false }
        - { name: 'git', install: true }
      when: item.install
    
    # Nested loops (using include_tasks)
    - name: Nested loop example
      debug:
        msg: "Outer: {{ item.0 }}, Inner: {{ item.1 }}"
      loop: "{{ ['a', 'b', 'c'] | product(['1', '2', '3']) | list }}"
    
    # Loop with register
    - name: Check multiple services
      service:
        name: "{{ item }}"
        state: started
      loop:
        - nginx
        - mysql
        - redis
      register: service_results
    
    - name: Display service results
      debug:
        msg: "{{ item.item }} is {{ item.state }}"
      loop: "{{ service_results.results }}"
    
    # Loop until condition
    - name: Retry until success
      uri:
        url: "http://{{ item }}/health"
        status_code: 200
      register: result
      until: result.status == 200
      retries: 5
      delay: 10
      loop:
        - app1.example.com
        - app2.example.com
    
    # Loop with subelements
    - name: Create users with SSH keys
      authorized_key:
        user: "{{ item.0.name }}"
        key: "{{ item.1 }}"
      loop: "{{ users | subelements('ssh_keys') }}"
      vars:
        users:
          - name: alice
            ssh_keys:
              - ssh-rsa AAAAB3...alice@host
          - name: bob
            ssh_keys:
              - ssh-rsa AAAAB3...bob@host1
              - ssh-rsa AAAAB3...bob@host2
```

## Best Practices

### Variables Best Practices

```yaml
# 1. Sử dụng tên biến có ý nghĩa và nhất quán
# TỐT
app_name: myapp
app_version: "2.1.0"
database_host: db.example.com

# KHÔNG TỐT
an: myapp
v: "2.1.0"
dbh: db.example.com

# 2. Tổ chức variables theo môi trường
# group_vars/production.yml
environment: production
database_host: prod-db.example.com
backup_enabled: true

# group_vars/staging.yml
environment: staging
database_host: stage-db.example.com
backup_enabled: false

# 3. Sử dụng defaults cho optional variables
# roles/myapp/defaults/main.yml
app_port: 8080
enable_ssl: false
max_connections: 100

# 4. Tránh hardcode values
# TỐT
- name: Configure application
  template:
    src: app.conf.j2
    dest: "{{ app_config_path }}/app.conf"

# KHÔNG TỐT
- name: Configure application
  template:
    src: app.conf.j2
    dest: /opt/myapp/config/app.conf

# 5. Sử dụng YAML multiline cho giá trị dài
description: >
  This is a long description that spans
  multiple lines but will be rendered as
  a single line with spaces.

script: |
  #!/bin/bash
  echo "This is a multi-line script"
  echo "that preserves newlines"
```

### Naming Conventions

```yaml
# 1. Playbook naming
# TỐT: mô tả rõ ràng mục đích
deploy-application.yml
configure-webservers.yml
backup-databases.yml

# 2. Task naming: Luôn đặt tên tasks
# TỐT
- name: Install nginx web server
  apt:
    name: nginx
    state: present

- name: Copy nginx configuration file
  copy:
    src: nginx.conf
    dest: /etc/nginx/nginx.conf

# KHÔNG TỐT
- apt:
    name: nginx
    state: present

# 3. Role naming: sử dụng lowercase và dashes
roles/
  web-server/
  database/
  monitoring/

# 4. Variable naming: snake_case
database_hostname: db.example.com
max_connection_count: 100
enable_ssl_certificate: true

# 5. Handler naming: Mô tả hành động
handlers:
  - name: restart nginx
  - name: reload systemd
  - name: restart application service
```

### Idempotency

```yaml
# 1. Sử dụng state modules thay vì command
# TỐT - Idempotent
- name: Ensure nginx is installed
  apt:
    name: nginx
    state: present

# KHÔNG TỐT - Không idempotent
- name: Install nginx
  command: apt-get install -y nginx

# 2. Sử dụng creates/removes với command
# TỐT
- name: Extract application archive
  command: tar -xzf /tmp/app.tar.gz -C /opt/
  args:
    creates: /opt/app/

# 3. Sử dụng lineinfile thay vì echo/append
# TỐT
- name: Add configuration line
  lineinfile:
    path: /etc/hosts
    line: "192.168.1.100 app.local"
    state: present

# KHÔNG TỐT
- name: Add configuration line
  shell: echo "192.168.1.100 app.local" >> /etc/hosts

# 4. Check trước khi thay đổi
- name: Check if service exists
  stat:
    path: /etc/systemd/system/myapp.service
  register: service_file

- name: Configure service
  template:
    src: myapp.service.j2
    dest: /etc/systemd/system/myapp.service
  when: not service_file.stat.exists or force_update | default(false)
```

### Version Control

```bash
# 1. Cấu trúc thư mục Git
ansible-project/
  .git/
  .gitignore
  README.md
  ansible.cfg
  inventory/
    production/
    staging/
  group_vars/
  host_vars/
  roles/
  playbooks/
  files/
  templates/

# 2. .gitignore file
# .gitignore
*.retry
*.log
.vault_pass
.ansible/
__pycache__/
*.pyc
.DS_Store

# Chỉ ignore vault password file
.vault_pass

# Không ignore encrypted files
!group_vars/*/vault.yml

# 3. Commit best practices
# Good commit messages
git commit -m "Add nginx role for web servers"
git commit -m "Update database backup playbook to support MySQL 8.0"
git commit -m "Fix: Correct permissions for application config files"

# 4. Branching strategy
# Main branches
main          # Production-ready code
develop       # Development branch

# Feature branches
feature/add-monitoring-role
feature/update-docker-deployment
fix/nginx-config-syntax

# 5. Tags cho releases
git tag -a v1.0.0 -m "Release version 1.0.0"
git tag -a v1.1.0 -m "Add monitoring features"
```

### Security with Ansible Vault

```bash
# 1. Tạo encrypted file
ansible-vault create group_vars/production/vault.yml

# 2. Encrypt existing file
ansible-vault encrypt group_vars/production/secrets.yml

# 3. Edit encrypted file
ansible-vault edit group_vars/production/vault.yml

# 4. Decrypt file (temporary)
ansible-vault decrypt group_vars/production/vault.yml

# 5. Rekey (thay đổi password)
ansible-vault rekey group_vars/production/vault.yml

# 6. View encrypted file
ansible-vault view group_vars/production/vault.yml
```

```yaml
# Encrypted variables file
# group_vars/production/vault.yml (encrypted)
---
vault_database_password: supersecretpassword
vault_api_key: abc123xyz789
vault_ssl_key_password: anothersecret

# Regular variables file referencing vault vars
# group_vars/production/vars.yml (plain text)
---
database_host: prod-db.example.com
database_user: appuser
database_password: "{{ vault_database_password }}"

api_endpoint: https://api.example.com
api_key: "{{ vault_api_key }}"
```

```bash
# 7. Run playbook with vault
# Prompt for password
ansible-playbook -i inventory/production playbook.yml --ask-vault-pass

# Using password file
ansible-playbook -i inventory/production playbook.yml --vault-password-file .vault_pass

# Using multiple vault passwords
ansible-playbook playbook.yml --vault-id prod@prompt --vault-id dev@.vault_pass_dev

# 8. Vault password file
# .vault_pass
mysupersecretpassword

# Set file permissions
chmod 600 .vault_pass

# 9. Environment variable
export ANSIBLE_VAULT_PASSWORD_FILE=.vault_pass
ansible-playbook playbook.yml
```

### Testing Playbooks

```bash
# 1. Syntax check
ansible-playbook playbook.yml --syntax-check

# Kiểm tra tất cả playbooks
find . -name "*.yml" -type f -exec ansible-playbook {} --syntax-check \;

# 2. Dry run (check mode)
ansible-playbook playbook.yml --check

# Check mode với diff
ansible-playbook playbook.yml --check --diff

# 3. Step mode - chạy từng task một
ansible-playbook playbook.yml --step

# 4. Start at specific task
ansible-playbook playbook.yml --start-at-task="Install nginx"

# 5. Tags để chạy specific tasks
# Playbook với tags
---
- name: Configure web server
  hosts: webservers
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
      tags:
        - packages
        - nginx

    - name: Copy nginx config
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      tags:
        - config
        - nginx

    - name: Start nginx
      service:
        name: nginx
        state: started
      tags:
        - service
        - nginx

# Chạy specific tags
ansible-playbook playbook.yml --tags "nginx"
ansible-playbook playbook.yml --tags "config"
ansible-playbook playbook.yml --tags "packages,config"

# Skip tags
ansible-playbook playbook.yml --skip-tags "service"

# List tags
ansible-playbook playbook.yml --list-tags

# 6. List tasks
ansible-playbook playbook.yml --list-tasks

# List hosts
ansible-playbook playbook.yml --list-hosts

# 7. Limit hosts
ansible-playbook playbook.yml --limit webservers
ansible-playbook playbook.yml --limit web1.example.com
ansible-playbook playbook.yml --limit "web*"

# 8. Extra variables for testing
ansible-playbook playbook.yml -e "enable_debug=true"
ansible-playbook playbook.yml -e "@test_vars.yml"

# 9. Verbose output
ansible-playbook playbook.yml -v    # verbose
ansible-playbook playbook.yml -vv   # more verbose
ansible-playbook playbook.yml -vvv  # very verbose
ansible-playbook playbook.yml -vvvv # connection debugging

# 10. Best practice testing workflow
# Step 1: Syntax check
ansible-playbook deploy.yml --syntax-check

# Step 2: Dry run
ansible-playbook deploy.yml --check --diff

# Step 3: Test on single host
ansible-playbook deploy.yml --limit staging-web1

# Step 4: Test on staging
ansible-playbook -i inventory/staging deploy.yml

# Step 5: Production deployment
ansible-playbook -i inventory/production deploy.yml
```

### Complete Example

```yaml
# deploy-application.yml
---
- name: Deploy web application
  hosts: webservers
  become: yes
  vars_files:
    - vars/common.yml
    - "vars/{{ environment }}.yml"
  
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
      tags: always
  
  roles:
    - role: common
      tags: common
    - role: nginx
      tags: nginx
    - role: application
      tags: app
  
  tasks:
    - name: Verify application is responding
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      retries: 5
      delay: 10
      tags: verify
  
  post_tasks:
    - name: Send deployment notification
      debug:
        msg: "Application {{ app_name }} v{{ app_version }} deployed successfully"
      tags: always

# Usage:
# ansible-playbook deploy-application.yml -i inventory/production -e "environment=production"
```

## Kết Luận

Ansible là một công cụ mạnh mẽ và linh hoạt cho việc tự động hóa IT infrastructure. Với kiến thức cơ bản về:

- **Kiến trúc và thành phần**: Control node, inventory, playbooks, modules
- **Cài đặt và cấu hình**: Multiple installation methods, ansible.cfg
- **Inventory management**: Static và dynamic inventories
- **Ad-hoc commands**: Quick operations trên remote hosts
- **Modules**: Powerful building blocks cho automation
- **Variables**: Flexible configuration management
- **Conditionals và Loops**: Control flow trong playbooks
- **Best Practices**: Security, testing, version control

Bạn đã có nền tảng vững chắc để bắt đầu tự động hóa infrastructure của mình với Ansible. Tiếp theo, bạn nên tìm hiểu về Playbooks, Roles và Collections để xây dựng các automation workflows phức tạp hơn.

## Tài Liệu Tham Khảo

- [Ansible Official Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Ansible GitHub Repository](https://github.com/ansible/ansible)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
