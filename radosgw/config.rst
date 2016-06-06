=====================
配置 Ceph 对象网关
=====================

要配置一个 Ceph 对象网关需要一个运行着的 Ceph 存储集群，以及启用了 FastCGI 模块\
的 Apache web服务器。

Ceph 对象网关是 Ceph 存储集群的一个客户端，作为 Ceph 存储集群的客户端，它需要：

- 需要为网关实例配置一个名字，在本手册中我们使用 ``gateway`` .
- 存储集群的一个用户名，并且该用户在keyring中有合适的权限.
- 存储数据的资源池.
- 网关实例的一个数据目录.
- Ceph 配置文件中有一个实例配置入口.
- web 服务器有一个配置文件跟 FastCGI 交互.


新建用户和 Keyring
=========================

每一个实例必须有一个用户名和key来跟 Ceph 存储集群通信.在下面的步骤中,我们使用管理\
节点来新建keyring.然后,我们新建一个客户端用户名和key.然后我们将这个key添加到 Ceph \
存储集群中.最后,分发这个 key ring到包含这个网关实例的节点.


.. topic:: Monitor Key CAPS

   当你给 key 分配 CAPS 的时候,你必须提供读权限.然而,你也可以选择为 monitor 提供\
   写的权限. 这是一个重要的抉择.如果你给 key 分配了写权限, Ceph 对象网关将具备自动\
   新建资源池的能力; 此时,它将会根据默认的 PG 值(不是最合适的)或者你在 Ceph 配置文\
   件中指定的 PG 数. 如果你允许 Ceph 对象网关自动新建资源池，请确保你首先定义好了合\
   理的默认 PG 值. 查看 `资源池配置`_ 获取详细信息.


关于 Ceph 认证请参考\ `用户管理`_\ 。

#. 为每一个实例生成一个 Ceph 对象网关用户名和key. 举一个典型实例, 我们将使用 ``client.radosgw`` \
   后使用 ``gateway`` 作为用户名:: 

        sudo ceph auth get-or-create client.radosgw.gateway osd 'allow rwx' mon 'allow rwx' -o /etc/ceph/ceph.client.radosgw.keyring

#. 分发生成的keyring到网关实例所在节点. ::

	sudo scp /etc/ceph/ceph.client.radosgw.keyring  ceph@{hostname}:/home/ceph
	ssh {hostname}
	sudo mv ceph.client.radosgw.keyring /etc/ceph/ceph.client.radosgw.keyring

   .. note:: 如果 ``admin node`` 就是 ``gateway host`` ，那第 2 步就没必要。



创建存储池
==========

Ceph 对象网关需要 Ceph 存储集群资源池来存储特定的网关数据. 如果你新建的用户有权限, 网关\
将会自动新建所需资源池. 然而,你需要确保在你的 Ceph 配置文件中给资源池设置了合理的默认 PG 值.

.. note:: Ceph 对象网关有多个存储池，考虑到所有存储池会共用相同的 CRUSH 分级\
   结构，所以 PG 数不要设置得过大，否则会影响性能。

当使用默认的 region 和 zone 时,资源池的命名规则通常省略了 region 和 zone 的名字，但是\
你可以使用任何你想要的命名规则. 举例如下:


- ``.rgw``
- ``.rgw.root``
- ``.rgw.control``
- ``.rgw.gc``
- ``.rgw.buckets``
- ``.rgw.buckets.index``
- ``.rgw.buckets.extra``
- ``.log``
- ``.intent-log``
- ``.usage``
- ``.users``
- ``.users.email``
- ``.users.swift``
- ``.users.uid``


查看 `配置参考——存储池`_ 获取关于网关默认资源池的详细信息. 查看 `资源池`_ 获取新建资源池的详\
细信息. 正如前面所说，如果具备写权限,Ceph 对象网关将会自动新建资源池.
如果想手动新建资源池,执行下面的命令::

	ceph osd pool create {poolname} {pg-num} {pgp-num} {replicated | erasure} [{erasure-code-profile}]  {ruleset-name} {ruleset-number}

.. tip:: Ceph 允许多级 CRUSH 和多种 CRUSH 规则集，这样在配置你自己的网关时就\
   有很大的灵活性。像 ``rgw.buckets.index`` 这样的存储池就可以利用 SSD 来做存\
   储池以获取高性能；后端存储也可以从更经济的纠删编码的存储中获益，还可利用缓存层\
   提升性能。

