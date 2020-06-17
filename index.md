# 声明
本文档是基于backtrader官方文档的翻译  
官方文档地址 https://www.backtrader.com/docu/  

# 1.介绍

## backtrader的2个目标
1. 使用简单
2. 参考1

## backtrader的运行流程
1. 制定策略  
    1.1 确定潜在的可调参数  
    1.2 实例化您在策略中需要的指标  
    1.3 写下进入/退出市场的逻辑  
  
2. 创建Cerebro引擎（西班牙语大脑的意思）  
    2.1 注入策略  
    2.2 使用cerebro.adddata加载回测数据  
    2.3 执行cerebro.run  
    2.4 使用cerebro.plot绘制可视化图表  
  
backtrader是高度可配置的，希望大家发现其中的乐趣。  
  
# 2.安装
除了绘图以外，backtrader不需要任何外部依赖包  

## 版本要求
基本要求是：  
Python 2.7  
Python 3.2 / 3.3 / 3.4 / 3.5  
如有绘图需求，则要求 Matplotlib> = 1.4.1  

## 兼容性
backtrader兼容Python2.x/3.x。  
backtrader同时在Python2.7和Python3.4下进行过开发和测试。并在Travis下通过持续集成来检查与3.2/3.3/3.5的兼容性测试。  

## 从pypi安装
pip install backtrader  
如果希望绘图功能，请使用[plotting]选项,这将会安装matplotlib和相关依赖包。  
pip install backtrader[plotting]  

## 从源码安装
首先从github网站下载一个版本或最新的压缩包：  
https://github.com/mementum/backtrader  

解压并将源代码拷贝到项目目录 
命令如下： 
```shell  
tar xzf backgrader.tgz  
cd backtrader  
cp -r backtrader project_directory  
python setup.py install  
```
如有绘图需求，请手动安装matplotlib。  

# 3.快速开始
让我们从零开始来跑一些完整的例子，在此之前，先来了解两个基本概念。  
  
## 折线（Line） 
交易数据（Data Feeds）、技术指标（Indicators）和策略（Strategies）都是折线（Line）。  
折线（Line）是由一系列的点组成的。  
  
通常交易数据（Data Feeds）包含以下几个组成部分：  
开盘价（Open）、最高价（High）、最低价（Low）、收盘价（Close）、成交量（Volume）、持仓量（OpenInterest）等。  
比如：所有的开盘价（Open）按时间组成一条折线（Line），那么一组交易数据（Data Feeds）就应该包含了6条折线（Line）。  

再加上时间（DateTime）一共有7条折线（Line）。时间，一般用作一组交易数据的主键。  

## 索引从0开始
当访问一条折线（Line）的数据时，会默认从下标为0的位置开始，最后一个数据通过下标-1来获取。这样的设计和Python的迭代器是一致的，所以折线（Line）是可以迭代遍历的。  

例如：创建一个简单移动平均值的策略（均值策略）：  
`self.sma = SimpleMovingAverage(.....)`
访问此移动平均线的当前值的最简单方法：  
`av = self.sma[0]`
所以在回测过程中，无需知道已经处理了多少条/分钟/天/月，“0”一直指向当前值。  

按照Python遍历数组的方式，用下标-1来访问最后一个值：  
previous_value = self.sma[-1]  
同理：-2、-3下标也是可以照常使用。  

## 从0到100

先跑一个基本框架
```python
# 导入backtrader框架
import backtrader as bt
if __name__ == '__main__':
	# 创建Cerebro引擎
    cerebro = bt.Cerebro()
    #Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000
    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())
```
运行后，结果输出：
```
组合期初资金: 10000.00
组合期末资金: 10000.0
```

## 设定现金
在金融世界中，可以肯定的是，只有Losser才以1万开始。让我们更改现金并再次运行该示例。  
```python
#导入backtrader框架
import backtrader as bt
if __name__ == '__main__':
	# 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 设置投资金额100000.0
    cerebro.broker.setcash(100000.0)
    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())
```
运行后，结果输出：
```
组合期初资金: 100000.00
组合期末资金: 100000.0
```
任务完成。Let’s move to tempestuous waters.（意译：让我们再整点牛逼点的）  

## 加入交易数据

拥有现金是一件快乐的事，这一切背后最终的目的就是: 不动一根手指，就能让自动化策略通过操作资产的交易数据来增加现金。  
无数据，不快乐，让我们继续加入交易数据。  
```python
import datetime  # 
import os.path  # 路径管理
import sys  # 获取当前运行脚本的路径 (in argv[0])

#导入backtrader框架
import backtrader as bt
if __name__ == '__main__':
	# 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 获取当前运行脚本所在目录
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    # 拼接加载路径
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    # 创建交易数据集
    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        # 数据必须大于fromdate
        fromdate=datetime.datetime(2000, 1, 1),
        # 数据必须小于todate
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    # 加载交易数据
    cerebro.adddata(data)


    # 设置投资金额100000.0
    cerebro.broker.setcash(100000.0)
    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())
```
运行后，结果输出：
```
组合期初资金: 100000.00
组合期末资金: 100000.0
```

代码量略有增加，因为我们添加了：  
找出数据文件所在的路径  
通过datetime对象过滤我们想要操作的数据  

>注意:  
>Yahoo的价格数据有点非主流，它是以时间倒序排列的。datetime.datetime()中的reversed=True参数是将顺序反转一次，这样就得到了我们想要的正序数据。  

## 第一个策略
资金和交易数据都有了，探险即将开始。让我们给策略加点代码，打印出每天的”收盘价”。  
DataSeries（交易数据的基类）对象，可以直接访问到 OHLC (开盘价、最高价、最低价、收盘价)等数据。这使我们打印数据很方便。  

