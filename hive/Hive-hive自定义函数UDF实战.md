# Hive-hive自定义函数UDF实战

## 一、概述
    
很多时候hive自带的函数和方法无法满足用户一些特定的需求，有的特定需求无法使用hql语句直接实现，所以hive提供了接口，以供用户自定义某些特殊的需求。编写成的函数则是UDF(用户自定义函数)。


---

## 二、UDF类型
**udf分为三种类型。**
- UDF：操作单个数据行，产生单个数据行;
- UDAF：操作多个数据行，产生一个数据行;
- UDTF：操作一个数据行，产生多个数据行一个表作为输出;

---

## 三、编写自定义UDF

### 3.1  添加pom依赖
    
    <dependency>
    	<groupId>org.apache.hive</groupId>
    	<artifactId>hive-exec</artifactId>
    	<version>1.2.1</version>
	</dependency>
	<dependency>
    	<groupId>org.apache.hadoop</groupId>
    	<artifactId>hadoop-common</artifactId>
    	<version>2.7.3</version>
	</dependency>

### 3.2 编写udf

```java
//继承UDF类,并重写evaluate方法
public class StockTestUDF extends UDF {
    //输入参数为hive表某个字段，返回为自定义数据类型
    /*
    *输入股票代码，返回每只股票所属板块。
    */
    public String evaluate(String str){
        if(str.length()==0 || str == null) return "代码错误";
        String word = str.substring(0,1);
        switch(word){
            case "0":
                return "深证中小板";
            case "3":
                return "深证创业板";
            case "6":
                return "上证主板";
            default:
                return "代码错误";
        }
    }
}
```

### 3.3 打包成jar包并上传至服务器，添加到HIVE
    --指定jar包的位置
    hive> add jar /opt/chenyh/jar/HiveUDF.jar
    
    --创建hive临时函数，指定函数名字和函数入口
    hive>create temporary function getStockType as 'hadoop.cyril.StockTestUDF'

### 3.4 Hive下调用

```hive
hive> select stockcode,stockname,getstocktype(stockcode) from dw.dw_stock_buss_info_d where part_month='201911' and part_day='19' limit 10;
OK
600891  *ST秋林 上证主板
603328  依顿电子        上证主板
600671  天目药业        上证主板
600446  金证股份        上证主板
600036  招商银行        上证主板
600702  舍得酒业        上证主板
600746  江苏索普        上证主板
600793  宜宾纸业        上证主板
600455  博通股份        上证主板
600682  南京新百        上证主板
Time taken: 0.167 seconds, Fetched: 10 row(s)
```

4