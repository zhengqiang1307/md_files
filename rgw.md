# Ceph RGW 的数据分布和还原用户数据
## Ceph数据存储原理 
- Ceph的对象存储接口是由RGW组件提供的。RGW为用户提供了一套兼容S3及Swift协议的API。用户通过RGW API完成上传、下载等数据相关操作。

- 在RGW提供了整体上传和分段上传两种文件上传方式。
- 每部分RGW上传的文件片段根据配置进行切块，形成若干个Object，Object逻辑归属于某个PG中，最终Object存储在PG映射的一组OSD磁盘上，Ceph最终存储的是对象（内容+属性）。
- 无论是块存储、文件存储、还是对象存储存入Ceph的数据都以如下方式组织：

  ![avatar](https://note.youdao.com/yws/api/personal/file/52ED215EC1524C63911E6E82DB4040D3?method=download&shareKey=65d61350f7c45b767cc80b0cb07e9961)

- File：client读写的文件
- object：是将File切块后的存储实体
- oid：（object id） = ino（File的ID）+ono（切块序号）
- mask：PG总数m(m为2的整数幂)-1
- PG（Placement Group）：放置组（标识为 PGID）是一个逻辑的概念，一个PG存放多个对象，每个存储节点有上百个PG。
- OSD（Object Storage Device）：对象存储设备，提供存储资源。

 ## Ceph RGW对象存储模型
1. RGW对象整体上传
   ![](https://note.youdao.com/yws/api/personal/file/46F61194C6E54BE4844ECCD1C09A86F0?method=download&shareKey=7784a74f014b77d992774404ea012edb)

    ![](https://note.youdao.com/yws/api/personal/file/86D3AAECF98C42CD884FD2510E08B920?method=download&shareKey=884d4f39533d0d7183e8366f0174a299)

- rgw_max_chunk_size：首对象大小（默认1M）
- rgw_obj_stripe_size：除首对象外的其他分片大小(默认8M)
- rgw_max_put_size：整体上传文件限制（默认512G）
- 整体上传的文件，将会切分为一个大小为rgw_max_chunk_size的首部Rados对象，及若干个大小为rgw_obj_stripe_size的中间Rados对象（最后一个对象可以小于rgw_obj_stripe_size）。其中，扩展属性存在于首部Rados对象的扩展属性中，包含了分片信息。
- 首部对象命名：{桶ID}{上传对象名}
- 条带对象命名：{桶ID}\_shadow\_.{32位字符串}_{分片号}
1. RGW对象分段上传
![](https://note.youdao.com/yws/api/personal/file/C381A8FDFB8D4605AE58249123FEF373?method=download&shareKey=364629990aa576f47adb9111de8aec17)

![](https://note.youdao.com/yws/api/personal/file/CCDFFF3C06504293B729E37414EF5C4D?method=download&shareKey=1730079c9bc3ac7669337aaa7bbc2231)

- rgw_multipart_min_part_size：分段最小大小（默认1M）
- rgw_multipart_part_upload_limit:最大分段数（默认10000）
- 分段上传，用户上传时自主对上传文件进行分段，除最后一个分段外必须大于rgw_multipart_min_part_size，每段文件独立上传，每段都可看作一个整体上传。上传完成后，会生成一个内容为空的首部Rados对象，用于存储RGW对象的属性信息，其中包含分段信息。
- 每个分段的第一个对象命名：{桶ID}\_multipart\_{上传对象名}.{上传ID}.{分段号}
- {bucketindexid}+\__multipart_+{objectname}+.+{uploadid}+.+{partnumber}
- 条带对象命名：{桶ID}\_shadow\_{上传对象名}.{上传ID}.{分段号}\_{分片号}
- {bucketindexid}+__shadow_+{objectname}+.+{uploadid}+.+{partnumber}+_+{shardnum}
## Ceph RGW对象存储实践
### 整体上传
![](https://note.youdao.com/yws/api/personal/file/FCC81C56D4244887B2A3152B3461DC34?method=download&shareKey=416b876600ee9f730d2fdf3f50202b30)



- 10M文件
```
[root@ceph01 my-cluster]# dd if=/dev/zero of=./file10M.dat bs=1MB count=10
记录了10+0 的读入
记录了10+0 的写出
10000000字节(10 MB)已复制，0.0166375 秒，601 MB/秒
```
- 创建桶
```
[root@ceph01 my-cluster]# s3cmd mb s3://bucket
Bucket 's3://bucket/' created
```
- 上传文件
```
[root@ceph01 my-cluster]# s3cmd put file10M.dat s3://bucket
upload: 'file10M.dat' -> 's3://bucket/file10M.dat'  [1 of 1]
 10000000 of 10000000   100% in    1s     7.34 MB/s  done
```
- 定位首部对象,查找bucket marker
```
[root@node my-cluster]# radosgw-admin bucket stats|grep marker
        "marker": "33a99195-43b6-4969-8606-d1603eb2e69d.4137.1",
        "max_marker": "0#",
```
- 查找file10M.dat的首部和条带对象
```
[root@node my-cluster]# rados -p default.rgw.buckets.data ls
33a99195-43b6-4969-8606-d1603eb2e69d.4137.1_file10M.dat
33a99195-43b6-4969-8606-d1603eb2e69d.4137.1__shadow_.mC4tn648n3JWaaazuNpmmJ_7vPY8cTg_2
33a99195-43b6-4969-8606-d1603eb2e69d.4137.1__shadow_.mC4tn648n3JWaaazuNpmmJ_7vPY8cTg_1
```

- 查看首部对象存储位置
```
[root@node my-cluster]# ceph osd map default.rgw.buckets.data 33a99195-43b6-4969-8606-d1603eb2e69d.4137.1_file10M.dat
osdmap e30 pool 'default.rgw.buckets.data' (6) object '33a99195-43b6-4969-8606-d1603eb2e69d.4137.1_file10M.dat' -> pg 6.f8a5403a (6.2) -> up ([1], p1) acting ([1], p1)
```


- 查看osd所在主机
```
[root@node my-cluster]# ceph osd find osd.1
{
    "osd": 1,
    "ip": "10.0.2.15:6805/8485",
    "osd_fsid": "af42a7f3-9d52-434a-9249-de98d727919b",
    "crush_location": {
        "host": "node",
        "root": "default"
    }
}
```

- osd存储路径，定位首部对象
```
[root@node current]# cd /var/lib/ceph/osd/ceph-1/current/6.2_head/
[root@node 6.2_head]# ls -lh|grep file10
-rw-r--r--. 1 ceph ceph 4.0M 4月   8 17:49 33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
```
- 查看首部对象的扩展属性列表
```
[root@node 6.2_head]# attr -l ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\\ufile10M.dat__head_F8A5403A__6 
Attribute "selinux" has a 36 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "cephos.spill_out" has a 2 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._" has a 250 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._@1" has a 60 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.acl" has a 117 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.content_type" has a 25 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.etag" has a 32 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.idtag" has a 44 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.manifest" has a 250 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.manifest@1" has a 60 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.pg_ver" has a 8 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.source_zone" has a 4 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.tail_tag" has a 44 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
Attribute "ceph._user.rgw.x-amz-content-sha256" has a 65 byte value for ./33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\ufile10M.dat__head_F8A5403A__6
```
- 其中几个属性
```
“ceph.”：对象信息
“ceph._user.rgw.acl”：对象ACL
“ceph._user.rgw.manifest”：对象rgw相关信息，包含分片信息。
```

- 通过首部对象的扩展属性“ceph._user.rgw.manifest”确定其它分片前缀信息
```
[root@node 6.2_head]# attr -q -g ceph._user.rgw.manifest 33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\\ufile10M.dat__head_F8A5403A__6 > /tmp/file10M._user.rgw.manifest.attr
[root@node 6.2_head]# attr -q -g ceph._user.rgw.manifest@1 33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\\ufile10M.dat__head_F8A5403A__6 >> /tmp/file10M._user.rgw.manifest.attr
[root@node 6.2_head]# ceph-dencoder type RGWObjManifest import /tmp/file10M._user.rgw.manifest.attr decode dump_json | grep prefix
    "prefix": ".mC4tn648n3JWaaazuNpmmJ_7vPY8cTg_",
                "override_prefix": ""
        "cur_override_prefix": "",
        "cur_override_prefix": "",
```

- 通过prefix查询除了首部对象外的其它分片对象的存储位置
```
[root@node 6.2_head]# rados -p default.rgw.buckets.data ls | grep mC4tn64
33a99195-43b6-4969-8606-d1603eb2e69d.4137.1__shadow_.mC4tn648n3JWaaazuNpmmJ_7vPY8cTg_2
33a99195-43b6-4969-8606-d1603eb2e69d.4137.1__shadow_.mC4tn648n3JWaaazuNpmmJ_7vPY8cTg_1
[root@node 6.2_head]# 
```
- 查询第分片存储位置
```
[root@node 6.2_head]# ceph osd map default.rgw.buckets.data 33a99195-43b6-4969-8606-d1603eb2e69d.4137.1__shadow_.mC4tn648n3JWaaazuNpmmJ_7vPY8cTg_2
osdmap e30 pool 'default.rgw.buckets.data' (6) object '33a99195-43b6-4969-8606-d1603eb2e69d.4137.1__shadow_.mC4tn648n3JWaaazuNpmmJ_7vPY8cTg_2' -> pg 6.9ee8e9f9 (6.1) -> up ([1], p1) acting ([1], p1)

[root@node ceph-1]# ceph osd map default.rgw.buckets.data 33a99195-43b6-4969-8606-d1603eb2e69d.4137.1__shadow_.mC4tn648n3JWaaazuNpmmJ_7vPY8cTg_1
osdmap e30 pool 'default.rgw.buckets.data' (6) object '33a99195-43b6-4969-8606-d1603eb2e69d.4137.1__shadow_.mC4tn648n3JWaaazuNpmmJ_7vPY8cTg_1' -> pg 6.666f6925 (6.5) -> up ([0], p0) acting ([0], p0)
```

- 将首部与各个分片拷贝至同一个目录，按照顺序合并，并与原文件md5对比
```
[root@node my-cluster]# cat 33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\\ufile10M.dat__head_F8A5403A__6 > file10M.dat.manual
[root@node my-cluster]# cat 33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\\u\\ushadow\\u.mC4tn648n3JWaaazuNpmmJ\\u7vPY8cTg\\u1__head_666F6925__6 >> file10M.dat.manual
[root@node my-cluster]# cat 33a99195-43b6-4969-8606-d1603eb2e69d.4137.1\\u\\ushadow\\u.mC4tn648n3JWaaazuNpmmJ\\u7vPY8cTg\\u2__head_9EE8E9F9__6 >> file10M.dat.manual
[root@node my-cluster]# md5sum file10M.dat*
311175294563b07db7ea80dee2e5b3c6  file10M.dat
311175294563b07db7ea80dee2e5b3c6  file10M.dat.manual
```

- 从上面实践结果可以看到，我们手动构建的文件MD5和原文件一致。

### 分片上传
![](https://note.youdao.com/yws/api/personal/file/8CBD4DE4881E45D2A327FEB00FF7431B?method=download&shareKey=3eaa915cd200cfa03b13284e920904a1)

- 分段上传文件
```
[root@node2 my-cluster]# s3cmd put file30M.dat s3://test-bucket
upload: 'file30M.dat' -> 's3://test-bucket/file30M.dat'  [part 1 of 2, 15MB] [1 of 1] 15728640 of 15728640   100% in    0s    82.13 MB/s  done
upload: 'file30M.dat' -> 's3://test-bucket/file30M.dat'  [part 2 of 2, 13MB] [1 of 1] 14271360 of 14271360   100% in    0s    80.49 MB/s  done
```
- 查找首部对象名
```
[root@node2 my-cluster]# radosgw-admin bucket stats --bucket=test-bucket|grep marker    
    "marker": "b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1",
    "max_marker": "0#",
[root@node my-cluster]# rados -p default.rgw.buckets.data ls|grep b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1|grep file30M
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~2TXo985bMP-7pLrAAnSAQYaLCxwaLPJ.1_2
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~3VmS03To4n4MKLC-6GIZQTp9TnUf29E.1_2
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1_file30M.dat
```
- 查找首部对象存储位置
```
[root@node2 my-cluster]# ceph osd map default.rgw.buckets.data b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1_file30M.dat
osdmap e35 pool 'default.rgw.buckets.data' (7) object 'b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1_file30M.dat' -> pg 7.c48c7e68 (7.0) -> up ([1], p1) acting ([1], p1)
```

- 查看osd所在主机
```
[root@node2 my-cluster]# ceph osd find osd.1
{
    "osd": 1,
    "ip": "10.0.2.15:6805/20940",
    "osd_fsid": "49c471ea-a939-4132-a3b5-9b75eb5e3d57",
    "crush_location": {
        "host": "node2",
        "root": "default"
    }
}
```
- 登陆到主机，查找osd存储路径
```
[root@node my-cluster]# df -h | grep -i ceph-1
/dev/dm-4                8.0G  125M  7.9G    2% /var/lib/ceph/osd/ceph-1
[root@node my-cluster]# cd /var/lib/ceph/osd/ceph-1/current/
[root@node current]# ls -l|grep -i 7.0
drwxr-xr-x. 2 ceph ceph  376 4月   9 11:47 7.0_head
drwxr-xr-x. 2 ceph ceph    6 4月   9 11:46 7.0_TEMP
[root@node current]# cd 7.0_head/
[root@node 7.0_head]# ls -l
总用量 5748
-rw-r--r--. 1 ceph ceph       0 4月   9 11:47 b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
-rw-r--r--. 1 ceph ceph 4194304 4月   9 11:46 b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\u\ushadow\ufile30M.dat.2~2TXo985bMP-7pLrAAnSAQYaLCxwaLPJ.1\u2__head_52ADBD00__7
-rw-r--r--. 1 ceph ceph 1688448 4月   9 11:46 b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\u\ushadow\ufile30M.dat.2~2TXo985bMP-7pLrAAnSAQYaLCxwaLPJ.2\u3__head_B80CF968__7
-rw-r--r--. 1 ceph ceph       0 4月   9 11:46 __head_00000000__7
```
- 查看首部对象扩展属性
```
[root@node2 7.0_head]# attr -l ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\\ufile30M.dat__head_C48C7E68__7 
Attribute "selinux" has a 36 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "cephos.spill_out" has a 2 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._" has a 250 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._@1" has a 60 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.acl" has a 135 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.content_type" has a 25 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.etag" has a 34 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.idtag" has a 45 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.manifest" has a 250 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.manifest@1" has a 106 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.pg_ver" has a 8 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.source_zone" has a 4 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.tail_tag" has a 45 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
Attribute "ceph._user.rgw.x-amz-content-sha256" has a 65 byte value for ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\ufile30M.dat__head_C48C7E68__7
```

- 通过首部对象的扩展属性“ceph._user.rgw.manifest”确定其它分片前缀信息
```
[root@node2 7.0_head]# attr -q -g ceph._user.rgw.manifest ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\\ufile30M.dat__head_C48C7E68__7 > /tmp/file30M._user.rgw.manifest.attr
[root@node2 7.0_head]# attr -q -g ceph._user.rgw.manifest@1 ./b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1\\ufile30M.dat__head_C48C7E68__7 >> /tmp/file30M._user.rgw.manifest.attr
[root@node2 7.0_head]# ceph-dencoder type RGWObjManifest import /tmp/file30M._user.rgw.manifest.attr decode dump_json | grep prefix
    "prefix": "file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8",
                "override_prefix": ""
                "override_prefix": ""
        "cur_override_prefix": "",
        "cur_override_prefix": "",
```

- 通过prefix查询除了首部对象外的其它分片对象
```
[root@node2 7.0_head]# rados -p default.rgw.buckets.data ls | grep file30M.dat.2~X6Y9
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1_3
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1_1
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.2_2
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.2_1
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1_2
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__multipart_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.2
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.2_3
b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__multipart_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1
```

- 查询分片存储位置
```
[root@node2 7.0_head]# ceph osd map default.rgw.buckets.data b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1_3
osdmap e35 pool 'default.rgw.buckets.data' (7) object 'b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1_3' -> pg 7.2828fed8 (7.0) -> up ([1], p1) acting ([1], p1)
[root@node2 7.0_head]# ceph osd map default.rgw.buckets.data b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1_1
osdmap e35 pool 'default.rgw.buckets.data' (7) object 'b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1_1' -> pg 7.4c516104 (7.4) -> up ([0], p0) acting ([0], p0)
[root@node2 7.0_head]# ceph osd map default.rgw.buckets.data b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1_2
osdmap e35 pool 'default.rgw.buckets.data' (7) object 'b7f871f0-65f7-4459-a860-eaa9eaa63d09.4141.1__shadow_file30M.dat.2~X6Y9xXlbAca0o9Vr4h8CiC9o_K2FUd8.1_2' -> pg 7.fbcbb6f9 (7.1) -> up ([0], p0) acting ([0], p0)
......
```
- 将首部与各个分片拷贝至同一个目录，按照顺序合并，并与原文件md5对比，与整体上传一致。

总结：通过研究Ceph RGW及Rados存储原理，手动还原了用户数据在对象存储中完整分布

一个存储桶对应一个RADOS对象？
