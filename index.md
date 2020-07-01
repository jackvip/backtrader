# 声明

本文档是基于backtrader官方文档的翻译  
官方文档地址 https://www.backtrader.com/docu/  
文档回测数据文件 https://github.com/jackvip/backtrader/blob/master/orcl-1995-2014.txt

# 介绍

## backtrader的2个目标

1. 使用简单
2. 参考1

## backtrader的运行流程

1. 制定策略1.1 确定潜在的可调参数1.2 实例化您在策略中需要的指标1.3 写下进入/退出市场的逻辑
2. 创建Cerebro引擎（西班牙语大脑的意思）
   2.1 注入策略
   2.2 使用cerebro.adddata加载回测数据
   2.3 执行cerebro.run
   2.4 使用cerebro.plot绘制可视化图表

backtrader是高度可配置的，希望大家发现其中的乐趣。

# 安装

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

# 快速开始

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

代码量略有增加，因为我们添加了：找出数据文件所在的路径通过datetime对象过滤我们想要操作的数据

> 注意:
> Yahoo的价格数据有点非主流，它是以时间倒序排列的。datetime.datetime()中的reversed=True参数是将顺序反转一次，这样就得到了我们想要的正序数据。

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

> 注意：
> 没有将柱的下标传给next()方法，怎么知道已经经过了5个柱了呢？ 这里用了Python的len()方法获取它Line数据的长度。
> 交易发生时记下它的长度，后边比较大小，看是否经过了5个柱。

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

```python
# 参数的定义像这样: 
params = (('myparam', 27), ('exitbars', 5),)
```

这个 tuple 嵌套看着不方便，格式化一下:

```python
params = (
    ('myparam', 27),
    ('exitbars', 5),
)
```

将策略添加到引擎的时候，可以指定刚才定义的参数:

```python
# 指定参数
cerebro.addstrategy(TestStrategy, myparam=20, exitbars=7)
```

> 注意：
> 下面的 setsizing 方法已经被弃用。这里还保留是因为还有一些老示例在用。方法已改为下面这种:
> cerebro.addsizer(bt.sizers.FixedSize, stake=10)
> 请参考 sizers 章节。

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

之前我提到过indicators（技术指标），下一步就该添加他们了，要做的肯定比“三连跌”这种复杂点。借用PyAlgoTrade这个框架的一个使用移动平均线的例子：

* 收盘价高于平均价的时候，以市价买入
* 持有仓位的时候，如果收盘价低于平均价，卖出
* 只有一个待执行的订单

大多数代码不用改变，在 __init__ 方法中加入移动平均线的实例化:

```python
# 加入移动平均线
self.sma = bt.indicators.MovingAverageSimple(self.datas[0], period=self.params.maperiod)
```

当然买入卖出的逻辑依赖平均价，具体代码如下。

> 注意：
> 起始金额为1000元，无手续费，这和 PyAlgoTrade 保持一致。

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

> 注意：
> 同样的交易逻辑和数据，和PyAlgoTrade输出的结果并不完全一致，当然只是轻微不一致。最可疑的原因是因为：小数点
> 处理”调整后价格”（分红、拆股后调整）时，PyAlgoTrade并不对小数点进行四舍五入。
> 在对价格进行调整后，backtrader的数据引擎将Yahoo价格数据的价格小数点缩减到2位。虽然输出看起来差不多，但积少成多结果就不同了。
> 将价格小数点缩减到2位是合理的，一般交易所只允许价格保留小数点后面2位。

> 从 1.8.11.99 版本开始，backtrader的Yahoo数据引擎可以设置是否做小数点位数保留，还可以设置保留多少位。

## 可视化:绘图

文字日志虽然能看到细节，但人们还是喜欢看可视化的东西，所以有必要将结果绘制成图表。
绘图很容易使用，只需添加一行代码:

```python
# 绘制图像
cerebro.plot()
```

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

> 注意：
> 即使指标没有被显式地声明为成员变量（如 self.sma = MovingAverageSimple…）， 它们还是会被自动注册到策略类中，并影响开始执行 next 的最小周期，而且会被绘制。
> 在例子中，只有RSI的指标被赋予了一个rsi的变量，供后边为它创建移动平均线使用。

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

> 注意：
> 前面提到过，框架会等待所有指标数据到位之后，才会运行next函数。 在上例中，MACD是最后一个数据到位的指标（它的3条线都完成了输出）。
> 所以第一笔下单已经不是2000年1月份了，而是2000年2月份末。

