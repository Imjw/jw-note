

[TOC]

------



# 环境准备

## 编译环境

```shell
yum install git gcc gcc-c++ make automake autoconf libtool pcre pcre-devel zlib zlib-devel openssl-devel -y
```
## 目录

|说明|位置|
|---|---|
|所有安装包|/usr/local/src|
|tracker跟踪服务器数据|/fastdfs/tracker|
|storage存储服务器数据|/fastdfs/storage|
```shell
mkdir -p /fastdfs/tracker  #创建跟踪服务器数据目录
mkdir -p /fastdfs/storage  #创建存储服务器数据目录
 #切换到安装目录准备下载安装包
cd /usr/local/src 
```
## 安装libfatscommon

```shell
git clone https://github.com/happyfish100/libfastcommon.git --depth 1
cd libfastcommon/
./make.sh && ./make.sh install
```
## 安装FastDFS

```shell
git clone https://github.com/happyfish100/fastdfs.git --depth 1
cd fastdfs/
./make.sh && ./make.sh install
#配置文件准备
cp /etc/fdfs/tracker.conf.sample /etc/fdfs/tracker.conf
cp /etc/fdfs/storage.conf.sample /etc/fdfs/storage.conf
cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf #客户端文件，测试用
cp /usr/local/src/fastdfs/conf/http.conf /etc/fdfs/ #供nginx访问使用
cp /usr/local/src/fastdfs/conf/mime.types /etc/fdfs/ #供nginx访问使用
```
## 安装fastdfs-nginx-module

```shell
git clone https://github.com/happyfish100/fastdfs-nginx-module.git --depth 1
cp /usr/local/src/fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs
```
## 安装nginx

```shell
wget http://nginx.org/download/nginx-1.12.2.tar.gz
tar -zxvf nginx-1.12.2.tar.gz
cd nginx-1.12.2/
#添加fastdfs-nginx-module模块
./configure --add-module=/usr/local/src/fastdfs-nginx-module/src/ --prefix=/usr/local/nginx/
make && make install
```
> --prefix=/xx/xx  该选项在一台服务器上装有多个nginx的时候需要指定不同的目录，同时nginx.conf配置文件中需要设置不同的监听端口

# 单机部署

## tracker配置

```shell
vim /etc/fdfs/tracker.conf
#需要修改的内容如下
port=22122  # tracker服务器端口（默认22122,一般不修改）
base_path=/fastdfs/tracker  # 存储日志和数据的根目录
#保存后启动
/etc/init.d/fdfs_trackerd start #启动tracker服务
chkconfig fdfs_trackerd on #自启动tracker服务
```
## storage配置

```shell
vim /etc/fdfs/storage.conf
#需要修改的内容如下
port=23000  # storage服务端口（默认23000,一般不修改）
base_path=/fastdfs/storage  # 数据和日志文件存储根目录
store_path0=/fastdfs/storage  # 第一个存储目录
tracker_server=10.3.22.43:22122  # tracker服务器IP和端口
http.server_port=8097  # http访问文件的端口(默认8888,看情况修改,和nginx中保持一致)
#保存后启动
/etc/init.d/fdfs_storaged start #启动storage服务
chkconfig fdfs_storaged on #自启动storage服务
```
## client测试

```shell
vim /etc/fdfs/client.conf
#需要修改的内容如下
base_path=/fastdfs/tracker
tracker_server=10.3.22.43:22122    #tracker IP地址
#保存后测试,返回ID表示成功 eg:group1/M00/00/00/wKgAQ1pysxmAaqhAAA76tz-dVgg.tar.gz
fdfs_upload_file /etc/fdfs/client.conf /usr/local/src/nginx-1.12.2.tar.gz
```
## 配置nginx访问

```shell
vim /etc/fdfs/mod_fastdfs.conf
#需要修改的内容如下
tracker_server=10.3.22.43:22122
url_have_group_name=true
store_path0=/fastdfs/storage
#配置nginx.config
vi /usr/local/nginx/conf/nginx.conf
#添加如下配置
server {
    listen       8097;    ## 该端口为storage.conf中的http.server_port相同
    server_name  localhost;
    location ~/group[0-9]/ {
        ngx_fastdfs_module;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
    root   html;
    }
}
#启动nginx
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
#测试下载，用外部浏览器访问刚才已传过的nginx安装包,引用返回的ID
http://10.3.22.43:8097/group1/M00/00/00/wKgAQ1pysxmAaqhAAA76tz-dVgg.tar.gz
#弹出下载单机部署全部跑通，否则首先检查防火墙，再检查其他配置。
service iptables status
#若不通，可查看iptables端口是否开启，可以关闭防火墙：
#永久性生效：
#开启：
chkconfig iptables on
#关闭：
chkconfig iptables off
#即时生效，重启后失效
#开启：
service iptables start
#关闭：
service iptables stop

#测试远程服务器端口
telnet 10.3.22.43 22122
```
# 集群部署

