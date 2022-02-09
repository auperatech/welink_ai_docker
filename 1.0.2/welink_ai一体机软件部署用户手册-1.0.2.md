[toc]
# 1 修订记录

| 时间       | 版本  | 描述                                        |
| ---------- | ----- | ------------------------------------------- |
| 2022/01/21 | 1.0.2 | 该文档对于的docker版本为: `welink_ai:1.0.2` |



# 2 版本信息

该版本不支持人脸识别




# 3 软件部署流程

## 3.1 基础环境搭建

- centos 安装命令如下
```
yum update;
yum install make automake gcc gcc-c++  kernel-devel
yum -y install nfs-utils rpcbind;
systemctl start rpcbind;
systemctl enable rpcbind;
systemctl start nfs;
systemctl enable nfs;
yum install docker
systemctl start docker
systemctl enable docker
yum install pciutils;
```
- ubuntu安装命令如下
```
sudo apt update;
sudo apt install make build-essential nfs-kernel-server     docker docker-containerd docker.io;
sudo service rpcbind restart
sudo service nfs-kernel-server restart
```

## 3.2 Docker容器安装步骤

- 再搭建好基础环境后，可以在此基础上安装该docker镜像，先创建一个临时的容器，用于拷贝一些配置文件模板，你在拷贝完这些数据后可以删除这个容器

  ```
  #load docker image
  sudo docker load-i welink.tar
  #cp message
  sudo docker cp <CONTAINER_NAME>:/root/driver <NFS_ABS_PATH>
  sudo docker cp <CONTAINER_NAME>:/root/firmware <NFS_ABS_PATH>
  sudo docker cp <CONTAINER_NAME>:/root/drm <NFS_ABS_PATH>
  sudo docker cp <CONTAINER_NAME>:/root/config <NFS_ABS_PATH>
  sudo docker container rm <CONTAINER_NAME>
  ```

  - 安装驱动 xdma
  ```
  cd {NFS_ABS_PATH} /driver
  ./install
  ```

- 人流统计配置文档说明
```
# this is configs template for welink ai application
# to use this configs, please copy this template to same folder and rename it to 'app.ini'
# then reboot device

# for app service
# max support value length is 200
# so we split app ticket to 3 segments, every segment max length 128
# do not disrupt the order of app ticket
[app]
appid =  
app_key = 
app_ticket_1 = 
app_ticket_2 = 
app_ticket_3 =
app_server = 

# message config
[message]
to_appid=
;modelId = ; defalut value is 200206

# video gateway
[video]
gateway_server = http://10.10.10.2:58888
;sharp_enable = 

[persistence]
enable = 1               ;task persistence. Default value is 1
recover_delay = 60       ;after the manager is started, it will be delayed for a period of time to read the persistent data and start the task. Deafult value is 30s

[api]
port =  44999              ; http server port. Default data is 44999
encrypt = 0                ; encrypt data 
;token =                   ; if encrypt =1 , please set token. encrypt http data by token. private method

[log]
enable = 1     ;log will be save to /tmp/welink_crowd_flow_*.log. Default value is 0

[task]
recover = 1                 ;if the task stops abnormally and has been running for more than 60s. if recover = 1, try to restart task
report_status = 0           ;if the task stops abnormally, send task status to welink server
disable_delete_check_id = 0 ;

[ai_param]
#max_cluster_size = 
```
- 配置`DRM`模式和序列号

  需要先从Aupera公司获取DRM序列号，获取到序列号文件`cred.json`文件后，把该文件拷贝到x86主机对应的目录

  DRM激活有两种模式，即`floating`和`nodelocked`模式

  ```
  # 拷贝到cred.json文件到此目录
  {NFS_ABS_PATH}/drm 
  
  # 把对应的配置文件拷贝到此目录
  cp ./floating/conf.json .
  cp ./nodelocked/conf.json .
  ```

- 算法配置

  | 类型     | 环境变量 | 环境变量描述 | LAN访问端口 | LAN访问端口描述 |
  | -------- | -------- | ---- | ---- | ---- |
  | 人脸识别 | ENABLE_AI_FR | yes/no | 44998 | 配置了proxy的话不需要映射此端口 |
  | 人流统计 | ENABLE_AI_CF | yes/no | 44999 | 配置了proxy的话不需要映射此端口 |
  | 视频网关 | ENABLE_VIDEO_GATEWAY | yes/no | NA | NA |


