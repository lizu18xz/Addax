# Addax TxtFileReader 说明

## 1 快速介绍

TxtFileReader提供了读取本地文件系统数据存储的能力。在底层实现上，TxtFileReader获取本地文件数据，并转换为Addax传输协议传递给Writer。

**本地文件内容存放的是一张逻辑意义上的二维表，例如CSV格式的文本信息。**

## 2 功能与限制

TxtFileReader实现了从本地文件读取数据并转为Addax协议的功能，本地文件本身是无结构化数据存储，对于Addax而言，TxtFileReader实现上类比OSSReader，有诸多相似之处。目前TxtFileReader支持功能如下：

1. 支持且仅支持读取TXT的文件，且要求TXT中shema为一张二维表。

2. 支持类CSV格式文件，自定义分隔符。

3. 支持多种类型数据读取(使用String表示)，支持列裁剪，支持列常量

4. 支持递归读取、支持文件名过滤。

5. 支持文本压缩，且自动猜测压缩格式

6. 多个File可以支持并发读取。

我们暂时不能做到：

1. 单个File支持多线程并发读取，这里涉及到单个File内部切分算法。二期考虑支持。

2. 单个File在压缩情况下，从技术上无法支持多线程并发读取。

## 3 功能说明

### 3.1 配置样例

```json
{
  "job": {
    "setting": {
      "speed": {
        "channel": 2,
        "bytes": -1
      }
    },
    "content": [
      {
        "reader": {
          "name": "txtfilereader",
          "parameter": {
            "path": [
              "/tmp/data"
            ],
            "encoding": "UTF-8",
            "column": [
              {
                "index": 0,
                "type": "long"
              },
              {
                "index": 1,
                "type": "boolean"
              },
              {
                "index": 2,
                "type": "double"
              },
              {
                "index": 3,
                "type": "string"
              },
              {
                "index": 4,
                "type": "date",
                "format": "yyyy.MM.dd"
              }
            ],
            "fieldDelimiter": ","
          }
        },
        "writer": {
          "name": "txtfilewriter",
          "parameter": {
            "path": "/tmp/result",
            "fileName": "txt_",
            "writeMode": "truncate",
            "format": "yyyy-MM-dd"
          }
        }
      }
    ]
  }
}
```

### 3.2 参数说明

| 配置项            | 是否必须 | 默认值         | 描述                                                                   |
| :---------------- | :------: | -------------- | --------------------------------------------------------------------|
| path            |    是    | 无             | 本地文件系统的路径信息，注意这里可以支持填写多个路径,详细描述见下文                |
| column            |    是    | 默认String类型 | 读取字段列表，type指定源数据的类型，详见下文                                 |
| fieldDelimiter    |    是    | `,`            | 描述：读取的字段分隔符                                                  |
| encoding          |    否    | utf-8          | 读取文件的编码配置                                                     |
| skipHeader        |    否    | false          | 类CSV格式文件可能存在表头为标题情况，需要跳过。默认不跳过                    |
| csvReaderConfig   |    否    | 无             | 读取CSV类型文件参数配置，Map类型。不配置则使用默认值,详见下文 |

#### path

本地文件系统的路径信息，注意这里可以支持填写多个路径。

- 当指定单个本地文件，TxtFileReader暂时只能使用单线程进行数据抽取。二期考虑在非压缩文件情况下针对单个File可以进行多线程并发读取
- 当指定多个本地文件，TxtFileReader支持使用多线程进行数据抽取。线程并发数通过通道数指定
- 当指定通配符，TxtFileReader尝试遍历出多个文件信息。例如: 指定 `/*`代表读取 `/` 目录下所有的文件，指定 `/bazhen/*` 代表读取bazhen目录下游所有的文件。目前只支持 `*` 作为文件通配符。

特别需要注意的是，Addax会将一个作业下同步的所有Text File视作同一张数据表。用户必须自己保证所有的File能够适配同一套schema信息。读取文件用户必须保证为类CSV格式，并且提供给Addax权限可读。

特别需要注意的是，如果Path指定的路径下没有符合匹配的文件抽取，Addax将报错。

从 3.2.3 版本起， `path` 下允许混合不同压缩格式的文件，插件会尝试自动猜测压缩格式并自动解压，目前支持的压缩格式有：

- zip
- bzip2
- gzip
- LZ4
- PACK200
- XZ
- Compress

#### column

读取字段列表，type指定源数据的类型，index指定当前列来自于文本第几列(以0开始)，value指定当前类型为常量，不从源头文件读取数据，而是根据value值自动生成对应的列。 <br />

默认情况下，用户可以全部按照String类型读取数据，配置如下：

```json
{
  "column": [
    "*"
  ]
}
```

用户可以指定Column字段信息，配置如下：

```json
[
  {
    "type": "long",
    "index": 0
  },
  {
    "type": "string",
    "value": "alibaba"
  }
]
```

对于用户指定Column信息，type必须填写，index/value必须选择其一。

从 `4.0.1` 开始，表示字段除了使用 `index` 来指定字段的顺序外，还支持 `name` 方式，这需要所读取的文件的都包含了文件头，插件会尝试将指定的 `name` 去匹配从文件读取的文件头，
然后得到对应的 `index` 值，并回写到 配置文件中。同时，`index` 和 `name` 可以在不同的列上进行混合使用，比如下面这样：

```json
[
  {
    "type": "long",
    "index": 0
  },
  {
    "name": "region",
    "type": "string"
  },
  {
    "type": "string",
    "value": "alibaba"
  }
]
```

注： 这种方式以为这在准备阶段就要尝试读取文件，因为会有一定的性能损失，如非必要，不建议配置 `name` 方式。

#### csvReaderConfig

读取CSV类型文件参数配置，Map类型。读取CSV类型文件使用的CsvReader进行读取，会有很多配置，不配置则使用默认值。

常见配置：

```json
{
  "csvReaderConfig": {
    "safetySwitch": false,
    "skipEmptyRecords": false,
    "useTextQualifier": false
  }
}
```

所有配置项及默认值,配置时 csvReaderConfig 的map中请**严格按照以下字段名字进行配置**：

```ini
boolean caseSensitive = true;
char textQualifier = 34;
boolean trimWhitespace = true;
boolean useTextQualifier = true;//是否使用csv转义字符
char delimiter = 44;//分隔符
char recordDelimiter = 0;
char comment = 35;
boolean useComments = false;
int escapeMode = 1;
boolean safetySwitch = true;//单列长度是否限制100000字符
boolean skipEmptyRecords = true;//是否跳过空行
boolean captureRawRecord = true;
```

### 3.3 类型转换

本地文件本身不提供数据类型，该类型是Addax TxtFileReader定义：

| Addax 内部类型| 本地文件 数据类型    |
| -------- | -----  |
|
| Long     |Long |
| Double   |Double|
| String   |String|
| Boolean  |Boolean |
| Date     |Date |

其中：

- Long 是指本地文件文本中使用整形的字符串表示形式，例如"19901219"。
- Double 是指本地文件文本中使用Double的字符串表示形式，例如"3.1415"。
- Boolean 是指本地文件文本中使用Boolean的字符串表示形式，例如"true"、"false"。不区分大小写。
- Date 是指本地文件文本中使用Date的字符串表示形式，例如"2014-12-31"，Date可以指定format格式。
