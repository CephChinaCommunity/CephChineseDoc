:orphan:

=======================================
 rbd -- 管理 RADOS 块设备（ RBD ）映像
=======================================

.. program:: rbd

摘要
====

| **rbd** [ -c *ceph.conf* ] [ -m *monaddr* ] [--cluster *cluster name*]
  [ -p | --pool *pool* ] [--size *size* ] [ --order *bits* ] [ *command* ... ]


描述
====

**rbd** 是个操纵 rados 块设备（ RBD ）映像的工具， QEMU/KVM 通过 \
Linux rbd 内核驱动和 rbd 存储驱动来使用。 RBD 映像是简单的块设备，\
它被条带化成小块对象后存储于 RADOS 对象存储集群。条带化后的对象尺\
寸必须是 2 的幂。


选项
====

.. option:: -c ceph.conf, --conf ceph.conf

   指定 ceph.conf 配置文件，而不是用默认的 /etc/ceph/ceph.conf 来确\
   定启动时需要的 monitor 地址。

.. option:: -m monaddress[:port]

   连接到指定 monitor，无需通过 ceph.conf 查找。

.. option:: --cluster cluster name

   使用自定义的集群名字，即不是默认的 *ceph* 。

.. option:: -p pool-name, --pool pool-name

   在指定存储池下操作，大多数命令都得指定。

.. option:: --no-progress

   不显示进度信息（有些命令会默认输出到标准错误）。


参数
====

.. option:: --image-format format-id

   选择使用哪个对象布局，默认为 1 。

   * format 1 - （废弃）新建 rbd 映像时使用最初的格式。此格式兼容所有版本的 \
     librbd 和内核模块，但是不支持诸如克隆这样较新的功能。

   * format 2 - 使用第二版 rbd 格式， librbd 和 3.11 版以上内核模块\
     才支持（除非是分拆的模块）。此格式增加了对克隆的支持，日后扩展以增加新功能\
     也变得更加容易。

.. option:: --size size-in-M/G/T

   指定新 rbd 映像的大小，单位可以是 M/G/T 。
   
.. option:: --object-size B/K/M

   指定 object 对象的大小，单位可以是 B/K/M ，会向上取整为 2 的整数次幂。默认值\
   是 4 MB，最小值是 4 KB，最大值是 32 MB。   

.. option:: --stripe-unit size-in-B/K/M

   指定条带单元尺寸，单位可以是 B/K/M 。详情见下面的条带化一节。

.. option:: --stripe-count num

   指定条带化要至少跨越多少对象后才能转回第一个。详情见条带化一节。

.. option:: --snap snap

   某些操作需要指定快照名。

.. option:: --id username

   指定 map 命令要用到的用户名（不含 ``client.`` 前缀）。

.. option:: --keyring filename

   指定 map 命令所需的包含特定用户密钥的文件。如果未指定，去默认密\
   钥环位置查找。

.. option:: --keyfile filename

   指定 map 命令所需的包含 ``--id user`` 用户密钥的文件。\
   如果同时指定了 ``--keyring`` 选项（常用），本选项就会被覆盖。

.. option:: --shared lock-tag

   `lock add` 选项允许使用同一标签的多个客户端同时锁住某个\
   映像。标签是任意字符串。当需要从多个客户端同时打开某映像时，此选\
   项很有用，像热迁移虚拟机时、或者在集群文件系统下使用时。

.. option:: --format format

   指定输出格式，默认： plain 、 json 、 xml 。

.. option:: --pretty-format

   使 json 或 xml 格式的输出更易读。

.. option:: -o map-options, --options map-options

   指定 map 映像时所用的选项。选项格式为逗号分隔的字符串（类似于 \
   mount(8) 的挂载选项）。详情见下面的 map 选项小节。

.. option:: --read-only

   以只读方式 map 映像，等价于 -o ro 。

