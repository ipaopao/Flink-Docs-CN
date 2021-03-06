# Connectors

## 从文件系统中读取

Flink内置支持以下文件系统：

| 文件系统 | 方案 | 描述 |
| :--- | :--- | :--- |
| Hadoop分布式文件系统（HDFS） | `hdfs://` | 支持所有HDFS版本 |
| 亚马逊S3 | `s3://` | 通过Hadoop文件系统实现支持（见下文） |
| MapR文件系统 | `maprfs://` | 用户必须手动将所需的jar文件放在`lib/`目录中 |
| Alluxio | `alluxio://` | 通过Hadoop文件系统实现支持（见下文） |

## 使用Hadoop文件系统实现

Apache Flink允许用户使用任何实现该`org.apache.hadoop.fs.FileSystem` 接口的文件系统。有Hadoop `FileSystem`实现

* [S3](https://aws.amazon.com/s3/)（已测试）
* [适用于Hadoop的Google云端存储连接器](https://cloud.google.com/hadoop/google-cloud-storage-connector)（已测试）
* [Alluxio](http://alluxio.org/)（已测试）
* [XtreemFS](http://www.xtreemfs.org/)（已测试）
* FTP通过[Hftp](http://hadoop.apache.org/docs/r1.2.1/hftp.html)（未测试）
* 还有很多。

为了使用Flink的Hadoop文件系统，请确保：

* 在`flink-conf.yaml`已设定的`fs.hdfs.hadoopconf`属性将Hadoop配置目录。对于自动测试或从IDE运行，`flink-conf.yaml`可以通过定义`FLINK_CONF_DIR`环境变量来设置包含的目录。
* Hadoop配置（在该目录中）在文件中具有所需文件系统的条目`core-site.xml`。S3和Alluxio的示例链接/显示如下。
* `lib/`Flink安装的文件夹中提供了使用文件系统所需的类（在运行Flink的所有计算机上）。如果无法将文件放入目录，Flink还会尊重`HADOOP_CLASSPATH`环境变量以将Hadoop jar文件添加到类路径中。

### **亚马逊S3**

请参阅[部署和操作 - 部署 - AWS - S3：简单存储服务，](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/deployment/aws.html)了解可用的S3文件系统实现，其配置和所需的库。

### **Alluxio**

对于Alluxio支持，将以下条目添加到`core-site.xml`文件中：

```markup
<property>
  <name>fs.alluxio.impl</name>
  <value>alluxio.hadoop.FileSystem</value>
</property>
```

## 使用Hadoop的Input / OutputFormat包装器连接到其他系统

Apache Flink允许用户访问许多不同的系统作为数据源或接收器。该系统的设计非常容易扩展。与Apache Hadoop类似，Flink具有所谓的`InputFormat`s和`OutputFormat`s 的概念。

这些`InputFormat`的一个实现是`HadoopInputFormat`。这是一个包装器，允许用户使用Flink的所有现有Hadoop输入格式。

本节介绍将Flink连接到其他系统的一些示例。 [阅读有关Flink中Hadoop兼容性的更多信息](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/batch/hadoop_compatibility.html)。

## Avro对Flink的支持

Flink对[Apache Avro](http://avro.apache.org/)提供了广泛的内置支持。这允许使用Flink轻松读取Avro文件。此外，Flink的序列化框架能够处理从Avro架构生成的类。确保将Flink Avro依赖项包含在项目的pom.xml中。

```markup
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro</artifactId>
  <version>1.7.1</version>
</dependency>
```

要从Avro文件中读取数据，您必须指定一个`AvroInputFormat`。

**示例**：

```java
AvroInputFormat<User> users = new AvroInputFormat<User>(in, User.class);
DataSet<User> usersDS = env.createInput(users);
```

请注意，这`User`是Avro生成的POJO。Flink还允许执行这些POJO的基于字符串的**Key**选择。例如：

```java
usersDS.groupBy("name")
```

请注意，使用Flink可以使用genericData.record类型，但不推荐使用。因为记录包含完整的模式，所以它的数据非常密集，因此使用起来可能很慢。

Flink的POJO字段选择也适用于Avro生成的POJO。但是，只有在将字段类型正确写入生成的类时才可以使用。如果字段是类型`Object`，则不能将该字段用作连接或分组键。像这样在Avro中指定一个字段`{"name": "type_double_test", "type": "double"},`可以正常工作，但是将其指定为只有一个字段（`{"name": "type_double_test", "type": ["double"]},`）的UNION类型将生成一个类型的字段`Object`。请注意，指定可空类型（`{"name": "type_double_test", "type": ["null", "double"]},`）是可能的！

### 访问Microsoft Azure表存储

_注意：此示例从Flink 0.6-incubate以上可用_

此示例使用`HadoopInputFormat`包装器使用现有的Hadoop输入格式实现来访问[Azure的表存储](https://azure.microsoft.com/en-us/documentation/articles/storage-introduction/)。

1. 下载并编译`azure-tables-hadoop`项目。项目开发的输入格式尚未在Maven Central中提供，因此，我们必须自己构建项目。执行以下命令：

```text
   git clone https://github.com/mooso/azure-tables-hadoop.git
   cd azure-tables-hadoop
   mvn clean install
```

1. 使用快速入门设置新的Flink项目：

```text
   curl https://flink.apache.org/q/quickstart.sh | bash
```

1. 将以下依赖项（在本`<dependencies>`节中）添加到您的`pom.xml`文件中：

```text
   <dependency>
       <groupId>org.apache.flink</groupId>
       <artifactId>flink-hadoop-compatibility_2.11</artifactId>
       <version>1.7.1</version>
   </dependency>
   <dependency>
     <groupId>com.microsoft.hadoop</groupId>
     <artifactId>microsoft-hadoop-azure</artifactId>
     <version>0.0.4</version>
   </dependency>
```

`flink-hadoop-compatibility`是一个Flink包，提供Hadoop输入格式包装器。 `microsoft-hadoop-azure`将我们之前构建的项目添加到项目中。

该项目现在准备开始编码。我们建议将项目导入IDE，例如Eclipse或IntelliJ。（作为Maven项目导入！）。浏览到该`Job.java`文件的代码。它是Flink工作的空骨架。

将以下代码粘贴到其中：

```java
import java.util.Map;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.hadoopcompatibility.mapreduce.HadoopInputFormat;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import com.microsoft.hadoop.azure.AzureTableConfiguration;
import com.microsoft.hadoop.azure.AzureTableInputFormat;
import com.microsoft.hadoop.azure.WritableEntity;
import com.microsoft.windowsazure.storage.table.EntityProperty;

public class AzureTableExample {

  public static void main(String[] args) throws Exception {
    // set up the execution environment
    final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

    // create a  AzureTableInputFormat, using a Hadoop input format wrapper
    HadoopInputFormat<Text, WritableEntity> hdIf = new HadoopInputFormat<Text, WritableEntity>(new AzureTableInputFormat(), Text.class, WritableEntity.class, new Job());

    // set the Account URI, something like: https://apacheflink.table.core.windows.net
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.ACCOUNT_URI.getKey(), "TODO");
    // set the secret storage key here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.STORAGE_KEY.getKey(), "TODO");
    // set the table name here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.TABLE_NAME.getKey(), "TODO");

    DataSet<Tuple2<Text, WritableEntity>> input = env.createInput(hdIf);
    // a little example how to use the data in a mapper.
    DataSet<String> fin = input.map(new MapFunction<Tuple2<Text,WritableEntity>, String>() {
      @Override
      public String map(Tuple2<Text, WritableEntity> arg0) throws Exception {
        System.err.println("--------------------------------\nKey = "+arg0.f0);
        WritableEntity we = arg0.f1;

        for(Map.Entry<String, EntityProperty> prop : we.getProperties().entrySet()) {
          System.err.println("key="+prop.getKey() + " ; value (asString)="+prop.getValue().getValueAsString());
        }

        return arg0.f0.toString();
      }
    });

    // emit result (this works only locally)
    fin.print();

    // execute program
    env.execute("Azure Example");
  }
}
```

该示例显示了如何访问Azure表并将数据转换为Flink `DataSet`（更具体地说，是集的类型`DataSet<Tuple2<Text, WritableEntity>>`）。使用`DataSet`，您可以将所有已知的转换应用于DataSet。

## 访问MongoDB

这个[GitHub存储库记录了如何将MongoDB与Apache Flink一起使用（从0.7-incubating开始）](https://github.com/okkam-it/flink-mongodb-test)。