完成上述步骤之后，执行下列的命令确认前述存储池都已经创建了： ::

	rados lspools


为 Ceph 新增网关配置
====================

添加 Ceph 对象网关配置到 ``admin node`` 节点的 Ceph 配置文件中. Ceph 对象网关配\
置需要你指定 Ceph 对象网关实例. 然后你必须指定你安装有 Ceph 对象网关守护进程的节点\
的主机名, 一个 keyring (为了使用 cephx), FastCGI 的 socket 路径和日志文件.


对于使用 Apache 2.2 和 Apache 2.4 早期版本 (RHEL 6, Ubuntu
12.04, 14.04 etc) 的发行版, 追加下面的配置到 ``admin node`` 的文件 ``/etc/ceph/ceph.conf`` 中::

	[client.radosgw.gateway]
	host = {hostname}
	keyring = /etc/ceph/ceph.client.radosgw.keyring
	rgw socket path = ""
	log file = /var/log/radosgw/client.radosgw.gateway.log
	rgw frontends = fastcgi socket_port=9000 socket_host=0.0.0.0
	rgw print continue = false


.. note:: Apache 2.2 和早期 Apache 2.4 版本不使用 Unix Domain
   Sockets 而是用 localhost TCP.

对于使用 Apache 2.4.9 或者更新版版本的发行版 (RHEL 7, CentOS 7 等), 追加下面的配置\
到 ``admin node`` 的文件 ``/etc/ceph/ceph.conf`` 中::

	[client.radosgw.gateway]
	host = {hostname}
	keyring = /etc/ceph/ceph.client.radosgw.keyring
	rgw socket path = /var/run/ceph/ceph.radosgw.gateway.fastcgi.sock
	log file = /var/log/radosgw/client.radosgw.gateway.log
	rgw print continue = false


.. note:: ``Apache 2.4.9`` 支持 Unix Domain Socket (UDS) ，但是 ``Ubuntu 14.04`` \
   附带的 ``Apache 2.4.7`` 不支持 UDS ,并且默认配置使用 localhost TCP. 在``Ubuntu 14.04`` \
   中的 ``Apache 2.4.7`` 已经发现一个 bug ，并且已经申请一个补丁以支持 UDS.
   查看: `Ubuntu Trusty 中支持 UDS 的补丁`_

这里, ``{hostname}`` 是即将提供网关服务的节点的主机名 ( ``hostname -s`` 命令的输出), \
比如 ``gateway host``. 网关实例的 ``[client.radosgw.gateway]`` 部分标识了 Ceph 配置\
文件的这个部分是用来配置 Ceph 存储集群的一个客户端的，这个客户端的类型是 Ceph 对象网关 \
(比如, ``radosgw``).


.. note:: 配置文件的最后一行 ``rgw print continue = false`` 是用来避免 ``PUT`` 操\
   作可能出现的问题.

一旦你完成了这些安装过程, 如果你所做的配置遇到了问题, 你可以在 Ceph 配置文件的 ``[global]`` \
部分添加调试选项, 然后重启你的网关来帮忙排错,找到可能的配置问题. 举例如下::

	[global]
	#append the following in the global section.
	debug ms = 1
	debug rgw = 20


分发更新后的 Ceph 配置文件
==========================================

更新后的 Ceph 配置文件需要从 ``admin node`` 分发到 Ceph 集群节点上.

它包含以下步骤:

#. 将更新后 ``ceph.conf`` 从 ``/etc/ceph/`` 目录拷贝到管理节点上新建集群时的根目录中 \
   (比如. ``my-cluster`` 目录). 因此 ``my-cluster`` 中``ceph.conf`` 将会被覆盖. \
   为此,执行下面的命令::

		ceph-deploy --overwrite-conf config pull {hostname}

   这里, ``{hostname}`` 是 Ceph 管理节点的精简主机名.

#. 将更新后 ``ceph.conf`` f文件从管理节点推送到集群中其他所有节点，包含 ``gateway host``::

		ceph-deploy --overwrite-conf config push [HOST] [HOST...]

   使用其他 Ceph 节点的主机名代替上面命令中的 ``[HOST] [HOST...]``.


