---
layout: post
title: Why I use Arch Linux
---


# Loding and Saving Your Data

主要介绍Spark对于下面3类数据源的处理：

1. File formats and filesystems
2. Structured data sources through Spark SQL
3. Databases and key/value stores

## File Formats

Spark可以很容易地加载存储很多文件格式，从非结构化，半结构化到结构化的文件格式：

- Text files
- JSON
- CSV
- SequenceFiles
- Protocal buffers
- Object files

除了Spark内置的输出机制，我们也可以使用Hadoop的keyed data的API。

### Text Files

当我们加载一份文本文件作为一个RDD，那么每一行将会成为RDD的一个元素。我们也可以同时加载很多文本文件到pair RDD中，其中key是文件名，value是文件的内容。

#### Loading text files

```scala
val input = sc.textFile("file:///home/holden/repos/spark/README.md")
```

如果需要将一个文件夹中的多份文件加载，那么可以使用textFile，参数是文件夹的路径，有时候我们也需要知道输入中的一份文件的信息，或者使用整份文件，那么可以使用`SparkContext.wholeTextFiles()`,将会得到pair RDD，key就是文件名。

```scala
val input = sc.wholeTextFiles("file:///home/holden/salesFiles")
val result = input.mapValues{y => 
  val nums = y.split(" ").map(x => x.toDouble)
  nums.sum / nums.size.toDouble
}
```

#### Saving text files

```scala
result.saveAsTextFile(outputFile)
```

`outputFile`是文件夹的路径，Spark将会在这个文件夹下写入文本文件。

### JSON

JSON是很流行的半结构数据格式。最简单的加载JSON数据是作为文本文件加载，然后使用JSON parser来映射数据。

#### Loading JSON

上面最简单的方式要求是你的JSON文件每一行是一个记录，如果你有multiline JSON files，那么需要加载整份文件并且parse每一份文件。可以使用mapPartitions()重用parser。

在scala或java中，通常将记录加载到一个类中来表示它们的schemas。

```
import com.fasterxml.jackson.module.scala.DefaultScalaModule
import com.fasterxml.jackson.module.scala.experimental.ScalaObjectMapper
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.DeserializationFeature
...
case class Person(name: String, lovesPandas: Boolean) // Must be a top-level class
...
// Parse it into a specific case class. We use flatMap to handle errors
// by returning an empty list (None) if we encounter an issue and a
// list with one element if everything is ok (Some(_)).
val result = input.flatMap(record => {
  try {
    Some(mapper.readValue(record, classOf[Person]))
  } catch {
    case e: Exception => None
}})
```

#### Saving JSON

我们可以使用将String的RDD转换为parsed JSON数据的库，将结构数据的RDD转换为string的RDD，然后就能使用Spark的文本文件的API。

```scala
result.filter(p => P.lovesPandas).map(mapper.writeValueAsString(_)).saveAsTextFile(outputFile)
```

### SCV and TSV

#### Loading CSV

对于Scala和Java，可以使用opencsv，Hadoop也有一个InputFormat，即CSVInputFormat，它并不支持记录中包含newlines。

如果你的CSV数据没有包含newlines，你可以用textFile()加载数据，parse数据。

```scala
import Java.io.StringReader
import au.com.bytecode.opencsv.CSVReader
...
val input = sc.textFile(inputFile)
val result = input.map{ line =>
  val reader = new CSVReader(new StringReader(line));
  reader.readNext();
}
```

如果记录中有newlines，就需要完整读入每个文件，然后解析各个字段。

```scala
case class Person(name: String, favoriteAnimal: String)
val input = sc.wholeTextFiles(inputFile)
val result = input.flatMap{ case (_, txt) =>
  val reader = new CSVReader(new StringReader(txt));
  reader.readAll().map(x => Person(x(0), x(1)))
}
```

#### Saving CSV

```scala
pandaLovers.map(person => List(person.name, person.favoriteAnimal).toArray)
.mapPartitions{people =>
  val stringWriter = new StringWriter();
  val csvWriter = new CSVWriter(stringWriter);
  csvWriter.writeAll(people.toList)
  Iterator(stringWriter.toString)
}.saveAsTextFile(outFile)
```

如果只有一小部分输入文件，你需要使用wholeFile()方法，可能还需要对输入数据进行重新分区是的Spark能够更高效地并行化执行后续操作。

### SequenceFile

SequenceFile是由没有相对关系结构的键值对文件组成的常用Hadoop格式。**SequenceFile文件有同步标记，Spark可以用它来定位到文件中的某个点，然后再与记录的边界对齐。**这可以让Spark使用多个节点高效地并行读取SequenceFile文件。

SequenceFile是由实现Hadoop的Writable接口的元素组成，如果你无法为要写出的数据找到对应的Writable类型，你可以重载Writable中的readFields和write来实现自己的Writable类。

#### Loading SequenceFile

```scala
val data = sc.sequenceFile(inFile, classOf[Text], classOf[IntWritable]).
  map{case (x, y) => (x.toString, y.get())}
```

有一个函数可以自动地将Writable对象转为相应的Scala类型。`sequence[Key, Value](path, minPartition)`。

#### Saving SequenceFile

我们已经进行了将许多Scala的原生类型转为Hadoop Writable的隐式转换。如果不能自动转为Writable类型，就可以对数据进行映射操作，在保存前进行类型转换。

```scala
val data = sc.parallelize(List(("Panda", 3), ("Kay", 6), ("Snail", 2)))
data.saveAsSequenceFile(outputFile)
```
