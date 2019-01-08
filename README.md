[AWS Database Migration Service](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html)是一项云服务，可轻松迁移关系数据库、数据仓库、NoSQL 数据库及其他类型的数据存储。它的优势是：
- 支持多种数据库之间的同步。
- 支持异构数据库的同步。比如：MySql到Redshift, Oracle到AWS Aurora等。
- 增量实时的同步。由于采用读取数据库的log来进行同步数据，所以理论上，可以做到实时的同步。而且，这种方式的同步对Source数据库的压力比较小。
- 支持Insert, Update, Delete的操作的数据同步。虽然在Source数据库中，绝大多数的操作是Insert，但一般会有少量的Delete和Update。
- 简单可靠。由于是AWS官方的服务，是比较可靠的。虽然是黑箱，但可以通过日志来获取详细信息。和其他类似工具来比较，AWS DMS不是最易用的，但如果Target数据库是AWS自己的数据库（Aurora，DynamoDB，Redshift等），AWS DMS无疑更加的可靠。

本文中详细描述了，如何使用DMS把MySql的数据同步到AWS Redshift。其他数据库之间的同步，仅供参考。

# 1. 背景分析
在我们的项目中，需要把mysql数据库同步到redshift中。mysql是业务数据库，数据会持续更新，所以数据同步不是一次性的工作。而reshift是数据仓库，未来需要同步多个Source数据库到redshift。

第一期，只需要同步10张表。除了几张小表，大部分表的数据将在百万级别以上，其中最大的两张表情况如下：

| table_name | table_size(MB) | table_rows | description          |
|------------|----------------|------------|----------------------|
| A          | 4,743.67       | 29,452,042 | 包含一个text类型字段，里面有长文本。 |
| B          | 3,496.37       | 19,794,750 |                   |

全部的table size是10G左右，table rows是6000w左右。
总体而言，数据量并不大。但对于第一次同步，由于需要同步全量数据，还是有些压力的。

实际的全量同步测试中，表A，B的性能不是很好，尤其是表A，其数据同步速度只有每小时500,000-1,500,000行。这意味着要完成A表的同步，需要1到3天的时间。由于业务数据库在白天的访问量还是很大的，1-3天同步时间是无法接受的。业务数据库的空闲时段是03:00-06:00（没有任何的数据变化），只有三个小时，所以全量同步的时间最好不要超过3个小时。

经过监控并分析，我们还发现：

1. mysql数据库重启后，再进行数据同步，性能提高100-300倍。最快可以达到每小时150,000,000行。实际中A表可以在15分钟之内完成同步。重试了几次，发现都是这个情况，真是有点儿匪夷所思，没法解释。不知道我们的测试环境有什么特别的设置，所以不能保证这种情况会在你的环境里完全重现。
2. replciation instance是否有足够的IOPS。当同步的数据量比较大的时候，IOPS有会持续达到3000。默认情况下，在credit用完后（第一次能用30-60分钟），IOPS会下降到磁盘容量的3倍。比方，如果replication instance的磁盘是100G，则基准的IOPS是300。IOPS从3000到300，性能会有5-10倍的降低。也就是说对于大数据量的全量同步，一个大的磁盘非常重要。建议是1000G，这样IOPS可以持续达到3000，而不会衰减。
3. LOB(Large Binary Objects, 比如text, mediumtext,  longblob等)对性能的影响还是比较大的。实际也发现，表A比表B慢很多。
4. 在多张表同步的过程中，replication instance 的CPU占用率会比较高。对于多表的全量导入，需要选用配置高一些的cpu，建议是dms.c4.xlarge以上。对于增量的同步，如果每天的增量在1G以内，dms.t2.medium够用的。
5. 全量同步时，最好选择在MySql的空闲时段。这是因为如果MySql有频繁的数据变化，DMS读取MySql数据的性能会下降，replciation instance也需要更大的内存来缓存增量数据，而这些都会进一步降低同步的性能。

综合分析，以上几点点对性能的影响是比较大的。在方案中我们将会重点考虑。

除了以上的重点，也可以关注以下几点。

1. Source数据库的负载。实际测试全量同步时，Source MySql没有任何的数据变化，只有DMS的数据读取，从监控看，其负载完全正常。Read IPOS一般在。由于我们测试的只有10张表，实际数据库中可能是50-100张，甚至更多，所以在全量同步时，一般认为对于Source MySql的访问还是尽量要控制，以便不会对Source产生过多的性能压力
2. Target数据库的负载。实际测试全量同步时，Target Redshift的负载完全正常。Write IPOS一般在，
3. Replication Instance的内存。实际测试全量同步时，4G内存是够用的。监控发现在replication instance上，FreeableMemory超过1G，SwapUsage接近0。
4. Redshfit数据库，仅支持从S3拷贝数据，这个过程可能会对性能有影响。数据的整个流程是：
    - DMS拷贝Soure数据到replciation isntance的本地磁盘
    - replciation isntance对数据进行进行整理转换。
    - 把整理后的数据上传到S3。和其他数据库比较，这是增加的一个步骤。qi's由于没有发现metric来量化这步时间，目前不确定是否会影响性能。
    - Copy S3数据到Redshfit    



