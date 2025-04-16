<!--
 * @Author: hugo2046 shen.lan123@gmail.com
 * @Date: 2025-04-09 14:19:59
 * @LastEditors: hugo2046 shen.lan123@gmail.com
 * @LastEditTime: 2025-04-11 13:51:28
 * @FilePath: /workspace/DolphinDBScript/ExchTDFData/README.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->
# 说明

使用ddb读取TDFToCSV生成的csv数据并将其同步至数据库。

TDFToCSV储存的文件在`/data1/hugo/tdf/`中。

# 每日文件结构

每日的文件结构如下

```
20250317
├── SH-2-0
│   ├── Order
│   │   ├── 010609.csv
│   │   ├── ...
│   │   └── 900957.csv
│   ├── OrderQueue
│   │   ├── 000300.csv
│   │   ├── ...
│   │   └── 999999.csv
│   ├── Tick2
│   │   ├── 000300.csv
│   │   ├── ...
│   │   └── 999999.csv
│   └── Transaction
│       ├── 019319.csv
│       ├── ...
│       └── 900957.csv
└── SZ-2-0
    ├── Order
    │   ├── 000006.csv
    │   ├── ...
    │   └── 524071.csv
    ├── OrderQueue
    │   ├── 000001.csv
    │   ├── ...
    │   └── 988007.csv
    ├── Tick2
    │   ├── 000001.csv
    │   ├── ...
    │   └── 988107.csv
    └── Transaction
        ├── 000001.csv
        ├── ...
        └── 563638.csv
```
# 配置
系统配置通过 `config.dos` 文件管理，该文件定义了名为 `setConfig` 的函数，用于返回包含所有配置选项的字典。

## 配置项说明

`config.dos` 中的主要配置项包括：

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| dbName | 数据库名称 | dfs://TickBase |
| tick | 行情快照表名 | stockTick |
| order | 逐笔委托表名 | stockOrder |
| orderqueue | 逐笔委托队列表名 | stockOrderQueue |
| transaction | 逐笔成交表名 | stockTransaction |
| filePath | 源数据文件路径 | /store/tdf/data/ |

## 使用方法

在其他模块中，可以通过以下方式使用配置：

```cpp
use DolphinDBScript::ExchTDFData::config

// 获取配置
cfg = setConfig()

// 访问特定配置项
dbName = cfg['dbName']
tickTable = cfg['tick']
```

## 配置修改

如需修改默认配置，可以直接编辑 `config.dos` 文件中的默认值，或在运行时通过以下方式覆盖默认配置：

```cpp
use DolphinDBScript::ExchTDFData::config

// 获取默认配置
cfg = setConfig()

// 修改配置项
cfg['filePath'] = "/path/to/new/data/directory/"
```

# 模块介绍

## 数据表结构

schema文件夹下的模块根据wind盘后数据脚本获取的csv表整理取得。该文件夹按照数据源格式，包含一下几个模块文件:

- `tickSchema`用于指定 Level-2 快照行情数据存入数据库的数据格式以及 DolphinDB 读取 CSV 文件时的数据格式。
- `orderSchema`用于指定逐笔委托数据存入数据库的数据格式以及 DolphinDB 读取 CSV 文件时的数据格式。
- `orderQueueSchema`用于指定逐笔委托队列数据存入数据库的数据格式以及 DolphinDB 读取 CSV 文件时的数据格式。
- `transactionSchema`用于指定逐笔成交数据存入数据库的数据格式以及 DolphinDB 读取 CSV 文件时的数据格式。

## 创建数据库和分区表

数据库和分区表创建可参考 createTB.dos，其用于创建存储交易所数据的分布式库表。根据业务需求，这里对沪深股票 Level-2 高频行情数据采用一库四表的建库建表方案，分区方案如下：
|表名|分区方案|分区列|排序列|
|--|--|--|--|
|tick|时间维度按天分区+证券代码维度HASH 25分区|date、wind_code|wind_code和date、time|
|order|时间维度按天分区+证券代码维度HASH 25分区|date、wind_code|wind_code和date、time|
|orderqueue|时间维度按天分区+证券代码维度HASH 25分区|date、wind_code|wind_code和date、time|
|transaction|时间维度按天分区+证券代码维度HASH 25分区|date、wind_code|wind_code和date、time|

