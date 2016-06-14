====================
 块设备与 OpenStack
====================

.. index:: Ceph Block Device; OpenStack

通过 ``libvirt`` 你可以把 Ceph 块设备用于 OpenStack ，它配置了 QEMU 到 \
``librbd`` 的接口。 Ceph 把块设备映像条带化为对象并分布到集群中，这意味着大容量\
的 Ceph 块设备映像其性能会比独立服务器更好。

要把 Ceph 块设备用于 OpenStack ，必须先安装 QEMU 、 ``libvirt`` 和 OpenStack 。我\
们建议用一台独立的物理主机安装 OpenStack ，此主机最少需 8GB 内存和一个 4 核 CPU 。\
下面的图表描述了 OpenStack/Ceph 技术栈。


.. ditaa::  +---------------------------------------------------+
            |                    OpenStack                      |
            +---------------------------------------------------+
            |                     libvirt                       |
            +------------------------+--------------------------+
                                     |
                                     | configures
                                     v
            +---------------------------------------------------+
            |                       QEMU                        |
            +---------------------------------------------------+
            |                      librbd                       |
            +---------------------------------------------------+
            |                     librados                      |
            +------------------------+-+------------------------+
            |          OSDs          | |        Monitors        |
            +------------------------+ +------------------------+

.. important:: 要让 OpenStack 使用 Ceph 块设备，你必须有相应的 Ceph 集群访问权限。

OpenStack 里有三个地方可以和 Ceph 块设备结合：

- **Images：** OpenStack 的 Glance 管理着 VM 的 image 。Image 相对恒定， OpenStack 把它\
  们当作二进制文件、并以此格式下载。

- **Volumes：** Volume 是块设备， OpenStack 用它们引导虚拟机、或挂载到运行中的虚拟机上。 \
  OpenStack 用 Cinder 服务管理 Volumes 。

- **Guest Disks**: Guest disks 是装有客户操作系统的磁盘。默认情况下，启动一台虚拟机时，\
  它的系统盘表现为 hypervisor 文件系统的一个文件（通常位于 ``/var/lib/nova/instances/<uuid>/``）。\
  在 Openstack Havana 版本前，在 Ceph 中启动虚拟机的唯一方式是使用 Cinder 的 boot-from-volume 功能.
  不过，现在能够在 Ceph 中直接启动虚拟机而不用依赖于 Cinder，这一点是十分有\
  益的，因为可以通过热迁移更方便地进行维护操作。另外，如果你的 hypervisor 挂掉了，也可以\
  很方便地触发 ``nova evacuate`` ，并且几乎可以无缝迁移虚拟机到其他地方。

你可以用 OpenStack Glance 把 image 存储到 Ceph 块设备中，还可以使用 Cinder 通过 image 的写\
时复制克隆来启动虚拟机。

下面将详细指导你配置 Glance 、 Cinder 和 Nova ，虽然它们不一定一起用。你可以在\
本地硬盘上运行 VM 、却把 image 存储于 Ceph 块设备，反之亦可。

.. important:: Ceph 不支持 QCOW2 格式的虚拟机磁盘，所以，如果想要在 Ceph 中启动虚拟机\
  （ 临时后端或者从卷启动），Glance 镜像必须是 ``RAW`` 格式。

.. tip:: 本文档描述了在 Openstack Havana中使用 Ceph 块设备。更早的版本请参考
   `块设备与 OpenStack (Dumpling)`_。

.. index:: pools; OpenStack

创建存储池
==========

默认情况下， Ceph 块设备使用 ``rbd`` 存储池。你可以用任何可用存储池。建议为 \
Cinder 和 Glance 单独创建池。确保 Ceph 集群在运行，然后创建存储池。 ::

	ceph osd pool create volumes 128
	ceph osd pool create images 128
	ceph osd pool create backups 128
	ceph osd pool create vms 128

参考\ `创建存储池`_\ 为存储池指定归置组数量，参考\ `归置组`_\ 确定应该为存储池设定\
多少归置组。

.. _创建存储池: ../../rados/operations/pools#createpool
.. _归置组: ../../rados/operations/placement-groups


配置 OpenStack 的 Ceph 客户端
=============================

运行着 ``glance-api`` 、 ``cinder-volume`` 、 ``nova-compute`` 或 \
``cinder-backup`` 的主机被当作 Ceph 客户端，它们都需要 ``ceph.conf`` 文件。 ::

	ssh {your-openstack-server} sudo tee /etc/ceph/ceph.conf </etc/ceph/ceph.conf