```python
import datetime  # 
import os.path  # 路径管理
import sys  # 获取当前运行脚本的路径 (in argv[0])

#导入backtrader框架
import backtrader as bt

# 创建策略继承bt.Strategy
class TestStrategy(bt.Strategy):

    def log(self, txt, dt=None):
        # 记录策略的执行日志
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 保存收盘价的引用
        self.dataclose = self.datas[0].close

    def next(self):
        # 记录收盘价
        self.log('Close, %.2f' % self.dataclose[0])


if __name__ == '__main__':    
	# 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 为Cerebro引擎添加策略
    cerebro.addstrategy(TestStrategy)

    # 获取当前运行脚本所在目录
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    # 拼接加载路径
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    # 创建交易数据集
    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        # 数据必须大于fromdate
        fromdate=datetime.datetime(2000, 1, 1),
        # 数据必须小于todate
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    # 加载交易数据
    cerebro.adddata(data)


    # 设置投资金额100000.0
    cerebro.broker.setcash(100000.0)
    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())
```
运行后的输出结果为：  
```
组合期初资金: 100000.00
2000-01-03T00:00:00, Close, 27.85
2000-01-04T00:00:00, Close, 25.39
2000-01-05T00:00:00, Close, 24.05
...
...
...
2000-12-26T00:00:00, Close, 29.17
2000-12-27T00:00:00, Close, 28.94
2000-12-28T00:00:00, Close, 29.29
2000-12-29T00:00:00, Close, 27.41
组合期末资金: 100000.0
 ```

有人说股市有风险，看起来不像啊。  
一些解释说明：  
 
框架在调用init时，该策略已经具有一个数据列表datas，这是标准的Python列表，可以按插入顺序访问数据。  
列表中的第一个数据self.datas[0]是用于交易操作，并且策略中的所有元素都是由框架的系统时钟进行同步的。  
由于只需访问收盘价数据，于是使用 self.dataclose = self.datas[0].close将第一条价格数据的收盘价赋值给新变量。  
系统时钟当经过一个K线柱的时候，策略的next()方法就会被调用一次。这一过程将一直循环，直到其他指标信号出现为止。此时，便会输出最终结果。关于这些，后继内容会讲到。

## 给策略加点逻辑
如果K线收盘价出现三连跌，则买入。
```python
import datetime  # 
import os.path  # 路径管理
import sys  # 获取当前运行脚本的路径 (in argv[0])

#导入backtrader框架
import backtrader as bt

# 创建策略继承bt.Strategy
class TestStrategy(bt.Strategy):

    def log(self, txt, dt=None):
        # 记录策略的执行日志
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 保存收盘价的引用
        self.dataclose = self.datas[0].close

    def next(self):
        # 记录收盘价
        self.log('Close, %.2f' % self.dataclose[0])
        # 今天的收盘价 < 昨天收盘价 
        if self.dataclose[0] < self.dataclose[-1]:
            # 昨天收盘价 < 前天的收盘价
            if self.dataclose[-1] < self.dataclose[-2]:
                # 买入
                self.log('买入, %.2f' % self.dataclose[0])
                self.buy()
        

if __name__ == '__main__':    
	# 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 为Cerebro引擎添加策略
    cerebro.addstrategy(TestStrategy)

    # 获取当前运行脚本所在目录
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    # 拼接加载路径
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    # 创建交易数据集
    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        # 数据必须大于fromdate
        fromdate=datetime.datetime(2000, 1, 1),
        # 数据必须小于todate
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    # 加载交易数据
    cerebro.adddata(data)


    # 设置投资金额100000.0
    cerebro.broker.setcash(100000.0)
    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())
```
运行后的输出结果为：  
```
组合期初资金: 100000.00
2000-01-03, Close, 27.85
2000-01-04, Close, 25.39
2000-01-05, Close, 24.05
2000-01-05, 买入, 24.05
2000-01-06, Close, 22.63
2000-01-06, 买入, 22.63
2000-01-07, Close, 24.37
...
...
...
2000-12-20, 买入, 26.88
2000-12-21, Close, 27.82
2000-12-22, Close, 30.06
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-27, 买入, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
组合期末资金: 99725.08
 ```

若干个买入操作被执行，所以余额在减少。  
细心的朋友可能会问，买了多少？买的什么？订单怎么被执行的？BackTrader框架替我们做了这些事：  
如果没有指定的话，self.datas[0]即是标的物当前交易数据。  
交易数量=仓位数量，默认值等于1，后面例子我们会修改此参数。  

订单被以”市价”成交了。 Broker（经纪人，之前提到过）使用了下一个交易日的开盘价，因为是broker在当前的交日易收盘后天提交的订单，下一个交易日开盘价是他接触到的第一个价格。  
这里没有为订单设置佣金费，后边会加上。  


## 不光有买入，还得有卖出
在知道如何买入（做多）之后，需要知道如何卖出，并且还需要了解该策略是否在市场中。  
Strategy类有一个变量position保存当前持有的资产数量（可以理解为金融术语中的头寸）， 
buy()和sell()会返回被创建的订单(尚未执行的)，
订单状态的更改将通过notify方法通知给策略Strategy。

卖出逻辑也很简单：
5个K线柱后（第6个K线柱）不管涨跌都卖。   
请注意，这里没有指定具体时间，而是指定的柱的数量。一个柱可能代表1分钟、1小时、1天、1星期等等，这取决于你价格数据文件里一条数据代表的周期。  
虽然我们知道每个柱代表一天，但策略并不知道。  
```
因为买入的时候，用的是交易日，所以这里应翻译为：5个交易日（第6个交易日）不管涨跌都卖。
```
另外，当还有头寸的时候，就不再买入了。  

>注意：  
>没有将柱的下标传给next()方法，怎么知道已经经过了5个柱了呢？ 这里用了Python的len()方法获取它Line数据的长度。  
>交易发生时记下它的长度，后边比较大小，看是否经过了5个柱。  


