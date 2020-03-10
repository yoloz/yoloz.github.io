---
title: Lucene的Index文档模型
comments: false
toc: false
date: 2019-02-18 17:06:43
categories: lucene
tags:
---

Lucene中包含了四种基本数据类型，分别是：

* __Index__：索引，由很多的Document组成。
* __Document__：由很多的Field组成，是Index和Search的最小单位。
* __Field__：由很多的term组成，包括field_name和field_value。
* __Term__：由很多的字节组成，可以分词，分词之后每个词即为一个term。term是索引的最小单元。

Lucene中存储的索引主要分为三种类型：

* __Invert Index__，即倒排索引, 通过term可以快速查找到包含该term的doc_id。如果Field配置分词，则分词后的每个term都会进入倒排索引，如果Field不指定分词，那该Field的value值则会作为一个term进入倒排。（这里需要注意的是term的长度是有限制的，如果对一个Field不采取分词，那么不建议该Field存储过长的值）
* __DocValues__，即正排索引, 采用的是类似数据库的列式存储。对于一些特殊需求的字段可以选择这种索引方式。
* __Store__，即原文, 存储整个完整Document的原始信息。

<! --more-->

## Invert Index

倒排索引是lucene的核心索引类型，采用链表的数据结构，倒排索引中的key就是一个term，value就是以doc_id形成的链表结构。如下：

| Term  | Doc_1 | Doc_2 |
| :-------| :----- | :------ |
| Quick |       |     X |
| The   |   X   |       |
| brown |   X   |     X |
| dog   |   X   |       |
| dogs  |       |     X |
 
现在，如果我们想搜索 quick brown ，我们只需要查找包含每个词条的文档：

| Term  | Doc_1 | Doc_2 |
|:-------|:-----|:------|
| brown |   X   |     X |
| Quick |       |     X |
| total |   2   |     1 |

这里分别匹配到了doc1和doc2，但是doc1匹配度要高于doc2。

倒排索引中的value有四种存储类型：

* __DOCS__：只存储doc_id。
* __DOCS_AND_FREQS__：存储doc_id和词频（Term Freq）。
* __DOCS_AND_FREQS_AND_POSITIONS__：存储doc_id、词频（Term Freq）和位置。
* __DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS__：存储doc_id、词频（Term Freq）、位置和偏移(offset)

> org.apache.lucene.index.IndexOptions

``` java
public enum IndexOptions {
  NONE,
  DOCS,
  DOCS_AND_FREQS,
  DOCS_AND_FREQS_AND_POSITIONS,
  DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS,
}
```

## DocValues

正排索引类似关系型数据库的存储模式，作用是通过doc_id和field_name可以快速定位到指定doc的特定字段值。DocValues的key是doc_id+field_name，value是field_value。

> 如果域要进行facet，gourp，highlight等查询可以使用docValue，节省内存。

## Store

存储的是Document的完整信息，包括所有field_name和field_value。Store的key是doc_id，value是field_name+field_value。对于上述中需要聚合和排序的Field并没有开启DocValues的情况，依然可以实现排序和聚合，会从Store中获取要排序聚合的字段值。

## 索引过程

总体来说，索引过程为：

1. **提取**：从原文提取，并创建Document和Field对象。
2. **分析**：分析文本将文本数据分割成字汇单元串，然后对它们执行一些可选操作。
3. **建立索引**：通过IndexWriter的addDocument写入到索引中，lucene将输入数据以一种倒排索引(inverted index)的数据结构存储。

## 索引操作

* __创建IndexWriter__

 `IndexWriter(dir,new WhiteSpaceAnalyser(),IndexWriter.MaxField.UNLIMITED)`

> dir是索引的保存路径，WhiteSpaceAnalyser是基于空白的分词 ，最后部限定Field的数量。

* __创建文档Document和Field__

``` java
Document doc = new Document();
doc.add(new Filed(key,value,STORE?,INDEX?)
```

> key是field的检索字段名，value是待写入/分析的文本;

* __更新索引__

 `updateDocument(xxxx)` Luncene只支持全部替换，即整个Docuemnt要被替换掉，没法更新单独的Field

## 域(Field)选项

### 域索引选项

域索引选项(Field. Index.*)通过倒排索引来控制域文本是否可被搜索。具体选项如下：

* __Index. ANALYZED__：使用分析器将域值分解成独立的词汇单元流，并使每个词汇单元能被搜索。该选项适用于普通文本域(如正文、标题、摘要等)；
* __Index. NOT_ANALYZED__：对域进行索引，但不对String值进行分析。该操作实际上将域值作为单一词汇单元并使之能被搜索。该选项适用于索引那些不能被分解的域值，如URL，文件路径，日期，人名，社保号码和电话号码等。该选项尤其适用于“精确匹配”搜索；
* __Index. ANALYZED_NO_NORMS__：这是Index. ANALYZED选项的一个变体，它不会在索引中存储norms信息。norms记录了索引中的index-time boost信息，但是当你进行搜索时可能会比较耗费内存；
* __Index. NOT_ANALYZED_NO_NORMS__：与Index. NOT_ANALYZED类似，但也是不存储norms。该选项常用于在搜索期间节省索引空间和减少内存耗费，因为single-token域并不需要norms信息，除非它们已被进行加权操作；
* __Index. NO__: 使对应的域值不被搜索。

