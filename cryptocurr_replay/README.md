### 加密货币逐笔交易数据回放

对加密货币逐笔交易数据的回放展示，有助于交易员复盘，加深对市场的洞察。对于以每天千万条记录持续递增的盘口和逐笔交易数据集，DolphinDB可实现可视化的高速回放，以及对回放结果逐点查询的功能。回放过程中，DolphinDB可以关联盘口和交易，实现两个数据表的同步回放。

#### 1. 数据回放案例使用说明

DolphinDB支持将多个分布式表同步回放并发布到流数据表。基于这一特性，前端JavaScript使用DolphinDB Web API来轮询回放输出的流数据表，实现盘口和交易数据的可视化回放。DolphinDB自带web服务器，因此整个系统可以在DolphinDB内完成，无外部依赖。

通过几个步骤来运行案例：

* 部署DolphinDB节点
* 下载加密货币盘口和交易数据
* 导入数据到DolphinDB数据库
* 安装基于web的回放页面
* 通过浏览器访问回放页面

#### 2. 部署DolphinDB节点
请访问[DolphinDB官网](https://www.dolphindb.cn/downloads.html)，下载最新版本DolphinDB。具体部署的可以参考[教程](https://github.com/dolphindb/Tutorials_CN/blob/master/single_machine_cluster_deploy.md)中的单机集群方式部署。

通过GUI连接集群中任意一个数据节点作为客户端节点，本文中使用 `[host]`,`[port]`,`[nodeAlias]`来分别代表此节点的IP, 端口，别名。
#### 3. 下载盘口和逐笔交易数据
本案例的使用的是火币研究院提供的加密货币交易数据，数据获取方式参考[火币数据API](https://huobiapi.github.io/docs/spot/v1/cn/), 获取数据的示例代码可以参考 [python 示例代码](https://github.com/huobiapi/Futures-Python-demo) | [java 示例代码](https://github.com/huobiapi/Futures-Java-demo)。
#### 4. 导入数据到DolphinDB

本案例中使用的数据为csv文件，包含orderBook和tick数据，可以通过火币研究院提供的[API]来获取生成。DolphinDB提供了loadText的方式来快速加载csv文件，在GUI中运行下面DolphinDB脚本，即可以实现数据的导入。

**下文`/hdd/data/orderBook`和`/hdd/data/tick`指原始数据文件所在目录,用户在运行时请自行替换为本地目录** 

* 案例使用的数据文件中第一行是无关信息，可以采用下述脚本进行数据的预处理,处理好的文件保存到 `/hdd/data/orderBook-processed, /hdd/data/tick-processed`目录下。若数据文件正常无需处理的，可以略过这一步。
```kdb
//删除数据文件第一行无关信息
def dataPreProcess(DIR){
	if(!exists(DIR+ "-processed/"))
		mkdir(DIR+ "-processed/")
	fileList = exec filename from files(DIR) where isDir = false, filename like "%.csv"
	for(filename in fileList){
		f = file(DIR + "/" + filename)
		y = f.readLines(1000000).removeHead!(1)
		saveText(y, DIR+ "-processed/" + filename)
	}
}
dataPreProcess("/hdd/data/orderBook")
dataPreProcess("/hdd/data/tick")
```

* 按照查询频率以及数据量，按照产品代码和业务时间对数据做复合分区。在本演示案例中，为方便起见，数据库名称强制为`dfs://huobiDB`。 如果需要修改，用户必须同时修改回放网页中的数据库名称。
```kdb
def createDB(){
	if(existsDatabase("dfs://huobiDB"))
		dropDatabase("dfs://huobiDB")
    //按照数据集的时间跨度，请自行调整VALUE分区日期范围
	db1 = database(, VALUE, 2018.09.01..2018.09.30)
	db2 = database(, HASH, [SYMBOL,20])
	db = database("dfs://huobiDB", COMPO, [db1,db2])
}

def createTick(){
	tick = table(100:0, `aggregate_ID`server_time`price`amount`buy_or_sell`first_trade_ID`last_trade_ID`product , [INT,TIMESTAMP,DOUBLE,DOUBLE,CHAR,INT,INT,SYMBOL])
	db = database("dfs://huobiDB")
	return db.createPartitionedTable(tick, `tick, `server_time`product)
}