图表如下:
![backtrader第一个策略](https://www.backtrader.com/docu/quickstart/quickstart10.png)

## 参数调优

许多交易书籍都会说每个市场、每只股票（或期货等等）都有不同的节奏，也就是说没有一个参数能适应所有。在之前的例子里，策略里使用的默认参数是15。这个参数可以被更换并进行测试，以评估什么值更适合于市场。

> 注意：
> 大量文献讨论了关于优化的优缺点。一般建议都会指向同一方向：不要过度优化。如果策略不理想，而在拟合上下功夫，则可能产生一个在回测数据上非常优秀的参数，但这个参数在将来表现可能并不好。

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

这次没有调用addstrategy，而是用optstrategy函数将策略添加到Cerebro。传入的是要测试的一系列值，而不是单个值。
在策略类中添加了stop 方法，它将在每轮回测之后被调用，我们用它来打印回测结束之后的资产余额。
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

> 注意
> 在上面的例子中，移除了多余的用来绘图的指标，数据开始回测的时间仅取决于我们添加的简单移动平均线。所以周期为15的回测结果和之前的略有不同。

总结

上面的教程，我们从一个头开始，一步步搭建了一个能运行的回测系统，并且具备绘制结果和优化参数功能。
除此之外，还能做一些提高胜率的事情：
自定义指标：创建自定义指标很容易，绘制它们同样简单；
下单数量：资金管理是交易成功的关键之一；
委托单类型（限价单、止损单、限价止损单）等等。

阅读后续章节，获取相关功能的介绍。
运气很重要，祝好运～

# 一些概念

这是backtrader一些概念的集合。了解这些概念对，对于使用平台很有帮助。

## 开始之前

所有的代码示例，需要导入以下库才能使用：

```python
import backtrader as bt
import backtrader.indicators as btind
import backtrader.feeds as btfeeds
```

> 注意：
> 访问子模块的另一种语法：
> 以bt形式导入backtrader
> 然后：
> thefeed = bt.feeds.OneOfTheFeeds（...）
> theind = bt.indicators.SimpleMovingAverage（...）

## 传递交易数据

该平台的基础工作都是由“策略”完成。 将交易数据传递给策略，用户无需关心怎么接收收据。
交易数据以数组的形式传递给“策略”，并作为“策略”的成员变量，可以通过数组下标的方式快捷访问。
快速预览一下策略派生类的声明和框架的运行

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.datas[0], period=self.params.period)
    ...

cerebro = bt.Cerebro()
...
data = btfeeds.MyFeed(...)
cerebro.adddata(data)
...
cerebro.addstrategy(MyStrategy, period=30)
...
```

请注意以下几点：策略的构造方法__init__中并未接收到* args或** kwargs任何参数（但是它们仍可以使用），成员变量self.datas，该成员变量为数组/列表/可迭代，至少要要有一条记录（否则将引发异常）。就是这样，交易数据已添加到框架中，并且将按照添加到系统中的顺序显示在策略中。

> 注意：
> 这种方式，同样适用于框架源码中的现有指标，或者用户开发的自定义指标。

### 交易数据快捷访问

可以使用其他自动成员变量直接访问self.datas数组项：

* 通过self.data访问self.datas[0]
* 通过self.dataX访问self.datas[X]
  例如：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.data, period=self.params.period)
    ...
```

### 省略交易数据

上面的示例可以进一步简化为：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(period=self.params.period)
    ...
```

self.data已从SimpleMovingAverage的调用中完全删除，SimpleMovingAverage默认首个参数就是self.data也即self.data0（self.data[0]）。

### 一切皆是数据源

不仅仅交易数据是数据源，可以传递给测策略， 指标和操作结果同样也是数据源。

在前面的示例中，SimpleMovingAverage接收self.datas[0]作为要进行操作的输入。下面看一个操作结果和额外指标的示例：

```python
class MyStrategy(bt.Strategy):
    params = dict(period1=20, period2=25, period3=10, period4)

    def __init__(self):
        sma1 = btind.SimpleMovingAverage(self.datas[0], period=self.p.period1)
      
        # 第二移动平均线使用sma1作为参数，均线的均线
        sma2 = btind.SimpleMovingAverage(sma1, period=self.p.period2)

        # 通过算术运算创建的新数据
        something = sma2 - sma1 + self.data.close

        # 第三移动平均线使用something作为参数
        sma3 = btind.SimpleMovingAverage(something, period=self.p.period3)

        # 比较sma3和sma1...
        greater = sma3 > sma1

        # 并没有实际意义的均值
        # 第四移动平均线使用greater做为参数
        sma3 = btind.SimpleMovingAverage(greater, period=self.p.period4)
    ...