安装 Ceph 客户端软件包
----------------------

在运行 ``glance-api`` 的节点上你需要 ``librbd`` 的 Python 绑定： ::

	sudo apt-get install python-rbd
	sudo yum install python-rbd

在 ``nova-compute`` 、 ``cinder-backup`` 和 ``cinder-volume`` 节点上，要安装 \
Python 绑定和客户端命令行工具： ::

	sudo apt-get install ceph-common
	sudo yum install ceph


配置 Ceph 客户端认证
--------------------

如果你启用了 `cephx 认证`_\ ，需要分别为 Nova/Cinder 和 Glance 创建新用户。命令如下： ::

	ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images'
	ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
	ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups'

把 ``client.cinder`` 、 ``client.glance`` 和 ``client.cinder-backup`` 的\
密钥环复制到适当的节点，并更改所有权： ::

	ceph auth get-or-create client.glance | ssh {your-glance-api-server} sudo tee /etc/ceph/ceph.client.glance.keyring
	ssh {your-glance-api-server} sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring
	ceph auth get-or-create client.cinder | ssh {your-volume-server} sudo tee /etc/ceph/ceph.client.cinder.keyring
	ssh {your-cinder-volume-server} sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
	ceph auth get-or-create client.cinder-backup | ssh {your-cinder-backup-server} sudo tee /etc/ceph/ceph.client.cinder-backup.keyring
	ssh {your-cinder-backup-server} sudo chown cinder:cinder /etc/ceph/ceph.client.cinder-backup.keyring

运行 ``nova-compute`` 的节点，其进程需要密钥环文件： ::

	ceph auth get-or-create client.cinder | ssh {your-nova-compute-server} sudo tee /etc/ceph/ceph.client.cinder.keyring

还得把 ``client.cinder`` 用户的密钥存进 ``libvirt`` 。 libvirt 进程从 \
Cinder 挂载块设备时要用它访问集群。

在运行 ``nova-compute`` 的节点上创建一个密钥的临时副本： ::

	ceph auth get-key client.cinder | ssh {your-compute-node} tee client.cinder.key

然后，在计算节点上把密钥加进 ``libvirt`` 、然后删除临时副本： ::

	uuidgen
	457eb676-33da-42ec-9a8c-9293d545c337

	cat > secret.xml <<EOF
	<secret ephemeral='no' private='no'>
	  <uuid>457eb676-33da-42ec-9a8c-9293d545c337</uuid>
	  <usage type='ceph'>
		<name>client.cinder secret</name>
	  </usage>
	</secret>
	EOF
	sudo virsh secret-define --file secret.xml
	Secret 457eb676-33da-42ec-9a8c-9293d545c337 created
	sudo virsh secret-set-value --secret 457eb676-33da-42ec-9a8c-9293d545c337 --base64 $(cat client.cinder.key) && rm client.cinder.key secret.xml

保留密钥的 uuid ，稍后配置 ``nova-compute`` 时要用。

.. important:: 所有计算节点上的 UUID 不一定非要一样。但考虑到平台的一致性，
   最好使用同一个 UUID 。

.. _cephx 认证: ../../rados/operations/authentication


配置 OpenStack 使用 Ceph
========================

配置 Glance
-----------

Glance 可使用多种后端存储 image 。要让它默认使用 Ceph 块设备，按如下配置 Glance 。

Juno 之前的版本
~~~~~~~~~~~~~~~

编辑 ``/etc/glance/glance-api.conf`` 并把下列内容加到 ``[DEFAULT]`` 段下： ::

	default_store = rbd
	rbd_store_user = glance
	rbd_store_pool = images
	rbd_store_chunk_size = 8


Juno 版
~~~~~~~

编辑 ``/etc/glance/glance-api.conf`` 并把下列内容加到 ``[glance_store]`` 段下： ::

	[DEFAULT]
	...
	default_store = rbd
	...
	[glance_store]
	stores = rbd
	rbd_store_pool = images
	rbd_store_user = glance
	rbd_store_ceph_conf = /etc/ceph/ceph.conf
	rbd_store_chunk_size = 8

关于 Glance 里可用的其它配置选项见 http://docs.openstack.org/trunk/config-reference/content/section_glance-api.conf.html。

.. important:: Glance 还没完全迁移到 'store' ，所以我们还得在 DEFAULT 段下配\
   置 store 。


任意版 OpenStack
~~~~~~~~~~~~~~~~

