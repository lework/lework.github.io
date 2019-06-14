---
layout: post
title: "Ansible Tower系列 二（安装 Tower）"
date: "2016-11-19 21:26:34"
categories: Ansible
excerpt: "文档：http://docs.ansible.com/ansible-tower/ 安装前检查 python版本为2.6 保持网络畅通 内存预留..."
auth: lework
---
* content
{:toc}

文档：http://docs.ansible.com/ansible-tower/

## 安装前检查
---

1. python版本为2.6
2. 保持网络畅通
3. 内存预留充足
4. 安装用户为root

## 软件下载
---

下载地址：http://releases.ansible.com/ansible-tower/setup/
含有包文件的版本：http://releases.ansible.com/ansible-tower/setup-bundle/

```
wget http://releases.ansible.com/ansible-tower/setup-bundle/ansible-tower-setup-bundle-latest.el6.tar.gz

tar zxf ansible-tower-setup-bundle-latest.el6.tar.gz
cd ansible-tower-setup-bundle-3.0.3-1.el6/
```
## 部署
---

设置主机信息
```
sed -i "s#password=''#password='admin'#g" inventory
sed -i "s#host=''#host='127.0.0.1'#g" inventory 
sed -i "s#port=''#port='5432'#g" inventory 
```
修改yum源
```
sed -i 's#dl.fedoraproject.org/pub#mirrors.ustc.edu.cn#g' roles/packages_el/defaults/main.yml
sed -i 's/#baseurl=/baseurl=/g' roles/packages_el/files/epel-6.repo
sed -i 's/mirrorlist=/#mirrorlist=/g' roles/packages_el/files/epel-6.repo
sed -i 's#download.fedoraproject.org/pub#mirrors.ustc.edu.cn#g' roles/packages_el/files/epel-6.repo

yum -y install centos-release-scl-rh centos-release-scl
sed -i 's#mirror.centos.org#centos.ustc.edu.cn#g' /etc/yum.repos.d/CentOS-SCLo-scl.repo
sed -i 's#mirror.centos.org#centos.ustc.edu.cn#g' /etc/yum.repos.d/CentOS-SCLo-scl-rh.repo
yum -y install supervisor
```
安装ansible
```
./setup.sh
```

在
`TASK [awx_install : Migrate the Tower database schema (may take awhile when upgrading).] ***`
这一步会出现错误，提示信息是数据库连接不上

启动postgresql

```
service postgresql-9.4 initdb
service postgresql-9.4 start
```

创建用户
```
su - postgres
psql
CREATE ROLE awx CREATEDB PASSWORD 'admin' LOGIN; 
\q

sed -i 's#peer#md5#g' /var/lib/pgsql/9.4/data/pg_hba.conf
sed -i 's#ident#md5#g' /var/lib/pgsql/9.4/data/pg_hba.conf
service postgresql-9.4 restart
```

测试awx用户连接，输入密码连接，并创建数据库
```
psql -U awx -d postgres -h 127.0.0.1
create database awx;
\q
```

再次` ./setup.sh `进行安装tower


## Web配置
---

**访问web页面**

`https://192.168.77.128/#/`

![Paste_Image.png](/assets/images/Ansible/3629406-d51b13d66f7ebbd5.png)

用户名/密码为admin admin

**导入license**
没有的话，点击REQUEST LICENSE，去官方申请免费试用。

![Paste_Image.png](/assets/images/Ansible/3629406-62c3606cdccb5870.png)

提交license后，就进入了DASHBOARD页面啦



![Paste_Image.png](/assets/images/Ansible/3629406-030c231fe3cb20a6.png)

## Tower无限hosts的License修改
---

> 仅供实验测试使用，切勿挪作他用。

下载反编译工具： http://sourceforge.net/projects/easypythondecompiler/

反编译task_engine.pyc文件
```
find / -name task_engine.pyc
/var/lib/awx/venv/tower/lib/python2.7/site-packages/awx/main/task_engine.pyc
```

![Paste_Image.png](/assets/images/Ansible/3629406-1d9703441ade7942.png)


反编译后的文件为task_engine.pyc_dis，文件重命名为task_engine.py

**修改内容**

`89`行和`186`行代码
available_instances = int(self.attributes['instance_count']) 为
`available_instances = 10000`

![Paste_Image.png](/assets/images/Ansible/3629406-db0139247b895202.png)

![Paste_Image.png](/assets/images/Ansible/3629406-e1db4c32f95f4dd3.png)

`247`行代码，把相应的功能由`False`改为`True`

![Paste_Image.png](/assets/images/Ansible/3629406-cc80ff232ddcbb66.png)

删除task_engine.pyc task_engine.pyo ，将修改后的task_engine.py文件上传到tower上，重启tower服务
```
rm -f  /var/lib/awx/venv/tower/lib/python2.7/site-packages/awx/main/task_engine.py*
cp task_engine.py /var/lib/awx/venv/tower/lib/python2.7/site-packages/awx/main/
ansible-tower-service restart
```

查看license信息

![Paste_Image.png](/assets/images/Ansible/3629406-0484440406179c24.png)

## 安装时遇到的错误
---
```
fatal: [localhost]: FAILED! => {"changed": false, "failed": true, "msg": "This machine does not have sufficient RAM to run Ansible Tower."}
```
解决：机器内存不足，增加内存
```
fatal: [localhost]: FAILED! => {"changed": false, "failed": true, "msg": "no service or tool found for: supervisord"}
```
解决：yum -y install supervisor
```
出现Is the server running on host \"localhost\" (127.0.0.1) and accepting\n\tTCP/IP connections on port 5432?
```
是postgresql服务没启动
```
service postgresql-9.4 initdb
service postgresql-9.4 start

# 创建用户
su - postgres
psql
CREATE ROLE awx CREATEDB PASSWORD 'admin' LOGIN; 
\q
exit

sed -i 's#peer#md5#g' /var/lib/pgsql/9.4/data/pg_hba.conf
sed -i 's#ident#md5#g' /var/lib/pgsql/9.4/data/pg_hba.conf
service postgresql-9.4 restart

# 测试awx用户连接，输入密码连接，并创建数据库
psql -U awx -d postgres -h 127.0.0.1
create database awx;
\q
```
---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
