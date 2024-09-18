# Ansible

## 部署
### 服务器准备
```shell
master01    192.168.178.120
node01      192.168.178.138
node02      192.168.178.139
```

### master 服务器安装
```shell

# master 节点安装 ansible
yum install epel-release -y
yum install ansible libselinux-python -y

# 检查安装情况
rpm -ql ansible | grep -E '^/etc|^/usr/bin'

# 查看 ansible 版本
ansible --version
```

### node 服务器准备
```shell
yum install ansible libselinux-python -y
```

## ansible 管理方式
### 命令行中指定 ssh 密码问询的方式

### 在 /etc/ansible/hosts 中配置密码
缺点：会暴露密钥

### ssh 免密
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