如果你想允许使用 image 的写时复制克隆，再添加下列内容到 ``[DEFAULT]`` \
段下： ::

	show_image_direct_url = True

注意，这会通过 Glance API 暴露后端存储位置，所以此选项启用时 endpoint 不应该\
被公开访问。

禁用 Glance 缓存管理，以免 image 被缓存到 ``/var/lib/glance/image-cache/`` \
下，假设你的配置文件里有 ``flavor = keystone+cachemanagement`` ： ::

	[paste_deploy]
	flavor = keystone


Image 属性
~~~~~~~~~~

建议配置如下 image 属性：

- ``hw_scsi_model=virtio-scsi``: 添加 virtio-scsi 控制器以获得更好的性能、\
  并支持 discard 操作；
- ``hw_disk_bus=scsi``: 把所有 cinder 块设备都连到这个控制器；
- ``hw_qemu_guest_agent=yes``: 启用 QEMU guest agent （访客代理）
- ``os_require_quiesce=yes``: 通过 QEMU guest agent 发送 fs-freeze/thaw 调用


配置 Cinder
-----------

OpenStack 需要一个驱动和 Ceph 块设备交互。还得指定块设备所在的存储池名。编辑 \
OpenStack 节点上的 ``/etc/cinder/cinder.conf`` ，添加： ::

    [DEFAULT]
    ...
    enabled_backends = ceph
    ...
    [ceph]
    volume_driver = cinder.volume.drivers.rbd.RBDDriver
    rbd_pool = volumes
    rbd_ceph_conf = /etc/ceph/ceph.conf
    rbd_flatten_volume_from_snapshot = false
    rbd_max_clone_depth = 5
    rbd_store_chunk_size = 4
    rados_connect_timeout = -1
    glance_api_version = 2

如果你使用了 `cephx 认证`_\ ，还需要配置用户及其密钥（前述文档中存进了 ``libvirt`` ）\
的 uuid ： ::

	[ceph]
	...
	rbd_user = cinder
	rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337

注意如果你为 cinder 配置了多后端， ``[DEFAULT]`` 节中必须有 ``glance_api_version = 2`` 。


配置 Cinder Backup
------------------

OpenStack 的 Cinder Backup 需要一个特定的守护进程，不要忘记安装它。编辑 Cinder \ 
Backup 节点的 ``/etc/cinder/cinder.conf`` 添加： ::

	backup_driver = cinder.backup.drivers.ceph
	backup_ceph_conf = /etc/ceph/ceph.conf
	backup_ceph_user = cinder-backup
	backup_ceph_chunk_size = 134217728
	backup_ceph_pool = backups
	backup_ceph_stripe_unit = 0
	backup_ceph_stripe_count = 0
	restore_discard_excess_bytes = true


配置 Nova 来挂载 Ceph RBD 块设备
--------------------------------

为了挂载 Cinder 块设备（块设备或者启动卷），必须告诉 Nova 挂载设备时使用的用户\
和 uuid 。libvirt会使用该用户来和 Ceph 集群进行连接和认证。 ::

	rbd_user = cinder
	rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337

这两个标志同样用于 Nova 的临时后端。


配置 Nova
---------

要让所有虚拟机直接从 Ceph 启动，必须配置 Nova 的临时后端。

建议在 Ceph 配置文件里启用 RBD 缓存（从 Giant 起默认启用）。另外，\
启用管理套接字对于故障排查来说大有好处，给每个使用 Ceph 块设备的虚拟机\
分配一个套接字有助于调查性能和/或异常行为。

可以这样访问套接字： ::

	ceph daemon /var/run/ceph/ceph-client.cinder.19195.32310016.asok help

编辑所有计算节点上的 Ceph 配置文件： ::

	[client]
		rbd cache = true
		rbd cache writethrough until flush = true
		admin socket = /var/run/ceph/guests/$cluster-$type.$id.$pid.$cctid.asok
		log file = /var/log/qemu/qemu-guest-$pid.log
		rbd concurrent management ops = 20

调整这些路径的权限： ::

	mkdir -p /var/run/ceph/guests/ /var/log/qemu/
	chown qemu:libvirtd /var/run/ceph/guests /var/log/qemu/

要注意， ``qemu`` 用户和 ``libvirtd`` 组可能因系统不同而不同，前面的实例\
基于 RedHat 风格的系统。

.. tip:: 如果你的虚拟机已经跑起来了，重启一下就能得到套接字。


Havana and Icehouse
~~~~~~~~~~~~~~~~~~~

