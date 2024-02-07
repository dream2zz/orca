虚拟机群
===

## 环境准备

```
wget http://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-1907_01.VirtualBox.box
mv CentOS-7-x86_64-Vagrant-1907_01.VirtualBox.box /root/boxes/CentOS-7.box
```

## 新建虚拟机

```
mkdir /root/cad2.0 -p && cd /root/cad2.0
wget http://172.22.3.3/CloudNativeTechs/orca/raw/master/Multiple-Machines/Vagrantfile # 项目需要具备一个 Vagrant Project，通过克隆来更新这些资产
vagrant up
```

## 清除虚拟机

如果启动失败，则需要检查是否已经装配了虚拟机，并清除掉它们：
```
vagrant global-status
vagrant destroy -f server # 或ID
```


