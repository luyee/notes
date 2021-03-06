## innobackupex  备份5.6.23-72.1-log （Server A）指定表恢复至5.7.19(Server B)

Server A

```
mysql> select version();
+-----------------+
| version()       |
+-----------------+
| 5.6.23-72.1-log |
+-----------------+
1 row in set (0.01 sec)

mysql> 

mysql> show variables like 'innodb_file_per_table'; 
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |
+-----------------------+-------+
1 row in set (0.00 sec)
```
### innobackupex 备份指定数据库的表

```
innobackupex --defaults-file=/etc/my.cnf --user=root --password=root123 --no-timestamp  --tables-file=/root/oel_table.txt   /data/oel_table
```
cat oel_table.txt 

```
# cat oel_table.txt 
oel.yyhb_contract_info
oel.yyhb_outlet_info
```

### Server B

```
scp -r serverA:/data/oel_table  ./oel_table
innobackupex --apply-log --export oel_table
```

### Server B建立数据库，创建表

```
mysql> select version();
+-----------+
| version() |
+-----------+
| 5.7.19    |
+-----------+
1 row in set (0.03 sec)

mysql> 

CREATE TABLE `yyhb_outlet_info` (
  `STAT_DT` varchar(20) DEFAULT NULL COMMENT '数据日期',
  `OUTLET_CODE` varchar(100) DEFAULT NULL COMMENT '门店编号',
  `OUTLET_NAME` varchar(200) DEFAULT NULL COMMENT '门店名',
  `SERVICE_MANAGER_ID` varchar(100) DEFAULT NULL COMMENT '服务经理ID',
  `CLIENT_MANAGER_ID` varchar(100) DEFAULT NULL COMMENT '服务经理ID',
  `OUTLET_STATUS` varchar(10) DEFAULT NULL COMMENT '门店状态',
  KEY `YYHB_OUTLET_INFO_IX_1` (`STAT_DT`),
  KEY `YYHB_OUTLET_INFO_IX_2` (`STAT_DT`,`OUTLET_CODE`),
  KEY `YYHB_OUTLET_INFO_IX_3` (`OUTLET_CODE`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8  row_format=compact  COMMENT='门店表';



CREATE TABLE `yyhb_contract_info` (
  `STAT_DT` varchar(20) DEFAULT NULL COMMENT '数据日期',
  `CONTRACT_NO` varchar(50) DEFAULT NULL COMMENT '合同号',
  `OUTLET_CODE` varchar(100) DEFAULT NULL COMMENT '门店号',
  `SERVICE_MANAGER_ID` varchar(100) DEFAULT NULL COMMENT '当前服务经理ID',
  `CLIENT_MANAGER_ID` varchar(100) DEFAULT NULL COMMENT '当前客户经理ID',
  `PER_SERVICE_MANAGER_ID` varchar(100) DEFAULT NULL COMMENT '业绩服务经理ID',
  `PER_CLIENT_MANAGER_ID` varchar(100) DEFAULT NULL COMMENT '业绩客户经理ID',
  `APPLY_DATE` varchar(20) DEFAULT NULL COMMENT '申请日期',
  `ACTIVED_DATE` varchar(20) DEFAULT NULL COMMENT '激活日期',
  `APPLY_FLAG` int(3) DEFAULT NULL COMMENT '申请标志',
  `APPROVED_FLAG` int(3) DEFAULT NULL COMMENT '通过标志',
  `ACTIVED_FLAG` int(3) DEFAULT NULL COMMENT '激活标志',
  `PREPAY_FLAG` int(3) DEFAULT NULL COMMENT '是否提前还款',
  `PROMOTE_FLAG` int(3) DEFAULT NULL COMMENT '是否促销',
  `COMMON_FLAG` int(3) DEFAULT NULL COMMENT '是否普通产品',
  `INSUR_FLAG` int(3) DEFAULT NULL COMMENT '是否包含保险服务包',
  `VIP_FLAG` int(3) DEFAULT NULL COMMENT '是否包含VIP服务包',
  `FSTPD30_FLAG` int(3) DEFAULT NULL COMMENT 'FSTPD30标志',
  `FPD10_FLAG` int(3) DEFAULT NULL COMMENT 'FPD10标志',
  `FPD30_FLAG` int(3) DEFAULT NULL COMMENT 'FPD30标志',
  `CUSTOMER_NAME` varchar(100) DEFAULT NULL COMMENT '姓名',
  `CUSTOMER_PHONE` varchar(30) DEFAULT NULL COMMENT '手机号',
  `ACTIVED_AMOUNT` decimal(24,2) DEFAULT NULL COMMENT '贷款金额',
  `OVERDUEDAYS` int(11) DEFAULT NULL COMMENT '逾期天数',
  `MFCUSTOMERID` varchar(40) DEFAULT NULL COMMENT '客户号（客户唯一标识）',
  `APPLY_AFTER_FLAG` int(3) DEFAULT NULL COMMENT '面审通过后申请标志',
  KEY `YYHB_CONTRACT_INFO_IX_1` (`STAT_DT`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 row_format=compact  COMMENT='合同明细表';
```

### Server B DISCARD TABLESPACE

```
 ALTER TABLE yyhb_contract_info DISCARD TABLESPACE;
 ALTER TABLE yyhb_outlet_info DISCARD TABLESPACE;
```

### Server B拷贝需要的文件到指定的数据库目录
 
 ```
 cp oel_table/oel/*.cfg /var/lib/mysql/oel/
 cp oel_table/oel/*.ibd  /var/lib/mysql/oel/
 ```
 
 ### Server B IMPORT TABLESPACE
 
 ```
 ALTER TABLE yyhb_contract_info  IMPORT TABLESPACE;
 ALTER TABLE yyhb_outlet_info  IMPORT TABLESPACE;
 ```

###  注意事项

1. 拷贝过去的*.idb,*.cfg文件的目录的权限   chown -R mysql:mysql   chown -R mysql:mysql  /var/lib/mysql/oel/
 
2.这里注意：  在目标机器create创建表时必须设置row_format=compact否则IMPORT TABLESPACE 会报错
 
 ```
 alter table yyhb_contract_info IMPORT TABLESPACE
 ERROR 1808 (HY000): Schema mismatch (Table flags don't match, server table has 0x5 and the meta-data file has 0x1)
 or
 ERROR 1808 (HY000): Schema mismatch (Table has ROW_TYPE_DYNAMIC row format, .ibd file has ROW_TYPE_COMPACT row format.)
 ```
 
 [Restoring Individual Tables](https://www.percona.com/doc/percona-xtrabackup/2.2/innobackupex/restoring_individual_tables_ibk.html)   
 [Transporting tablespace from MySQL 5.6 to MySQL 5.7 (case study)](https://www.percona.com/blog/2015/12/01/how-to-transport-tablespace-from-mysql-5-6-to-mysql-5-7/)   
 [mysql5.6新功能transportable tablespaces（可传输表空间）进行远程备份数据库](http://blog.csdn.net/xiaoyi23000/article/details/53150776)   
 [Partial backup 备份指定表/库](http://blog.csdn.net/ashic/article/details/52280967)   
 [MySQL5.6下使用xtrabackup部分备份恢复到MySQL5.7](http://www.cnblogs.com/Bccd/p/5939729.html)   
 [Percona Xtrabackup 2.4 恢复指定表](http://m.blog.itpub.net/26506993/viewspace-2123281/)
