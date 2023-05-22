# SSB性能测试

## 1 测试方法和结论

Star schema benchmark（以下简称SSB）是学术界和工业界广泛使用的一个星型模型测试集，通过这个测试集合可以方便的对比各种OLAP产品的基础性能指标。本报告记录了Cloudwave、StarRocks在SSB多表数据集上进行了性能对比测试的结果。测试结论如下：

在多表测试的13个查询中，Cloudwave平均比Starrocks快0.31倍。

以下是具体测试结果：

![1](ssb%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E5%9B%BE%E7%89%87/1.png)   

## 2 测试准备

### 2.1 硬件环境

| 机器     | 4台  阿里云主机                                              |
| -------- | ------------------------------------------------------------ |
| CPU      | 64core    Intel(R)  Xeon(R) Platinum 8269CY CPU @ 2.50GHz   cachesize : 36608 KB |
| 内存     | 256GB                                                        |
| 网络带宽 | 10Gbits/s                                                    |
| 磁盘     | ESSD pl1高效云盘                                             |

### 2.2 软件环境

| 内核版本     | Linux 3.10.0-957.21.3.el7.x86_64 |
| ------------ | -------------------------------- |
| 操作系统版本 | CentOS Linux release 7.6.1810    |
| 软件版本     | Cloudwave 4.0，StarRocks 3.0     |

## 3 测试数据与结果

### 3.1 测试数据

| 表名      | 行数          | 解释          |
| --------- | ------------- | ------------- |
| lineorder | 5,999,989,709 | SSB商品订单表 |
| customer  | 30,000,000    | SSB客户表     |
| part      | 2,000,000     | SSB 零部件表  |
| supplier  | 2,000,000     | SSB 供应商表  |
| dates     | 2,556         | 日期表        |

### 3.2 测试SQL

| SQL编号 | SQL语句                                                      |
| ------- | ------------------------------------------------------------ |
| Q1.1    | select sum(lo_revenue) as revenue from lineorder,dates  where lo_orderdate = d_datekey and d_year = 1993 and lo_discount between 1  and 3 and lo_quantity < 25; |
| Q1.2    | select sum(lo_revenue) as revenue from  lineorder,dates where lo_orderdate = d_datekey and d_yearmonthnum = 199401  and lo_discount between 4 and 6 and lo_quantity between 26 and 35; |
| Q1.3    | select sum(lo_revenue) as revenue from  lineorder,dates where lo_orderdate = d_datekey and d_weeknuminyear = 6 and  d_year = 1994 and lo_discount between 5 and 7 and lo_quantity between 26 and  35; |
| Q2.1    | select sum(lo_revenue) as lo_revenue,  d_year, p_brand from lineorder ,dates,part,supplier where lo_orderdate =  d_datekey and lo_partkey = p_partkey and lo_suppkey = s_suppkey and  p_category = 'MFGR#12' and s_region = 'AMERICA' group by d_year, p_brand  order by d_year, p_brand; |
| Q2.2    | select sum(lo_revenue) as lo_revenue,  d_year, p_brand from lineorder,dates,part,supplier where lo_orderdate =  d_datekey and lo_partkey = p_partkey and lo_suppkey = s_suppkey and p_brand  between 'MFGR#2221' and 'MFGR#2228' and s_region = 'ASIA' group by d_year,  p_brand order by d_year, p_brand; |
| Q2.3    | select sum(lo_revenue) as lo_revenue,  d_year, p_brand from lineorder,dates,part,supplier where lo_orderdate =  d_datekey and lo_partkey = p_partkey and lo_suppkey = s_suppkey and p_brand =  'MFGR#2239' and s_region = 'EUROPE' group by d_year, p_brand order by d_year,  p_brand; |
| Q3.1    | select c_nation, s_nation, d_year,  sum(lo_revenue) as lo_revenue from lineorder,dates,customer,supplier where  lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey =  s_suppkey and c_region = 'ASIA' and s_region = 'ASIA'and d_year >= 1992  and d_year <= 1997 group by c_nation, s_nation, d_year order by d_year  asc, lo_revenue desc; |
| Q3.2    | select c_city, s_city, d_year,  sum(lo_revenue) as lo_revenue from lineorder,dates,customer,supplier where  lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey = s_suppkey  and c_nation = 'UNITED STATES' and  s_nation = 'UNITED STATES' and d_year >= 1992 and d_year <= 1997 group  by c_city, s_city, d_year order by d_year asc, lo_revenue desc; |
| Q3.3    | select c_city, s_city, d_year,  sum(lo_revenue) as lo_revenue from lineorder,dates,customer,supplier where  lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey =  s_suppkey and (c_city='UNITED KI1' or c_city='UNITED KI5') and  (s_city='UNITED KI1' or s_city='UNITED KI5') and d_year >= 1992 and d_year  <= 1997 group by c_city, s_city, d_year order by d_year asc, lo_revenue  desc; |
| Q3.4    | select c_city, s_city, d_year,  sum(lo_revenue) as lo_revenue from lineorder,dates,customer,supplier where  lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey =  s_suppkey and (c_city='UNITED KI1' or c_city='UNITED KI5') and  (s_city='UNITED KI1' or s_city='UNITED KI5') and d_yearmonth = 'Dec1997' group by c_city, s_city, d_year  order by d_year asc, lo_revenue desc; |
| Q4.1    | select d_year, c_nation, sum(lo_revenue)  - sum(lo_supplycost) as profit from lineorder,dates,customer,supplier,part  where lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey =  s_suppkey and lo_partkey = p_partkey and c_region = 'AMERICA' and s_region =  'AMERICA' and (p_mfgr = 'MFGR#1' or p_mfgr = 'MFGR#2') group by d_year,  c_nation order by d_year, c_nation; |
| Q4.2    | select d_year, s_nation, p_category,  sum(lo_revenue) - sum(lo_supplycost) as profit from  lineorder,dates,customer,supplier,part where lo_orderdate = d_datekey and  lo_custkey = c_custkey and lo_suppkey = s_suppkey and lo_partkey = p_partkey  and c_region = 'AMERICA'and s_region = 'AMERICA' and (d_year = 1997 or d_year  = 1998) and (p_mfgr = 'MFGR#1' or p_mfgr = 'MFGR#2') group by d_year,  s_nation, p_category order by d_year, s_nation, p_category; |
| Q4.3    | select d_year, s_city, p_brand,  sum(lo_revenue) - sum(lo_supplycost) as profit from  lineorder,dates,customer,supplier,part where lo_orderdate = d_datekey and  lo_custkey = c_custkey and lo_suppkey = s_suppkey and lo_partkey = p_partkey  and c_region = 'AMERICA'and s_nation = 'UNITED STATES' and (d_year = 1997 or  d_year = 1998) and p_category = 'MFGR#14' group by d_year, s_city, p_brand  order by d_year, s_city, p_brand; |