从管理节点拷贝 ceph.client.admin.keyring 到网关主机
==============================================================

因为 ``gateway host`` 有可能是集群之外的其他节点,所以需要将 ``ceph.client.admin.keyring`` \
从 ``admin node`` 拷贝到 ``gateway host``. 为此,在 ``admin node`` 上执行下面的命令::

	sudo scp /etc/ceph/ceph.client.admin.keyring  ceph@{hostname}:/home/ceph
	ssh {hostname}
	sudo mv ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring


.. note:: 如果你的 ``admin node`` 就是 ``gateway host`` 则无需执行上面的步骤.


新建数据目录
=====================
 
部署脚本不会创建默认的 Ceph 对象网关数据目录. 因此需要为每一个 ``radosgw`` 守护进程实例新建\
数据目录. Ceph 配置文件中的 ``host`` 变量决定了在哪一个主机上运行 ``radosgw`` 守护进程实例.\
数据目录的典型命名规则是指定 ``radosgw`` 守护进程,集群名和守护进程 ID.

执行下面的命令在 ``gateway host`` 上新建所需的数据目录::

	sudo mkdir -p /var/lib/ceph/radosgw/ceph-radosgw.gateway


调整 Socket 目录权限
===================================

在一些发行版中, ``radosgw`` 守护进程是以没有什么权利的 UID 为 ``apache`` 的用户运行的, 但\
是这个 UID 必须在运行时写入 socket 文件的目录中具备写权限.

在 ``gateway host`` 上执行下面的命令来给上述 UID 授予默认 socket 位置的权限::

	sudo chown apache:apache /var/run/ceph


改变日志文件所有者
=====================

在一些发行版中, ``radosgw`` 守护进程是以没有什么权利的 UID 为 ``apache`` 的用户运行的,
但是日志文件默认是 ``root`` 用户所有的. 你必须日志文件的将所有者修改 ``apache`` 用户,以\
便 Apache 可以在这里写入日志文件. 为此，执行下面的命令::

	sudo chown apache:apache /var/log/radosgw/client.radosgw.gateway.log


启动 radosgw 服务
=====================

Ceph 对象网关守护进程需要启动. 为此，在 ``gateway host`` 上执行下面的命令:

在基于 Debian 的发行版上::

	sudo /etc/init.d/radosgw start

在基于 RPM 的发行版上::

	sudo /etc/init.d/ceph-radosgw start


新建一个网关配置文件
===================================

在你安装了 Ceph 对象网关的主机上, 比如 ``gateway host``, 新建一个 ``rgw.conf`` 文件, 对\
于 ``Debian-based`` 发行版将该文件放在 ``/etc/apache2/conf-available`` 目录下,对于 \
``RPM-based`` 发行版则将其放在 ``/etc/httpd/conf.d`` 目录下. 这个是 ``radosgw`` 所需\
的一个 Apache 配置文件. 对于 web 服务器而言这个文件必须是可读的.

按照下面的步骤执行:

#. 新建文件:

   对于基于 Debian 的发行版, 执行::

	sudo vi /etc/apache2/conf-available/rgw.conf

   对于基于 RPM 的发行版, 执行::

	sudo vi /etc/httpd/conf.d/rgw.conf

#. 对于使用 Apache 2.2 或者 Apache 2.4 早期版本的使用 localhost TCP 并且不支持 \
   Unix Domain Socket 的发行版而言，添加下面的内容到该文件中::

	<VirtualHost *:80>
	ServerName localhost
	DocumentRoot /var/www/html

	ErrorLog /var/log/httpd/rgw_error.log
	CustomLog /var/log/httpd/rgw_access.log combined

	# LogLevel debug

	RewriteEngine On

	RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization},L]

	SetEnv proxy-nokeepalive 1

	ProxyPass / fcgi://localhost:9000/

	</VirtualHost>

   .. note:: For Debian-based distros replace ``/var/log/httpd/``
      with ``/var/log/apache2``.

