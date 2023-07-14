建议采用容器化部署，本教程作了一定简化，使其更简单，需要了解更多，查看[链接](https://www.freebuf.com/articles/wireless/268397.html)

（1）要求：稳定访问github

（2）本视频分为两个：将要介绍的是Open5GS、Free5GC基于Docker容器部署，具体细节不在视频中录制，最后将展示如何抓取核心网数据包。以下安装步骤经过实际验证

（3）问题：ubuntu20.04，在UE通过RAN接入核心网，无法访问baidu.com，建议使用Ubuntu18.06

现在开始的是Open5GS安装，关于UERANSIM的配置与Free5GC类似

### 一、准备工作

1. 更新配置

   `sudo apt update`

2. 安装一些工具

   `sudo apt-get install vim openssh-server -y`

   `Vim`是从`vi`发展出来的一个文本编辑器。

   `openssh-server`安装ssh，用于远程连接。

3. 更换内核

   桌面版更换内核，安装synaptic，安装命令`sudo apt-get install synaptic`，之后`sudo synaptic`运行软件，从中查找linux-image、linux-headers（注意两个的版本一直，为generic），选中安装，之后按如下更改配置文件。

   在`/boot/grub/grub.cfg`查看内核配置。查看上一级menuentry和安装的内核的位置

   在`/etc/default/grub`配置与以下类似的形式

   ```normal
   GRUB_DEFAULT="gnulinux-advanced-2c6a2b50-95b6-4dbf-be3e-4171d06e7f46>gnulinux-5.4.0-125-generic-advanced-2c6a2b50-95b6-4dbf-be3e-4171d06e7f46"
   ```

   保存，执行`sudo update-grub`更新配置，重启

   ubuntu18.04可以执行

   ```
   sudo apt install 'linux-image-5.0.0-23-generic'
   sudo apt install 'linux-headers-5.0.0-23-generic'
   ```

   安装后同上操作更换

   ```
   gnulinux-advanced-05e7e2cd-f462-4169-9eb7-ca46461edeb4>gnulinux-5.0.0-23-generic-advanced-05e7e2cd-f462-4169-9eb7-ca46461edeb4
   ```

   

4. 安装其他主见

   ```
   sudo apt install git-all curl make gcc g++ autoconf libtool pkg-config libmnl-dev libyaml-dev -y 
   ```

5. 安装docker

   ```
   curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
   ```

6. 安装docker-compose

   ```sh
   # 可以更换版本，到里面的地址去看看https://github.com/docker/compose/releases
   sudo curl -L "https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   
   sudo chmod +x /usr/local/bin/docker-compose
   # 下面命令可能不用执行
   sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
   ```

   

7. 配置docker加速镜像，`sudo vim /etc/docker/daemon.json`

   ```json
   {
   	"registry-mirrors": ["https://hub-mirror.c.163.com"]
   }
   ```

   重启docker，`sudo systemctl restart docker`

8. 安装cmake

   `sudo snap install cmake --classic`

9. 安装mongodb，并启动（非容器部署有用，容器化部署不执行）

   `sudo apt -y install mongodb wget git`

   `sudo systemctl start mongodb`

10. 安装yarn（非容器部署执行）

    ```sh
    curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
    sudo echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
    
    sudo apt update && sudo apt install yarn
    ```

11. 安装go语言环境

    这里选择go1.13安装，https://golang.google.cn/dl/

    ```sh
    sudo wget https://golang.google.cn/dl/go1.13.9.linux-amd64.tar.gz
    
    sudo tar -C /usr/local -zxvf go1.13.9.linux-amd64.tar.gz
    
    ```

    配置环境变量，`/etc/profile`最后添加

    ```
    export PATH=$PATH:/usr/local/go/bin
    ```

    执行生效命令，`source /etc/profile`

### 二、Free5GC安装

free5gc安装教程如下，不在详述

> free5gc docker环境部署配置下载地址：https://github.com/free5gc/free5gc-compose

1. 构建GTP5G模块

   ```sh
   cd /opt/module
   sudo git clone https://github.com/PrinzOwO/gtp5g
   cd gtp5g
   
   # 编译代码
   sudo make
   sudo make install
   ```

   注：gtp5g模块是free5gc模拟核心网的内核模块，无论何种部署方案，都==必须安装==

2. 容器化部署Free5GC

   ```sh
   cd /opt/module
   sudo git clone https://github.com/free5gc/free5gc-compose.git
   cd free5gc-compose
   
   sudo make base
   docker-compose -f docker-compose-build.yaml build
   ```

3. 修改docker-compose，为了以后在也不需要在nr-gnb执行更改amf 的ip，修改未知ports，修改位置`- "38412:38412/sctp"`

   ```
     free5gc-amf:
       container_name: amf
       image: free5gc/amf:v3.2.1
       command: ./amf -c ./config/amfcfg.yaml
       expose:
         - "8000"
       ports:
         - "38412:38412/sctp"
       volumes:
         - ./config/amfcfg.yaml:/free5gc/config/amfcfg.yaml
       environment:
         GIN_MODE: release
       networks:
         privnet:
           aliases:
             - amf.free5gc.org
       depends_on:
         - free5gc-nrf
   ```

   

4. UERANSIM模拟设备安装

   ```sh
   cd /opt/module
   git clone https://github.com/aligungr/UERANSIM
   
   # 依赖安装，包括sctp协议依赖等
   sudo apt update
   sudo apt install libsctp-dev lksctp-tools iproute2 -y
   
   # 编译代码
   cd /opt/module/UERANSIM
   sudo make
   ```

5. 更改gnb配置文件

   ngapIp、gtpIp、amfconfig一项下的address为amf的ip 都修改为本机IP

   ```yaml
   
   mcc: '208'          # Mobile Country Code value
   mnc: '93'           # Mobile Network Code value (2 or 3 digits)
   
   nci: '0x000000010'  # NR Cell Identity (36-bit)
   idLength: 32        # NR gNB ID length in bits [22...32]
   tac: 1              # Tracking Area Code
   
   linkIp: 192.168.59.129   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
   ngapIp: 192.168.59.129   # gNB's local IP address for N2 Interface (Usually same with local IP)
   gtpIp: 192.168.59.129    # gNB's local IP address for N3 Interface (Usually same with local IP)
   
   # List of AMF address information
   amfConfigs:
     - address: 192.168.59.129
       port: 38412
   
   # List of supported S-NSSAIs by this gNB
   slices:
     - sst: 0x1
       sd: 0x010203
   
   # Indicates whether or not SCTP stream number errors should be ignored.
   ignoreStreamIds: true
   ```

6. 修改nr-ue配置文件

   ```yaml
   gnbSearchList:
     - 192.168.59.129
   ```

   

### 三、全套环境运行

1. 启动Free5GC核心网

   ```sh
   cd /opt/module/free5gc-compose
   sudo docker-compose -f docker-compose-build.yaml up -d
   
   ```

2. 模拟基站启动

   ```
   cd /opt/module/UERANSIM/
   sudo ./build/nr-gnb -c ./config/free5gc-gnb.yaml
   ```

3. 模拟UE启动

   ```
   cd /opt/module/UERANSIM/
   sudo ./build/nr-ue -c ./config/free5gc-ue.yaml
   ```



接下来看演示访问

### 四、Open5GS安装

https://github.com/herlesupreeth/docker_open5gs

```sh
git clone https://github.com/herlesupreeth/docker_open5gs
cd docker_open5gs/base
docker build --no-cache --force-rm -t docker_open5gs .
cd ../ims_base
docker build --no-cache --force-rm -t docker_kamailio .
cd ..
source .env
docker-compose -f sa-deploy.yaml build
docker-compose -f sa-deploy.yaml up -d
```

```
Username : admin
Password : 1423
```