def createOrderBook(){
	orderData = table(100:0, `lastUpdateId`server_time`buy_1_price`buy_2_price`buy_3_price`buy_4_price`buy_5_price`buy_6_price`buy_7_price`buy_8_price`buy_9_price`buy_10_price`buy_11_price`buy_12_price`buy_13_price`buy_14_price`buy_15_price`buy_16_price`buy_17_price`buy_18_price`buy_19_price`buy_20_price`sell_1_price`sell_2_price`sell_3_price`sell_4_price`sell_5_price`sell_6_price`sell_7_price`sell_8_price`sell_9_price`sell_10_price`sell_11_price`sell_12_price`sell_13_price`sell_14_price`sell_15_price`sell_16_price`sell_17_price`sell_18_price`sell_19_price`sell_20_price`buy_1_amount`buy_2_amount`buy_3_amount`buy_4_amount`buy_5_amount`buy_6_amount`buy_7_amount`buy_8_amount`buy_9_amount`buy_10_amount`buy_11_amount`buy_12_amount`buy_13_amount`buy_14_amount`buy_15_amount`buy_16_amount`buy_17_amount`buy_18_amount`buy_19_amount`buy_20_amount`sell_1_amount`sell_2_amount`sell_3_amount`sell_4_amount`sell_5_amount`sell_6_amount`sell_7_amount`sell_8_amount`sell_9_amount`sell_10_amount`sell_11_amount`sell_12_amount`sell_13_amount`sell_14_amount`sell_15_amount`sell_16_amount`sell_17_amount`sell_18_amount`sell_19_amount`sell_20_amount`product,[INT,TIMESTAMP,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,DOUBLE,SYMBOL])
	db = database("dfs://huobiDB")
	return db.createPartitionedTable(orderData, `orderBook, `server_time`product)
}

```

* 从文件目录中读取数据文件并导入数据库。
```kdb
def loadTick(path, filename, mutable tb){
	tmp = filename.split("_")
	product = tmp[1]
	file = path + "/" + filename
	t = loadText(file)
	t[`product]=product
	tb.append!(t)
}

def loopLoadTick(mutable tb, path){
	fileList = exec filename from files(path,"%.csv")
	for(filename in fileList){
		print filename
		loadTick(path, filename, tb)
	}
}


def loadOrderBook(path, filename, mutable tb){
	tmp = filename.split("_")
	product = tmp[1]
	file = path + "/" + filename
	t = loadText(file)
	t[`product] = product
	tb.append!(t)
}

def loopLoadOrderBook(mutable tb, path){
	fileList = exec filename from files(path, "%.csv")
	for(filename in fileList){
		print filename
		loadOrderBook(path, filename, tb)
	}
}

