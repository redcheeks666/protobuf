# ProtoBuf

## quick start

当前版本proto 3.17.3

安装proto代码生成工具

***https://github.com/protocolbuffers/protobuf/releases/tag/v3.17.3***

下载安装完成后

定义一个proto文件

```protobuf
//如果是proto3以上版本 此处必须声明 否则默认按v2版本规范
syntax = "proto3";

message PersonProto {

  int32 id = 1;

  string name = 2;
}
```

进入protobuf安装目录,命令行生成对应语言模型类代码

```shell
#proto文件夹路径不包含文件名且与文件名之间保留一个空格 否则无法找到文件 此处巨坑 
#proto文件名不要和message名相同
#各语言命令参数名略有差异
protoc --java_out=${生成对应代码的文件路径}  --proto_path=${proto的文件夹路径}   person.proto
```

使用生成模型类中的parseFrom(input) 即可将接收到的二进制数据解析为对应的模型类对象。

相关依赖

```xml
<dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>3.17.3</version>
</dependency>
```

```java
Person.PersonProto value = Person.PersonProto.parseFrom(input)
```

### 集成Flink Dstream

以消费kafka数据为例，首先自定义反序列化的模型类。

```java
@PublicEvolving
public class ProtoDeserialization implements KafkaDeserializationSchema<byte[]> {
    private static final long serialVersionUID = 1509391548173891955L;
    private final static Logger log = LoggerFactory.getLogger(ProtoDeserialization2.class);
    private final boolean includeMetadata;
    public ProtoDeserialization(boolean includeMetadata) {
        this.includeMetadata = includeMetadata;
    }
    //反序列化方法,将二进制数据原封不动返回
    public byte[] deserialize(ConsumerRecord<byte[], byte[]> record) {
        return record.value();
    }
    public boolean isEndOfStream(byte[] nextElement) {
        return false;
    }
    public TypeInformation<byte[]> getProducedType() {
        return TypeExtractor.getForClass(byte[].class);
    }
}
```

之后在下游进行解析数据

```scala
stream.map(x=>CreateUserReq.parseFrom(x))
```

注意，在解析之前，需要在flink环境中注册相关信息。

**https://flink.apache.org/news/2020/04/15/flink-serialization-tuning-vol-1.html#protobuf-via-kryo**

需要导入依赖

```xml
<dependency>
            <groupId>com.twitter</groupId>
            <artifactId>chill-protobuf</artifactId>
            <version>0.7.6</version>
            <!-- exclusions for dependency conversion -->
            <exclusions>
                <exclusion>
                    <groupId>com.esotericsoftware.kryo</groupId>
                    <artifactId>kryo</artifactId>
                </exclusion>
            </exclusions>
</dependency>
```

```scala
//注册相关模型类
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.getConfig.registerTypeWithKryoSerializer(classOf[CreateUserReq],classOf[ProtobufSerializer]);
```

### 集成Flink Sql

### Flink Sql中使用protobuf协议的解决方案

#### 方案1

在kafka连接器中,新增protobuf的format的类型。通过指定对应的protobuf的模型类。直接将二进制数据解析为对应字段。

此方案社区已经有实现，暂时未开放，据说计划在1.12中上线，但目前1.13版本依旧未提供。预计1.14会开放。

目前已有的实现(可供参考) **https://github.com/apache/flink/pull/14376**

相关讨论 **https://issues.apache.org/jira/browse/FLINK-18202?filter=-4&jql=project%20%3D%20FLINK%20AND%20issuetype%20%3D%20%22New%20Feature%22%20AND%20text%20~%20protobuf%20order%20by%20created%20DESC**

#### 方案2

通过目前kafka提供的VARBINARY数据类型，将二进制数据进行消费, 下游通过udtf(表值函数)方式进行解析处理，返回对应各个字段。

```sql
CREATE TEMPORARY TABLE test_proto (
    log VARBINARY
  )
WITH (
    'connector' = 'kafka',
    'topic' = 't_topic',
    'properties.bootstrap.servers' = 'xxx.xxx.x.xxx:9092',
    'properties.group.id' = 'g_01',
    'properties.auto.offset.reset' = 'earliest',
    'scan.startup.mode' = 'earliest-offset',
    'value.format' = 'raw'
);
select user_id,labels,properties  from  test_proto,lateral table(PROTO_USER(log,'reg')) as T(user_id,labels,properties);
```

udtf示例如下

```java
public class PROTO_USER extends TableFunction<Tuple3<String,String,String>> {

    public void  eval(byte[] bts,String type) throws InvalidProtocolBufferException {
            if (type=="reg"){
                CreateUserReq createUserReq = CreateUserReq.parseFrom(bts);
                String userId = createUserReq.getUserId();
                Map<String, String> pmap = createUserReq.getPropertiesMap();
                Map<String, Integer> lmap = createUserReq.getLabelsMap();

                String labels=null;
                String properties=null;
                if (lmap!=null) labels = JSONObject.toJSON(lmap).toString();
                if (pmap!=null) properties = JSONObject.toJSON(pmap).toString();

                collect(Tuple3.of(userId,labels,properties));
            }
}
```
































