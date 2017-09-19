
TBLPROPERTIES 设置

```
CREATE TABLE various_encodings
(
  id BIGINT PRIMARY KEY,
  c1 BIGINT ENCODING PLAIN_ENCODING,
  c2 BIGINT ENCODING AUTO_ENCODING,
  c3 TINYINT ENCODING BIT_SHUFFLE,
  c4 DOUBLE ENCODING BIT_SHUFFLE,
  c5 BOOLEAN ENCODING RLE,
  c6 STRING ENCODING DICT_ENCODING,
  c7 STRING ENCODING PREFIX_ENCODING
) PARTITION BY HASH(id) PARTITIONS 2 STORED AS KUDU TBLPROPERTIES ('kudu.num_tablet_replicas' = '1');
```

 [Managing Kudu](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_mc_kudu_service.html#impala_dependency   
 [Using Impala to Query Kudu Tables](https://www.cloudera.com/documentation/enterprise/5-11-x/topics/impala_kudu.html#kudu_benefits)
 [Installing Kudu](https://www.cloudera.com/documentation/enterprise/5-11-x/topics/kudu_install_cm.html)   
 [Install Kudu Using Parcels-5-11-x](https://www.cloudera.com/documentation/enterprise/5-11-x/topics/kudu_install_cm.html#install_parcels)