当lucene建立起倒排索引后，默认情况下它会保存所有必要信息以实施Vector Space Modal。该Modal需要计算文档中出现的term数，以及它们出现的位置(这是必要的，比如通过词组搜索时用到)。但有时候这些域只是在布尔搜索时用到，它们并不为相关评分做贡献，一个常见的例子是域只是被用作过滤，如权限过滤和日期过滤。在这种情况下，可以通过调用 `Field.setOmitTermFreqAndPositions(true)` 方法让lucene跳过对该项的出现频率和出现位置的索引。该方法可以节省一些索引在磁盘上的存储空间，还可以加速搜索和过滤过程，但会悄悄地阻止需要位置信息的搜索，如阻止PhraseQuery和SpanQuery类的运行。

> 加权基准(Norms)  

在索引期间，文档中域的所有加权都被合并成一个单一的浮点数。除了域，文档也有自己的加权值，lucene会基于域的词汇单元数量自动计算出这些加权值(更短的域具有更高的加权)。这些加权被合并到一处，并被编码(量化)成一个单一的字节值，作为域或文档信息的一部分存储起来。在搜索期间，被搜索域的norms都被加载到内存并被解码还原为浮点数，然后用于计算相关性评分(relevance score)。
需要注意如果索引进行一半时关闭norms选项，那么必须对整个索引进行重建，因为即使只有一个文档在索引时包含了norms选项，那么随后的段合并操作中，这个情况会“扩散”，从而使得所有文档都会占用一个字节norms空间，即使它们在此前的索引操作中挂比了norms选项。

### 域存储选项

域存储选项(Field. Store.*)用来确定是否需要存储域的真实值，以便后续搜索时能恢复这个值。

* __Store. YES__: 指定存储域值。该情况下，原始的字符串值全部被保存在索引中，并可以由IndexReader类恢复。该选项对于需要展示搜索结果的一些域很有用(如UEL，标题或数据库主键)。如果索引的大小在搜索程序考虑之列的话，不要存储太大的域值，因为存储这些域值会消耗掉索引的存储空间;
* __Store. NO__: 指定不存储域值。该选项通常跟Index. ANALYZED选项共同用来索引大的文本域值，通常这些域值不用恢复为初始格式，如Web页面的正文或其他类型的文本文档。

lucene包含一个很实用的工具类 `CompressionTools` ，该类提供静态方法压缩和解压缩字节数组。该类运行时会后台调用java内置的 `java.util.zip` 类。你可以使用它在存储域值之前进行压缩。注意，尽管该方法可以为索引节省一些空间，但节省的幅度跟域值的可被压缩程度有关，而且该方法会降低索引和搜索速度。其实是通过消耗更多CPU计算能力来换取更多的磁盘空间，对于很多程序来说，需要仔细权衡一下，如果域值占用空间不是很大，建议少用压缩。

### 域的项向量选项

域的项向量(Field. TermVector.*)主要用于高亮显示及相似搜索，它是介于索引域和存储域的一个中间结构。

* __TermVector. YES__: 记录Term Vector; 
* __TermVector. WITH_POSITIONS__: 记录Term Vector以及每个Term出现的位置;
* __TermVector. WITH_OFFSETS__: 记录Term Vector以及每个Term出现的偏移;
* __TermVector. WITH_POSITIONS_OFFSETS__: 记录Term Vector以及出现的位置+偏移;
* __TermVector. NO__: 不存储TermVector  

> 如果Index选择了No，则TermVector必须选择No

### Reader、TokenStream和byte[]域值

Field对象还有其他几个初始化方法，允许传入除String以外的其他参数。

* `Field(String name, Reader reader, IndexableFieldType type)` 域值是不能被存储的(域存储选项被硬编码成Store. NO), 并且该域会一直用于分析和索引(Index. ANALYZES)。如果内存中保存String代价高或者不太方便时，如果存储的域值较大时，使用这个初始化方法则比较有效。
* `Field(String name, TokenStream tokenStream, IndexableFieldType type)` 允许程序对域值进行预分析并生成TokenStream对象。此外这个域不会被存储并将一直用于分析和索引。
* `Field(String name, byte[] value, IndexableFieldType type)`

### 域选项组合

3类域选项(索引、存储和项向量)设置完会形成若干可能的组合，常见如下：

|索引选项|存储选项|项向量|使用范例|
|:-|:-|:-|:-|
|NOT_ANALYZED_NO_NORMS|YES|NO|标识符(文件名、主键)，电话号码和社会安全号码、URL、姓名、日期、用于排序的文本域|
|ANALYZED|YES|WITH_POSITIONS_OFFSETS|文档标题、摘要|
|ANALYZED|NO|WITH_POSITIONS_OFFSETS|文档正文|
|NO|YES|NO|文档类型、数据库主键(如果没有用于搜索)|
|NOT_ANALYZED|NO|NO|隐藏的关键词|

### 域排序选项

用于排序的域是必须进行索引的，而且每个对应文档必须包含一个词汇单元。通常这意味着使用 `Field.Index.NOT_ANALYZED` 或 `Field.Index.NOT_ANALYZED_NO_NORMS` 。

### 多值域

## 对数字、日期、时间等进行索引

lucene为我们装备了一个处理日期的利器：DateTools, 通过它我们可以便捷的把Date型转换成String型。DateTools可以把日期和时间转换成 YYYYMMDDhhmmss 的格式，并根据指定的resolution去除相应后缀; 也可以将其转成long值处理;

[样例Demo](https://github.com/yoloz/searchSSDB.git)

[lucene开发](http://codepub.cn/tags/Lucene/)