```python
import datetime  # 
import os.path  # 路径管理
import sys  # 获取当前运行脚本的路径 (in argv[0])

#导入backtrader框架
import backtrader as bt

# 创建策略继承bt.Strategy
class TestStrategy(bt.Strategy):

    def log(self, txt, dt=None):
        # 记录策略的执行日志
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 保存收盘价的引用
        self.dataclose = self.datas[0].close
        # 跟踪挂单
        self.order = None


    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # broker 提交/接受了，买/卖订单则什么都不做
            return

        # 检查一个订单是否完成
        # 注意: 当资金不足时，broker会拒绝订单
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log('已买入, %.2f' % order.executed.price)
            elif order.issell():
                self.log('已卖出, %.2f' % order.executed.price)

            # 记录当前交易数量
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('订单取消/保证金不足/拒绝')

        # 其他状态记录为：无挂起订单
        self.order = None

    def next(self):
        # 记录收盘价
        self.log('Close, %.2f' % self.dataclose[0])

        # 如果有订单正在挂起，不操作
        if self.order:
            return

        # 如果没有持仓则买入
        if not self.position:
	        # 今天的收盘价 < 昨天收盘价 
	        if self.dataclose[0] < self.dataclose[-1]:
	            # 昨天收盘价 < 前天的收盘价
	            if self.dataclose[-1] < self.dataclose[-2]:
	                # 买入
	                self.log('买入, %.2f' % self.dataclose[0])
	                 # 跟踪订单避免重复
	                self.order = self.buy()
	    else:
            # 如果已经持仓，且当前交易数据量在买入后5个单位后
            if len(self) >= (self.bar_executed + 5):
                # 全部卖出
                self.log('卖出, %.2f' % self.dataclose[0])
                # 跟踪订单避免重复
                self.order = self.sell()

        

if __name__ == '__main__':    
	# 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 为Cerebro引擎添加策略
    cerebro.addstrategy(TestStrategy)

    # 获取当前运行脚本所在目录
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    # 拼接加载路径
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    # 创建交易数据集
    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        # 数据必须大于fromdate
        fromdate=datetime.datetime(2000, 1, 1),
        # 数据必须小于todate
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    # 加载交易数据
    cerebro.adddata(data)


    # 设置投资金额100000.0
    cerebro.broker.setcash(100000.0)
    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())
```
运行后的输出结果为：  
```
组合期初资金: 100000.00
2000-01-03T00:00:00, Close, 27.85
2000-01-04T00:00:00, Close, 25.39
2000-01-05T00:00:00, Close, 24.05
2000-01-05T00:00:00, 买入, 24.05
2000-01-06T00:00:00, 已买入, 23.61
2000-01-06T00:00:00, Close, 22.63
2000-01-07T00:00:00, Close, 24.37
2000-01-10T00:00:00, Close, 27.29
2000-01-11T00:00:00, Close, 26.49
2000-01-12T00:00:00, Close, 24.90
2000-01-13T00:00:00, Close, 24.77
2000-01-13T00:00:00, 卖出, 24.77
2000-01-14T00:00:00, 已卖出, 25.70
2000-01-14T00:00:00, Close, 25.18
...
...
...
2000-12-15T00:00:00, 卖出, 26.93
2000-12-18T00:00:00, 已卖出, 28.29
2000-12-18T00:00:00, Close, 30.18
2000-12-19T00:00:00, Close, 28.88
2000-12-20T00:00:00, Close, 26.88
2000-12-20T00:00:00, 买入, 26.88
2000-12-21T00:00:00, 已买入, 26.23
2000-12-21T00:00:00, Close, 27.82
2000-12-22T00:00:00, Close, 30.06
2000-12-26T00:00:00, Close, 29.17
2000-12-27T00:00:00, Close, 28.94
2000-12-28T00:00:00, Close, 29.29
2000-12-29T00:00:00, Close, 27.41
2000-12-29T00:00:00, 卖出, 27.41
组合期末资金: 100018.53
 ```

竟然盈利了，是不是哪里出错了？  

## 经纪人说：手续费呢？
这比费用叫做佣金。让我们设定一个常见的费率0.1%，买卖都要收（经纪人就是这么贪）。
一行代码就能搞定:
```Python
cerebro.broker.setcommission(commission=0.001) # 0.001即是0.1%
```
让我看看加不加手续费，结果到底有什么区别。

```python
import datetime  # 
import os.path  # 路径管理
import sys  # 获取当前运行脚本的路径 (in argv[0])

#导入backtrader框架
import backtrader as bt

# 创建策略继承bt.Strategy
class TestStrategy(bt.Strategy):

    def log(self, txt, dt=None):
        # 记录策略的执行日志
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 保存收盘价的引用
        self.dataclose = self.datas[0].close
        # 跟踪挂单
        self.order = None
        # 买入价格和手续费
        self.buyprice = None
        self.buycomm = None

    # 订单状态通知，买入卖出都是下单
    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # broker 提交/接受了，买/卖订单则什么都不做
            return

        # 检查一个订单是否完成
        # 注意: 当资金不足时，broker会拒绝订单
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    '已买入, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            elif order.issell():
                self.log('已卖出, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
            # 记录当前交易数量
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('订单取消/保证金不足/拒绝')

        # 其他状态记录为：无挂起订单
        self.order = None

    # 交易状态通知，一买一卖算交易
    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log('交易利润, 毛利润 %.2f, 净利润 %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        # 记录收盘价
        self.log('Close, %.2f' % self.dataclose[0])

        # 如果有订单正在挂起，不操作
        if self.order:
            return

        # 如果没有持仓则买入
        if not self.position:
            # 今天的收盘价 < 昨天收盘价 
            if self.dataclose[0] < self.dataclose[-1]:
                # 昨天收盘价 < 前天的收盘价
                if self.dataclose[-1] < self.dataclose[-2]:
                    # 买入
                    self.log('买入单, %.2f' % self.dataclose[0])
                     # 跟踪订单避免重复
                    self.order = self.buy()
        else:
            # 如果已经持仓，且当前交易数据量在买入后5个单位后
            if len(self) >= (self.bar_executed + 5):
                # 全部卖出
                self.log('卖出单, %.2f' % self.dataclose[0])
                # 跟踪订单避免重复
                self.order = self.sell()

        

if __name__ == '__main__':    
    # 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 为Cerebro引擎添加策略
    cerebro.addstrategy(TestStrategy)

    # 获取当前运行脚本所在目录
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    # 拼接加载路径
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    # 创建交易数据集
    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        # 数据必须大于fromdate
        fromdate=datetime.datetime(2000, 1, 1),
        # 数据必须小于todate
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    # 加载交易数据
    cerebro.adddata(data)


    # 设置投资金额100000.0
    cerebro.broker.setcash(100000.0)
    # 设置佣金为0.001,除以100去掉%号
    cerebro.broker.setcommission(commission=0.001)

    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())
```
运行后输出
```
组合期初资金: 100000.00
2000-01-03T00:00:00, Close, 27.85
2000-01-04T00:00:00, Close, 25.39
2000-01-05T00:00:00, Close, 24.05
2000-01-05T00:00:00, 买入单, 24.05
2000-01-06T00:00:00, 已买入, 价格: 23.61, 费用: 23.61, 佣金 0.02
2000-01-06T00:00:00, Close, 22.63
2000-01-07T00:00:00, Close, 24.37
2000-01-10T00:00:00, Close, 27.29
2000-01-11T00:00:00, Close, 26.49
2000-01-12T00:00:00, Close, 24.90
2000-01-13T00:00:00, Close, 24.77
2000-01-13T00:00:00, 卖出单, 24.77
2000-01-14T00:00:00, 已卖出, 价格: 25.70, 费用: 25.70, 佣金 0.03
2000-01-14T00:00:00, 操作利润, 毛利润 2.09, 净利润 2.04
2000-01-14T00:00:00, Close, 25.18
...
...
...
2000-12-15T00:00:00, 卖出单, 26.93
2000-12-18T00:00:00, 已卖出, 价格: 28.29, 费用: 28.29, 佣金 0.03
2000-12-18T00:00:00, 操作利润, 毛利润 -0.06, 净利润 -0.12
2000-12-18T00:00:00, Close, 30.18
2000-12-19T00:00:00, Close, 28.88
2000-12-20T00:00:00, Close, 26.88
2000-12-20T00:00:00, 买入单, 26.88
2000-12-21T00:00:00, 已买入, 价格: 26.23, 费用: 26.23, 佣金 0.03
2000-12-21T00:00:00, Close, 27.82
2000-12-22T00:00:00, Close, 30.06
2000-12-26T00:00:00, Close, 29.17
2000-12-27T00:00:00, Close, 28.94
2000-12-28T00:00:00, Close, 29.29
2000-12-29T00:00:00, Close, 27.41
2000-12-29T00:00:00, 卖出单, 27.41
组合期末资金: 100016.98
```