```

基本上，所有内容都会转换为一个对象，一旦对其进行操作，就可以用作数据源。

## 参数

通常，平台中的所有其他类都支持参数的概念。
参数和默认值一起声明为类的属性（元组或类似字典的对象）。
扫描关键字args（** kwargs）以查找匹配的参数，如果找到则将它们从** kwargs中删除，并将值分配给相应的参数。
通过访问成员变量self.params（简写为self.p），最终可以在类的实例中使用参数。
先前的简洁策略已经包含一个参数示例，这里我们再次聚焦参数的使用。

通过元组方式：

```python
class MyStrategy(bt.Strategy):
    params = (('period', 20),)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.data, period=self.p.period)
```

通过字典方式：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.data, period=self.p.period)
```

## Lines（线群）

同样，平台中几乎所有对象都是使用了Lines（线群）对象。 从用户的角度来看，这意味着：

* Lines（线群）对象可以容纳一个或多个线，线是由一组数据组成的数组，这组值在图表中放在一起就可以形成一条线。
  线的一个很好的例子是由股票的收盘价形成的线，这实际上就是我们众所周知的收盘价曲线。

框架中对线的使用通常就是读取操作， 前面的小例子少加改造如下：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):

        self.movav = btind.SimpleMovingAverage(self.data, period=self.p.period)

    def next(self):
        if self.movav.lines.sma[0] > self.data.lines.close[0]:
            print('移动平均线大于收盘价')
```

展现了拥有线的两个对象：

* self.data具有一个lines属性，该属性包含一个close属性
* self.movav是一个SimpleMovingAverage指标，也具有一个lines属性，该属性包含一个sma属性
  close和sma通过访问(索引0)来比较值的大小

> 注意：
> 由此可见，线只是一个名字而已。可以按照声明时的顺序访问它，但这仅适用于指标开发的过程中。

也可以通过快捷方式访问线：

* xxx.lines可以缩短为xxx.l
* xxx.lines.name可以缩写为xxx.lines_name
* 诸如策略和指标之类的复杂对象可以快速访问线数据
* + self.data_name是对self.data.lines.name的直接访问
* + 同样适用于编号的data变量：self.data1_name -> self.data1.lines.name

此外，也可以通过以下方式直接访问线的属性：

* self.data.close
* self.movav.sma
  但是，实际开发中，这种快捷访问的含义不如之前的清晰。

> 注意：
> 不支持使用后面的两种标记方式来给lines赋值

### Lines（线群）声明

如果开发一个指标，则该指标所拥有的Lines必须要声明。
就像params一样，但是只能作为元组类型的成员属性，不支持字典，因为Lines不按照插入顺序存储内容。
对于简单移动平均线，可以这样进行声明：

```python
class SimpleMovingAverage(Indicator):
    lines = ('sma',)

    ...
```

> 注意：
> 如果您将单个字符串传递给元组，则在元组中声明后的添加逗号。否则，字符串中的每个字母都将被解释为要添加到元组的项。 这可能是Python语法中少数不合理的几个地方之一。

如上例所示，该声明在指标中创建了一条sma线，以后可以在该策略进行访问（并可能传递给其他指标来创建更复杂的指标）。

对于开发而言，通过非命名的方式访问lines会很有用，这也是使用数字索引访问的方便之处：

* self.lines[0] 指向 self.lines.sma

如果定义了更多的线，可以将使用索引1、2或者更高的索引进行访问。当然，也确实存在快捷访问版本：

* self.line 指向 to self.lines[0]
* self.lineX 指向 self.lines[X]
* self.line_X 指向 self.lines[X]

交易数据对象也可以通过数字快速访问对象内部的lines：

* self.dataY 指向 self.data.lines[Y]
* self.dataX_Y 指向 self.dataX.lines[X] 这是 self.datas[X].lines[Y] 的缩写版本。

### 访问交易数据中的lines（线群）

在交易数据内部，也可以省略的方式来访问lines，比如可以用比较自然的方式访问收盘价：

```python
data = btfeeds.BacktraderCSVData(dataname='mydata.csv')
...
class MyStrategy(bt.Strategy):
    ...
    def next(self):
        if self.data.close[0] > 30.0:
            ...
