==========
 预检
==========

.. versionadded:: 0.60

谢谢您尝试 Ceph ！我们建议安装一个 ``ceph-deploy`` 管理\ :term:`节点`\ 和一\
个三节点的\ :term:`Ceph 存储集群`\ 来研究 Ceph 的基本特性。这篇\ **预检**\ 会\
帮你准备一个 ``ceph-deploy`` 管理节点、以及三个\
\ Ceph 节点（或虚拟机），以此构成 Ceph 存储集群。在进行下一步之前，请参\
见\ `操作系统推荐`_\ 以确认你安装了合适的 Linux 发行版。如果你在\
整个生产集群中只部署了单一 Linux 发行版的同一版本，那么在排查生产环境中\
遇到的问题时就会容易一点。

在下面的描述中\ :term:`节点`\ 代表一台机器。

.. include:: quick-common.rst


安装 Ceph 部署工具
===================

把 Ceph 仓库添加到 ``ceph-deploy`` 管理节点，然后安装 ``ceph-deploy`` 。

Debian / Ubuntu
---------------

在 Debian 和 Ubuntu 发行版上，执行下列步骤：

#. 添加 release key ： ::

	wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -

#. 添加Ceph软件包源，用Ceph稳定版（如 ``hammer`` ， ``jewel`` ，等等）替换掉 \
   ``{ceph-stable-release}`` 。例如： ::

	echo deb https://download.ceph.com/debian-{ceph-stable-release}/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list

#. 更新你的仓库，并安装 ``ceph-deploy`` ： ::

	sudo apt-get update && sudo apt-get install ceph-deploy

.. note:: 你也可以从欧洲镜像 eu.ceph.com 下载软件包，只需把 \
   ``http://ceph.com/`` 替换成 ``http://eu.ceph.com/`` 即可。


RHEL / CentOS
-------------

对于 CentOS 7 ，执行下列步骤：

#. 在 RHEL7 上，用 ``subscription-manager`` 注册你的目标机器，确认你的订阅， \
   并启用安装依赖包的“Extras”软件仓库。例如 ： ::
   
	sudo subscription-manager repos --enable=rhel-7-server-extras-rpms
   
#. 安装并启用 Extra Packages for Enterprise Linux (EPEL) 软件仓库。 \
   请查阅 `EPEL wiki`_ 获取更多信息。
   
#. 在 CentOS 上，可以执行下列命令： ::

	sudo yum install -y yum-utils && sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && sudo yum install --nogpgcheck -y epel-release && sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && sudo rm /etc/yum.repos.d/dl.fedoraproject.org*

#. 把软件包源加入软件仓库。用文本编辑器创建一个 YUM (Yellowdog \
   Updater, Modified) 库文件，其路径为 \
   ``/etc/yum.repos.d/ceph.repo`` 。例如： ::

	sudo vim /etc/yum.repos.d/ceph.repo

   把如下内容粘帖进去，用 Ceph 的最新主稳定版名字替换 \
   ``{ceph-stable-release}`` （如 ``jewel`` ），用你的Linux发行版\
   名字替换 ``{distro}`` （如 ``el7`` 为 CentOS 7 ）。最后保存到 ``/etc/yum.repos.d/ceph.repo`` 文件中。 ::

	[ceph-noarch]
	name=Ceph noarch packages
	baseurl=https://download.ceph.com/rpm-{ceph-release}/{distro}/noarch
	enabled=1
	gpgcheck=1
	type=rpm-md
	gpgkey=https://download.ceph.com/keys/release.asc


#. 更新软件库并安装 ``ceph-deploy`` ： ::

	sudo yum update && sudo yum install ceph-deploy


.. note:: 你也可以从欧洲镜像 eu.ceph.com 下载软件包，只需把 \
   ``http://ceph.com/`` 替换成 ``http://eu.ceph.com/`` 即可。


openSUSE
--------

目前 Ceph 尚未发布针对 openSUSE 的 RPM 包，但是默认的 update 仓\
库包含了 Ceph 的稳定版，所以安装 Ceph 只需要： ::

	sudo zypper install ceph
	sudo zypper install ceph-deploy
	
如果发行版本号过时了，你可以在 https://bugzilla.opensuse.org/index.cgi \
上报告一个 bug ，然后试试下面的仓库：

#. Hammer::

        https://software.opensuse.org/download.html?project=filesystems%3Aceph%3Ahammer&package=ceph

#. Jewel::

        https://software.opensuse.org/download.html?project=filesystems%3Aceph%3Ajewel&package=ceph

	
Ceph 节点安装
=============

你的管理节点必须能够通过 SSH 无密码地访问各 Ceph 节点。如果 \
``ceph-deploy`` 以某个普通用户登录，那么这个用户必须有无密码\
使用 ``sudo`` 的权限。


