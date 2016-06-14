:orphan:

===================================
 rbd-fuse -- 把 rbd 映像展现为文件
===================================

.. program:: rbd-fuse

摘要
====

| **rbd-fuse** [ -p pool ] [-c conffile] *mountpoint* [ *fuse options* ]


描述
====

**rbd-fuse** 是个 rbd 映像的用户空间文件系统（ File system in USErspace，FUSE ）客户端。\
给定一个包含 rbd 映像的存储池，它就可以在用户空间把那些映像挂载到 **mountpoint** 下，并\
显示为普通文件。

用下列命令卸载文件系统： ::

        fusermount -u mountpoint

或者向 ``rbd-fuse`` 进程发送 ``SIGINT`` 信号。


选项
====

rbd-fuse 无法识别的选项将传递给 libfuse 。

.. option:: -c ceph.conf

   用指定的 *ceph.conf* 配置文件、而非默认的 ``/etc/ceph/ceph.conf`` 来确定启动 \
   monitor 时所用的地址。

.. option:: -p pool

   在 *pool* 存储池中搜索 rbd 映像，默认为 ``rbd`` 。


使用范围
========

**rbd-fuse** 是 Ceph 的一部分，这是个大规模可伸缩、开源、分布式的\
存储系统，更多信息参见 http://ceph.com/docs 或 http://docs.ceph.org.cn/。


另请参阅
========

fusermount(8),
:doc:`rbd <rbd>`\(8)