- 启动命令行示例

  - [x] 开启视频网关功能
  - [x] 开启人流统计功能
  - [ ] 开启人脸识别功能
  - [x] 开启人流统计信令服务器功能

- 启动docker前要部署网络
  ```
  docker network create -d bridge welink_ai-1.0.1
  docker network ls
  ```

  ```
  sudo docker run -dit --name welink_ai_1.0.1 --privileged=true -v /opt/aupera/welink/:/opt/aupera/welink/ -e NFS_ABS_PATH=/opt/aupera/welink/ -e ENABLE_VIDEO_GATEWAY=yes -e ENABLE_AI_PROXY=yes -e ENABLE_AI_CF=yes -e ENABLE_AI_FR=yes -p 44999:44999 -p 44998:44998 welink_ai:1.0.1 bash
  ```
  
- 启动容器后，开始执行软件的安装脚本，使能U30的软件

  ```
  sudo docker exet -it welink_ai_1.0.1 bash start.sh
  ```



# 4 软件维护

## 4.1 集群管理软件的维护

- `x86`设备支持安装多个`U30`卡，算法任务有`manager`和`worker`两个角色，`manager`运行在`x86`主机，负责接收外部的`http`请求，把对应的AI任务分配到`worker`节点，`worker`节点运行在`U30`卡上

运行 `ps -x`查看manager和worker是否启动

```
SCREEN -S aupm -dm mc_manager --manager-port 43777 --worker-port 43888 --engine welink_crowd_flow_manager
mc_manager --manager-port 43777 --worker-port 43888 --engine welink_crowd_flow_manager
```

注意要在docker里面运行`screen -r aupm`

- 目前该版本限制的AI任务路数为8路，根据测试，稳定运行的路数为4路，输入流为`1080P, 25fps`

  
### 4.1.1 `manager`的状态

**注意**：需要进入docker容器才能执行该指令，如果修改了配置文件，需要重新启动算法APP

查看

- 获取状态指令

  ````
  app status
  ````
  
- 重启指令

  ```
  app restart
  ```
  
- 启动指令
  
  ```
  app start
  ```
  
 - 停止指令

   ```
   app stop
   ```



### 4.1.2 `worker`的状态

**注意**：worker的状态需要进入`manager`的会话才能确认，主要是确认`worker`处于激活状态

登录板子运行`screen -r aupw`

```
54.31 - [INFO] Node0(0.0.0.0), state=ACTIVE, codecUsage=0, ipv4:10.10.10.2
54.31 - [INFO] state=ACTIVE, powermode=PERFORMANCE
54.31 - [INFO] ramdiskAvail=1906MB, storageAvail=172024MB, codecUsgae=0%
54.31 - [INFO] idle tasks=512, active tasks=0, idle workers=64, busy workers=0
Task Pool capacity 512 tasks, size 265.23KiB
idle:512/512 pending:0/0 assigning:0/0 assigned:0/0 accepted:0/0 working:0/0 finished:0/0 failed:0/0 
```



### 4.1.3 `proxy`的状态

**注意**：如果你为人流服务配置了代理服务器，则需要查看这个状态

- 获取状态指令

  ````
  proxy status
  ````
  
- 重启指令

  ```
  proxy restart
  ```
  
- 启动指令
  
  ```
  proxy start
  ```
  
 - 停止指令

   ```
   proxy stop
   ```



# 5 FAQ

## 5.1 算法任务异常停止

- 查看pipeline log，需要知道pipeline在哪个设备`worker`运行

  确认开启了pipeline的输出存储文件log,

  ```
  cat /D0/.welink_crowd_flow/app.ini
  [log]
  enable = 1     ;log will be save to /tmp/welink_crowd_flow_*.log. Default value is 0
  1为开启，0为不开启
  ```

- 登录设备查找对应的log文件

 ```
 ssh root@10.10.10.6 
 cat /tmp/welink_crowd_flow_53881.log
 ```



## 5.2 算法任务正常运行，即视客户端没有结果显示


- 查看是否有通知上传记录，进入容器查看log文件`/var/log/welink_crowd_flow/welink_upload.log`

  - 如果log显示告警发送失败，先检查log`/var/log/welink_crowd_flow/welink_login.log`查看AI网关是否登录成功
  - 如果log显示告警发送成功，则发送upload的log信息给welink开发人员，需要服务器配置指标