天哪！竟然还是盈利。  
在继续之前，让我们看看这些带有盈亏的操作。  

```
2000-01-14T00:00:00, 操作利润, 毛利润 2.09, 净利润 2.04
2000-02-07T00:00:00, 操作利润, 毛利润 3.68, 净利润 3.63
2000-02-28T00:00:00, 操作利润, 毛利润 4.48, 净利润 4.42
2000-03-13T00:00:00, 操作利润, 毛利润 3.48, 净利润 3.41
2000-03-22T00:00:00, 操作利润, 毛利润 -0.41, 净利润 -0.49
2000-04-07T00:00:00, 操作利润, 毛利润 2.45, 净利润 2.37
2000-04-20T00:00:00, 操作利润, 毛利润 -1.95, 净利润 -2.02
2000-05-02T00:00:00, 操作利润, 毛利润 5.46, 净利润 5.39
2000-05-11T00:00:00, 操作利润, 毛利润 -3.74, 净利润 -3.81
2000-05-30T00:00:00, 操作利润, 毛利润 -1.46, 净利润 -1.53
2000-07-05T00:00:00, 操作利润, 毛利润 -1.62, 净利润 -1.69
2000-07-14T00:00:00, 操作利润, 毛利润 2.08, 净利润 2.01
2000-07-28T00:00:00, 操作利润, 毛利润 0.14, 净利润 0.07
2000-08-08T00:00:00, 操作利润, 毛利润 4.36, 净利润 4.29
2000-08-21T00:00:00, 操作利润, 毛利润 1.03, 净利润 0.95
2000-09-15T00:00:00, 操作利润, 毛利润 -4.26, 净利润 -4.34
2000-09-27T00:00:00, 操作利润, 毛利润 1.29, 净利润 1.22
2000-10-13T00:00:00, 操作利润, 毛利润 -2.98, 净利润 -3.04
2000-10-26T00:00:00, 操作利润, 毛利润 3.01, 净利润 2.95
2000-11-06T00:00:00, 操作利润, 毛利润 -3.59, 净利润 -3.65
2000-11-16T00:00:00, 操作利润, 毛利润 1.28, 净利润 1.23
2000-12-01T00:00:00, 操作利润, 毛利润 2.59, 净利润 2.54
2000-12-18T00:00:00, 操作利润, 毛利润 -0.06, 净利润 -0.12
```
净收益加起来是:15.83  
但系统最后的余额是:100016.98  
很明显 15.83 不等于 16.98。其实没错，净收益指的是已经落到口袋里的钱。  

造成差异的原因是，最后一天还持有头寸。其实卖单已经发出去了，但还没来得及执行。Broker净收益率是按照2000-12-29收盘价算的。实际应该按下一个交易日2001-01-02价格算。  

```
2001-01-02T00:00:00, 已卖出, 价格: 27.87, 费用: 27.87, 佣金 0.03
2001-01-02T00:00:00, 操作利润, 毛利润 1.64, 净利润 1.59
2001-01-02T00:00:00, Close, 24.87
2001-01-02T00:00:00, 买入单, 24.87
组合期末资金: 100017.41
```
加起来之前的净收益:5.83 + 1.59 = 17.42，这个净收益率17.42和最后的余额100017.41就对上了（忽略小数点误差）。

## 自定义策略:技术指标参数
在实战中，一般不将参数写死到策略中。Parameters（参数）就是用来处理这个的。
参数的定义像这样:
`params = (('myparam', 27), ('exitbars', 5),)`
这个 tuple 嵌套看着不方便，格式化一下:
```python
params = (
    ('myparam', 27),
    ('exitbars', 5),
)
```
将策略添加到引擎的时候，可以指定刚才定义的参数:
`cerebro.addstrategy(TestStrategy, myparam=20, exitbars=7)`
>注意
>下面的 setsizing 方法已经被弃用。这里还保留是因为还有一些老示例在用。方法已改为下面这种:
>cerebro.addsizer(bt.sizers.FixedSize, stake=10)
>请参考 sizers 章节。
 