.. option:: --image-feature feature-name

   指定创建 format 2 格式的 RBD 映像时，要启用的特性。可以通过重复\
   此选项来启用多个特性。当前支持下列特性：

   * layering: 支持分层
   * striping: 支持条带化 v2
   * exclusive-lock: 支持独占锁
   * object-map: 支持对象映射（依赖 exclusive-lock ）
   * fast-diff: 快速计算差异（依赖 object-map ）
   * deep-flatten: 支持快照扁平化操作
   * journaling: 支持记录 IO 操作（依赖独占锁）

.. option:: --image-shared

   指定该映像将被多个客户端同时使用。此选项将禁用那些依赖于独占映像\
   所有权的功能。

.. option:: --whole-object

   把 diff 操作范围限定在完整的对象级别，而非对象内差异。当某一\
   映像启用了 object-map 功能时，把 diff 操作限定到对象级别会显著地\
   提高性能，因为通过检查驻留于内存中的对象映射就可以计算出差异，而\
   无需针对映像内的各个对象去查询 RADOS 。


命令
====

.. TODO rst "option" directive seems to require --foo style options, parsing breaks on subcommands.. the args show up as bold too

:command:`ls` [-l | --long] [*pool-name*]
  列出 rbd_directory 对象中的所有 rbd 映像。加 -l 选项后也显示快照，\
  并用长格式输出，包括大小、父映像（若是克隆）、格式等等。

:command:`du` [-p | --pool *pool-name*] [*image-spec* | *snap-spec*]
  计算指定存储池内所有映像及其相关快照的磁盘使用量，包括预分配的和实\
  际使用的。此命令也可用于单个映像和快照。

  如果没启用 RBD 映像的 fast-diff 特性，此操作则需向多个 OSD \
  查询此映像涉及的每个对象。

:command:`info` *image-spec* | *snap-spec*
  显示指定 rbd 映像的信息（如映像大小和对象大小）。若映像是克隆品，会显示\
  其父快照的信息。若指定了快照，会显示是否被保护。

:command:`create` (-s | --size *size-in-M/G/T*) [--image-format *format-id*] [--object-size *B/K/M*] [--stripe-unit *size-in-B/K/M* --stripe-count *num*] [--image-feature *feature-name*]... [--image-shared] *image-spec*
  新建 rbd 映像。必须用 --size 指定尺寸。 --strip-unit 和 \
  --strip-count 参数是可选项，但必须同时使用。

:command:`clone` [--object-size *B/K/M*] [--stripe-unit *size-in-B/K/M* --stripe-count *num*] [--image-feature *feature-name*] [--image-shared] *parent-snap-spec* *child-image-spec*
  创建一个父快照的克隆（写时复制子映像）。若不指定 object size，会采用\
  父映像的值。克隆映像的大小和父快照相同。参数 --stripe-unit 和 \
  --stripe-count 是可选的，但必须同时使用。

  父快照必须已被保护（见 `rbd snap protect` ）。 只适用于 format 2 格式\
  的映像。

:command:`flatten` *image-spec*
  如果映像是个克隆，就从父快照拷贝所有共享的块，使子快照独立于父\
  快照，切断父子快照间的联系。如果没有克隆映像引用此父快照了，就可以\
  取消保护并删除它。

  只适用于 format 2 格式的映像。

:command:`children` *snap-spec*
  列出此映像指定快照的克隆品。它会检查各存储池、并输出存储池名/映像\
  名。

  只适用于 format 2 格式的映像。

:command:`resize` (-s | --size *size-in-M/G/T*) [--allow-shrink] *image-spec*
  调整 rbd 映像的大小。必须指定 size 参数。 --allow-shrink 选项允许缩小。

:command:`rm` *image-spec*
  删除 rbd 映像，包括所有数据块。如果映像有快照，此命令会失效。

:command:`export` (*image-spec* | *snap-spec*) [*dest-path*]
  把映像导出到目的路径，用 - （短线）输出到标准输出。

