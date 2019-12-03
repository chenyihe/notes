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

**注意事项**
- 1.通过这种方法创建的临时函数只对本次会话有效，关闭此会话会失效，下次使用许重新创建。
- 2.一个UDF必须继承org.apache.hadoop.hive.ql.exec.UDF;类
- 3.一个UDF必须要包含有evaluate()方法，但是该方法并不存在于UDF中。evaluate的参数个数以及类型都是用户自定义的。在使用的时候，Hive会调用UDF的evaluate()方法。
- 4.UDF的evaluate()方法可以重载。


## 四、自定义UDAF


**两篇关于UDAF不错的帖子**

[Hive UDAF开发详解](https://blog.csdn.net/kent7306/article/details/50110067
)

[Hive UDAF开发--个人补充理解](https://blog.csdn.net/w124374860/article/details/81021474/)

[Hive ObjectInspector接口解析](https://blog.csdn.net/weixin_39469127/article/details/89739285)

UDAF是聚合函数，相当于reduce，将多行数据聚合成一行数据。
UDAF是需要在hive的sql语句和group by联合使用，hive的group by 对于每个分组，只能返回一条记录，这点和mysql不一样。

开发UDAF的步骤：
1. 继承AbstractGenericUDAFResolver抽象类，resolver负责类型检查，操作符重载，里面创建evaluator类对象；
2. 继承GenericUDAFevaluator类，重写类的方法。evaluator真正实现UDAF的逻辑；
3. add jar /opt/chenyh/jar/HiveUDAF.jar
4. create temporary function stringlen as 'hadoop.cyril.StringUDAF';
```java
package hadoop.cyril;

import org.apache.hadoop.hive.ql.exec.UDFArgumentLengthException;
import org.apache.hadoop.hive.ql.exec.UDFArgumentTypeException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.parse.SemanticException;
import org.apache.hadoop.hive.ql.udf.generic.AbstractGenericUDAFResolver;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDAFEvaluator;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
import org.apache.hadoop.hive.serde2.objectinspector.PrimitiveObjectInspector;
import org.apache.hadoop.hive.serde2.typeinfo.TypeInfo;
import org.apache.hadoop.hive.serde2.typeinfo.TypeInfoUtils;


/**
 * @author ：Cyril
 * @date ：Created in 2019/12/2 20:44
 * @description： 自定义聚合函数(UDAF) ,求某一列字符串所有数据的长度和
 * @modified By：
 */


/**
 * Hive UDAF需要继承两个重要的类AbstractGenericUDAFResolver,GenericUDAFEvaluator
 */
public class StringUDAF extends AbstractGenericUDAFResolver {

    @Override
    public GenericUDAFEvaluator getEvaluator(TypeInfo[] info) throws SemanticException {
        /**
         * 判断输入参数长度，输入参数的长度即是函数参数长度
         */
        if(info.length!=1){
            throw new UDFArgumentLengthException("Only one Argument support here");
        }
        /**
         * ObjectInspector ObjectInspector实例代表了一个类型的数据在内存中存储的特定类型和方法
         */
        ObjectInspector io = TypeInfoUtils.getStandardJavaObjectInspectorFromTypeInfo(info[0]);

        /**
         * 判断是否为原始类型，如果不是原始数据类型抛出异常
         */
        if(io.getCategory()!= ObjectInspector.Category.PRIMITIVE){
            throw new UDFArgumentTypeException(0,"Only PRIMITIVE type support here");
        }
        /**
         * PrimitiveObjectInspector 实现ObjectInspector接口，此类里面枚举了所有原始数据类型
         */
        PrimitiveObjectInspector in = (PrimitiveObjectInspector) io;

        /**
         * 判断输入参数是否为原始STRING类型，如果是则返回为String类型实现的StringEvaluator实例，否则抛出异常
         */
        switch (in.getPrimitiveCategory()){
            case STRING:
                return new StringEvaluator();
            default:
                throw new UDFArgumentTypeException(0,"Only String support here,"+((PrimitiveObjectInspector)info[0]).getPrimitiveCategory()+"is passed");
        }
    }

    /**
     * 继承GenericUDAFEvaluator类，并重写其中的方法。
     */
    public static class StringEvaluator extends GenericUDAFEvaluator{

        /**
         * 定义每个阶段输入参数的类型
         */
        private PrimitiveObjectInspector input;
        private PrimitiveObjectInspector output;

        /**
         * 定义函数输出类型
         */
        private ObjectInspector allout;
        private long total;

        /**
         * 存放中间结果的类，需要实现AggregationBuffer接口
         */
        public static class StringAggregation implements AggregationBuffer{
            long sum=0;
            void add(long num){
                this.sum += num;
            }
        }

        /**
         *
         * @param m  枚举类型Mode
         *           PARTIAL1: 这个是mapreduce的map阶段:从原始数据到部分数据聚合 将会调用iterate()和terminatePartial()
         *           PARTIAL2: 这个是mapreduce的map端的Combiner阶段，负责在map端合并map的数据::从部分数据聚合到部分数据聚合: 将会调用merge() 和 terminatePartial()
         *           FINAL: mapreduce的reduce阶段:从部分数据的聚合到完全聚合 将会调用merge()和terminate()
         *           COMPLETE: 如果出现了这个阶段，表示mapreduce只有map，没有reduce，所以map端就直接出结果了:从原始数据直接到完全聚合 将会调用 iterate()和terminate()
         * @param parameters
         * @return
         * @throws HiveException
         */
        @Override
        public ObjectInspector init(Mode m, ObjectInspector[] parameters) throws HiveException {
            //判断输入参数长度是否为1
            assert (parameters.length==1);
            super.init(m, parameters);
            /**
             * 可以根据不同阶段，定义每个阶段的输入输出参数类型
             */
            if(m==Mode.PARTIAL1 || m==Mode.COMPLETE){
                //当m为Mode.PARTIAL1 或 Mode.COMPLETE 输入参数为String类型
                input = (PrimitiveObjectInspector) parameters[0];
            }else{
                //m为Mode.PARTIAL1 或 Mode.COMPLETE 之外，输入参数为Long类型
                output = (PrimitiveObjectInspector) parameters[0];
            }
            //每个阶段的输出参数都是Long类型
            allout = ObjectInspectorFactory.getReflectionObjectInspector(Long.class, ObjectInspectorFactory.ObjectInspectorOptions.JAVA);
            return allout;
        }

        /**
         * 保存结果的类
         */
        @Override
        public AggregationBuffer getNewAggregationBuffer() throws HiveException {
            StringAggregation agg  = new StringAggregation();
            agg.sum = 0;
            return agg;
        }

        /**
         * @description 重置中间结果
         * @param aggregationBuffer
         * @throws HiveException
         */
        @Override
        public void reset(AggregationBuffer aggregationBuffer) throws HiveException {
            StringAggregation agg  = (StringAggregation)aggregationBuffer;
            agg.sum=0;
        }

        /**
         * @description
         * @param aggregationBuffer 保存中间结果的类
         * @param objects 输入参数
         * @throws HiveException
         */
        @Override
        public void iterate(AggregationBuffer aggregationBuffer, Object[] objects) throws HiveException {
            assert (objects.length==1);
            /**
             * 注意此处的格式转换
             */
//            input = (StringObjectInspector) objects[0];
            Object obj = input.getPrimitiveJavaObject(objects[0]);
            StringAggregation agg = (StringAggregation) aggregationBuffer;
            agg.add(String.valueOf(obj).length());
        }

        @Override
        public Object terminatePartial(AggregationBuffer aggregationBuffer) throws HiveException {
            StringAggregation myagg = (StringAggregation) aggregationBuffer;
            total += myagg.sum;
            return total;
        }

        /**
         * 合并中间结果
         * @param aggregationBuffer
         * @param o
         * @throws HiveException
         */
        @Override
        public void merge(AggregationBuffer aggregationBuffer, Object o) throws HiveException {
            if(o!=null){
                long len = (Long) output.getPrimitiveJavaObject(o);
                StringAggregation agg = (StringAggregation) aggregationBuffer;
                StringAggregation agg1 = new StringAggregation();
                agg1.add(len);
                agg.add(agg1.sum);
            }
        }

        /**
         * 输出最终结果
         * @param aggregationBuffer
         * @return
         * @throws HiveException
         */
        @Override
        public Object terminate(AggregationBuffer aggregationBuffer) throws HiveException {
            StringAggregation agg = (StringAggregation) aggregationBuffer;
            total += agg.sum;
            return total;
        }
    }

}



```