login("admin","123456")
tb = createOrderBook()
loopLoadOrderBook(tb, "/hdd/data/orderBook-processed")
tb = createTick()
loopLoadTick(tb, "/hdd/data/tick-processed")
```
* 在GUI中执行上述代码，即可将数据载入到DolphinDB的数据库中，[点击下载](https://github.com/dolphindb/applications/blob/master/cryptocurr_replay/importData.dos)完整代码。

* 定义回放自定义函数

```python
def replayData(productCode, startTime, length, rate){
    login('admin', '123456');
    tick = loadTable('dfs://huobiDB', 'tick');
    orderbook = loadTable('dfs://huobiDB', 'orderBook');

    schTick = select name,typeString as type from  tick.schema().colDefs;
    schOrderBook = select name,typeString as type from  orderbook.schema().colDefs;
	
    share(streamTable(100:0, schOrderBook.name, schOrderBook.type), `outOrder);
    share(streamTable(100:0, schTick.name, schTick.type), `outTick);
    enableTablePersistence(objByName(`outOrder), true,true, 100000);
    enableTablePersistence(objByName(`outTick), true,true, 100000);
    clearTablePersistence(objByName(`outOrder));
    clearTablePersistence(objByName(`outTick));
                                                
    share(streamTable(100:0, schOrderBook.name, schOrderBook.type), `outOrder);
    share(streamTable(100:0, schTick.name, schTick.type), `outTick);
    enableTablePersistence(objByName(`outOrder), true,true, 100000);
    enableTablePersistence(objByName(`outTick), true,true, 100000);

    endTime = temporalAdd(startTime, length, "m")
    sqlTick = sql(sqlCol("*"), tick,  [<product=productCode>, <server_time between timestamp(pair(startTime, endTime))>]);
    sqlOrder = sql(sqlCol("*"), orderbook,  [<product=productCode>, <server_time between timestamp(pair(startTime, endTime))>]);
    cutCount = length * 60 / 20
    trs = cutPoints(timestamp(startTime..endTime), cutCount);
    rds = replayDS(sqlTick, `server_time , , trs);
    rds2 = replayDS(sqlOrder, `server_time , , trs);
    return submitJob('replay_huobi','replay_huobi',  replay,  [rds,rds2],  [`outTick,`outOrder],`server_time ,, rate);
}

addFunctionView(replayData);
```



#### 5. 使用数据回放界面

* 使用数据回放界面，还需要下载回放的界面[html文件压缩包](https://github.com/dolphindb/applications/raw/master/cryptocurr_replay/replay.zip)，将下载的解压包部署到dolphindb的web根目录下，通过`http://[host]:[port]/replay.html` 进入数据回放界面。
* 数据回访界面主要包含`回放`和 `点查询`两个功能。
* 回放： 按照设置的产品代码，时间段，按照指定速率值回放时间段内的数据，回放数据包含盘口数据和实时交易数据。回放可以设置的参数主要是三个：
    *  产品代码： 加密货币产品代码
    *  时间段： 设置回放的起始时间和播放时长(单位:分钟)
    *  播放速率: 播放速率值是每秒钟播放的实时交易记录数，若正常市场交易是每秒钟产生100笔交易，这里设置1000就是加快10倍回放。
* 点查询：当回访完成并点击结束按钮之后，用户通过单击价格趋势图表中的点，可以让表格中显示该时间之前的10笔数据。

* 具体操作请参考下图：

![image](https://github.com/dolphindb/Tutorials_CN/blob/master/images/replay/v.gif?raw=true)

#### 6. 使用docker快速体验回放功能
我们提供了一个包含DolphinDB Server以及演示数据的docker容器，并打包成tar文件提供下载。用户仅需要[安装docker环境](https://docs.docker.com/install/)，[下载](https://www.dolphindb.cn/downloads/cryptocurr_replay.tar.gz)打包文件，运行下面的命令就可以快速完成演示环境部署。

```bash
gunzip cryptocurr_replay.tar.gz
##docker若没有赋予非管理访问权限，可以使用 sudo docker
docker import cryptocurr_replay.tar ddb/replay:v1
##生成并启动容器
docker run -dt -p 8888:8848 --name replay1 ddb/replay:v1 /bin/bash /dolphindb/start.sh

```

启动容器后，docker内DolphinDB的访问端口被映射到宿主机8888端口，打开浏览器访问`http://[宿主机ip]:8888/replay.html`，进入到回放演示界面。

为了控制docker容器大小，方便下载，演示数据仅包含加密货币编号为`ETHUSDT,ETHBTC,BTCUSDT` 的交易数据，日期为`2018.09.17`一天。

注意：因为内置的license文件会过期, 需要从官网下载最新的社区版license替换。下载社区版解压后进入server目录下，拷贝dolphindb.lic文件覆盖docker中的/dolphindb/目录下同名文件
```
sudo docker cp ./dolphindb.lic replay1:/dolphindb/dolphindb.lic
```
重启docker
```
sudo docker restart replay1
```
#### 7. 注意事项

* 本案例当前仅限单用户使用，不支持多用户同时回放。
* 为了简化操作，数据库名称和数据库用户信息均固化在网页中，若有需要请自行修改replay.html文件

**使用中有任何问题欢迎加入[DolphinDB技术交流群](https://zhuanlan.zhihu.com/p/55382440)沟通**