### 3.3 测试结果

| SQL  | Cloudwave用时（ms） | StarRocks 用时（ms） | Cloudwave/StarRocks耗时比例 |
| ---- | ------------------- | -------------------- | --------------------------- |
| Q1.1 | 187                 | 94                   | 1.989                       |
| Q1.2 | 48                  | 85                   | 0.565                       |
| Q1.3 | 44                  | 77                   | 0.571                       |
| Q2.1 | 408                 | 1032                 | 0.395                       |
| Q2.2 | 245                 | 948                  | 0.258                       |
| Q2.3 | 207                 | 879                  | 0.235                       |
| Q3.1 | 2070                | 2024                 | 1.023                       |
| Q3.2 | 557                 | 967                  | 0.576                       |
| Q3.3 | 259                 | 838                  | 0.309                       |
| Q3.4 | 120                 | 157                  | 0.764                       |
| Q4.1 | 1505                | 2127                 | 0.708                       |
| Q4.2 | 1261                | 682                  | 1.849                       |
| Q4.3 | 967                 | 443                  | 2.183                       |

## 4 测试流程

### 4.1 数据生成

下载ssb-poc工具包并编译

```
wget https://starrocks-public.oss-cn-zhangjiakou.aliyuncs.com/ssb-poc-0.10.0.zip 
unzip ssb-poc-0.10.0.zip 
cd ssb-poc-0.10.0 
cd ssb-poc make && make install  
```

所有相关工具安装到output目录，进入output目录，生成数据。

```
cd output 
bin/gen-ssb.sh 1000 data_dir
```

### 4.2 创建数据导入配置文件

将下面的json字符串保存为 loadplan.json。

