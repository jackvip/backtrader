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
tar xzf backgrader.tgz  
cd backtrader  
cp -r backtrader project_directory  
python setup.py install  
   
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
  
self.sma = SimpleMovingAverage(.....)  
访问此移动平均线的当前值的最简单方法：  
av = self.sma[0]  
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


运行后，结果输出：

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

注意:  
Yahoo的价格数据有点非主流，它是以时间倒序排列的。datetime.datetime()中的reversed=True参数是将顺序反转一次，这样就得到了我们想要的正序数据。  

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

由于只需访问收盘价数据，于是使用 self.dataclose = self.datas[0].close 将第一条价格数据的收盘价赋值给新变量。  

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

Strategy类有一个变量position保存当前持有的资产数量，（可以理解为金融术语中的头寸）  

buy()和sell()会返回被创建的订单(尚未执行的)  
订单状态的更改将通过notify方法通知给策略Strategy  


卖出逻辑也很简单：
5个K线柱后（第6个K线柱）不管涨跌都卖。   
请注意，这里没有指定具体时间，而是指定的柱的数量。一个柱可能代表1分钟、1小时、1天、1星期等等，这取决于你价格数据文件里一条数据代表的周期。  
虽然我们知道每个柱代表一天，但策略并不知道。  
```
因为买入的时候，用的是交易日，所以这里应翻译为：5个交易日（第6个交易日）不管涨跌都卖。
```
另外，当还有头寸的时候，就不再买入了。  



注意：  
没有将柱的下标传给next()方法，怎么知道已经经过了5个柱了呢？ 这里用了Python的len()方法获取它Line数据的长度。  
交易发生时记下它的长度，后边比较大小，看是否经过了5个柱。  


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

```python
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

陆续更新中...