```

这种方式肯定比 self.data.lines.close[0] > 30.0: 更加自然些。但是这不适用于以下指标：

* 指标只具有一个属性close，该属性包含一个中间计算，该中间计算随后的结果赋值给lines中的close
  对于交易数据，不会进行任何计算，因为它只是一个数据源。

### 线的长度

Lines是一组点的集合，并且在执行过程中会动态增长，因此可以通过调用Python中的len函数随时测量长度。
这适用于：

* 交易数据
* 策略
* 指标

交易数据预加载后，data属性也可使用buflen方法，返回交易数据的柱线数。

len和buflen之间的区别：

* len报告已处理了多少条
* buflen报告已为交易数据加载的柱线总数

如果两个都返回相同的值，则要么没有数据被预加载，要么当前处理已消耗了所有预加载的数据（除非系统连接到实时交易数据，否则将意味着处理结束）。

### 线和参数的继承

框架提供了一种元语言来支持参数和线的声明。我们尽量使其与Python的继承规则兼容。

参数继承

* 支持多重继承
* 基类的参数被继承
* 如果多个基类定义相同的参数，则使用继承列表中最后一个类的默认值
* 如果在子类中重新定义了相同的参数，则新的默认值将覆盖基类的默认值。

Lines继承

* 支持多重继承
* 所有基类的Lines均被继承。被命名为Lines的情况下，如果在基类中多次使用相同的名称，则Lines中只有一个该名称的属性。

## 索引0和-1

如前所述，Lines是线群，线是一组点的集合，这些点在绘制在一起形成一条线（例如，沿着时间轴将所有收盘价连在一起就形成收盘价曲线）

要在常规代码中访问这些点，一般通过0索引的方式对当前点进行get/set操作。
策略只能读取数据， 指标既可以读取也可以写入数据。

回顾前面简单的示例，策略中的next方法：

```python
def next(self):
    if self.movav.lines.sma[0] > self.data.lines.close[0]:
        print('简单移动平均线大于收盘价')
```

通过索引0获得移动平均线的当前值和当前收盘价，并比较它们的大小。

> 注意：
> 实际上对于索引0，可直接进行逻辑/算术运算操作，如下所示：

```python
if self.movav.lines.sma > self.data.lines.close:
    ...
```

更多相关说明请参阅文档后面的《操作章节》。

在指标开发的应用中，会出现赋值操作。
例如SimpleMovingAverage的当前值可以通过如下方式进行读写：

```python
def next(self):
  self.line[0] = math.fsum(self.data.get(0, size=self.p.period)) / self.p.period
```

访问前一个点集合可以按照Python访问数组索引为-1的方式：

* 它指向数组的最后一项

框架认为最后一项为（读写当前点的前一个点）索引值为-1。
因此，在策略中比较当前收盘价与前一个收盘价是通过 0 vs -1的方式。例如：

```python
def next(self):
    if self.data.close[0] > self.data.close[-1]:
        print('今天收盘价更高')
```

同理，使用-1，-2，-3，...便访问-1之前项的价格。

## 切片
backtrader不支持对线对象进行切片，这是遵循[0]和[-1]索引方案的设计决策。 使用常规的可索引Python对象，可以执行以下操作：
```python
# 从开始到结尾的切片
myslice = self.my_sma[0:]  
```
但是请记住，选择0…实际上是当前开始传递的值，之后也没有任何值。
```python
# 从开始到结尾的切片
myslice = self.my_sma[0:-1] 
```
同样，…0是当前值，而-1是先前交付的值。 这就是为什么从0->-1进行的切片反向操作就毫无意义的原因。  
如果可以反向操作，那么切片可能应该这样：
```python
# 从当前点向前的切片
myslice = self.my_sma[:0] 
# 从最后的值到当前值
myslice = self.my_sma[-1:0]  
# 从最后的值到倒数第3个值
myslice = self.my_sma[-3:-1]  
```

### 获取切片
可以获得具有最新值的数组，语法：
```python
# 显示默认值
myslice = self.my_sma.get(ago=0, size=1)  
```
返回一个数组，该数组的大小为1，当前时刻为0，向后获取。  
要从当前时间点获取10个值（即：最后10个值）：
```python
# ago的默认值为0
myslice = self.my_sma.get(size=10)  
```
常规数组具有你所期望的顺序。最左边的值是最旧的值，最右边的值是最新的值（这是常规的python数组，而不是lines对象）。  

```python
# 跳过当前点获取最后10个值
myslice = self.my_sma.get(ago=-1, size=10)
```
## 线的延迟索引
[]运算符可用于在next逻辑阶段提取单个值。lines对象支持附加的符号，以便在\_\_init\_\_阶段通过延迟的线对象寻址取值。  
假设一条逻辑是将先前的收盘价与简单移动平均线的实际值进行比较。无需在每次next迭代中进行手动操作，而是可以生成预定义的lines对象：
```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):

        self.movav = btind.SimpleMovingAverage(self.data, period=self.p.period)
        self.cmpval = self.data.close(-1) > self.sma

    def next(self):
        if self.cmpval[0]:
            print('上一个收盘价高于当前移动平均值')
