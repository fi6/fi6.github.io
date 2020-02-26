# Initialize a New VPS
When you are a root user:


```bash
useradd fi
passwd fi
echo -e "\nfi ALL=(ALL) NOPASSWD: ALL\n" >> /etc/sudoers
```

Or if you just wan to use password for more safety:
```bash
echo -e "\nfi ALL=(ALL) ALL\n" >> /etc/sudoers
```

```bash
tail -3 /etc/sudoers
sudo su -l fi
mkdir .ssh && chmod 700 .ssh && cd .ssh
nano authorized_keys
chmod 600 authorized_keys
```



## Disable password log in and root user log in

!!! danger Must Ensure!
    Make sure your are now able to log in with ssh key

```bash
sudo vi /etc/ssh/sshd_config
```

将 26 行左右的 #PermitRootLogin yes 改为 PermitRootLogin no ，
50 行左右的 #PasswordAuthentication yes 改为 PasswordAuthentication no。
修改端口, 将 #Port 22 改为 Port 端口号数字(比如： Port 7564)。

```bash
sudo service sshd restart
```

## Enable firewall

!!! danger Must Ensure!
    Make sure your ssh port will maintain accessible before starting firewall

```bash
sudo systemctl enable firewalld 
sudo systemctl start firewalld && firewall-cmd --zone=public --add-port=22/tcp --permanent && firewall-cmd --reload
```

## Install docker

Install docker, enable auto startup, add current user to docker group.

```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install docker-ce docker-ce-cli containerd.io

sudo systemctl start docker

sudo systemctl enable docker && sudo usermod -aG docker $USER

newgrp docker 

docker run hello-world
```

## Install docker-compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version
```

## SS-libev(if needed)

### docker only

```bash
docker pull shadowsocks/shadowsocks-libev
docker run -e PASSWORD=<password> -p<server-port>:8388 -p<server-port>:8388/udp -d shadowsocks/shadowsocks-libev
```

### using docker-compose

docker-compose.yml
```yaml
ss-libev:
  ports:
    - "8388:8388"
    - "8388:8388/udp"
  image: "shadowsocks/shadowsocks-libev"
  environment:
    - PASSWORD=mypassword
```
The content below comes from:\
https://www.hawu.me/operation/1481


> ## 使用iptables监控端口流量
> 
> 1. 在iptables的INPUT链插入两条规则：监控8379,8380端口的tcp输入流量
> 
>     注意 -I 与 -A 的区别，-I表示在规则链首部插入规则，-A表示在规则链尾部追加规则
>
>     由于这两条规则没有-j参数，即对匹配到的流量不执行任何目标动作（ACCEPT/DROP/REJECT），只做记录。所以不会影响原有规则
> 
>     ```bash
>     iptables -I INPUT -p tcp --dport 8379
>     iptables -I INPUT -p tcp --dport 8380
>     ```
> 
> 2. 在iptables的OUTPUT链插入两条规则：监控8379,8380端口的tcp输出流量
> 
>     ```bash    
>     iptables -I OUTPUT -p tcp --sport 8379
>     iptables -I OUTPUT -p tcp --sport 8380
>     ```
>
> 3. 检查新增的规则（-n参数表示以数值方式显示地址与端口, -v表示显示包括流量在内的详细信息）\
>     ```bash
>     iptables -n -v -L
>     ```
> 
> 4. 保存iptables，否则重启就没了\
>     ```bash
>     service iptables save
>     ```
> 
> 5. 统计端口流量
>     ```bash
>     iptables -nv -L INPUT | grep 8379
>     iptables -nv -L INPUT | grep 8380
>     iptables -nv -L OUTPUT | grep 8379
>     iptables -nv -L OUTPUT | grep 8380
>     ```
> 
> ### iptables删除规则
> 
> 1. 先列出规则号
>     ```bash
>     iptables -nvL INPUT --line-numbers
>     iptables -nvL OUTPUT --line-numbers
>     ```
> 2. 删除第n号规则
>     ```bash
>     iptables -D INPUT 1
>     iptables -D OUTPUT 3
>     ```
> ## 使用shell脚本输出iptables统计信息
> 
> 为了方便查看流量统计，写了一个shell脚本，并添加到crontab中每月自动运行
> 
> 
> ss_statistics.sh:
>   ```bash
> #!/bin/bash
> # 这是一个利用iptables进行端口流量统计的脚本
> # 首先请保证已经在iptables规则中加入了要监控的端口
> 
> # 需要统计的端口
> PORTS=(8379 8380)
> 
> # 打印分割线
> echo "**************************************************"
> 
> # 打印日期
> date
> 
> # 统计iptables的INPUT链
> echo "------------------------------"
> echo "INPUT chain:"
> for PORT in ${PORTS[@]};
> do
> iptables -nv -L INPUT | grep $PORT
> done
> 
> # 统计iptables的OUTPUT链
> echo "------------------------------"
> echo "OUTPUT chain:"
> for PORT in ${PORTS[@]};
> do
> iptables -nv -L OUTPUT | grep $PORT
> done
> 
> # 输出换行
> echo -e "\n"
> ```
> 
> ## crontab作业
> 
> crontab -e
> 
> ```bash
> SHELL=/bin/bash
> PATH=/sbin:/bin:/usr/sbin:/usr/bin
> MAILTO=root
> 
> # 每月1号02:00统计ss端口流量
> 0 2 1 * * /root/ss_statistics.sh >> /root/ss_traffic.txt
> ```