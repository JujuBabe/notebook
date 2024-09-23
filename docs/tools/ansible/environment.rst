环境搭建
=========
本示例将按惯例通过docker来搭建联系环境

服务器准备
---------
Dockerfile 文件::

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

镜像操作::

    # 构建
    docker build -t ansible_vm:v1 -f Dockerfile .
    # 准备ansible机器
    docker run -d -e TZ=Asia/Shanghai -v $PWD:/deploy --name ansible_vm ansible_vm:v1
    # 准备5台服务器
    for i in `seq 1 5`;do docker run -d -e TZ=Asia/Shanghai --name ansible_vm_$i ansible_vm:v1;done
    # 释放容器
    for i in `seq 1 5`;do docker stop ansible_vm_$i && docker rm ansible_vm_$i;done

准备托管机器配置文件::

    # 获取所有测试节点的IP
    for i in `seq 1 5`;do docker inspect ansible_vm_$i -f {{.NetworkSettings.Networks.bridge.IPAddress}};done > ansible_vm_ips
    # 第一行添加分组名 docker(linux)
    sed -i '1 i[docker]' ansible_vm_ips
    # 第一行添加分组名 docker(mac os)
    sed -i "" '1i\'$'\n''[docker]'$'\n' ansible_vm_ips
    # 按官方习惯重命名
    mv ansible_vm_ips inventory.cfg

安装 ansible
````````````

::

    # 进入 ansible 服务器
    docker exec -it ansible_vm sh
    # 安装
    apt update
    apt install software-properties-common -y
    apt-add-repository -y -u ppa:ansible/ansible
    apt install ansible -y
    ansible --version


建立 ansible 服务器到托管服务器的免密访问
`````````````````````````````````````
::

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


ping 测试
````````````
::

    # 进入 ansible 服务器
    docker exec -it ansible_vm sh
    # 进入配置文件目录
    cd /deploy
    # ping
    ansible docker -i ./inventory.cfg -m ping

Done!