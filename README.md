# s3fs部署指南

## 概述

s3fs使用fuse挂载s3 bucket到Linux或Mac系统,并且支持在Docker容器内部以非特权用户挂载s3 bucket。

## 安装s3fs


在容器内使用s3fs,需要在Docker镜像中安装s3fs,以下是主流发行版安装方法:


* Debian 9 and Ubuntu 16.04 or newer:

  ```
  sudo apt install s3fs
  ```

* RHEL and CentOS 7 or newer through via EPEL:

  ```
  sudo yum install epel-release
  sudo yum install s3fs-fuse
  ```


## 参数说明

1. /etc/fuse.conf文件中配置说明

   | Parameter                         | Description                                      | Default                         |
   | --------------------------------- | ------------------------------------------------ | ------------------------------- |
   | user_allow_other                  | 允许非root用户使用allow_other 挂载选项             |no value                         |
  

   <hr>

2. 挂载选项说明

   | Parameter                      | Description           | Default                                 |
   | ----------------------------- | --------------------- | --------------------------------------- |
   | use_path_request_style    | 非AWS实现的S3服务,设置此参数,配合url使用   | no value                                  |
   | url | S3服务的URL       | http://obs.cn-north-4.myhuaweicloud.com                                 |
   | allow_other   | 允许非root用户挂载     |  no vaule |


<hr>

## 部署服务

- 通过fstab挂载

  ```shell
  echo ACCESS_KEY_ID:SECRET_ACCESS_KEY > ~/.passwd-s3fs
  sed -i 's/\#user_allow_other/user_allow_other/g' /etc/fuse.conf
  mkdir /mnt/s3
  rke-test /mnt/s3 fuse.s3fs _netdev,allow_other,use_path_request_style,url=http://obs.cn-north-4.myhuaweicloud.com / 0 0
  ```

  

- 通过命令挂载

  ```shell
  echo ACCESS_KEY_ID:SECRET_ACCESS_KEY > ~/.passwd-s3fs
  sed -i 's/\#user_allow_other/user_allow_other/g' /etc/fuse.conf
  mkdir /mnt/s3
  s3fs rke-test /mnt/s3 -o _netdev -o allow_other -o use_path_request_style -o url=http://obs.cn-north-4.myhuaweicloud.com
  ```

- 容器内命令挂载

  ```shell
  docker run -d --name ranger  --cap-add mknod --cap-add sys_admin --security-opt apparmor:unconfined --device=/dev/fuse reg.chebai.org/paas/ranger:latest
  docker exec -it -u 1000 ranger bash
  echo ACCESS_KEY_ID:SECRET_ACCESS_KEY > ~/.passwd-s3fs
  sed -i 's/\#user_allow_other/user_allow_other/g' /etc/fuse.conf
  mkdir /mnt/s3
  s3fs rke-test /mnt/s3 -o _netdev -o allow_other -o use_path_request_style -o url=http://obs.cn-north-4.myhuaweicloud.com
  ```


## 测试

1. 确定已成功挂载
```bash
mount | grep s3
```

2. 复制文件到挂载点

```shell
cp file /mnt/s3
```

2. 在华为云控制台检查bucket中是否存在该文件

## 常见问题

1. fuse: device not found, try 'modprobe fuse' first
[k8s中，容器需要设置特权模式。否则会引起此报错](https://github.com/s3fs-fuse/s3fs-fuse/issues/1314#issuecomment-647482118)

2. 启用此功能需要节点安装fuse组件么
不需要，只需要容器安装`s3fs`，调用容器内的fuse组件。fuse是user-space的组件，只需要容器内安装。之后需要特权模式sys-admin。则能正常使用。
[用户空间文件系统FUSE工作原理](https://zhuanlan.zhihu.com/p/106719192?utm_source=wechat_session)

3. s3fs: credentials file /root/.passwd-s3fs should not have others permissions.
文件权限问题，为是s3认证信息保密，将权限由644改为600。`chmod 600 ～/.passwd-s3fs`。

4. 文件只读性能还可以，但一点有写操作，性能很差。
a服务修改了文件，会重新上传文件至s3，其他节点则是从s3同步下修改的文件。

## reference

[s3fs-fuse github](https://github.com/s3fs-fuse)
[fuse 概念扫盲](https://zhuanlan.zhihu.com/p/106719192?utm_source=wechat_session)