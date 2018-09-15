# Thomas
## Environment Requirements
**OS**：[Ubuntu 16.04 Server](https://www.ubuntu.com.cn/download/server)

**Software**：[Docker 17.03.0-ce](https://docs.docker.com/release-notes/docker-ce/) or higher

**Hardware**：NVIDIA GPU, such as TESLA P4

**Docker Image**：docker.io/superapricot/thomas:20180913081313

## Deploy
### 1. install gpu driver
##### download gpu driver to machine
```
sudo wget https://cn.download.nvidia.com/tesla/390.12/nvidia-diag-driver-local-repo-ubuntu1604-390.12_1.0-1_amd64.deb
```
##### install with follow command
```
sudo dpkg -i nvidia-diag-driver-local-repo-ubuntu1604-390.12_1.0-1_amd64.deb
```
##### add apt public key
```
sudo apt-key add /var/nvidia-diag-driver-local-repo-390.12/7fa2af80.pub
```
##### update apt source and install cuda software
```
sudo apt-get update && apt-get -y install cuda-drivers
```
##### reboot machine after above install finished
```
sudo reboot
```
##### log on machine after reboot. verify the gpu/cuda driver is ok by follow commandline <code>nvidia-smi</code>，expected outputs is as follow
```
Wed Sep 12 13:34:46 2018
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.30                 Driver Version: 390.30                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla P4            On   | 00000000:00:07.0 Off |                    0 |
| N/A   28C    P8     7W /  75W |     10MiB /  7611MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+

```

### 2. install cuda toolkit
##### download cuda toolkit to machine
```
sudo wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_9.1.85-1_amd64.deb
```
##### install with follow command
```
sudo dpkg -i cuda-repo-ubuntu1604-9-1-local_9.1.85-1_amd64.deb
```
##### update apt source and install cuda toolkit
```
sudo apt-get update && apt-get -y install cuda
```
##### reboot machine after above install finished
```
sudo reboot
```

### 3. install [docker-ce](https://docs.docker.com/release-notes/docker-ce/)
##### if machine exist docker.io, uninstall it first
```
sudo apt-get -y remove \
  docker-engine \
  docker.io
```
##### install [docker-ce](https://docs.docker.com/release-notes/docker-ce/)dependence
```
sudo apt-get -y install \
  aufs-tools \
  subversion \
  libcurl4-nss-dev \
  cgroup-lite
```
##### update apt source
```
sudo apt-get update
```
##### allow apt use https
```
sudo apt-get -y install \
  apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common
```
##### add docker officer GPG key
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
##### add docker officer stable source
```
add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
```
##### update apt source again and install [docker-ce](https://docs.docker.com/release-notes/docker-ce/)
```
sudo apt-get update && apt-get -y install docker-ce
```

### 4. configure gpu image run environment
 **Notice**：If you have modified <code>/etc/docker/daemon.json</code> before, follow process will overwrite your modify.
 
##### add nvidia-docker source
```
sudo curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
  sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
sudo curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```
##### update apt source and install nvidia-docker2
```
sudo apt-get update && apt-get install -y nvidia-docker2
```
##### reload the docker daemon configuration
```
sudo pkill -SIGHUP dockerd
```

### 5. download the thomas docker image and run

##### download the thomas image to your machine
```
docker pull superapricot/thomas:20180913081313
```
##### create db by follow command 
```
sudo apt-get install mysql-server
sudo apt install mysql-client
sudo apt install libmysqlclient-dev
```
#### set db remote access private
```
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
add # before bind-address = 127.0.0.1（comment it）
```
#### grant db right, asumme your db password is root123, you can change to your own.(and replace root123 to your own the all the follow step)
```
mysql -uroot -proot123
grant all on *.* to root@'%' identified by 'root123' with grant option;
flush privileges;
service mysql restart
```
#### create database and table
```
DROP DATABASE IF EXISTS d_thomas;
CREATE DATABASE d_thomas;

DROP TABLE IF EXISTS `d_thomas`.`job_info`;
CREATE TABLE `d_thomas`.`job_info` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) COLLATE utf8mb4_bin NOT NULL,
  `input_url` varchar(2048) COLLATE utf8mb4_bin NOT NULL,
  `input_url_backup` varchar(2048) COLLATE utf8mb4_bin NOT NULL,
  `output_url` varchar(2048) COLLATE utf8mb4_bin NOT NULL,
  `scale`  varchar(256) COLLATE utf8mb4_bin NOT NULL,
  `vcodec_type` int(11) NOT NULL COMMENT '0 h264; 1 h265',
  `vcodec_bitrate` int(11) NOT NULL,
  `watermark_text` varchar(255) COLLATE utf8mb4_bin NOT NULL DEFAULT '',
  `watermark_param` varchar(255) COLLATE utf8mb4_bin NOT NULL DEFAULT '' COMMENT 'fontsize:width:height',
  `audio_param`  varchar(255) COLLATE utf8mb4_bin NOT NULL DEFAULT '',
  `gpu_number` int(11) NOT NULL,
  `gpu_deint` int(11) NOT NULL,
  `worker` varchar(256) COLLATE utf8mb4_bin NOT NULL,
  `op` int(11) NOT NULL COMMENT '0 none; 1 start; 2 restart; 3 stop',
  `state` int(11) NOT NULL COMMENT '0 none; 1 todo; 2 doing; 3 done',
  `score` int(11) NOT NULL,
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `modify_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  INDEX `state` (`state`) USING BTREE,
  INDEX `op` (`op`) USING BTREE,
  INDEX `worker` (`worker`) USING BTREE,
  INDEX `update_time` (`update_time`) USING BTREE,
  INDEX `create_time` (`create_time`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
```
##### create log&config path by follow command 
```
mkdir -p /data/conf/thomas
cd /data/conf/thomas/
vim app.conf, set app.conf as follow
#app.conf begin---------------------------------------------------------------------------
appname = Thomas
httpaddr = 172.16.236.141
httpport = 8080
runmode =dev
autorender = false
recoverpanic = false
viewspath = html

mysql_string = "root:root123@tcp(YOUR_IP:3306)/d_thomas?charset=utf8&loc=Local"
#app.conf end---------------------------------------------------------------------------

mkdir -p /data/conf/scheduler
cd /data/conf/scheduler
vim scheduler.conf, set scheduler.conf as follow
#scheduler.conf begin---------------------------------------------------------------------------
max_update_interval_s = 30
mysql_string = "root:root123@tcp(YOUR_IP:3306)/d_thomas?charset=utf8&loc=Local"
#scheduler.conf end---------------------------------------------------------------------------

AccessKey=$AccessKey
```
##### run thomas docker image, to start transcoder server
```
docker run -d --privileged=true --network=host --runtime=nvidia -v /data/logs/:/data/logs/:rw -v /data/conf/thomas:/app/thomas/conf:ro -v /data/conf/scheduler:/app/scheduler/conf:ro -v /usr/lib/nvidia-390:/usr/lib/nvidia-390:ro superapricot/thomas:20180915162848 /init
```

## 3. Try & Test

##### access the system by follow url 
```
http://YOUR_IP:YOUR_PORT
```