:command:`import` [--image-format *format-id*] [--object-size *B/K/M*] [--stripe-unit *size-in-B/K/M* --stripe-count *num*] [--image-feature *feature-name*]... [--image-shared] *src-path* [*image-spec*]
  创建映像，并从目的路径导入数据，用 - （短线）表示从标准输入导入。如\
  果可能的话，导入操作会试着创建稀疏映像。如果从标准输入导入，稀疏\
  化单位将是目标映像的数据块尺寸（即 object size ）。

  参数 --stripe-unit 和 --stripe-count 是可选的，但必须同时使用。

:command:`export-diff` [--from-snap *snap-name*] [--whole-object] (*image-spec* | *snap-spec*) *dest-path*
  导出映像的增量差异，用 - 表示导出到标准输出。若指定了起始快照，就只包含\
  与此快照的差异部分；否则包含映像的所有数据部分。结束快照用 --snap \
  选项或 @snap 语法（见下文）指定。此映像的差异格式包含了映像尺寸变更的\
  元数据、起始和结束快照，它高效地表达了被丢弃的或映像内的全 0 区域。

:command:`merge-diff` *first-diff-path* *second-diff-path* *merged-diff-path*
  把两个连续的增量差异合并为单个差异。前一个差异的结束快照必须与后一\
  个差异的起始快照相同。前一个差异可以是标准输入 - ，合并后的差异可\
  以是标准输出 - ，这样就可以合并多个差异文件，像这样： \
  'rbd merge-diff first second - | rbd merge-diff - third result' 。\
  注意，当前此命令仅支持 stripe_count == 1 这样的源增量差异。

:command:`import-diff` *src-path* *image-spec*
  导入一个映像的增量差异并应用到当前映像。如果此差异是在起始快照基础\
  上生成的，我们会先校验那个已存在快照再继续；如果指定了结束快照，\
  在应用这些变更前，我们还要检查它是否不存在，完成后再创建结束快照。

:command:`diff` [--from-snap *snap-name*] [--whole-object] *image-spec* | *snap-spec*
  打印出从指定快照点起、或从映像创建点起，映像内的变动区域。输出的\
  各行都包含起始偏移量（按字节）、数据块长度（按字节）、还有 zero \
  或 data ，用来指示此区域是 0 还是包含有其它数据。

:command:`cp` (*src-image-spec* | *src-snap-spec*) *dest-image-spec*
  把源映像内容复制进新建的目标映像，目标映像和源映像将有相同的大小、\
  object size 和格式。

:command:`mv` *src-image-spec* *dest-image-spec*
  映像重命名。注：不支持跨存储池。

:command:`image-meta list` *image-spec*
  显示此映像持有的元数据。第一列是关键字、第二列是值。

:command:`image-meta get` *image-spec* *key*
  获取关键字对应的元数据值。

:command:`image-meta set` *image-spec* *key* *value*
  设置元数据关键字的值，会显示在 `metadata-list` 中。

:command:`image-meta remove` *image-spec* *key*
  删除元数据关键字及其值。

:command:`object-map rebuild` *image-spec* | *snap-spec*
  为指定映像重建无效的对象映射关系。指定映像快照时，将为此快照重建\
  无效的对象映射关系。

:command:`snap ls` *image-spec*
  列出指定映像内的快照。

:command:`snap create` *snap-spec*
  新建快照。需指定快照名。

:command:`snap rollback` *snap-spec*
  把指定映像回滚到某个快照。此动作会递归整个块阵列，并把数据头内容更新\
  到快照版本。

:command:`snap rm` *snap-spec*
  删除指定快照。

:command:`snap purge` *image-spec*
  删除某个映像的所有快照。

:command:`snap protect` *snap-spec*
  保护快照，以防删除，这样才能从它克隆（见 `rbd clone` ）。做克隆前\
  必须先保护快照，保护意味着存在有依赖于此快照的克隆。 \
  `rbd clone` 不能在未保护的快照上操作。

  只适用于 format 2 格式的映像。

:command:`snap unprotect` *snap-spec*
  取消对快照的保护（撤销 `snap protect` ）。如果还有克隆出的子快照\
  尚在， `snap unprotect` 命令会失效。（注意克隆和其父快照可能会位于\
  不同的存储池。）

  只适用于 format 2 格式的映像。