Havana 和 Icehouse 需要补丁来实现写时复制克隆、修复 rbd 临时磁盘的镜像大小\
和热迁移中的缺陷。这些补丁可从基于 Nova `stable/havana`_  和 `stable/icehouse`_ \
的分支中获取。虽不是强制性的，但**强烈建议**使用这些补丁，以便能充分利用写时复制\
克隆功能的优势。

编辑所有计算节点上的 ``/etc/nova/nova.conf`` 文件，添加： ::

	libvirt_images_type = rbd
	libvirt_images_rbd_pool = vms
	libvirt_images_rbd_ceph_conf = /etc/ceph/ceph.conf
	libvirt_disk_cachemodes="network=writeback"
	rbd_user = cinder
	rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337

禁用文件注入也是一个好习惯。启动一个实例时， Nova 通常试图打开虚拟机的根文件\
系统。然后， Nova 会把比如密码、 ssh 密钥等值注入到文件系统中。然而，最好依赖\
元数据服务和 ``cloud-init`` 。

编辑所有计算节点上的 ``/etc/nova/nova.conf`` 文件，添加： ::

	libvirt_inject_password = false
	libvirt_inject_key = false
	libvirt_inject_partition = -2

为确保热迁移能顺利进行，要使用如下标志： ::

	libvirt_live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"


Juno
~~~~

在 Juno 版中， Ceph 块设备移到了 ``[libvirt]`` 段下。编辑所有计算节点上\
的 ``/etc/nova/nova.conf`` 文件，在 ``[libvirt]`` 段下添加： ::

	[libvirt]
	images_type = rbd
	images_rbd_pool = vms
	images_rbd_ceph_conf = /etc/ceph/ceph.conf
	rbd_user = cinder
	rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
	disk_cachemodes="network=writeback"


禁用文件注入也是一个好习惯。启动一个实例时， Nova 通常试图打开虚拟机的根文件\
系统。然后， Nova 会把比如密码、 ssh 密钥等值注入到文件系统中。然而，最好依赖\
元数据服务和 ``cloud-init`` 。

编辑所有计算节点上的 ``/etc/nova/nova.conf`` 文件，在 ``[libvirt]`` 段下添加： ::

	inject_password = false
	inject_key = false
	inject_partition = -2

为确保热迁移能顺利进行，要使用如下标志（在 ``[libvirt]`` 段下）： ::

	live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"

Kilo
~~~~

为虚拟机的临时根磁盘启用 discard 功能： ::

	[libvirt]
	...
	...
	hw_disk_discard = unmap # 启用 discard 功能（注意性能）


重启 OpenStack
==============

要激活 Ceph 块设备驱动、并把块设备存储池名载入配置，必须重启 \
OpenStack 。在基于 Debian 的系统上去对应节点执行下列命令： ::

	sudo glance-control api restart
	sudo service nova-compute restart
	sudo service cinder-volume restart
	sudo service cinder-backup restart

在基于 Red Hat 的系统上执行： ::

	sudo service openstack-glance-api restart
	sudo service openstack-nova-compute restart
	sudo service openstack-cinder-volume restart
	sudo service openstack-cinder-backup restart

一旦 OpenStack 启动并运行正常，应该就可以创建卷并用它启动虚拟机了。


从块设备引导
============

你可以用 Cinder 命令行工具从弄个 image 创建卷： ::

	cinder create --image-id {id of image} --display-name {name of volume} {size of volume}

注意 image 必须是 RAW 格式，你可以用 `qemu-img`_ 转换格式，如： ::

	qemu-img convert -f {source-format} -O {output-format} {source-filename} {output-filename}
	qemu-img convert -f qcow2 -O raw precise-cloudimg.img precise-cloudimg.raw

Glance 和 Cinder 都使用 Ceph 块设备，此镜像又是个写时复制克隆，就能非常\
快地创建一个新卷。在 OpenStack 操作面板里就能从那个启动虚拟机，步骤如下：

#. 启动新实例。
#. 选择与写时复制克隆关联的 image 。
#. 选择 'boot from volume' 。
#. 选择你刚创建的卷。


.. _qemu-img: ../qemu-rbd/#running-qemu-with-rbd
.. _块设备与 OpenStack (Dumpling): http://ceph.com/docs/dumpling/rbd/rbd-openstack
.. _stable/havana: https://github.com/jdurgin/nova/tree/havana-ephemeral-rbd
.. _stable/icehouse: https://github.com/angdraug/nova/tree/rbd-ephemeral-clone-stable-icehouse