安装 NTP
--------

我们建议在所有 Ceph 节点上安装 NTP 服务（特别是 Ceph Monitor 节点），\
以免因时钟漂移导致故障，详情见\ `时钟`_\ 。

在 CentOS / RHEL 上，执行： ::

	sudo yum install ntp ntpdate ntp-doc

在 Debian / Ubuntu 上，执行： ::

	sudo apt-get install ntp

确保在各 Ceph 节点上启动了 NTP 服务，并且要使用同一个 NTP 服务器，\
详情见 `NTP`_ 。


安装 SSH 服务器
---------------

在\ **所有** Ceph 节点上执行如下步骤：

#. 在各 Ceph 节点安装 SSH 服务器（如果还没有）： ::

	sudo apt-get install openssh-server

   或者 ::

	sudo yum install openssh-server

#. 确保\ **所有** Ceph 节点上的 SSH 服务器都在运行。


创建部署 Ceph 的用户
--------------------

``ceph-deploy`` 工具必须以普通用户登录 Ceph 节点，且此用户拥有无密码使用 \
``sudo`` 的权限，因为它需要在安装软件及配置文件的过程中，不必输入密码。

较新版的 ``ceph-deploy`` 支持用 ``--username`` 选项提供可无密码\
使用 ``sudo`` 的用户名（包括 ``root`` ，虽然\ **不建议**\ 这样\
做）。使用 ``ceph-deploy --username {username}`` 命令时，指定的用\
户必须能够通过无密码 SSH 连接到 Ceph 节点，因为 ``ceph-deploy`` \
中途不会提示输入密码。

我们建议在集群内的\ **所有** Ceph 节点上给 ``ceph-deploy`` 创\
建一个特定的用户，但\ **不要**\ 用 “ceph” 这个名字。全集群统一的\
用户名可简化操作（非必需），然而你应该避免使用知名用户名，因为黑\
客们会用它做暴力破解（如 ``root`` 、 ``admin`` 、 ``{productname}`` ）。\
后续步骤描述了如何创建无 ``sudo`` 密码的用\
户，你要用自己取的名字替换 ``{username}`` 。

.. note:: 从 `Infernalis 版`_\ 起，用户名 "ceph" 保留给了 Ceph \
   守护进程。如果 Ceph 节点上已经有了 "ceph" 用户，升级前必须先\
   删掉这个用户。

#. 在各 Ceph 节点创建新用户。 ::

	ssh user@ceph-server
	sudo useradd -d /home/{username} -m {username}
	sudo passwd {username}

#. 确保各 Ceph 节点上新创建的用户都有 ``sudo`` 权限。 ::

	echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
	sudo chmod 0440 /etc/sudoers.d/{username}


允许无密码 SSH 登录
-------------------

正因为 ``ceph-deploy`` 不支持输入密码，你必须在管理节点上生成 \
SSH 密钥并把其公钥分发到各 Ceph 节点。 ``ceph-deploy`` 会尝试给\
初始 monitors 生成 SSH 密钥对。

#. 生成 SSH 密钥对，但不要用 ``sudo`` 或 ``root`` 用户。提示 \
   "Enter passphrase" 时，直接回车，口令即为空： ::

	ssh-keygen

	Generating public/private key pair.
	Enter file in which to save the key (/ceph-admin/.ssh/id_rsa):
	Enter passphrase (empty for no passphrase):
	Enter same passphrase again:
	Your identification has been saved in /ceph-admin/.ssh/id_rsa.
	Your public key has been saved in /ceph-admin/.ssh/id_rsa.pub.

#. 把公钥拷贝到各 Ceph 节点，把下列命令中的 ``{username}`` 替换\
   成前面\ `创建部署 Ceph 的用户`_\ 里的用户名。 ::

	ssh-copy-id {username}@node1
	ssh-copy-id {username}@node2
	ssh-copy-id {username}@node3

#. （推荐做法）修改 ``ceph-deploy`` 管理节点上的 \
   ``~/.ssh/config`` 文件，这样 ``ceph-deploy`` 就能用你所建的\
   用户名登录 Ceph 节点了，而无需每次执行 ``ceph-deploy`` 都要指定 \
   ``--username {username}`` 。这样做同时也简化了 ``ssh`` 和 \
   ``scp`` 的用法。把 ``{username}`` 替换成你创建的用户名。 ::

	Host node1
	   Hostname node1
	   User {username}
	Host node2
	   Hostname node2
	   User {username}
	Host node3
	   Hostname node3
	   User {username}


引导时联网
----------

Ceph 的各 OSD 进程通过网络互联并向 Monitors 上报自己的状态。如果\
网络默认为 ``off`` ，那么 Ceph 集群在启动时就不能上线，直到你打开网络。