## 行情数据存储模型设计
### Tick数据
| 字段含义                     | 入库字段名       | 入库数据类型 |
| ---------------------------- | ---------------- | ------------ |
| 原始代码                     | code             | SYMBOL       |
| Wind 代码(上交所SH,深交所SZ) | wind_code        | SYMBOL       |
| 证券简称                     | name             | SYMBOL       |
| 交易日                       | date             | DATE         |
| 时间(HHMMSSmmm)              | time             | TIME         |
| 最新价                       | price            | LONG         |
| 成交量                       | volume           | LONG         |
| 成交金额                     | turover          | LONG         |
| 成交笔数                     | match_items      | LONG         |
| 持仓总量(期货)               | interest         | LONG         |
| 港股交易标志                 | trade_flag       | LONG         |
| 买卖方向                     | bs_flag          | LONG         |
| 累计成交量                   | accvolume        | LONG         |
| 累计成交金额                 | accturover       | LONG         |
| 最高价                       | high             | LONG         |
| 最低价                       | low              | LONG         |
| 开盘价                       | open             | LONG         |
| 前收盘价                     | pre_close        | LONG         |
| 今结算(期货)                 | settle           | LONG         |
| 持仓量                             | position         | LONG         |
| 今虚实度(期权)               | curDelta         | LONG         |
| 昨结算(期货)                 | preSettle        | LONG         |
| 昨持仓                        | prePosition      | LONG         |
| 申卖价                       | ask1~asize10     | LONG         |
| 申卖量                       | asize1~asize1    | LONG         |
| 申买价                       | bid1~bid10       | LONG         |
| 申买量                       | bsize1~bsize10   | LONG         |
| 委卖订单数量                 | aorder1~aorder10 | LONG         |
| 委买订单数量                 | border1~border10 | LONG         |
| 加权平均委卖价格             | ask_av_price     | LONG         |
| 加权平均委买价格             | bid_av_price     | LONG         |
| 加权平均委卖价格             | total_ask_volume | LONG         |
| 委托买入总量                 | total_bid_volume | LONG         |
| 成交编号                     | index            | LONG         |
| 品种总数                      | stocks           | LONG         |
| 上涨品种数                    | ups              | LONG         |
| 下跌品种数                     | downs            | LONG         |
| 持平品种数                     | holdLines        | LONG         |
| 均价                         | avgPrice         | LONG         |
| 盘后价格(科创板有使用到)     | afterPrice       | LONG         |
| 盘后量(科创板有使用到)       | afterVolume      | LONG         |
| 盘后成交金额(科创板有使用到) | afterTurnover    | LONG         |
| 盘后成交笔数(科创板有使用到) | afterMatchItems  | LONG         |
| 涨停价                       | HighLimit        | LONG         |
| 跌停价                       | LowLimit         | LONG         |
| 申购笔数(ETF)                | etfBuyNumber     | LONG         |
| 赎回笔数(ETF)                | etfSellNumber    | LONG         |
| 申购数量(ETF)                | etfBuyAmount     | LONG         |
| 申购金额(ETF)                | etfBuyMoney      | LONG         |
| 赎回数量(ETF)                | etfSellAmount    | LONG         |
| 赎回金额(ETF)                | etfSellMoney     | LONG         |

