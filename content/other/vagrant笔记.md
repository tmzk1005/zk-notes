---
title: "vagrant笔记"
date: 2021-01-30T11:27:39+08:00
draft: false
tags: ["vagrant"]
toc: true
---

# 1. 新建默认Vagrantfile

在一个空的目录里执行`vagrant init`,将会在这个目录产生一个`Vagrantfile`文件，其内容如下(去掉了注释)：

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "base"
end
```

Vagrant使用ruby语言开发，配置文件Vagrantfile也是使用ruby语言配置，上面的`config`就是一个ruby对象，可以通过设置此对象的属性来对虚拟机进行配置，而`"2"`是config对象的版本，目前的版本就是“2”。

上面配置了虚拟机box名为“base”，其他的配置都是默认的。我们想自定义虚拟机的操作系统，可以自定义box，比如设置为ubuntu-21.04:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
end
```

可以手动修改Vagrantfile文件得到，也可以在执行`vagrant init`是加参数：`vagrant init "ubunut/focal64"`

## 1.1 配置虚拟机名字

```ruby
config.vm.define "ubuntu-1"
```

这个名字不是主机名，而是给虚拟机本身取的名字，会显示在执行`vagrant global-status`时显示的name列，不配置的话，默认是“default”

## 1.2 配置是否自动更新box

```ruby
# 不要自动更新
config.vm.box_check_update = false
```

# 2. 虚拟机常用配置