```
这里使用"()"延迟符号：  
* 这提供了收盘价的副本，但延迟了-1。    
比较self.data.close（-1）> self.sma 会生成另一个line对象，如果条件为True，则返回1，否则为0

## 线(群)的耦合
运算符()可以与延迟的数值一起使用，以提供延迟的line对象。  
如果使用中不提供延迟数值，则返回LinesCoupler对象。这是为了在操作具有不同时间范围的数据指标之间建立耦合。  
不同时间范围的交易数据具有不同的长度，并且指标在操作这些数据时会复制数据的长度。例如：
* 日交易数据每年约有250条
* 周交易数据每年有52条
  
尝试创建一个比较两个简单移动平均线的操作，每次操作在引用数据时都有可能被中断。因为系统不知道如何在250条的日交易数据和52条的周交易数据之间进行匹配。
  
读者可以通过找出一天和一周的对应关系进行比较，但是：
* 指标只是数学公式，没有日期时间信息  
他们对上下文环境一无所知，只要提供了足够的数据，就可以进行计算。  
  
于是()表示法（空调用）可用于解决这个问题：
```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):

        # data0 是日交易数据
        sma0 = btind.SMA(self.data0, period=15)  # 15天sma
        # data1 是周要以数据
        sma1 = btind.SMA(self.data1, period=5)  # 5周sma

        self.buysig = sma0 > sma1()

    def next(self):
        if self.buysig[0]:
            print('每日sma大于每周sma1')
```
在这里，较大的时间范围指标sma1通过sma1()与每日时间范围耦合。这将返回与更大数量的sma0兼容的对象，并复制sma1产生的值，从而有效地将52个周数据分散为250个日数据。

## 通过操作符构造对象
为了实现“简单易用”的目标，backtrader允许（在Python的语法范围内）使用操作符。为了进一步简单化，操作符的使用有两种情景。  
### 情景1-操作符创建对象
我们之前已经看到了一个例子。在指标和策略类的对象初始化阶段（\_\_init\_\_方法）中，操作符创建并保存可以持续使用的对象，供策略逻辑在评估阶段使用。  
将SimpleMovingAverage的潜在实现方式进一步细分为多个步骤。    
SimpleMovingAverage指标\_\_init\_\_内的代码可能如下：
```python
def __init__(self):
    # N个周期值的总和，数据总和是一个Lines对象
    # 在与运算符[]和索引0查询时
    # 返回当前总和
    datasum = btind.SumN(self.data, period=self.params.period)
    # datasum（虽然是单行，但仍是一个Lines对象）
    # 在这种情况下它可以除以int/float类型的数据。 
    # 但实际上它被除以后得到另外一个Lines对象。
    # 该操作返回分配给av对象
    # 当查询为[0]，则返回当前时间点的平均值
    av = datasum / self.params.period
    # av是对新的Lines对象的命名
    # 其他对象使用这个指标可以直接访问计算
    self.line.sma = av
```
策略初始化期间显示了更加完整的用法：
```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma = btind.SimpleMovinAverage(self.data, period=20)

        close_over_sma = self.data.close > sma

        sma_dist_to_high = self.data.high - sma

        sma_dist_small = sma_dist_to_high < 3.5

        # 不幸的是，"and"不能在Python中被重载
        # 在python中and不属于运算符，所以backtrader提供一个函数模拟这个功能
        sell_sig = bt.And(close_over_sma, sma_dist_small)
```
完成上述操作后，sell_sig是一个Lines对象，当指示满足条件时，可以直接在策略中使用。

### 情景2-逻辑操作符
首先，策略的next方法，系统要处理每个柱时都要调用该方法，这就是操作符处于情景2地方。以前面的示例为基础：
```python
class MyStrategy(bt.Strategy):
    def __init__(self):
        sma = btind.SimpleMovinAverage(self.data, period=20)

        close_over_sma = self.data.close > sma

        sma_dist_to_high = self.data.high - sma

        sma_dist_small = sma_dist_to_high < 3.5

        # 不幸的是，"and"不能在Python中被重载
        # 在python中and不属于运算符，所以backtrader提供一个函数模拟这个功能
        sell_sig = bt.And(close_over_sma, sma_dist_small)

    def next(self):
        # 尽管这看起来不像是“操作号”，但确实返回的是正在测试对象的True/False
        if self.sma > 30.0:
            print('sma大于30.0')

        if self.sma > self.data.close:
            print('sma高于收盘价')

        if self.sell_sig:  # if sell_sig == True: would also be valid
            print('卖出标志为True')
        else:
            print('卖出标志为False')

        if self.sma_dist_to_high > 5.0:
            print('sma到high的距离大于5.0')