在策略类中使用买卖数量参数很容易，它们被保存在 “params” 参数里。例如，参数已经传入，在策略类里的 __init__ 方法中这样调用就可以了:

```python
# 根据传入的参加设置买卖数量
self.sizer.setsizing(self.params.stake)
```
也可以直接将买卖数量传入buy和sell方法。
卖出的逻辑改为:
```python
# 已经持有，可以卖出了
if len(self) >= (self.bar_executed + self.params.exitbars)
```
代码修改为:
```python
import datetime  # 
import os.path  # 路径管理
import sys  # 获取当前运行脚本的路径 (in argv[0])

#导入backtrader框架
import backtrader as bt

# 创建策略继承bt.Strategy
class TestStrategy(bt.Strategy):
    params = (
        # 持仓够5个单位就卖出
        ('exitbars', 5),
    )

    def log(self, txt, dt=None):
        # 记录策略的执行日志
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 保存收盘价的引用
        self.dataclose = self.datas[0].close
        # 跟踪挂单
        self.order = None
        # 买入价格和手续费
        self.buyprice = None
        self.buycomm = None

    # 订单状态通知，买入卖出都是下单
    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # broker 提交/接受了，买/卖订单则什么都不做
            return

        # 检查一个订单是否完成
        # 注意: 当资金不足时，broker会拒绝订单
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    '已买入, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            elif order.issell():
                self.log('已卖出, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
            # 记录当前交易数量
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('订单取消/保证金不足/拒绝')

        # 其他状态记录为：无挂起订单
        self.order = None

    # 交易状态通知，一买一卖算交易
    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log('交易利润, 毛利润 %.2f, 净利润 %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        # 记录收盘价
        self.log('Close, %.2f' % self.dataclose[0])

        # 如果有订单正在挂起，不操作
        if self.order:
            return

        # 如果没有持仓则买入
        if not self.position:
            # 今天的收盘价 < 昨天收盘价 
            if self.dataclose[0] < self.dataclose[-1]:
                # 昨天收盘价 < 前天的收盘价
                if self.dataclose[-1] < self.dataclose[-2]:
                    # 买入
                    self.log('买入单, %.2f' % self.dataclose[0])
                     # 跟踪订单避免重复
                    self.order = self.buy()
        else:
            # 如果已经持仓，且当前交易数据量在买入后5个单位后
            # 此处做了更新将5替换为参数
            if len(self) >= (self.bar_executed + self.params.exitbars):
                # 全部卖出
                self.log('卖出单, %.2f' % self.dataclose[0])
                # 跟踪订单避免重复
                self.order = self.sell()

        

if __name__ == '__main__':    
    # 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 为Cerebro引擎添加策略
    cerebro.addstrategy(TestStrategy)

    # 获取当前运行脚本所在目录
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    # 拼接加载路径
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    # 创建交易数据集
    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        # 数据必须大于fromdate
        fromdate=datetime.datetime(2000, 1, 1),
        # 数据必须小于todate
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    # 加载交易数据
    cerebro.adddata(data)


    # 设置投资金额1000000.0
    cerebro.broker.setcash(1000000.0)

    # 每笔交易使用固定交易量
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    # 设置佣金为0.001,除以100去掉%号
    cerebro.broker.setcommission(commission=0.001)

    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())
```
运行后的输出为:
```
组合期初资金: 1000000.00

2000-01-03T00:00:00, Close, 27.85
2000-01-04T00:00:00, Close, 25.39
2000-01-05T00:00:00, Close, 24.05
2000-01-05T00:00:00, 买单, 24.05
2000-01-06T00:00:00, 已买入, 交易量 10, 价格: 23.61, 费用: 236.10, 佣金 0.24
2000-01-06T00:00:00, Close, 22.63
...
...
...
2000-12-20T00:00:00, 买入单, 26.88
2000-12-21T00:00:00, 已买入, 交易量 10, 价格: 26.23, 费用: 262.30, 佣金 0.26
2000-12-21T00:00:00, Close, 27.82
2000-12-22T00:00:00, Close, 30.06
2000-12-26T00:00:00, Close, 29.17
2000-12-27T00:00:00, Close, 28.94
2000-12-28T00:00:00, Close, 29.29
2000-12-29T00:00:00, Close, 27.41
2000-12-29T00:00:00, 卖出单, 27.41
组合期末资金: 100169.80
```
为了显示改变已生效，输出中显示了买卖数量。
买卖数量改为了原来的10倍，盈亏也变为了原来的10倍，变为了169.80 。

## 添加技术指标
之前我提到过indicators（技术指标），下一步就该添加他们了，要做的肯定比“三连跌”这种复杂点。
借用PyAlgoTrade这个框架的一个使用移动平均线的例子：
*收盘价高于平均价的时候，以市价买入
*持有仓位的时候，如果收盘价低于平均价，卖出
*只有一个待执行的订单

大多数代码不用改变，在 __init__ 方法中加入移动平均线的实例化:
`self.sma = bt.indicators.MovingAverageSimple(self.datas[0], period=self.params.maperiod)`
当然买入卖出的逻辑依赖平均价，具体代码如下。

>注意
>起始金额为1000元，无手续费，这和 PyAlgoTrade 保持一致。

