# Ansible

文档：
- [Ansible 中文权威指南](https://ansible-tran.readthedocs.io/en/latest/index.html)
- [Ansible「2.9」 中文官方文档](https://cn-ansibledoc.readthedocs.io/zh-cn/latest/)
- [YAML 语法](https://ansible-tran.readthedocs.io/en/latest/docs/YAMLSyntax.html)
- [整套 playbook 示例](https://github.com/ansible/ansible-examples)
## 环境搭建
本示例将按惯例通过docker来搭建联系环境
### 服务器准备
Dockerfile
```shell
FROM ubuntu:18.04
RUN sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y openssh-server && \
    mkdir /var/run/sshd && \
    echo 'root:123456' |chpasswd && \
    sed -ri 's/session required pam_loginuid.so/#session required pam_loginuid.so/g' /etc/pam.d/sshd && \
    sed -ri 's/^#?PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config && \
    mkdir /root/.ssh && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```
镜像操作
```shell
# 构建
docker build -t ansible_vm:v1 -f Dockerfile .
# 准备ansible机器
docker run -d -e TZ=Asia/Shanghai -v $PWD:/deploy --name ansible_vm ansible_vm:v1
# 准备5台服务器
for i in `seq 1 5`;do docker run -d -e TZ=Asia/Shanghai --name ansible_vm_$i ansible_vm:v1;done
# 释放容器
for i in `seq 1 5`;do docker stop ansible_vm_$i && docker rm ansible_vm_$i;done
```
### 准备托管机器配置文件
```shell
# 获取所有测试节点的IP
for i in `seq 1 5`;do docker inspect ansible_vm_$i -f {{.NetworkSettings.Networks.bridge.IPAddress}};done > ansible_vm_ips
# 第一行添加分组名 docker(linux)
sed -i '1 i[docker]' ansible_vm_ips
# 第一行添加分组名 docker(mac os)
sed -i "" '1i\'$'\n''[docker]'$'\n' ansible_vm_ips
# 按官方习惯重命名
mv ansible_vm_ips inventory.cfg
```
### 安装 ansible
```shell
# 进入 ansible 服务器
docker exec -it ansible_vm sh
# 安装
apt update
apt install software-properties-common -y
apt-add-repository -y -u ppa:ansible/ansible
apt install ansible -y
ansible --version
```

### 建立 ansible 服务器到托管服务器的免密访问
```shell
# 进入 ansible 服务器
docker exec -it ansible_vm sh
# 生成密钥
ssh-keygen -t rsa
# 分发公钥到托管服务器（托管服务器密码 123456）
ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.3
ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.4
ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.5
ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.6
ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.7
```

### ping 测试
```shell
# 进入 ansible 服务器
docker exec -it ansible_vm sh
# 进入配置文件目录
cd /deploy
# ping
ansible docker -i ./inventory.cfg -m ping
```
Done!

---

## 最佳实践
TODO...


---

## 理论
### Inventory 文件
Ansible 可同时操作属于一个组的多台主机,组和主机之间的关系通过 inventory 文件配置. 默认的文件路径为 /etc/ansible/hosts

多个 Inventory 文件需要通过 -i 参数来指定
```shell
ansible -i multi.host -u ubuntu group1 -m ping
# -i Inventory 配置文件
# -u 指定用户
# group1 Inventory 配置文件中的分组
# -m 指定模块
```

#### Inventory 参数说明
```text
ansible_ssh_host
      将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置.

ansible_ssh_port
      ssh端口号.如果不是默认的端口号,通过此变量设置.

ansible_ssh_user
      默认的 ssh 用户名

ansible_ssh_pass
      ssh 密码(这种方式并不安全,我们强烈建议使用 --ask-pass 或 SSH 密钥)

ansible_sudo_pass
      sudo 密码(这种方式并不安全,我们强烈建议使用 --ask-sudo-pass)

ansible_sudo_exe (new in version 1.8)
      sudo 命令路径(适用于1.8及以上版本)

ansible_connection
      与主机的连接类型.比如:local, ssh 或者 paramiko. Ansible 1.2 以前默认使用 paramiko.1.2 以后默认使用 'smart','smart' 方式会根据是否支持 ControlPersist, 来判断'ssh' 方式是否可行.

ansible_ssh_private_key_file
      ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理的情况.

ansible_shell_type
      目标系统的shell类型.默认情况下,命令的执行使用 'sh' 语法,可设置为 'csh' 或 'fish'.

ansible_python_interpreter
      目标主机的 python 路径.适用于的情况: 系统中有多个 Python, 或者命令路径不是"/usr/bin/python",比如  \*BSD, 或者 /usr/bin/python
      不是 2.X 版本的 Python.我们不使用 "/usr/bin/env" 机制,因为这要求远程用户的路径设置正确,且要求 "python" 可执行程序名不可为 python以外的名字(实际有可能名为python26).

      与 ansible_python_interpreter 的工作方式相同,可设定如 ruby 或 perl 的路径....
```
示例：
```shell
192.168.178.138 ansible_user=root ansible_ssh_pass=111111
```

### Playbook 
Playbook 格式是 Yaml 。

由一个或多个 ‘plays’ 组成，它的内容是一个以 ‘plays’ 为元素的列表。

在 play 之中,一组机器被映射为定义好的角色.在 ansible 中,play 的内容,被称为 tasks,即任务。

#### yaml 格式要点
1. 以 --- 为第一行
2. 列表
```yaml
---
- Apple
- Orange
- Strawberry
- Mango
```
3. 字典
```yaml
---
name: Example Developer
job: Developer
skill: Elite
```
或者
```yaml
---
{name: Example Developer, job: Developer, skill: Elite}
```

#### playbook 基础示例：
```yaml
---
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    yum: pkg=httpd state=latest
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
    notify:
    - restart apache
  - name: ensure apache is running
    service: name=httpd state=started
  handlers:
    - name: restart apache
      service: name=httpd state=restarted

# hosts 一个或多个组,多个的格式为 ['webservers1', 'webservers2']
```
#### Task / Handler Include Files And Encouraging Reuse
通过 include 实现 task / handler 的重用

在 Inventory 文件中通过 include 引入一个 task / handler include file
```yaml
# 省略其他
tasks:
  - include: tasks/foo.yml
  - include: handlers/handlers.yml
```
一个 task/handler include file 由一个普通的 task/handler 列表所组成，像这样:
```yaml
---
# possibly saved as tasks/foo.yml

- name: placeholder foo
  command: /bin/foo

- name: placeholder bar
  command: /bin/bar
```
```yaml
---
# 重启 apache 的 handler
# this might be in a file like handlers/handlers.yml
- name: restart apache
  service: name=apache state=restarted
```
传递变量：
```yaml
# 方式一：
tasks:
  - include: wordpress.yml wp_user=timmy
  - include: wordpress.yml wp_user=alice
```
```yaml
# 方式二：
tasks:
  - { include: wordpress.yml, wp_user: timmy, ssh_keys: [ 'keys/one.txt', 'keys/two.txt' ] }
```
```yaml
# 方式三：
tasks:

  - include: wordpress.yml
    vars:
      wp_user: timmy
      some_list_variable:
        - alpha
        - beta
        - gamma
```
变量的引用：
```yaml
{{ wp_user }}
```

#### task 的组织方式 -> roles
一个项目的结构如下:
```text
site.yml
webservers.yml
fooservers.yml
roles/
    common/
        files/
        templates/
        tasks/
        handlers/
        vars/
        defaults/
        meta/
    webservers/
        files/
        templates/
        tasks/
        handlers/
        vars/
        defaults/
        meta/
```
一个 playbook 如下:
```yaml
---
- hosts: webservers
  roles:
    - common
    - webservers
```
这个 playbook 为一个角色 ‘x’ 指定了如下的行为：

- 如果 roles/x/tasks/main.yml 存在, 其中列出的 tasks 将被添加到 play 中
- 如果 roles/x/handlers/main.yml 存在, 其中列出的 handlers 将被添加到 play 中
- 如果 roles/x/vars/main.yml 存在, 其中列出的 variables 将被添加到 play 中
- 如果 roles/x/meta/main.yml 存在, 其中列出的 “角色依赖” 将被添加到 roles 列表中 (1.3 and later)
- 所有 copy tasks 可以引用 roles/x/files/ 中的文件，不需要指明文件的路径。
- 所有 script tasks 可以引用 roles/x/files/ 中的脚本，不需要指明文件的路径。
- 所有 template tasks 可以引用 roles/x/templates/ 中的文件，不需要指明文件的路径。
- 所有 include tasks 可以引用 roles/x/tasks/ 中的文件，不需要指明文件的路径。

#### roles 触发条件
```yaml
---

- hosts: webservers
  roles:
    - { role: some_role, when: "ansible_os_family == 'RedHat'" }
```
#### 给 roles 分配指定的 tags
```yaml
---

- hosts: webservers
  roles:
    - { role: foo, tags: ["bar", "baz"] }
```
#### 执行顺序控制
```yaml
---

- hosts: webservers

  pre_tasks:
    - shell: echo 'hello'

  roles:
    - { role: some_role }

  tasks:
    - shell: echo 'still busy'

  post_tasks:
    - shell: echo 'goodbye'
```
#### 角色依赖
New in version 1.3.

“角色依赖” 使你可以自动地将其他 roles 拉取到现在使用的 role 中。”角色依赖” 保存在 roles 目录下的 meta/main.yml 文件中。这个文件应包含一列 roles 和 为之指定的参数，下面是在 roles/myapp/meta/main.yml 文件中的示例:
```yaml
---
dependencies:
  # 相对路径
  - { role: common, some_parameter: 3 }
  - { role: apache, port: 80 }
  # 绝对路径
  - { role: /path/to/common/roles/postgres, dbname: blarg, other_parameter: 12 }
```
执行顺序：dependencies 优先于 roles

重复执行：dependencies 默认只执行一次，多次执行需添加 allow_duplicates: yes

重复执行示例：
```yaml
---
dependencies:
- { role: wheel, n: 1 }
- { role: wheel, n: 2 }
- { role: wheel, n: 3 }
- { role: wheel, n: 4 }
```
wheel 角色的 meta/main.yml 文件包含如下内容:
```yaml
---
allow_duplicates: yes
dependencies:
- { role: tire }
- { role: brake }
```
最终执行顺序：
```text
tire(n=1)
brake(n=1)
wheel(n=1)
tire(n=2)
brake(n=2)
wheel(n=2)
...
car
```


## ansible 管理方式
1. 命令行中指定 ssh 密码问询的方式

2. 在 /etc/ansible/hosts 中配置密码
缺点：会暴露密钥

3. ssh 免密
需提前共享公钥

```shell
# 备份配置文件
cp /etc/ansible/hosts{,.ori}
# 添加远程主机 /etc/ansible/hosts
[dev]
192.168.178.138
192.168.178.139


# 管理方式1：命令行中指定 ssh 密码问询
# 注意：首次访问会有 fingerprint 问题，需要手动 ssh root@192.168.178.139 一次
ansible dev -m command -a 'hostname' -k -u root
# -m 指定功能模块，默认 command
# -a 指定具体命令
# -k ask pass 询问密码验证
# -u 指定运行用户


# 管理方式2：/etc/ansible/hosts 中指定远程服务器密码
# 修改 /etc/ansible/hosts
[dev]
192.168.178.138 ansible_user=root ansible_ssh_pass=111111
192.168.178.139 ansible_user=root ansible_ssh_pass=111111
# 远程访问
ansible dev -m command -a "hostname"

# 管理方式3：ssh 密钥方式批量管理主机
# ansible 服务器上创建ssh密钥对
ssh-keygen -f ~/.ssh/id_rsa -P "" > /dev/null 2>&1
# 分发公钥到远程服务器
SSH_PASS=11111 # 远程服务器密码
SSH_PATH=~/.ssh/id_rsa.pub
sshpass -p$SSH_PASS ssh-copy-id -i $KEY_PATH "-o StrictHostKeyChecking=no" 192.168.178.138
# 如果 ssh 有指定密码，可通过ssh-add缓存密码
ssh-agent bash
ssh-add $SSH_PATH
```
---
## ansible 管理模式
- ad-hoc 纯命令行
- playbook

### ad-hoc 模式
适用于可直接使用ansible命令行来处理的一些临时、简单的任务。如查看机器内存、负载、网络情况，分发配置文件等。
```shell
ansible dev -m command -a 'hostname'
```

### playbook 模式
适用于具体且较大的任务。如一键初始化服务器等

#### command 模块
作用：在远程节点上执行一个命令
```shell
# 查看该模块支持的参数
ansible-doc -s command
# chdir 在执行命令前，先通过cd进入该参数指定的目录
# creates 在创建一个文件之前，判断文件是否存在，如果存在则跳过前面的东西，如果不存在则执行前面的动作
# free_from 该参数可输入任何系统命令，实现远程执行和管理
# removes 定义一个文件是否存在，如果存在了则执行前面的动作，如果不存在则跳过动作
```
command 模块是默认模块，可省略不写，但是要注意如下的坑：
- 使用command模块，不得出现shell变量`$name`，也不得出现特殊符号`> < | ; &`，如需使用，请使用shell模块

```shell
ansible dev -m command -a 'uptime'

ansible dev -m command -a 'pwd chdir=/tmp/'
# 存在则跳过，不存在则执行
ansible dev -m command -a 'pwd creates=/opt/'
# 存在则执行，不存在则跳过
ansible dev -m command -a 'ls /ppt removes=/deploy'
# 忽略告警信息
ansible dev -m command -a 'chmod 000 /etc/hosts warn=False'
```

#### shell 模块
```shell
# 查看该模块支持的参数
ansible-doc -s shell
# chdir 在执行命令前，先通过cd进入该参数指定的目录
# creates 在创建一个文件之前，判断文件是否存在，如果存在则跳过前面的东西，如果不存在则执行前面的动作
# free_from 该参数可输入任何系统命令，实现远程执行和管理
# removes 定义一个文件是否存在，如果存在了则执行前面的动作，如果不存在则跳过动作
```

使用示例：
1. 批量查询进程信息(grep -v grep 排除掉 grep 进程本身)
```shell
ansible dev -m shell -a "ps -ef | grep vim | grep -v grep"
```
2. 批量写入文件信息
```shell
ansible dev -m shell -a "echo hello! > /tmp/hw.txt"
```
3. 批量远程执行脚本 (以下示例实际使用建议使用script模块)
```shell
# 注意：脚本需要在远程机器上存在
# 1.创建文件夹
# 2.创建脚本文件，写入脚本内容
# 3.赋予脚本可执行权限
# 4.执行脚本
# 5.忽略warning信息

ansible dev -m shell -a "mkdir -p /server/myscripts/;echo 'hostname' > /server/myscripts/hostname.sh;chmod +x /server/myscripts/hostname.sh; bash /server/myscripts/hostname.sh warn=False"
```

#### script 模块
```shell
# 查看该模块支持的参数
ansible-doc -s script
# chdir 在执行命令前，先通过cd进入该参数指定的目录
# creates 在创建一个文件之前，判断文件是否存在，如果存在则跳过前面的东西，如果不存在则执行前面的动作
# free_from 该参数可输入任何系统命令，实现远程执行和管理
# removes 定义一个文件是否存在，如果存在了则执行前面的动作，如果不存在则跳过动作
```
远程执行脚本，且在远程客户端不需要存在该脚本
```shell
ansible dev -m script -a "/myscripts/local_hostname.sh"
# /myscripts/local_hostname.sh 为管理服务器上的脚本路径
```