> 以下配置为双机，双tracker（未配nginx负载均衡），双storage（配nginx），单group配置，storage间会进行同步

## tracker配置

```shell
vim /etc/fdfs/tracker.conf
#需要修改的内容如下
port=22122  # tracker服务器端口（默认22122,一般不修改）
base_path=/fastdfs/tracker  # 存储日志和数据的根目录
#保存后启动
/etc/init.d/fdfs_trackerd start #启动tracker服务
chkconfig fdfs_trackerd on #自启动tracker服务
```

> 与单节点的配置相同

## storage配置

```shell
vim /etc/fdfs/storage.conf
#需要修改的内容如下
port=23000  # storage服务端口（默认23000,一般不修改）
base_path=/fastdfs/storage  # 数据和日志文件存储根目录
store_path0=/fastdfs/storage  # 第一个存储目录
tracker_server=10.3.22.43:22122  # tracker1服务器IP和端口
tracker_server=10.3.22.72:22122  # tracker2服务器IP和端口
http.server_port=8097  # http访问文件的端口(默认8888,看情况修改,和nginx中保持一致)
#保存后启动
/etc/init.d/fdfs_storaged start #启动storage服务
chkconfig fdfs_storaged on #自启动storage服务
```

## 配置nginx访问

```shell
vim /etc/fdfs/mod_fastdfs.conf
#需要修改的内容如下
tracker_server=10.3.22.43:22122
tracker_server=10.3.22.72:22122
url_have_group_name=true
store_path0=/fastdfs/storage
#配置nginx.config
vi /usr/local/nginx/conf/nginx.conf
#添加如下配置
server {
    listen       8097;    ## 该端口为storage.conf中的http.server_port相同
    server_name  localhost;
    location ~/group[0-9]/ {
        ngx_fastdfs_module;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
    root   html;
    }
}
upstream storage_server_group1{
	server 10.3.22.43:8097 weight=10;
	server 10.3.22.72:8888 weight=10;
}
#启动nginx
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
#弹出下载单机部署全部跑通，否则首先检查防火墙，再检查其他配置。
service iptables status
#若不通，可查看iptables端口是否开启，可以关闭防火墙：
#永久性生效：
#开启：
chkconfig iptables on
#关闭：
chkconfig iptables off
#即时生效，重启后失效
#开启：
service iptables start
#关闭：
service iptables stop

#测试远程服务器端口
telnet 10.3.22.43 22122
```

> 另一台机子配置与上面一致。***注：第二台启动的storage必须从空目录启动，其data目录中不能有数据，否则会出现storage数据不同步现象***

## client测试

```shell
vim /etc/fdfs/client.conf
#需要修改的内容如下
base_path=/fastdfs/tracker
tracker_server=10.3.22.43:22122    #tracker1 IP地址
tracker_server=10.3.22.72:22122    #tracker2 IP地址
#保存后测试,返回ID表示成功 eg:group1/M00/00/00/wKgAQ1pysxmAaqhAAA76tz-dVgg.tar.gz
fdfs_upload_file /etc/fdfs/client.conf /usr/local/src/nginx-1.12.2.tar.gz
#可以通过两天服务器任一地址访问都可获取文件
#http://10.3.22.43:8097/group1/M00/00/00/wKgAQ1pysxmAaqhAAA76tz-dVgg.tar.gz
#或者http://10.3.22.72:8097/group1/M00/00/00/wKgAQ1pysxmAaqhAAA76tz-dVgg.tar.gz
```

## 启动后查看状态