```python
import datetime  # 
import os.path  # 路径管理
import sys  # 获取当前运行脚本的路径 (in argv[0])

#导入backtrader框架
import backtrader as bt

# 创建策略继承bt.Strategy
class TestStrategy(bt.Strategy):
    params = (
        # 均线参数设置15天，15日均线
        ('maperiod', 15),
    )

    def log(self, txt, dt=None):
        # 记录策略的执行日志
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 保存收盘价的引用
        self.dataclose = self.datas[0].close
        # 跟踪挂单
        self.order = None
        # 买入价格和手续费
        self.buyprice = None
        self.buycomm = None
        # 加入均线指标
        self.sma = bt.indicators.SimpleMovingAverage(self.datas[0], period=self.params.maperiod)
 

    # 订单状态通知，买入卖出都是下单
    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # broker 提交/接受了，买/卖订单则什么都不做
            return

        # 检查一个订单是否完成
        # 注意: 当资金不足时，broker会拒绝订单
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    '已买入, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            elif order.issell():
                self.log('已卖出, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
            # 记录当前交易数量
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('订单取消/保证金不足/拒绝')

        # 其他状态记录为：无挂起订单
        self.order = None

    # 交易状态通知，一买一卖算交易
    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log('交易利润, 毛利润 %.2f, 净利润 %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        # 记录收盘价
        self.log('Close, %.2f' % self.dataclose[0])

        # 如果有订单正在挂起，不操作
        if self.order:
            return

        # 如果没有持仓则买入
        if not self.position:
            # 今天的收盘价在均线价格之上 
            if self.dataclose[0] > self.sma[0]: 
                # 买入
                self.log('买入单, %.2f' % self.dataclose[0])
                    # 跟踪订单避免重复
                self.order = self.buy()
        else:
            # 如果已经持仓，收盘价在均线价格之下
            if self.dataclose[0] < self.sma[0]:
                # 全部卖出
                self.log('卖出单, %.2f' % self.dataclose[0])
                # 跟踪订单避免重复
                self.order = self.sell()

        

if __name__ == '__main__':    
    # 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 为Cerebro引擎添加策略
    cerebro.addstrategy(TestStrategy)

    # 获取当前运行脚本所在目录
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    # 拼接加载路径
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    # 创建交易数据集
    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        # 数据必须大于fromdate
        fromdate=datetime.datetime(2000, 1, 1),
        # 数据必须小于todate
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    # 加载交易数据
    cerebro.adddata(data)


    # 设置投资金额1000.0
    cerebro.broker.setcash(1000.0)

    # 每笔交易使用固定交易量
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    # 设置佣金为0.0
    cerebro.broker.setcommission(commission=0.0)

    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())
```
让我们仔细地看一下出现在下面日志中的第一条记录：
不再是新千年的第一个交易日2000-01-03了。
变成了2000-01-24，怎么回事呢？
是因为，框架根据新代码做出了改变：
在策略中我们加入了移动平均技术指标。
移动平均需要有个均线周期参数，程序根据这个参数回看计算前边的X条价格数据然后进行开仓判断，例子中周期是15。
2000-01-24就是第15天
backtrader 框架假定策略加入这个技术指标是有正当理由的，比如做开平仓的决策。
框架不会在数据没到位的时候就进行下一步。
在技术指标产生第一条数据之后，next方法第一个被调用。
在示例中只有一个技术指标，其实策略支持添加多个技术指标。

运行后的输出为:
```
组合期初资金: 1000.00
2000-01-24T00:00:00, Close, 25.55
2000-01-25T00:00:00, Close, 26.61
2000-01-25T00:00:00, 买入单, 26.61
2000-01-26T00:00:00, 已买入, 数量 10, 价格: 26.76, 费用: 267.60, 佣金 0.00
2000-01-26T00:00:00, Close, 25.96
2000-01-27T00:00:00, Close, 24.43
2000-01-27T00:00:00, 卖出单, 24.43
2000-01-28T00:00:00, 已卖出, 数量 10, 价格: 24.28, 费用: 242.80, 佣金 0.00
2000-01-28T00:00:00, 操作利润, 毛利润 -24.80, 净利润 -24.80
2000-01-28T00:00:00, Close, 22.34
2000-01-31T00:00:00, Close, 23.55
2000-02-01T00:00:00, Close, 25.46
2000-02-02T00:00:00, Close, 25.61
2000-02-02T00:00:00, 买入单, 25.61
2000-02-03T00:00:00, 已买入, 数量 10, 价格: 26.11, 费用: 261.10, 佣金 0.00
...
...
...
2000-12-20T00:00:00, 卖出单, 26.88
2000-12-21T00:00:00, 已卖出, 数量 10, 价格: 26.23, 费用: 262.30, 佣金 0.00
2000-12-21T00:00:00, 操作利润, 毛利润 -20.60, 净利润 -20.60
2000-12-21T00:00:00, Close, 27.82
2000-12-21T00:00:00, 买入单, 27.82
2000-12-22T00:00:00, 已买入, 数量 10, 价格: 28.65, 费用: 286.50, 佣金 0.00
2000-12-22T00:00:00, Close, 30.06
2000-12-26T00:00:00, Close, 29.17
2000-12-27T00:00:00, Close, 28.94
2000-12-28T00:00:00, Close, 29.29
2000-12-29T00:00:00, Close, 27.41
2000-12-29T00:00:00, 卖出单, 27.41
组合期末资金: 973.90
```

一个盈利系统被改变之后开始亏损了，还是在手续费率设置为0的情况下。看来简单添加一个技术指标并不是万能的。 

>注意
>同样的交易逻辑和数据，和PyAlgoTrade输出的结果并不完全一致，当然只是轻微不一致。最可疑的原因是因为：小数点
>处理”调整后价格”（分红、拆股后调整）时，PyAlgoTrade并不对小数点进行四舍五入。
>在对价格进行调整后，backtrader的数据引擎将Yahoo价格数据的价格小数点缩减到2位。虽然输出看起来差不多，但积少成多结果就不同了。
>将价格小数点缩减到2位是合理的，一般交易所只允许价格保留小数点后面2位。
>从 1.8.11.99 版本开始，backtrader的Yahoo数据引擎可以设置是否做小数点位数保留，还可以设置保留多少位。

## 可视化:绘图
文字日志虽然能看到细节，但人们还是喜欢看可视化的东西，所以有必要将结果绘制成图表。
绘图很容易使用，只需添加一行代码:
`cerebro.plot()`

这行代码要放在 cerebro.run() 之后。
为方便使用，框架做了下面这些自动化的事情： 
将添加第二条指数移动平均线，默认将使用数据进行绘制（就像第1条）。
将添加第三条加权移动平均线，在单独区域绘制（也许看起来不合理）。
将添加一条Stochastic（慢），使用默认参数。
将添加一条MACD，使用默认参数。
将添加一条RSI指标，使用默认参数。
将添加一条RSI指标的简单移动平均线，使用默认参数（将和RSI一起被绘制）。
将添加一条ATR指标，修改了默认参数以避免被绘制。
 