### Transaction数据
| 字段含义                     | 入库字段名    | 入库数据类型 |
| ---------------------------- | ------------- | ------------ |
| 原始代码                     | code          | SYMBOL       |
| Wind 代码(上交所SH,深交所SZ) | wind_code     | SYMBOL       |
| 证券简称                     | name          | SYMBOL       |
| 交易日                       | date          | DATE         |
| 时间(HHMMSSmmm)              | time          | LONG         |
| 成交代码                     | function_code | LONG         |
| 成交类别                     | order_kind    | LONG         |
| 买卖方向                     | bs_flag       | LONG         |
| 成交价格                     | trade_price   | LONG         |
| 成交数量                     | trade_volume  | LONG         |
| 叫卖方委托序号               | ask_order     | LONG         |
| 叫买方委托序号               | bid_order     | LONG         |
| channel id                   | channel       | LONG         |
| 不加权指数                  | index         | LONG         |
| 业务编号                     | biz_index     | LONG         |

### Order
| 字段含义                     | 入库字段名    | 入库数据类型 |
| ---------------------------- | ------------- | ------------ |
| 原始代码                     | code          | SYMBOL       |
| Wind 代码(上交所SH,深交所SZ) | wind_code     | SYMBOL       |
| 证券简称                     | name          | SYMBOL       |
| 交易日                       | date          | DATE         |
| 时间(HHMMSSmmm)              | time          | TIME         |
| 委托号                       | order         | LONG         |
| 委托类别                     | order_kind    | LONG         |
| 委托代码                     | function_code | LONG         |
| 委托价格                     | order_price   | LONG         |
| 委托数量                     | order_volume  | LONG         |
| channel id                   | channel       | LONG         |
| 原始订单号                   | order_orino   | LONG         |
| 业务编号                     | biz_index     | LONG         |

### 
| 字段含义                     | 入库字段名  | 入库数据类型 |
| ---------------------------- | ----------- | ------------ |
| 原始代码                     | code        | SYMBOL       |
| Wind 代码(上交所SH,深交所SZ) | wind_code   | SYMBOL       |
| 证券简称                     | name        | SYMBOL       |
| 交易日                       | date        | DATE         |
| 时间(HHMMSSmmm)              | time        | LONG         |
| 买卖方向                     | side        | SYMBOL       |
| 委托价格                     | price       | LONG         |
| 订单数量                     | order_items | LONG         |
| 明细个数                     | ab_items    | LONG         |
| 叫卖叫卖的前五十笔               | ab1~ab50    | LONG         |

## 数据导入

数据导入部分涉及 ExchTDFData 文件夹和 ExchData.dos，作用如下：

ExchData 包含了 Order.dos OrderQueue.dos、Tick.dos、Transaction.dos 四个模块文件，分别用于导入沪深交易所的逐笔委托、逐笔委托队列、行情快照和逐笔成交 Level-2 高频行情数据。
ExchData.dos 用于导入指定目录下的所有交易所数据，是对前面所有模块的整合。
下面列出模块中的主要函数 ExchData 的语法和参数介绍。

### 数据导入接口
**语法**

`ExchTDFData(dbName,tbNames,filePath,startDate,endDate,dataTypes,market="ALL",deleteDuplicate=true,initialTB=false,initialDB=false)`

**详情**

将filePath路径下从startDate到endDate日期的dataTypes数据导入到`dbName`数据库中的`tbName`表中。

**参数**

- **dbName**字符串，数据库名称。
- **tbNames**字符串型的向量，分布式表名称。若需要导入行情数据，只需要传入导入的单一表名即可。
- **filePath**字符串，指定的存放数据的路径，需要确保和上述每日文件结构一致。
- **startDate**字符串，导入数据的起始日期，比如 2022.01.01(包括这一天)。如果为NULL，此时从上一个交易日开始导入。
- **endDate**字符串，导入数据的结束日期，比如 2022.12.31(包括这一天)。如果为NULL，此时从上一个交易日开始导入。
- **dataTypes**字符串型的向量，导入行情的数据源类型，"Tick","Order"等。
- **market**字符串，交易所，目前只能 "ALL", "SZ", "SH" 三选一。当 market="ALL" 时，会将沪深的数据全部导入一张名为 tableName 的分布式表；否则，会只导入一个交易所的数据。
- **deleteDuplicate**布尔值，表示是否需要删除数据库已导入的数据。默认值为 true，此时导入数据前不会删除库表中已存在的数据。
- **initialTB**布尔值，是否需要初始化数据库。如果已经存在名为 dbName 的数据库，当 initialDB=true 时，会删除原来的数据库并重新创建；否则会保留原来的数据库并输出 "[dbName] 数据库已经存在" 的提示。
- **initialDB**布尔值，是否需要初始化分布式表。如果在 dbName 数据库下已经存在名为 tbName 的表，当 initialTB=true 时，会删除原来的表并重新创建；否则会保留原来的表并输出 "数据库 [dbName] 已经存在表 [tbName]" 的提示。