```
这不是一个非常有用的策略，只是一个例子。在情景2中，操作符返回期望值（如果测试值为True，则返回布尔值；如果是浮点数进行比较，则返回浮点数），并且算术运算也返回期望值。  
>注意：  
>为了进一步简化，比较实际上没有使用操做符。  
>if self.sma > 30.0: …比较 self.sma[0] 和 30.0   
>if self.sma > self.data.close: … 比较 self.sma[0] 和 self.data.close[0]  

### 一些不可重载的运算符/函数
Python不允许重载所有内容，因此提供了一些功能函数来应对这种情况。
>注意：仅适用于情景1，以创建对象供后面使用。

操作符:  
* and -> And
* or -> Or

逻辑控制:
* if -> If

函数:
* any -> Any
* all -> All
* cmp -> Cmp
* max -> Max
* min -> Min
* sum -> Sum
* reduce -> Reduce

Sum实际上使用math.fsum作为底层操作，因为backtrader使用浮点数计算，如果用常规sum可能会影响精度。  
    
这些实用的操作符/函数可迭代使用。可迭代的元素可以是常规的Python数字类型（int，float等），也可以是带有Lines的对象。 例如一个非常原始的买入信号：
```python
class MyStrategy(bt.Strategy):
    def __init__(self):
        sma1 = btind.SMA(self.data.close, period=15)
        self.buysig = bt.And(sma1 > self.data.close, sma1 > self.data.high)

    def next(self):
        if self.buysig[0]:
            pass  # do something here
```
例如sma1高于最高价，则必高于收盘价，这里重点是说明bt.And的用法。  
bt.If用法：
```python
class MyStrategy(bt.Strategy):

    def __init__(self):
        # 在period=15的data.close上生成SMA
        sma1 = btind.SMA(self.data.close, period=15)
        # 如果sma的值大于close，则返回low，否则返回high
        high_or_low = bt.If(sma1 > self.data.close, self.data.low, self.data.high)
        sma2 = btind.SMA(high_or_low, period=15)
```
解释说明：
* 在period=15的data.close上生成sma1
* 如果sma1的值大于close，则返回low，否则返回high
* 调用bt.If时不会返回任何实际值。它返回一个Lines对象，就像SimpleMovingAverage一样，这些值将在稍后计算中会用到
* 然后将bt.If生成的Lines对象赋值给sma2该,sma2有时会使用最低价，有时会使用高价进行计算

这些函数也可以使用数值，修改后得到示例：
```python
class MyStrategy(bt.Strategy):
    def __init__(self):
        sma1 = btind.SMA(self.data.close, period=15)
        high_or_30 = bt.If(sma1 > self.data.close, 30.0, self.data.high)
        sma2 = btind.SMA(high_or_30, period=15)