:command:`map` [-o | --options *map-options* ] [--read-only] *image-spec* | *snap-spec*
  通过内核 rbd 模块把指定映像映射到某一块设备。

:command:`unmap` *image-spec* | *snap-spec* | *device-path*
  取消通过内核 rbd 模块的所做的块设备映射。

:command:`showmapped`
  显示通过内核 rbd 模块映射的 rbd 映像。

:command:`nbd map` [--device *device-path*] [--read-only] *image-spec* | *snap-spec*
  通过 rbd-nbd 工具把指定映像映射成一个块设备。

:command:`nbd unmap` *device-path*
  取消通过 rbd-nbd 工具映射的块设备。

:command:`nbd list`
  显示使用中的通过 rbd-nbd 工具映射的 nbd 设备。

:command:`status` *image-spec*
  显示映像状态，包括哪个客户端打开着它。

:command:`feature disable` *image-spec* *feature-name*...
  禁用指定镜像的某些特性，可以一次指定多个。

:command:`feature enable` *image-spec* *feature-name*...
  启用指定镜像的某些特性，可以一次指定多个。

:command:`lock list` *image-spec*
  显示加在映像上的锁，第一列是 `lock remove` 可以使用的锁名。

:command:`lock add` [--shared *lock-tag*] *image-spec* *lock-id*
  为映像加锁。锁标识是用户所选的任意名字。默认加的是互斥锁，这\
  意味着如果映像已经加过锁的话此命令会失败。 --shared 选项会改变此行为。\
  注意，加锁操作不影响除加锁之外的任何操作，也不会保护映像、防\
  止它被删除。

:command:`lock remove` *image-spec* *lock-id* *locker*
  释放映像上的锁。锁标识和其持有者来自 lock ls 。

:command:`bench-write` [--io-size *size-in-B/K/M/G/T*] [--io-threads *num-ios-in-flight*] [--io-total *total-size-to-write-in-B/K/M/G/T*] [--io-pattern seq | rand] *image-spec*
  向某一映像生成一系列写操作，以此测量写吞吐量和延时。默认参数为 \
  --io-size 4096 、 --io-threads 16 、 --io-total 1G 、 \
  --io-pattern seq 。

映像名和快照名规则
==================