```shell
#查询storage状态
/usr/bin/fdfs_monitor /etc/fdfs/storage.conf
#返回内容
[2018-06-28 00:26:20] DEBUG - base_path=/fastdfs/storage, connect_timeout=30, network_timeout=60, tracker_server_count=2, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=0, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0

server_count=2, server_index=1

tracker server is 10.3.22.72:22122

group count: 1

Group 1:
group name = group1
disk total space = 51175 MB
disk free space = 34641 MB
trunk free space = 0 MB
storage server count = 2
active server count = 2
storage server port = 23000
storage HTTP port = 8888
store path count = 1
subdir count per path = 256
current write server index = 0
current trunk file id = 0

	Storage 1:
		id = 10.3.22.43
		ip_addr = 10.3.22.43  ACTIVE
		http domain =
		version = 5.12
		join time = 2018-06-25 23:20:57
		up time = 2018-06-26 22:58:30
		total storage = 51175 MB
		free storage = 34641 MB
		upload priority = 10
		store_path_count = 1
		subdir_count_per_path = 256
		storage_port = 23000
		storage_http_port = 8098
		current_write_path = 0
		source storage id =
		if_trunk_server = 0
		connection.alloc_count = 256
		connection.current_count = 1
		connection.max_count = 2
		total_upload_count = 8
		success_upload_count = 8
		total_append_count = 0
		success_append_count = 0
		total_modify_count = 0
		success_modify_count = 0
		total_truncate_count = 0
		success_truncate_count = 0
		total_set_meta_count = 0
		success_set_meta_count = 0
		total_delete_count = 3
		success_delete_count = 3
		total_download_count = 0
		success_download_count = 0
		total_get_meta_count = 0
		success_get_meta_count = 0
		total_create_link_count = 0
		success_create_link_count = 0
		total_delete_link_count = 0
		success_delete_link_count = 0
		total_upload_bytes = 1973718
		success_upload_bytes = 1973718
		total_append_bytes = 0
		success_append_bytes = 0
		total_modify_bytes = 0
		success_modify_bytes = 0
		stotal_download_bytes = 0
		success_download_bytes = 0
		total_sync_in_bytes = 9
		success_sync_in_bytes = 9
		total_sync_out_bytes = 0
		success_sync_out_bytes = 0
		total_file_open_count = 9
		success_file_open_count = 9
		total_file_read_count = 0
		success_file_read_count = 0
		total_file_write_count = 15
		success_file_write_count = 15
		last_heart_beat_time = 2018-06-28 00:26:04
		last_source_update = 2018-06-27 09:38:43
		last_sync_update = 2018-06-27 09:32:08
		last_synced_timestamp = 2018-06-27 09:32:06 (0s delay)
	Storage 2:
		id = 10.3.22.72
		ip_addr = 10.3.22.72  ACTIVE
		http domain =
		version = 5.11
		join time = 2018-06-27 09:17:10
		up time = 2018-06-27 09:17:10
		total storage = 37861 MB
		free storage = 36314 MB
		upload priority = 10
		store_path_count = 1
		subdir_count_per_path = 256
		storage_port = 23000
		storage_http_port = 8888
		current_write_path = 0
		source storage id = 10.3.22.43
		if_trunk_server = 0
		connection.alloc_count = 256
		connection.current_count = 1
		connection.max_count = 2
		total_upload_count = 1
		success_upload_count = 1
		total_append_count = 0
		success_append_count = 0
		total_modify_count = 0
		success_modify_count = 0
		total_truncate_count = 0
		success_truncate_count = 0
		total_set_meta_count = 0
		success_set_meta_count = 0
		total_delete_count = 1
		success_delete_count = 1
		total_download_count = 0
		success_download_count = 0
		total_get_meta_count = 0
		success_get_meta_count = 0
		total_create_link_count = 0
		success_create_link_count = 0
		total_delete_link_count = 0
		success_delete_link_count = 0
		total_upload_bytes = 9
		success_upload_bytes = 9
		total_append_bytes = 0
		success_append_bytes = 0
		total_modify_bytes = 0
		success_modify_bytes = 0
		stotal_download_bytes = 0
		success_download_bytes = 0
		total_sync_in_bytes = 1973700
		success_sync_in_bytes = 1973700
		total_sync_out_bytes = 0
		success_sync_out_bytes = 0
		total_file_open_count = 7
		success_file_open_count = 7
		total_file_read_count = 0
		success_file_read_count = 0
		total_file_write_count = 13
		success_file_write_count = 13
		last_heart_beat_time = 2018-06-28 00:25:54
		last_source_update = 2018-06-27 09:32:06
		last_sync_update = 2018-06-27 09:38:52
		last_synced_timestamp = 2018-06-27 09:38:43 (0s delay)
```