#. 对于使用 Apache 2.4.9 或者更新版本的支持 Unix Domain Socket 的发行版而言，添加下面\
   的内容到该文件中
   add the following contents to the file::

	<VirtualHost *:80>
	ServerName localhost
	DocumentRoot /var/www/html

	ErrorLog /var/log/httpd/rgw_error.log
	CustomLog /var/log/httpd/rgw_access.log combined

	# LogLevel debug

	RewriteEngine On

	RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization},L]

	SetEnv proxy-nokeepalive 1

	ProxyPass / unix:///var/run/ceph/ceph.radosgw.gateway.fastcgi.sock|fcgi://localhost:9000/

	</VirtualHost>


重启 Apache
==============

Apache 服务需要重启来加载新的配置文件.

对于基于 Debian 的发行版, 执行::

	sudo service apache2 restart

对于基于 RPM 的发行版, 执行::

	sudo service httpd restart

或者::

	sudo systemctl restart httpd


使用网关
=================

为了使用 REST 接口, 首先需要为 S3 接口初始化一个 Ceph 对象网关用户. 然后为 Swift 接口\
新建一个子用户.
查看 `管理手册`_ 获取有关用户管理的详细信息.

为 S3 访问新建一个  radosgw 用户
------------------------------------

A ``radosgw`` 用户需要新建并且赋予访问权限. 命令 ``man radosgw-admin`` 将展示提供更多额外的命令选项信息.

为了新建用户, 在 ``gateway host`` 上执行下面的命令::

	sudo radosgw-admin user create --uid="testuser" --display-name="First User"

上述命令的输出结果类似下面这样::

	{"user_id": "testuser",
	"display_name": "First User",
	"email": "",
	"suspended": 0,
	"max_buckets": 1000,
	"auid": 0,
	"subusers": [],
	"keys": [
	{ "user": "testuser",
	"access_key": "I0PJDPCIYZ665MW88W9R",
	"secret_key": "dxaXZ8U90SXydYzyS5ivamEP20hkLSUViiaR+ZDA"}],
	"swift_keys": [],
	"caps": [],
	"op_mask": "read, write, delete",
	"default_placement": "",
	"placement_tags": [],
	"bucket_quota": { "enabled": false,
	"max_size_kb": -1,
	"max_objects": -1},
	"user_quota": { "enabled": false,
	"max_size_kb": -1,
	"max_objects": -1},
	"temp_url_keys": []}


.. note:: ``keys->access_key`` 和 ``keys->secret_key`` 两个值\
   在访问时是必需的，用来验证。


创建一个 Swift 用户
-------------------

如果要通过 Swift 访问，必须创建一个 Swift 子用户。需要分两步完成，\
第一步是创建用户，第二步创建密钥。

在 ``gateway host`` 主机上进行如下操作：

创建 Swift 用户： ::

	sudo radosgw-admin subuser create --uid=testuser --subuser=testuser:swift --access=full

此命令的输出类似如下： ::

	{ "user_id": "testuser",
	"display_name": "First User",
	"email": "",
	"suspended": 0,
	"max_buckets": 1000,
	"auid": 0,
	"subusers": [
	{ "id": "testuser:swift",
	"permissions": "full-control"}],
	"keys": [
	{ "user": "testuser:swift",
	"access_key": "3Y1LNW4Q6X0Y53A52DET",
	"secret_key": ""},
	{ "user": "testuser",
	"access_key": "I0PJDPCIYZ665MW88W9R",
	"secret_key": "dxaXZ8U90SXydYzyS5ivamEP20hkLSUViiaR+ZDA"}],
	"swift_keys": [],
	"caps": [],
	"op_mask": "read, write, delete",
	"default_placement": "",
	"placement_tags": [],
	"bucket_quota": { "enabled": false,
	"max_size_kb": -1,
	"max_objects": -1},
	"user_quota": { "enabled": false,
	"max_size_kb": -1,
	"max_objects": -1},
	"temp_url_keys": []}

创建用户的密钥： ::

	sudo radosgw-admin key create --subuser=testuser:swift --key-type=swift --gen-secret