这里只列出部分常用的配置，完全详细的参见[官方文档](https://www.vagrantup.com/docs/vagrantfile)

## 2.1 配置主机名

配置项是`config.vm.hostname`

eg: 配置主机名为“myubuntu”

```ruby
config.vm.hostname = "myubuntu"
```

## 2.2 配置虚拟硬件参数

配置项是`config.vm.provider`

比如配置底层的visualbox虚拟机的参数：

```ruby
config.vm.provider "virtualbox" do |v|
# 不要gui
v.gui = false
# 配置虚拟机名字
v.name = "ubuntu"
# 配置cpu核心数
v.cpus = 2
# 配置内存,单位MB
v.memory = 2048
# 可以直接调用shell命令配置vm，命令以及参数依此放入数组，这需要知道底层的虚拟机提供了什么命令
v.customize ["modifyvm", :id, "--cpuexecutioncap", "50"]
end
```

上面没有配置vagrant虚拟机的磁盘大小，vagrant的设计初衷是为了方便复制共享，因此box小一点更好，如果要大点的磁盘可以自己制作box,也可以用`v.customize`调用底层provider的命令新建磁盘安装到虚拟机。

## 2.3 配置虚拟机网络

配置项是`config.vm.network`

### 2.3.1 ip配置

eg1: dhcp网络，`vboxnet1`是host-only模式的虚拟网卡名
```ruby
config.vm.network "private_network", type: "dhcp", name: "vboxnet1"
```

eg2: 静态ip
```ruby
config.vm.network "private_network", ip: "192.168.50.4", name: "vboxnet1", hostname: true
```

### 2.3.2 配置宿主机端口转发

```ruby
config.vm.network "forwarded_port", guest: 80, host: 8080
```

### 2.3.3 配置public_network

vagrant默认的网络是“private_network", 宿主机只能用“vagrant ssh”连接到虚拟机内，不能用ssh命令连接，这样不能开多个终端窗口，宿主机所在的局域网的其他机器也不能联通虚拟机。要实现联通的话，得配置“public_network"

dhcp模式默认路由：

```bash
config.vm.network "public_network" use_dhcp_assigned_default_route: true
```

静态ip模式：
```bash
config.vm.network "public_network", ip: "192.168.0.17"
```

> 注意ip按实际网络情况填写，和宿主机ip在一个网段

静态ip模式下，启动时会交互式询问选择的网卡，可以自动设置以免去询问：

```bash
config.vm.network "public_network", ip: "192.168.0.17", bridge: "en0: Wi-Fi (AirPort)"
```

> bridge后面的网卡名，按实际的交互选择项的值填写

# 3. vagrant命令

执行`vagrant -h`可以查看命令帮助：

```txt
Usage: vagrant [options] <command> [<args>]

    -h, --help                       Print this help.

Common commands:
     box             manages boxes: installation, removal, etc.
     cloud           manages everything related to Vagrant Cloud
     destroy         stops and deletes all traces of the vagrant machine
     global-status   outputs status Vagrant environments for this user
     halt            stops the vagrant machine
     help            shows the help for a subcommand
     init            initializes a new Vagrant environment by creating a Vagrantfile
     login
     package         packages a running vagrant environment into a box
     plugin          manages plugins: install, uninstall, update, etc.
     port            displays information about guest port mappings
     powershell      connects to machine via powershell remoting
     provision       provisions the vagrant machine
     push            deploys code in this environment to a configured destination
     rdp             connects to machine via RDP
     reload          restarts vagrant machine, loads new Vagrantfile configuration
     resume          resume a suspended vagrant machine
     snapshot        manages snapshots: saving, restoring, etc.
     ssh             connects to machine via SSH
     ssh-config      outputs OpenSSH valid configuration to connect to the machine
     status          outputs status of the vagrant machine
     suspend         suspends the machine
     up              starts and provisions the vagrant environment
     upload          upload to machine via communicator
     validate        validates the Vagrantfile
     version         prints current and latest Vagrant version
     winrm           executes commands on a machine via WinRM
     winrm-config    outputs WinRM configuration to connect to the machine

For help on any individual command run `vagrant COMMAND -h`

Additional subcommands are available, but are either more advanced
or not commonly used. To see all subcommands, run the command
`vagrant list-commands`.
        --[no-]color                 Enable or disable color output
        --machine-readable           Enable machine readable output
    -v, --version                    Display Vagrant version
        --debug                      Enable debug output
        --timestamp                  Enable timestamps on log output
        --debug-timestamp            Enable debug output with timestamps
        --no-tty                     Enable non-interactive output
```

查看具体子命令的的帮助文档：`vagrant help [sub_command]`，例如执行`vagrant help box`输出：

```txt
Usage: vagrant box <subcommand> [<args>]

Available subcommands:
     add
     list
     outdated
     prune
     remove
     repackage
     update

For help on any individual subcommand run `vagrant box <subcommand> -h`
        --[no-]color                 Enable or disable color output
        --machine-readable           Enable machine readable output
    -v, --version                    Display Vagrant version
        --debug                      Enable debug output
        --timestamp                  Enable timestamps on log output
        --debug-timestamp            Enable debug output with timestamps
        --no-tty                     Enable non-interactive output
```

## 3.1 启动当前环境所有虚拟机

```bash
vagrant up
```

## 3.2 启动当前环境指定名字的虚拟机

```bash
vagrant up [name]
```

## 3.3 查看当前环境所有虚拟机的状态

```bash
vagrant status
```

使用此命令也可以知道当前环境到底有几台虚拟机，以及他们的名字和状态

## 3.4 停止指定名字的虚拟机

```bash
vagrant halt [name|id]
```

可以用name和id指定虚拟机，不加name和id参数则停止当前环境所有虚拟机

怎么知道虚拟机的id？，执行`vagrant global-status`,可以看到大致如下的输出：

```txt
id       name    provider   state    directory
-------------------------------------------------------------------------
02f05df  default virtualbox poweroff /Users/zk/vagrant-wsp

The above shows information about all known Vagrant environments
on this machine. This data is cached and may not be completely
up-to-date (use "vagrant global-status --prune" to prune invalid
entries). To interact with any of the machines, you can go to that
directory and run Vagrant, or you can use the ID directly with
Vagrant commands from any directory. For example:
"vagrant destroy 1a2b3c4d"
```

可以得知虚拟机的id，name，provider等属性，同时会有文档指导如何删除销毁虚拟机

## 3.5 停止并删除虚拟机

```bash
vagrant destroy [name|id]
```

可以用name和id指定虚拟机，不加name和id参数则停止并删除当前环境所有虚拟机。

> 删除后要更新`vagrantfile`，不然下次再执行`vagrant up`会重新新建虚拟机。


# 4. ubuntu虚拟机配置

## 4.1 设置用户密码

官方的ubuntu box安装并启动后默认的用户是“vagrant”，该用户已经被加入“sudoers”，并且执行“sudo”不需要输入密码。root用户的密码和vagrant的用户的密码都不知道，但是可以用“sudo passwd”重置：

```bash
sudo passwd root
sudo password vagrant
```

## 4.2 设置runlevel
另外，官方ubuntu box开机后运行在“level 5”，不需要gui的话可以设置开机运行在“level 3”：

```bash
sudo systemctl set-default multi-user.target
```

使用`sudo reboot`重启后，执行`runlevel`查看当前运行级别。

## 4.3 设置apt源

设置清华大学开源镜像站为apt源，参照[https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)设置

查看自己ubuntu的版本：

`cat /etc/issue`

或者

`sudo lsb_release -a`

对于“20.04”版本，备份“/etc/apt/sources.list”,写入如下内容：

```bash
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
```

然后执行`sudo apt update`


## 5 初始化脚本

像前面设置apt源这样的任务，可能每个新建的虚拟就都需要执行，如果每个都手动执行一次的话效率太低。vagrant提供了`Provisioning`机制来让虚拟机第一次初始化的时候
可以做一些动作，比如设置设置apt源可以用脚本自动完成。

```ruby
config.vm.provision "shell", path: "vagrant/myshell.sh", :args => [arg1, arg2, arg3]
```

其中，类型为"shell"时，path可以使用宿主机的绝对路径指定要指定的脚本的路径，也可以是用相对Vagrantfile的相对路径，args是传递给脚本的参数。注意脚本会被复制到
虚拟机中取执行，里面的逻辑当然要根据虚拟机内的环境来。

因为脚本不一定是幂等的，重复执行可能会有问题，所以provision配置的shell脚本默认是只在虚拟机第一次启动的时候执行，一般后面就不会在执行了，确实需要再次执行可以
加参数告诉vagrant，参见官方文档。


附修改apt源的脚本：

```bash
#!/bin/bash

cd /etc/apt

slfile=sources.list

if [ -f sources.list ]; then
    mv ${slfile} ${slfile}.backup.$(date +%s)
fi

cat <<EOF > ${slfile}
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse
EOF

apt update
```