```
现在，sma2使用30.0或最高价进行计算，具体取决于sma1和close的比较结果。
>注意：数值30在内部转换为伪迭代，始终返回30
  
  
# Cerebro(核心引擎)
## Cerebro
Cerebro类是backtrader的引擎，是以下几个方面的核心：

* 收集所有输入（Data Feeds），执行（Stratgegies），监控（Observers），评测（Analyzers）和记录（Writers），以确保系统随时运行。
* 执行回测或实盘交易
* 返回结果
* 绘图

### 收集输入数据
1.以创建cerebro开始:  
```python
# **kwargs参数支持某些控制执行，请参阅后面文档（相同的参数也应用于run方法）
cerebro = bt.Cerebro(**kwargs)
```

2.添加交易数据  
最常见的模式是 cerebro.adddata（data），其中data是已实例化的数据源。例：
```python
data = bt.BacktraderCSVData(dataname='mypath.days', timeframe=bt.TimeFrame.Days)
cerebro.adddata(data)
```
重新采样或数据重播遵循相同的模式：
```python
data = bt.BacktraderCSVData(dataname='mypath.min', timeframe=bt.TimeFrame.Minutes)
cerebro.resampledata(data, timeframe=bt.TimeFrame.Days)
# 或者
data = bt.BacktraderCSVData(dataname='mypath.min', timeframe=bt.TimeFrame.Minutes)
cerebro.replaydatadata(data, timeframe=bt.TimeFrame.Days)
```
系统可以接受任何数量的交易数据，包括将常规数据与重采样和/或重播的数据混合。当然，这些组合中的某些组合肯定会毫无意义，并且为了组合数据有意义增加了限制条件：根据时间条件限制。请参阅数据章节的多个时间范围、数据重采样和数据重播部分。

3.添加策略  
与交易数据不同的是交易数据已经是类的实例，而cerebro是直接使用策略类和参数。因为：在策略优化方案中，该类将被实例化多次并传递不同的参数。  
即使没有运行策略优化方案，该模式仍然适用：
```python
cerebro.addstrategy(MyStrategy, myparam1=value1, myparam2=value2)
# 优化时，必须将参数作为可迭代对象添加。有关详细说明，请参见“优化”部分。
cerebro.optstrategy(MyStrategy, myparam1=range(10, 20))
```
它将使用myparam1值从10到19的值运行MyStrategy10次（记住Python中的范围是半开的，不会取到20）

4.其他要素  
可以添加其他一些元素来增强回测体验，请参见相应的部分。方法是：  
* addwriter
* addanalyzer
* addobserver (or addobservermulti)

5.更换经纪人  
Cerebro将在backtrader中使用默认经纪人，但是可以被重写：
```python
broker = MyBroker()
cerebro.broker = broker  # property using getbroker/setbroker methods
```
6.接收通知  
如果交易数据和/或代理发送通知（被store provider创建的通知），则它们将通过Cerebro.notify_store方法接收。有三种处理通知的方法：
* 通过addnotifycallback(callback)方法将回调添加到cerebro实例中。 回调方法如下：  
callback(msg, *args, **kwargs)  
实际收到的msg，* args和** kwarg 由（data/broker/store）来决定，但通常它们是可打印的，以便进行接收和实验。 
* 在Strategy子类中的覆盖notify_store方法  
notify_store（self，msg，* args，** kwargs）
* 通过子类继承Cerebro并覆盖notify_store（与策略中的方法相同）  
最不推荐使用该方法

### 执行回测
运行回测比较简单，但是它支持多个选项来决定如何运行（可以在实例化时指定运行模式）：
```python
result = cerebro.run(**kwargs)
```
请参阅下文中相关部分，以了解可用的参数选项。

#### 标准观察者
cerebro（除非另有说明）自动实例化三个标准观察者  
* 经纪人观察者用来跟踪现金和投资组合的价值
* 交易观察者，会显示每笔交易所产生的影响
* 买/卖观察者主要对订单操作进行记录
  
如果希望使用更干净的绘图，可通过stdstats = False 来禁用这些观察者。  

### 返回结果
cerebro返回在回测期间创建的策略实例。由于策略中的所有元素都可访问，因此可以分析他们的操作：
```python
result = cerebro.run(**kwargs)
```
返回结果的格式将根据是否使用优化选项而有所不同（策略可以通过optstrategy方式添加到cerebro中）：
* 所有策略通过addstrategy添加  
result返回的是一个list
* 1个或多个策略通过optstrategy添加  
result返回的是list的list， 每个列表项是对应策略返回的list。

>注意  
> 为使得内核传递的消息更轻便，现在optimization默认返回分析器analyzers。

如果希望将整套策略作为返回值，请将参数optreturn设置为False。

### 便捷绘图
如果安装了matplotlib，则可以绘制策略结果。 通常的模式是：
```python
cerebro.plot（）
```
请阅读下面的参考中“绘图”部分。

###  回测逻辑
回测流程概述：  
  
1.传递store中的所有通知；
    
2.令数交易数据以柱集合方式传递给next方法；  
在版本1.9.0.99中已更改为：  
交易数据根据datetime同步检索出来，然后提供给next方法。当没有新时间段的交易数据时，就使用旧的数据，有则使用新的（指标中的计算也是如此）。   
如果过使用旧版本的默认行为，就在使用Cerebro时，将oldsync=True。    
    
插入到系统中的第一个数据是主数据，其他交易数据为从属数据，系统将等待第一次响应时钟，并且：  
* 如果下一时钟周期比datamaster已传递的数据时间要新，则该数据将不会被传递。
* 由于多种原因，可能会出现在没有提供新报价的情况下提前返回

该逻辑目的在于同步多个具有不同时间点的数据。   

3.将有关订单，交易和现金/价值的消息队列，通过经纪通知给策略； 
   
4.告知经纪人接受订单队列并使用新数据执行挂单操作；  
  
5.调用该策略的next方法，以使该策略评估新数据（以及可能发布在经纪人中排队的订单）  
根据阶段的不同，在满足策略/指标的最短期限要求之前（可能在prenext或者nextstart方法中），这些策略内部还将对观察者，指标，分析器和其他活跃要素产生影响；  
  
6.告诉任何记录器将数据写入目标。  
  
重要的考虑因素：  
> 注意:  
> 在上面的步骤1中，当交易数据传递新的一组柱线时，旧柱线被关闭，这意味着这些被关闭的数据已经发生过。

因此，无法使用步骤1中的数据执行步骤4中策略发出的订单。  
这意味着定单将以x + 1的概念执行，其中x是定单执行的柱线，x + 1是下一个定单，这是可能执行定单的最早时间。

### 相关介绍
__class backtrader.Cerebro()__
参数：
* preload (default: True) 是否为策略预先加载传递给cerebro的不同的交易数据
* runonce (default: True) 在矢量模式下运行指标，以加快整个系统的运行速度，策略和观察者将始终基于事件运行
* live (default: False) 如果没有实时数据（通过数据的islive方法，但用户仍希望以实时模式运行，可以将此参数设置为true），这将同时停用preload和runonce,它不会对内存节省方案产生影响。

> 未完待续...


# 交易数据
backtrader带有一组交易数据解析器（解析所有基于CSV格式的文件），可让你从不同来源加载数据。

* Yahoo（在线或已保存到文件）
* VisualChart（请参见www.visualchart.com）
* Backtrader CSV（backtrader用于测试的csv文件）
* 通用CSV支持

从快速入门指南中可以清楚地看到，你已将交易数据添加到Cerebro中。 交易数据将用于下不同的策略：

* 数组self.datas（按插入顺序）
* 数组对象的别名：
  * self.data和self.data0指向第一个元素
  * self.dataX指向数组中索引为X的元素

# 策略
Cerebro是backtrader的心核心，策略是用户的核心。  
  
研究一下策略的生命周期:   
>注意：  
>一个策略可能在出生时被来自backtrader.errors模块中的StrategySkipError异常所中断，这样可以避免在回测期间继续执行该策略，详情请参阅《异常》部分章节。

1. 初始化：\_\_init\_\_
这是在实例初始化期间调用的：指示和其他所需的属性将在此处创建。例如： 
```python 
def __init__（）：  
    self.sma = btind.SimpleMovingAverage（period = 15）