# 2. 解决方案

一般来说，数据同步主要分为两个步骤:
- 全量同步(Full Load)
- 增量同步(ongoing replication)

经过上面一节的分析，可以看到全量同步的速度是关键所在。基于此，我们的有几种方案：

1. 业务数据库直接同步。在MySql数据库空闲时段进行全量同步，全量同步完成后，再进行增量同步。这种方案的前提是全量同步能够在空闲期之内完成，只有这样，才能对业务数据库的影响降低到最低。
2. read replica节点同步。当业务数据库有read replica节点的情况，选择从read replica节点同步到Redshift。这种情况下，即使全量同步的速度有些慢，一般认为也是可以接受的。
3. backup节点同步。步骤如下：
    - 在业务数据库空闲时段进行数据备份。然后把这个备份恢复到一个新的MySql节点（backup节点）。同时记录下备份完成的时间点。
    - 从backup节点全量同步到Redshift中。完成后删除backup节点和相关资源。
    - 创建新的DMS Task，这些Task连接到业务数据库，设置从上面记录的时间点开始增量同步。

方案一，对全量同步的时间有要求。方案二，要求有read replcia。方案三，要求有业务空闲时段。综合考虑，我们选用方案三，好处如下：

- 对全量同步的时间没有绝对要求。当然时间越短越好。
- 对业务数据库的影响最小。在方案二中，一些报表或业务系统会使用read replica节点作为数据源，全量复制对于这些系统的性能还是有影响的。
- 便于验证数据同步的结果。和前两个方案比较，backup节点是没有任何数据变化的，这样便于比较Source和Target之间的数据是否完全一致。

# 3. 代码实现
## 3.1 准备


## 3.2 全量同步(Full Load)
```
# 1 配置文件编辑。
cat << EOF > dms.conf 
export rep_instance_class=dms.c4.large
export rep_instance_id=wechat-full
export source_endpoint_id=wechat-mysql
export target_endpoint_id=wechat-redshift
export allocated_storage=1000
export vpc_security_group_ids=
export avail_zone=
export rep_sg_id=
export tasks=

export source_db_type=mysql
export source_db_server=
export source_db_port=3306
export source_db_user=
export source_db_password=
export source_db_extra=""

export target_db_type=redshift
export target_db_server=
export target_db_name=wechat
export target_db_port=5439
export target_db_user=
export target_db_password=
export target_db_extra="acceptanydate=true;truncateColumns=true"
EOF

# 2. 创建replication instance.
# 创建需要几分钟，在console中查看。等到实例创建成功后，再执行下一步。
./create_rep.sh

# 3. 创建source和target endpoints
./create_endpoint.sh

# 4. 创建task
# 创建后，在控制台检查task状态是否是ready
./create_task.sh

# 5. start task
# 在console中启动task。希望能在2个小时之内完成。
# 1. customer-service-prod-messages-history  (13m)
#    如果性能不佳，尝试重新数据库服务器。不重启，数据同步的性能差100-300倍，原因何在？
# 2. asc-weixin-mp-menu-click-his            (7m)
# 3. customer-service-prod-others （4m）和 asc-weixin-mp-others (5m)

# 6. 删除所有task和replication instance
# 在控制台中依次删除。
```



## 3.3 增量同步(ongoing replication)
```
# 1 配置文件编辑。
cat << EOF > dms.conf 
export rep_instance_class=dms.t2.medium
export rep_instance_id=wechat-cdc
export source_endpoint_id=wechat-mysql
export target_endpoint_id=wechat-redshift
export allocated_storage=50
export vpc_security_group_ids=
export avail_zone=
export rep_sg_id=
export tasks=

export source_db_type=mysql
export source_db_server=d
export source_db_port=3306
export source_db_user=
export source_db_password=
export source_db_extra=""

export target_db_type=redshift
export target_db_server=
export target_db_name
export target_db_port=5439
export target_db_user=
export target_db_password=
export target_db_extra="acceptanydate=true;truncateColumns=true"
EOF

# 2. 创建replication instance.
# 创建需要几分钟，在console中查看。等到实例创建成功后，再执行下一步。
./create_rep.sh

# 3. 创建source和target endpoints
./test_endpoint.sh

# 4. 创建task
# 创建后，在控制台检查task状态是否是ready
./create_task.sh

# 5. start task
# 在console中启动task。当Full Load的task全部完成后。
# 1. cdc-customer-service-prod 和 cdc-asc-weixin-mp 
```

# 参考
- [AWS DMS Best Practices]( https://docs.aws.amazon.com/dms/latest/userguide/CHAP_BestPractices.htm)
- [How to Script a Database Migration](https://amazonaws-china.com/blogs/database/how-to-script-a-database-migration/)
- [DMS Available Commands](https://docs.aws.amazon.com/cli/latest/reference/dms/index.html#cli-aws-dms)
- [Monitoring AWS DMS Tasks](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Monitoring.html)