```
{
  "schema": "ssb1000",
  "sqls": [
    "CREATE TABLE CUSTOMER(C_CUSTKEY INTEGER PRIMARY KEY,C_NAME VARCHAR(25),C_ADDRESS VARCHAR(25),C_CITY VARCHAR(10),C_NATION VARCHAR(15),C_REGION VARCHAR(12),C_PHONE VARCHAR(15),C_MKTSEGMENT VARCHAR(10))",
    "CREATE TABLE DATES(D_DATEKEY INTEGER PRIMARY KEY,D_DATE VARCHAR(18),D_DAYOFWEEK VARCHAR(18),D_MONTH VARCHAR(9),D_YEAR INTEGER,D_YEARMONTHNUM INTEGER,D_YEARMONTH VARCHAR(7),D_DAYNUMINWEEK INTEGER,D_DAYNUMINMONTH INTEGER,D_DAYNUMINYEAR INTEGER,D_MONTHNUMINYEAR INTEGER,D_WEEKNUMINYEAR INTEGER,D_SELLINGSEASON VARCHAR(12),D_LASTDAYINWEEKFL INTEGER,D_LASTDAYINMONTHFL INTEGER,D_HOLIDAYFL INTEGER,D_WEEKDAYFL INTEGER)",
    "CREATE TABLE PART(P_PARTKEY INTEGER PRIMARY KEY,P_NAME VARCHAR(22) ,P_MFGR VARCHAR(6),P_CATEGORY VARCHAR(7),P_BRAND VARCHAR(9),P_COLOR VARCHAR(11),P_TYPE VARCHAR(25),P_SIZE TINYINT,P_CONTAINER VARCHAR(10))",
    "CREATE TABLE SUPPLIER(S_SUPPKEY INTEGER PRIMARY KEY,S_NAME VARCHAR(25),S_ADDRESS VARCHAR(25),S_CITY VARCHAR(10),S_NATION VARCHAR(15),S_REGION VARCHAR(12),S_PHONE VARCHAR(15))",
    "CREATE TABLE LINEORDER(LO_ORDERKEY LONG,LO_LINENUMBER TINYINT,LO_CUSTKEY INTEGER,LO_PARTKEY INTEGER,LO_SUPPKEY INTEGER,LO_ORDERDATE INTEGER,LO_ORDERPRIOTITY VARCHAR(15),LO_SHIPPRIOTITY TINYINT,LO_QUANTITY TINYINT,LO_EXTENDEDPRICE INTEGER,LO_ORDTOTALPRICE INTEGER,LO_DISCOUNT TINYINT,LO_REVENUE INTEGER,LO_SUPPLYCOST INTEGER,LO_TAX TINYINT,LO_COMMITDATE INTEGER,LO_SHIPMODE VARCHAR(10))",
    "create index lo_discount_idx on LINEORDER(LO_DISCOUNT)",
    "create index lo_quantity_idx on LINEORDER(LO_QUANTITY)",
    "create index p_category_idx on part(p_category)",
    "create index p_mfgr_idx on part(p_mfgr)",
    "create index p_brand_idx on part(p_brand)",
    "create index c_region_idx on customer(c_region)",
    "create index c_city_idx on customer(c_city)"
  ],
  "loads": [
    {"table": "customer", "file": "customer.*", "delimiter": "|"},
    {"table": "dates", "file": "dates.*", "delimiter": "|"},
    {"table": "part", "file": "part.*", "delimiter": "|"},
    {"table": "supplier", "file": "supplier.*", "delimiter": "|"},
    {"table": "lineorder", "file": "lineorder.*", "delimiter": "|"}
  ]
}
```

### 4.3 数据导入

1、在hdfs 中创建数据导入目录uploads

```
hdfs dfs -mkdir /wisdomdata/uploads
```

2、 将数据导入到hdfs

```
hdfs dfs -put /usr/local/cloudwave-ha/client/cloudwave_client/input/ssb1000 /wisdomdata/uploads/
```

说明：/usr/local/cloudwave-ha/client/cloudwave_client/input/ssb1000目录内部包含ssb1000数据文件以及loadplan.json 导入配置文件。

3、在 dbeaver 数据导入工具中，执行导入命令

```
loaddata ssb1000;
```

说明：执行完之后，可以等待执行结束，也可以关闭，数据库后台不会中止导入。可以通过select count(*) from ssb1000.lineorder 查看导入进度。

### 4.4 数据查询

在linux系统中执行下面的命令，执行并保存查询结果result.txt 文件中。

```
cat ../conf/SSB1000_SQL.txt | ./cplus.sh system/CHANGEME@cloudwave0 > result.txt 
```

说明，SSB1000_SQL.txt 的文件内容如下：