```

2. 启动：start
cerebro通知strategy开始运行，默认值是一个空方法。

3. 预循环：prenext  
初始化时，声明一个指标，指标限制了策略可使用的最小期限。例如：在__init__上方，创建了一个Period = 15的SimpleMovingAverage。  
只要系统看到的柱线数量少于15条，就会调用prenext（默认实现是空操作）

4. 循环：next  
一旦系统看到交易数据中有超过15条柱线，并且SimpleMovingAverage具有足够大的缓冲区，该策略可以真正执行。  
有一个nextstart方法，该方法仅被调用一次，以标记从prenext到next的切换。 nextstart的默认实现是简单地调用next方法。

5. 重复：none  
主要在进行参数优化时（使用不同的参数），系统会根据不同的参数重复实例化多次。

6. 停止：stop  
进行重置的时间到了，系统会告之该策略进行重置操作， 默认方法为空。

在大多数情况下，常规使用模式，如下：
```python
class MyStrategy(bt.Strategy):

    def __init__(self):
        self.sma = btind.SimpleMovingAverage(period=15)

    def next(self):
        if self.sma > self.data.close:
            # Do something
            pass

        elif self.sma < self.data.close:
            # Do something else
            pass
```
在此代码段中：
* 在\_\_init\_\_期间，为属性分配了一个指标
* 默认的空start方法不会被覆盖
* prenext和nexstart不会被覆盖
* 在next方法中，将指标的值与收盘价进行比较以执行某项操作
* 默认的空stop方法不会被覆盖

策略就像现实世界中的交易者一样，当有事件发生时会得到通知。实际上，回测过程中的每个周期next方法被调用一次。该策略将执行以下动作：
* 通过notify_order（order）通知订单中的任何状态更改
* 通过notify_trade（trade）通知任何开仓/更新/平仓交易
* 通过notify_cashvalue（cash, value）通知经纪人当前的现金和投资组合
* 通过notify_fund（cash, value, fundvalue, shares）通知经纪人当前的现金和投资组合以及基金价值和股票的交易
* 通过notify_store（msg，* args，** kwargs）实现特定的事件

请参阅Cerebro对有关store通知的说明，即使store通知传递给了Cerebro，也一样会传递给策略（使用重写的notify_store方法或通过回调的方式）。  

策略也希望交易者扑捉到市场中的机会，并在next方法中，通过以下操作来获利：
* buy方法可以做多或减少/关闭空头头寸
* sell方法可以做空或减少/关闭多头头寸
* close方法可以平仓
* cancel方法取消尚未执行的订单