**使用示例**
```cpp
use DolphinDBScript::ExchTDFData::ExchTDF
use DolphinDBScript::ExchTDFData::config

// 防止没用更新模块
clearCachedModules();
go;

// DolphinDB脚本导入交易所数据
cfg = setConfig();

dbName = cfg['dbName'];
market = "ALL";
filePath = cfg['filePath'];
startDate = 2025.03.28;
endDate = 2025.04.09;

// 导入tick数据
tbName = cfg['tick'];
jobId1 = submitJob("loadTickData","loadTickData",
ExchTDFData{dbName,tbName,filePath,startDate,endDate,"Tick",market,true,false,false})
getJobStatus(jobId1);
print(getJobMessage(jobId1));

// 导入order数据
tbName = cfg['order'];
jobId2 = submitJob("loadOrderData","loadOrderData",
ExchTDFData{dbName,tbName,filePath,startDate,endDate,"Order",market,true,false,false})
getJobStatus(jobId2);
print(getJobMessage(jobId2));

// 导入orderqueue数据
tbName = cfg['orderqueue'];
jobId3 = submitJob("loadOrderQueueData","loadOrderQueueData",
ExchTDFData{dbName,tbName,filePath,startDate,endDate,"OrderQueue",market,true,false,false})
getJobStatus(jobId3);
print(getJobMessage(jobId3));

// 导入transaction数据
tbName = cfg['transaction'];
jobId4 = submitJob("loadTransactionData","loadTransactionData",
ExchTDFData{dbName,tbName,filePath,startDate,endDate,"Transaction",market,true,false,false})
getJobStatus(jobId4);
print(getJobMessage(jobId4));
```

## 数据校验
在处理和分析交易所的 Level-2 历史行情数据时，针对原始数据的数据校验是一个至关重要的步骤。基于本模块的数据校验功能，可以监测交易所的 Level-2 历史行情数据是否存在数据遗漏、数据异常。

### 校验规则
ExchTDF模块的 *checkStockData.dos* 支持对沪深交易所的逐笔成交和逐笔委托数据做数据校验，校验逻辑包括：

- 检查导入的逐笔成交和逐笔委托数据量是否小于 1500 万，否则提示数据异常。
- 针对2023年以后的数据，检查逐笔数据的 ChannelNo 的取值范围，检查上交所的 ChannelNo 所有取值是否包含 1至6 、深交所的 ChannelNo 所有取值是否包含2011至2014。
- 检查逐笔数据每一支 ChannelNo 下的所有 ApplSeqNum 是否连续；若不连续检查是否存在重复数据，以及是否存在数据缺失的异常情况。

### 校验接口

**语法**
`checkAllData(startDate,endDate,market)`

**详情**
校验*startDate*和*endDate*期间的逐笔数据，若校验未通过将会返回统计信息表。

**参数**
- **startDate** 开始日期。
- **endDate** 结束日期。
- **market** 交易所类型，支持”SH”、”SZ”、”ALL”。

**使用示例**
如果检查2025.03.04的逐笔数据是否存在异常，结果如下：
```cpp
use ExchTDFData::checkStockData
go;

checkAllData(2025.03.04,2025.03.04,"ALL")
```