上面添加的这些指标，等于在策略类的 __init__ 方法中添加了以下语句:
```python
# 需要绘制的指标
bt.indicators.ExponentialMovingAverage(self.datas[0], period=25)
bt.indicators.WeightedMovingAverage(self.datas[0], period=25).subplot = True
bt.indicators.StochasticSlow(self.datas[0])
bt.indicators.MACDHisto(self.datas[0])
rsi = bt.indicators.RSI(self.datas[0])
bt.indicators.SmoothedMovingAverage(rsi, period=10)
bt.indicators.ATR(self.datas[0]).plot = False
```

>注意
>即使指标没有被显式地声明为成员变量（如 self.sma = MovingAverageSimple…）， 它们还是会被自动注册到策略类中，并影响开始执行 next 的最小周期，而且会被绘制。
>在例子中，只有RSI的指标被赋予了一个rsi的变量，供后边为它创建移动平均线使用。
 
现在程序变成了这样：

```python
import datetime  # 
import os.path  # 路径管理
import sys  # 获取当前运行脚本的路径 (in argv[0])

#导入backtrader框架
import backtrader as bt

# 创建策略继承bt.Strategy
class TestStrategy(bt.Strategy):
    params = (
        # 均线参数设置15天，15日均线
        ('maperiod', 15),
    )

    def log(self, txt, dt=None):
        # 记录策略的执行日志
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 保存收盘价的引用
        self.dataclose = self.datas[0].close
        # 跟踪挂单
        self.order = None
        # 买入价格和手续费
        self.buyprice = None
        self.buycomm = None
        # 加入均线指标
        self.sma = bt.indicators.SimpleMovingAverage(self.datas[0], period=self.params.maperiod)

        # 绘制图形时候用到的指标
        bt.indicators.ExponentialMovingAverage(self.datas[0], period=25)
        bt.indicators.WeightedMovingAverage(self.datas[0], period=25,subplot=True)
        bt.indicators.StochasticSlow(self.datas[0])
        bt.indicators.MACDHisto(self.datas[0])
        rsi = bt.indicators.RSI(self.datas[0])
        bt.indicators.SmoothedMovingAverage(rsi, period=10)
        bt.indicators.ATR(self.datas[0], plot=False)
 

    # 订单状态通知，买入卖出都是下单
    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # broker 提交/接受了，买/卖订单则什么都不做
            return

        # 检查一个订单是否完成
        # 注意: 当资金不足时，broker会拒绝订单
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    '已买入, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            elif order.issell():
                self.log('已卖出, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
            # 记录当前交易数量
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('订单取消/保证金不足/拒绝')

        # 其他状态记录为：无挂起订单
        self.order = None

    # 交易状态通知，一买一卖算交易
    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log('交易利润, 毛利润 %.2f, 净利润 %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        # 记录收盘价
        self.log('Close, %.2f' % self.dataclose[0])

        # 如果有订单正在挂起，不操作
        if self.order:
            return

        # 如果没有持仓则买入
        if not self.position:
            # 今天的收盘价在均线价格之上 
            if self.dataclose[0] > self.sma[0]: 
                # 买入
                self.log('买入单, %.2f' % self.dataclose[0])
                    # 跟踪订单避免重复
                self.order = self.buy()
        else:
            # 如果已经持仓，收盘价在均线价格之下
            if self.dataclose[0] < self.sma[0]:
                # 全部卖出
                self.log('卖出单, %.2f' % self.dataclose[0])
                # 跟踪订单避免重复
                self.order = self.sell()

        

if __name__ == '__main__':    
    # 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 为Cerebro引擎添加策略
    cerebro.addstrategy(TestStrategy)

    # 获取当前运行脚本所在目录
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    # 拼接加载路径
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    # 创建交易数据集
    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        # 数据必须大于fromdate
        fromdate=datetime.datetime(2000, 1, 1),
        # 数据必须小于todate
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    # 加载交易数据
    cerebro.adddata(data)


    # 设置投资金额1000.0
    cerebro.broker.setcash(1000.0)

    # 每笔交易使用固定交易量
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    # 设置佣金为0.0
    cerebro.broker.setcommission(commission=0.0)

    # 引擎运行前打印期出资金
    print('组合期初资金: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    # 引擎运行后打期末资金
    print('组合期末资金: %.2f' % cerebro.broker.getvalue())

    # 绘制图像
    cerebro.plot()
```
执行后的输出结果为:
```
组合期初资金: 1000.00
2000-02-18T00:00:00, Close, 27.61
2000-02-22T00:00:00, Close, 27.97
2000-02-22T00:00:00, 买入单, 27.97
2000-02-23T00:00:00, 已买入, 数量 10, 价格: 28.38, 费用: 283.80, 佣金 0.00
2000-02-23T00:00:00, Close, 29.73
...
...
...
2000-12-21T00:00:00, 买入单, 27.82
2000-12-22T00:00:00, 已买入, 数量 10, 价格: 28.65, 费用: 286.50, 佣金 0.00
2000-12-22T00:00:00, Close, 30.06
2000-12-26T00:00:00, Close, 29.17
2000-12-27T00:00:00, Close, 28.94
2000-12-28T00:00:00, Close, 29.29
2000-12-29T00:00:00, Close, 27.41
2000-12-29T00:00:00, SELL CREATE, 27.41
组合期末资金: 981.00
```

虽然策略逻辑没有变，但回测结果却变了。这是由于bar的数量发生了变化。