| *image-spec* 代表 [*pool-name*]/*image-name*
| *snap-spec*  代表 [*pool-name*]/*image-name*\ @\ *snap-name*

*pool-name* 的默认值是 rbd 。如果某个映像名包含斜杠字符（ ’/‘ ），\
那么还必须指定 *pool-name* 。

你可以用 --pool 、 --image 和 --snap 选项分别指定各个名字，但是并不\
鼓励这种用法，大家还是倾向于上面的语法规则。


条带化
======

RBD 映像被条带化为多个对象，然后存储到 Ceph 分布式对象存储\
（ RADOS ）集群中。因此，对此映像的读、写请求会分散到集群内的\
很多节点，也就避免了映像巨大或繁忙时可能出现的单节点瓶颈。

条带化由三个参数控制：

.. option:: object-size

   条带化产生的对象大小是 2 的幂。会向上取整为最接近 2 的幂的值。默认值\
   是 4 MB ，最小值是 4 KB，最大值是 32 MB。

.. option:: stripe_unit

   各 [*stripe_unit*] 是一段连续的字节，相邻地存储于同一对象，放满后\
   再去下一个对象。

.. option:: stripe_count

   我们写够 [*stripe_count*] 个长度为 [*stripe_unit*] 字节的对象后，\
   再转回到第一个对象写另一轮条带，直到达到对象的最大尺寸。此时，我们\
   再写下一轮 [*stripe_count*] 个对象。

默认情况下， [*stripe_unit*] 和对象尺寸相同、且 [*stripe_count*] \
为 1 。指定 [*stripe_unit*] 为其他值，需支持 STRIPINGV2 特性（ Ceph 0.53 起\
加入）并使用 format 2 格式的映像。


Map 选项
========

这里的大多数选项主要用于调试和基准测试。默认值设置于内核中，\
因此还与所用内核的版本有关。

libceph （每个客户端实例）选项：

* fsid=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee - 应该由客户端提供\
  的 FSID 。

* ip=a.b.c.d[:p] - 客户端使用的 IP 和可选端口。

* share - 允许与其它映射共享客户端实例（默认）。

* noshare - 禁止与其它映射共享客户端实例。

* crc - 在写入数据时计算 CRC32C 校验值（默认）。

* nocrc - 在写入数据时不计算 CRC32C 校验值。

* cephx_require_signatures - 要求对 cephx 消息签名（从 3.19 起，\
  默认开启）。

* nocephx_require_signatures - 不要求对 cephx 消息签名（从 3.19 \
  起）。

* tcp_nodelay - 在客户端套接字禁用 Nagle's 算法（从 4.0 起，默认开启）。

* notcp_nodelay - 在客户端套接字启用 Nagle's 算法（从 4.0 起）。

* cephx_sign_messages - 启用消息签名（从 4.4 起，默认开启）。

* nocephx_sign_messages - 禁用消息签名（从 4.4 起）。

* mount_timeout=x - 执行 `rbd map` 和 `rbd unmap` 时所涉及的各\
  操作步骤的超时值（默认为 60 秒）。特别是从 4.2 起，与集群间没\
  有连接时，使用此选项可确保 `rbd unmap` 操作最终超时了。

* osdkeepalive=x - OSD 保持连接的期限（默认为 5 秒）。

* osd_idle_ttl=x - OSD 闲置 TTL （默认为 60 秒）。

映射（每个块设备）选项：

* rw - 以读写方式映射映像（默认）。

* ro - 以只读方式映射映像，等价于 --read-only 。

* queue_depth=x - 队列深度（从 4.2 起，默认为 128 个请求）。


样例
====

新建一个 100GB 的 rbd 映像： ::

	rbd create mypool/myimage --size 102400

使用非默认对象尺寸（ 8 MB ）： ::

	rbd create mypool/myimage --size 102400 --object-size 8M

删除 rbd 映像（慎重！）： ::

	rbd rm mypool/myimage

新建快照： ::

	rbd snap create mypool/myimage@mysnap

创建已保护快照的写时复制克隆： ::

	rbd clone mypool/myimage@mysnap otherpool/cloneimage

查看快照有哪些克隆： ::

	rbd children mypool/myimage@mysnap

删除快照： ::

	rbd snap rm mypool/myimage@mysnap

通过启用了 cephx 的内核映射一映像： ::

	rbd map mypool/myimage --id admin --keyfile secretfile

通过内核，映射使用非默认名字 *ceph* 集群中的某一映像： ::

	rbd map mypool/myimage --cluster *cluster name*

取消映射： ::

	rbd unmap /dev/rbd0

创建一个映像并克隆： ::

	rbd import --image-format 2 image mypool/parent
	rbd snap create mypool/parent@snap
	rbd snap protect mypool/parent@snap
	rbd clone mypool/parent@snap otherpool/child

使用较小的 stripe_unit 值来创建映像（在某些情况下可更好地分布小块写）： ::

	rbd create mypool/myimage --size 102400 --stripe-unit 65536B --stripe-count 16

更改映像的格式，先导出、再导入为期望格式： ::

	rbd export mypool/myimage@snap /tmp/img
	rbd import --image-format 2 /tmp/img mypool/myimage2

给映像加互斥锁： ::

	rbd lock add mypool/myimage mylockid

释放锁： ::

	rbd lock remove mypool/myimage mylockid client.2485


使用范围
========

**rbd** 是 Ceph 的一部分，这是个大规模可伸缩、开源、分布式的存储\
系统，更多信息参见 http://ceph.com/docs 或 http://docs.ceph.org.cn/。


另请参阅
========

:doc:`ceph <ceph>`\(8),
:doc:`rados <rados>`\(8)