此命令的输出类似如下： ::

	{ "user_id": "testuser",
	"display_name": "First User",
	"email": "",
	"suspended": 0,
	"max_buckets": 1000,
	"auid": 0,
	"subusers": [
	{ "id": "testuser:swift",
	"permissions": "full-control"}],
	"keys": [
	{ "user": "testuser:swift",
	"access_key": "3Y1LNW4Q6X0Y53A52DET",
	"secret_key": ""},
	{ "user": "testuser",
	"access_key": "I0PJDPCIYZ665MW88W9R",
	"secret_key": "dxaXZ8U90SXydYzyS5ivamEP20hkLSUViiaR+ZDA"}],
	"swift_keys": [
	{ "user": "testuser:swift",
	"secret_key": "244+fz2gSqoHwR3lYtSbIyomyPHf3i7rgSJrF\/IA"}],
	"caps": [],
	"op_mask": "read, write, delete",
	"default_placement": "",
	"placement_tags": [],
	"bucket_quota": { "enabled": false,
	"max_size_kb": -1,
	"max_objects": -1},
	"user_quota": { "enabled": false,
	"max_size_kb": -1,
	"max_objects": -1},
	"temp_url_keys": []}


访问验证
========

然后你得验证一下刚创建的用户是否能访问网关。


测试 S3 访问
------------

你需要写一个 Python 测试脚本,并运行它以验证 S3 访问. S3 访问测试脚本\
将会连接 ``radosgw``, 然后新建一个新的 bucket 再列出所有的 buckets.\
``aws_access_key_id`` 和 ``aws_secret_access_key`` 的值就是前面\
``radosgw_admin`` 命令的返回值中的 ``access_key`` 和 ``secret_key``.

执行下面的步骤:

#. 首先你需要安装 ``python-boto`` 包.

   对于基于 Debian 的发行版请执行::

		sudo apt-get install python-boto

   对于基于 RPM 的发行版请执行::

		sudo yum install python-boto

#. 新建 Python 脚本::

	vi s3test.py

#. 添加下面的内容到该文件中::

	import boto
	import boto.s3.connection
	access_key = 'I0PJDPCIYZ665MW88W9R'
	secret_key = 'dxaXZ8U90SXydYzyS5ivamEP20hkLSUViiaR+ZDA'
	conn = boto.connect_s3(
	aws_access_key_id = access_key,
	aws_secret_access_key = secret_key,
	host = '{hostname}',
	is_secure=False,
	calling_format = boto.s3.connection.OrdinaryCallingFormat(),
	)
	bucket = conn.create_bucket('my-new-bucket')
	for bucket in conn.get_all_buckets():
		print "{name}\t{created}".format(
			name = bucket.name,
			created = bucket.creation_date,
	)

   将 ``{hostname}`` 替换为你配置了网关服务的主机的主机名,比如 ``gateway host``.

#. 运行这个脚本::

	python s3test.py

   输出类似下面的内容::

		my-new-bucket 2015-02-16T17:09:10.000Z

测试 swift 访问
-----------------

Swift 访问能够通过 ``swift`` 命令行客户端来验证. 命令 ``man swift`` 将提供更多、
可用命令行选项的详细信息.

执行下面的命令安装``swift`` 客户端:

   对于基于 Debian 的发行版::

		sudo apt-get install python-setuptools
		sudo easy_install pip
		sudo pip install --upgrade setuptools
		sudo pip install --upgrade python-swiftclient

   对于基于 RPM 的发行版::

		sudo yum install python-setuptools
		sudo easy_install pip
		sudo pip install --upgrade setuptools
		sudo pip install --upgrade python-swiftclient

执行下面的命令验证 swift 访问::

	swift -A http://{IP ADDRESS}/auth/1.0 -U testuser:swift -K ‘{swift_secret_key}’ list

将其中的 ``{IP ADDRESS}`` 替换为网关服务器的外网访问IP地址,将 ``{swift_secret_key}`` 替换为\
为 ``swift`` 用户所执行的 ``radosgw-admin key create`` 命令的输出.

举例如下::

	swift -A http://10.19.143.116/auth/1.0 -U testuser:swift -K ‘244+fz2gSqoHwR3lYtSbIyomyPHf3i7rgSJrF/IA’ list

输出如下::

	my-new-bucket




.. _配置参考——存储池: ../config-ref#pools
.. _资源池配置: ../../rados/configuration/pool-pg-config-ref/
.. _资源池: ../../rados/operations/pools
.. _用户管理: ../../rados/operations/user-management
.. _Ubuntu Trusty 中支持 UDS 的补丁: https://bugs.launchpad.net/ubuntu/+source/apache2/+bug/1411030
.. _管理手册: ../admin