>注意：
>前面提到过，框架会等待所有指标数据到位之后，才会运行next函数。 在上例中，MACD是最后一个数据到位的指标（它的3条线都完成了输出）。
>所以第一笔下单已经不是2000年1月份了，而是2000年2月份末。
图表如下: 
![backtrader第一个策略](https://www.backtrader.com/docu/quickstart/quickstart10.png)

## 参数调优

许多交易书籍都会说每个市场、每只股票（或期货等等）都有不同的节奏，也就是说没有一个参数能适应所有。
在之前的例子里，策略里使用的默认参数是15。这个参数可以被更换并进行测试，以评估什么值更适合于市场。

>注意
>大量文献讨论了关于优化的优缺点。一般建议都会指向同一方向：不要过度优化。如果策略不理想，而在拟合上下功夫，则可能产生一个在回测数据上非常优秀的参数，但这个参数在将来表现可能并不好。

修改了代码，以测试移动平均线的最优周期参数。为保持清新，删除了所有买入、卖出的输出。
修改后的例子：

```python
import datetime  # 
import os.path  # 路径管理
import sys  # 获取当前运行脚本的路径 (in argv[0])

#导入backtrader框架
import backtrader as bt

# 创建策略继承bt.Strategy
class TestStrategy(bt.Strategy):
    params = (
        # 均线参数设置15天，15日均线
        ('maperiod', 15),
        ('printlog', False),
    )

    def log(self, txt, dt=None):
        # 记录策略的执行日志
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 保存收盘价的引用
        self.dataclose = self.datas[0].close
        # 跟踪挂单
        self.order = None
        # 买入价格和手续费
        self.buyprice = None
        self.buycomm = None
        # 加入均线指标
        self.sma = bt.indicators.SimpleMovingAverage(self.datas[0], period=self.params.maperiod)
 

    # 订单状态通知，买入卖出都是下单
    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # broker 提交/接受了，买/卖订单则什么都不做
            return

        # 检查一个订单是否完成
        # 注意: 当资金不足时，broker会拒绝订单
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    '已买入, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            elif order.issell():
                self.log('已卖出, 价格: %.2f, 费用: %.2f, 佣金 %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
            # 记录当前交易数量
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('订单取消/保证金不足/拒绝')

        # 其他状态记录为：无挂起订单
        self.order = None

    # 交易状态通知，一买一卖算交易
    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log('交易利润, 毛利润 %.2f, 净利润 %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        # 记录收盘价
        self.log('Close, %.2f' % self.dataclose[0])

        # 如果有订单正在挂起，不操作
        if self.order:
            return

        # 如果没有持仓则买入
        if not self.position:
            # 今天的收盘价在均线价格之上 
            if self.dataclose[0] > self.sma[0]: 
                # 买入
                self.log('买入单, %.2f' % self.dataclose[0])
                    # 跟踪订单避免重复
                self.order = self.buy()
        else:
            # 如果已经持仓，收盘价在均线价格之下
            if self.dataclose[0] < self.sma[0]:
                # 全部卖出
                self.log('卖出单, %.2f' % self.dataclose[0])
                # 跟踪订单避免重复
                self.order = self.sell()
    
    # 测略结束时，多用于参数调优
    def stop(self):
        self.log('(均线周期 %2d)期末资金 %.2f' %
                 (self.params.maperiod, self.broker.getvalue()), doprint=True)


if __name__ == '__main__':    
    # 创建Cerebro引擎
    cerebro = bt.Cerebro()
    # Cerebro引擎在后台创建broker(经纪人)，系统默认资金量为10000

    # 为Cerebro引擎添加策略
    # cerebro.addstrategy(TestStrategy)

    # 为Cerebro引擎添加策略, 优化策略
    # 使用参数来设定10到31天的均线,看看均线参数下那个收益最好
    strats = cerebro.optstrategy(
        TestStrategy,
        maperiod=range(10, 31))

    # 获取当前运行脚本所在目录
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    # 拼接加载路径
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    # 创建交易数据集
    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        # 数据必须大于fromdate
        fromdate=datetime.datetime(2000, 1, 1),
        # 数据必须小于todate
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    # 加载交易数据
    cerebro.adddata(data)


    # 设置投资金额1000.0
    cerebro.broker.setcash(1000.0)

    # 每笔交易使用固定交易量
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    # 设置佣金为0.0
    cerebro.broker.setcommission(commission=0.0)


    cerebro.run()

```
这次没有调用addstrategy，而是用optstrategy函数将策略添加到 Cerebro。传入的是要测试的一系列值，而不是单个值。
在策略类中添加了stop 方法，它将在每轮回测之后被调用，我们用它来打印回测结束之后的资产余额（之前在Cerebro做的）。
框架将为策略测试每个参数值，下面是输出结果:
```
2000-12-29, (均线周期10) 期末资金 880.30
2000-12-29, (均线周期11) 期末资金 880.00
2000-12-29, (均线周期12) 期末资金 830.30
2000-12-29, (均线周期13) 期末资金 893.90
2000-12-29, (均线周期14) 期末资金 896.90
2000-12-29, (均线周期15) 期末资金 973.90
2000-12-29, (均线周期16) 期末资金 959.40
2000-12-29, (均线周期17) 期末资金 949.80
2000-12-29, (均线周期18) 期末资金 1011.90
2000-12-29, (均线周期19) 期末资金 1041.90
2000-12-29, (均线周期20) 期末资金 1078.00
2000-12-29, (均线周期21) 期末资金 1058.80
2000-12-29, (均线周期22) 期末资金 1061.50
2000-12-29, (均线周期23) 期末资金 1023.00
2000-12-29, (均线周期24) 期末资金 1020.10
2000-12-29, (均线周期25) 期末资金 1013.30
2000-12-29, (均线周期26) 期末资金 998.30
2000-12-29, (均线周期27) 期末资金 982.20
2000-12-29, (均线周期28) 期末资金 975.70
2000-12-29, (均线周期29) 期末资金 983.30
2000-12-29, (均线周期30) 期末资金 979.80
```
结果显示：
周期参数在18以下的亏损（在没有手续费的情况下）;
周期参数在18至26之间的盈利;
周期参数大于26的又会亏损;
 
对这个策略来说，最优的参数是：回看周期20，本金1000，盈利78元，收益率7.8%。
 
>注意
>在上面的例子中，移除了多余的用来绘图的指标，数据开始回测的时间仅取决于我们添加的简单移动平均线。所以周期为15的回测结果和之前的略有不同。

总结

上面的教程，我们从一个头开始，一步步搭建了一个能运行的回测系统，并且具备绘制结果和优化参数功能。
除此之外，还能做一些提高胜率的事情：
自定义指标：创建自定义指标很容易，绘制它们同样简单；
下单数量：资金管理是交易成功的关键之一；
委托单类型（限价单、止损单、限价止损单）等等。
 
 
阅读后续章节，获取相关功能的介绍。
运气很重要，祝好运～