```
use schema ssb1000;
select sum(lo_revenue) as revenue from lineorder,dates where lo_orderdate = d_datekey and d_year = 1993 and lo_discount between 1 and 3 and lo_quantity < 25; 
select sum(lo_revenue) as revenue from lineorder,dates where lo_orderdate = d_datekey and d_yearmonthnum = 199401 and lo_discount between 4 and 6 and lo_quantity between 26 and 35; 
select sum(lo_revenue) as revenue from lineorder,dates where lo_orderdate = d_datekey and d_weeknuminyear = 6 and d_year = 1994 and lo_discount between 5 and 7 and lo_quantity between 26 and 35; 
select sum(lo_revenue) as lo_revenue, d_year, p_brand from lineorder ,dates,part,supplier where lo_orderdate = d_datekey and lo_partkey = p_partkey and lo_suppkey = s_suppkey and p_category = 'MFGR#12' and s_region = 'AMERICA' group by d_year, p_brand order by d_year, p_brand; 
select sum(lo_revenue) as lo_revenue, d_year, p_brand from lineorder,dates,part,supplier where lo_orderdate = d_datekey and lo_partkey = p_partkey and lo_suppkey = s_suppkey and p_brand between 'MFGR#2221' and 'MFGR#2228' and s_region = 'ASIA' group by d_year, p_brand order by d_year, p_brand; 
select sum(lo_revenue) as lo_revenue, d_year, p_brand from lineorder,dates,part,supplier where lo_orderdate = d_datekey and lo_partkey = p_partkey and lo_suppkey = s_suppkey and p_brand = 'MFGR#2239' and s_region = 'EUROPE' group by d_year, p_brand order by d_year, p_brand; 
select c_nation, s_nation, d_year, sum(lo_revenue) as lo_revenue from lineorder,dates,customer,supplier where lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey = s_suppkey and c_region = 'ASIA' and s_region = 'ASIA'and d_year >= 1992 and d_year <= 1997 group by c_nation, s_nation, d_year order by d_year asc, lo_revenue desc; 
select c_city, s_city, d_year, sum(lo_revenue) as lo_revenue from lineorder,dates,customer,supplier where lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey = s_suppkey and  c_nation = 'UNITED STATES' and s_nation = 'UNITED STATES' and d_year >= 1992 and d_year <= 1997 group by c_city, s_city, d_year order by d_year asc, lo_revenue desc; 
select c_city, s_city, d_year, sum(lo_revenue) as lo_revenue from lineorder,dates,customer,supplier where lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey = s_suppkey and (c_city='UNITED KI1' or c_city='UNITED KI5') and (s_city='UNITED KI1' or s_city='UNITED KI5') and d_year >= 1992 and d_year <= 1997 group by c_city, s_city, d_year order by d_year asc, lo_revenue desc; 
select c_city, s_city, d_year, sum(lo_revenue) as lo_revenue from lineorder,dates,customer,supplier where lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey = s_suppkey and (c_city='UNITED KI1' or c_city='UNITED KI5') and (s_city='UNITED KI1' or s_city='UNITED KI5') and d_yearmonth  = 'Dec1997' group by c_city, s_city, d_year order by d_year asc, lo_revenue desc; 
select d_year, c_nation, sum(lo_revenue) - sum(lo_supplycost) as profit from lineorder,dates,customer,supplier,part where lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey = s_suppkey and lo_partkey = p_partkey and c_region = 'AMERICA' and s_region = 'AMERICA' and (p_mfgr = 'MFGR#1' or p_mfgr = 'MFGR#2') group by d_year, c_nation order by d_year, c_nation; 
select d_year, s_nation, p_category, sum(lo_revenue) - sum(lo_supplycost) as profit from lineorder,dates,customer,supplier,part where lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey = s_suppkey and lo_partkey = p_partkey and c_region = 'AMERICA'and s_region = 'AMERICA' and (d_year = 1997 or d_year = 1998) and (p_mfgr = 'MFGR#1' or p_mfgr = 'MFGR#2') group by d_year, s_nation, p_category order by d_year, s_nation, p_category; 
select d_year, s_city, p_brand, sum(lo_revenue) - sum(lo_supplycost) as profit from lineorder,dates,customer,supplier,part where lo_orderdate = d_datekey and lo_custkey = c_custkey and lo_suppkey = s_suppkey and lo_partkey = p_partkey and c_region = 'AMERICA'and s_nation = 'UNITED STATES' and (d_year = 1997 or d_year = 1998) and p_category = 'MFGR#14' group by d_year, s_city, p_brand order by d_year, s_city, p_brand;
```