某些发行版（如 CentOS ）默认关闭网络接口。所以需要确保网卡在系统启\
动时都能启动，这样 Ceph 守护进程才能通过网络通信。例如，在 Red \
Hat 和 CentOS 上，需进入 ``/etc/sysconfig/network-scripts`` 目录\
并确保 ``ifcfg-{iface}`` 文件中的 ``ONBOOT`` 设置成了 ``yes`` 。


确保联通性
----------

用 ``ping`` 短主机名（ ``hostname -s`` ）的方式确认网络联通性。\
解决掉可能存在的主机名解析问题。

.. note:: 主机名应该解析为网络 IP 地址，而非回环接口 IP 地址（即\
   主机名应该解析成非 ``127.0.0.1`` 的IP地址）。如果你的管理节点\
   同时也是一个 Ceph 节点，也要确认它能正确解析自己的主机名和 IP 地址\
   （即非回环 IP 地址）。


开放所需端口
------------

Ceph Monitors 之间默认使用 ``6789`` 端口通信， OSD 之间默认用 \
``6800:7300`` 这个范围内的端口通信。详情见\ `网络配置参考`_\ 。 \
Ceph OSD 能利用多个网络连接进行与客户端、monitors、其他 OSD 间的复制\
和心跳的通信。

某些发行版（如 RHEL ）的默认防火墙配置非常严格，你可能需要调整防火墙，\
允许相应的入站请求，这样客户端才能与 Ceph 节点上的守护进程通信。

对于 RHEL 7 上的 ``firewalld`` ，要对公共域开放 Ceph Monitors 使\
用的 ``6789`` 端口和 OSD 使用的 ``6800:7300`` 端口范围，并且要配置\
为永久规则，这样重启后规则仍有效。例如： ::

	sudo firewall-cmd --zone=public --add-port=6789/tcp --permanent

若使用 ``iptables`` ，要开放 Ceph Monitors 使用的 ``6789`` 端口和 \
OSD 使用的 ``6800:7300`` 端口范围，命令如下： ::

	sudo iptables -A INPUT -i {iface} -p tcp -s {ip-address}/{netmask} --dport 6789 -j ACCEPT

在每个节点上配置好 ``iptables`` 之后要一定要保存，这样重启之后才依然有效。\
例如： ::

	/sbin/service iptables save


终端（ TTY ）
-------------

在 CentOS 和 RHEL 上执行 ``ceph-deploy`` 命令时可能会报错。如果你的 Ceph \
节点默认设置了 ``requiretty`` ，执行 ``sudo visudo`` 禁用它，\
并找到 ``Defaults requiretty`` 选项，把它改为 ``Defaults:ceph !requiretty`` \
或者直接注释掉，这样 ``ceph-deploy`` 就可以用之前创建的用户（\
`创建部署 Ceph 的用户`_ ）连接了。

.. note:: 编辑配置文件 ``/etc/sudoers`` 时，必须用 \
   ``sudo visudo`` 而不是文本编辑器。


SELinux
-------

在 CentOS 和 RHEL 上， SELinux 默认为 ``Enforcing`` 开启状态。为简化\
安装，我们建议把 SELinux 设置为 ``Permissive`` 或者完全禁用，也\
就是在加固系统配置前先确保集群的安装、配置没问题。用下列命令把 \
SELinux 设置为 ``Permissive`` ： ::

	sudo setenforce 0

要使 SELinux 配置永久生效（如果它的确是问题根源），需修改其配置文件 \
``/etc/selinux/config`` 。


优先级/首选项
--------------

确保你的包管理器安装了优先级/首选项包且已启用。在 CentOS 上\
你也许得安装 EPEL ，在 RHEL 上你也许得启用可选软件库。 ::

	sudo yum install yum-plugin-priorities

比如在 RHEL 7 服务器上，可用下列命令安装 ``yum-plugin-priorities``\
\ 并启用 ``rhel-7-server-optional-rpms`` 软件库： ::

	sudo yum install yum-plugin-priorities --enablerepo=rhel-7-server-optional-rpms


总结
====

快速入门的预检部分到此结束，请继续\ `存储集群快速入门`_\ 。


.. _存储集群快速入门: ../quick-ceph-deploy
.. _操作系统推荐: ../os-recommendations
.. _网络配置参考: ../../rados/configuration/network-config-ref
.. _时钟: ../../rados/configuration/mon-config-ref#clock
.. _NTP: http://www.ntp.org/
.. _Infernalis 版: ../../release-notes/#v9-1-0-infernalis-release-candidate
.. _EPEL wiki: https://fedoraproject.org/wiki/EPEL
