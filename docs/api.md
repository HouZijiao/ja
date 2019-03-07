# doc\help
## API文档
感谢您使用JoinQuant（聚宽）量化交易平台，以下内容主要介绍聚宽量化交易平台的API使用方法，目录中带有"♠" 标识的API是 `"回测环境/模拟"`的专用API，**不能在研究模块中调用**。

内容较多，可使用`Ctrl+F`进行搜索。

如果以下内容仍没有解决您的问题，请您通过[社区提问](/community)的方式告诉我们，谢谢。

## 开始写策略
### 简单但是完整的策略
先来看一个简单但是完整的策略:
```python
def initialize(context):
    # 定义一个全局变量, 保存要操作的股票
    g.security = '000001.XSHE'
    # 运行函数
    run_daily(market_open, time='every_bar')

def market_open(context):
    if g.security not in context.portfolio.positions:
        order(g.security, 1000)
    else:
        order(g.security, -800)
```
一个完整策略只需要两步:

- 设置初始化函数: [initialize],上面的例子中, 只操作一支股票: '000001.XSHE', 平安银行
- 实现一个函数, 来根据历史数据调整仓位.

这个策略里, 每当我们没有股票时就买入1000股, 每当我们有股票时又卖出800股, 具体的下单API请看[order](#order-method)函数.

这个策略里, 我们有了交易, 但是只是无意义的交易, 没有依据当前的数据做出合理的分析

下面我们来看一个真正实用的策略

### 实用的策略
在这个策略里, 我们会根据历史价格做出判断:

- 如果上一时间点价格高出五天平均价1%, 则全仓买入
- 如果上一时间点价格低于五天平均价, 则空仓卖出

```python
# 导入聚宽函数库
import jqdata

# 初始化函数，设定要操作的股票、基准等等
def initialize(context):
    # 定义一个全局变量, 保存要操作的股票
    # 000001(股票:平安银行)
    g.security = '000001.XSHE'
    # 设定沪深300作为基准
    set_benchmark('000300.XSHG')
    # 开启动态复权模式(真实价格)
    set_option('use_real_price', True)
    # 运行函数
    run_daily(market_open, time='every_bar')

# 每个单位时间(如果按天回测,则每天调用一次,如果按分钟,则每分钟调用一次)调用一次
def market_open(context):
    security = g.security
    # 获取股票的收盘价
    close_data = attribute_history(security, 5, '1d', ['close'])
    # 取得过去五天的平均价格
    MA5 = close_data['close'].mean()
    # 取得上一时间点价格
    current_price = close_data['close'][-1]
    # 取得当前的现金
    cash = context.portfolio.cash

    # 如果上一时间点价格高出五天平均价1%, 则全仓买入
    if current_price > 1.01*MA5:
        # 用所有 cash 买入股票
        order_value(security, cash)
        # 记录这次买入
        log.info("Buying %s" % (security))
    # 如果上一时间点价格低于五天平均价, 则空仓卖出
    elif current_price < MA5 and context.portfolio.positions[security].closeable_amount > 0:
        # 卖出所有股票,使这只股票的最终持有量为0
        order_target(security, 0)
        # 记录这次卖出
        log.info("Selling %s" % (security))
    # 画出上一时间点价格
    record(stock_price=current_price)

```

## 用户需要实现的函数

###  initialize
```python
initialize(context)
```
初始化方法，在整个回测、模拟实盘中最开始执行一次，用于初始一些全局变量

**参数**
context: [Context](#context)对象, 存放有当前的账户/股票持仓信息

**返回**
None

**示例**
```python
def initialize(context):
    # g为全局变量
    g.security = "000001.XSHE"
```


### 定时运行函数, 可选

- run_monthly
- run_weekly
- run_daily

```python
def initialize(context):
    # 按月运行
    run_monthly(func, monthday, time='open', reference_security)
    # 按周运行
    run_weekly(func, weekday, time='open', reference_security)
    # 每天内何时运行
    run_daily(func, time='open', reference_security)
```

**具体参见[定时运行]**

**参数**

参数|解释
---|---
func|一个函数, 此函数必须接受context参数
monthday|每月的第几个交易日, 可以是负数, 表示倒数第几个交易日。如果超出每月总交易日个数，则取临近的交易日执行。（具体见下方注意中的示例）
weekday|每周的第几个交易日, 可以是负数, 表示倒数第几个交易日。如果超出每周总交易日个数，则取临近的交易日执行。（具体见下方注意中的示例）
time|一个字符串,可以是具体执行时间,支持 time 表达式。比如 "10:00", "01:00", 或者 "every_bar", "open", "before_open", "after_close", "morning" 和 "night"。(具体执行时间如见下方)<br><br>time 表达式具有 'base +/-offset' 的形式，如：'open-30m'表示开盘前30分钟，'close+1h30m'表示收盘后一小时三十分钟。当使用 'base +/-offset' 形式的表达式时， base 只能使用 open 或 close 二者之一。
reference_security|时间的参照标的。<br> 如参照 '000001.XSHG'，交易时间为 9:30\-15:00。<br>如参照'IF1512.CCFX'，2016-01-01之后的交易时间为 9:30\-15:00，在此之前为 9:15\-15:15。<br>如参照'A9999.XDCE'，因为有夜盘，因此开始时间为21:00，结束时间为15:00。


time|具体执行时间
---|---
具体时间|24小时内的任意时间，如"10:00", "01:00"
every_bar|只能在 run_daily 中调用； **按天**会在每天的开盘时调用一次，**按分钟**会在每天的每分钟运行
open|开盘时运行(等同于"9:30")
before_open|早上 9:00 运行
after_close|下午 15:30 运行
morning|早上 8:00 运行
night|晚上 20:00 运行


**返回**
None


**示例**

```python
def weekly(context):
    print 'weekly %s %s' % (context.current_dt, context.current_dt.isoweekday())


def monthly(context):
    print 'monthly %s %s' % (context.current_dt, context.current_dt.month)


def daily(context):
    print 'daily %s' % context.current_dt


def initialize(context):

    # 指定每月第一个交易日, 在开盘后一小时10分钟执行
    run_monthly(monthly, 1, 'open + 1h10m')

    # 指定每天收盘前10分钟运行
    run_daily(daily, 'close - 10m')

    # 指定每天收盘后执行
    run_daily(daily, 'after_close')

    # 指定在每天的10:00运行
    run_daily(daily, '10:00')

    # 参照股指期货的时间每分钟运行一次, 必须选择分钟回测, 否则每天执行
    run_daily(daily, 'every_bar', reference_security='IF1512.CCFX')
```

### handle_data, 可选
```python
handle_data(context, data)
```
该函数每个单位时间会调用一次, 如果按天回测,则每天调用一次,如果按分钟,则每分钟调用一次。

**该函数依据的时间是股票的交易时间，即 9:30 - 15:00. 期货请使用[定时运行]函数。**

该函数在回测中的非交易日是不会触发的（如回测结束日期为2016年1月5日，则程序在2016年1月1日-3日时，handle_data不会运行，4日继续运行）。

对于使用当日开盘价撮合的日级模拟盘，在9:25集合竞价完成时就可以获取到开盘价，出于减少并发运行模拟盘数量的目的，我们会提前到9:27~9:30之间运行, 策略类获取到逻辑时间(context.current_dt)仍然是 9:30。

**参数**
context: [Context](#context)对象, 存放有当前的账户/标的持仓信息
data: 一个字典(dict), key是股票代码, value是当时的[SecurityUnitData](#SecurityUnitData) 对象. 存放前一个单位时间(按天回测, 是前一天, 按分钟回测, 则是前一分钟) 的数据. **注意**:

- 为了加速, data 里面的数据是按需获取的, 每次 handle_data 被调用时, data 是空的 dict, 当你使用 `data[security]` 时该 security 的数据才会被获取.
- data 只在这一个时间点有效, 请不要存起来到下一个 handle_data 再用
- 注意, 要获取回测当天的开盘价/是否停牌/涨跌停价, 请使用 [get_current_data]

**返回**
None

**示例**
```python
def handle_data(context, data):
    order("000001.XSHE",100)
```

### before_trading_start, 可选
```python
before_trading_start(context)
```
该函数会在每天开始交易前被调用一次, 您可以在这里添加一些每天都要初始化的东西.

**该函数依据的时间是股票的交易时间，即该函数启动时间为 9:00. 期货请使用[定时运行]函数，time 参数设定为'before_open' 。**

**参数**
context: [Context](#context)对象, 存放有当前的账户/股票持仓信息

**返回**
None

**示例**
```python
def before_trading_start(context):
    log.info(str(context.current_dt))
```

### after_trading_end, 可选
```python
after_trading_end(context)
```
该函数会在每天结束交易后被调用一次, 您可以在这里添加一些每天收盘后要执行的内容. 这个时候所有未完成的订单已经取消.

**该函数依据的时间是股票的交易时间，即该函数启动时间为 15:30. 期货请使用[定时运行]函数，time 参数设定为'after_close' 。**

**参数**
context: [Context](#context)对象, 存放有当前的账户/股票持仓信息

**返回**
None

**示例**
```python
def after_trading_end(context):
    log.info(str(context.current_dt))
```

### process_initialize, 可选
```python
process_initialize(context)
```
该函数会在每次模拟盘/回测进程重启时执行, 一般用来初始化一些**不能持久化保存**的内容. 在 [initialize] 后执行.

因为模拟盘会每天重启, 所以这个函数会每天都执行.

**参数**
context: [Context](#context)对象, 存放有当前的账户/股票持仓信息

**返回**
None

**示例**
```python
def process_initialize(context):
    # query 对象不能被 pickle 序列化, 所以不能持久保存, 所以每次进程重启时都给它初始化
    # 以两个下划线开始, 系统序列化 [g] 时就会自动忽略这个变量, 更多信息, 请看 [g] 和 [模拟盘注意事项]
    g.__q = query(valuation)

def handle_data(context, data):
    get_fundamentals(g.__q)
```

### on_strategy_end, 可选
```python
def on_strategy_end(context)
```
在回测、模拟交易正常结束时被调用， 失败时不会被调用。

在模拟交易到期结束时也会被调用， 手动在到期前关闭不会被调用。

**参数**
context: [Context](#context)对象, 存放有当前的账户/股票持仓信息

**返回**
None

**示例**
```python
def on_strategy_end(context):
    print '回测结束'
```

### after_code_changed, 可选
```python
after_code_changed(context)
```
模拟盘在每天的交易时间结束后会休眠，第二天开盘时会恢复，如果在恢复时发现代码已经发生了修改，则会在恢复时执行这个函数。
具体的使用场景：可以利用这个函数修改一些模拟盘的数据。

注意: 因为一些原因, 执行回测时这个函数也会被执行一次, 在 [process_initialize] 执行完之后执行.

**参数**
context: [Context](#context)对象, 存放有当前的账户/股票持仓信息

**返回**
None

**示例**
```python
def after_code_changed(context):
    g.stock = '000001.XSHE'
```


## 回测引擎介绍

### 回测环境
印花税
1. 回测引擎运行在Python2.7之上, 请您的策略也兼容Python2.7
2. 我们支持所有的Python标准库和部分常用第三方库, 具体请看: [python库](#python库). 另外您可以把.py文件放在研究根目录, 回测中可以直接import, 具体请看: [自定义python库](#自定义python库)
3. 安全是平台的重中之重, 您的策略的运行也会受到一些限制, 具体请看: [安全](#安全)

### 回测过程
1. 准备好您的策略,  选择要操作的股票池, 实现handle_data函数
2. 选定一个回测开始和结束日期, 选择初始资金、调仓间隔(每天还是每分钟), 开始回测
3. 引擎根据您选择的股票池和日期, 取得股票数据, 然后每一天或者每一分钟调用一次您的handle_data函数, 同时告诉您现金、持仓情况和股票在上一天或者分钟的数据. 在此函数中, 您还可以调用函数获取任何多天的历史数据, 然后做出调仓决定.
4. 当您下单后, 我们会根据接下来时间的实际交易情况, 处理您的订单. 具体细节参见[订单处理](#订单处理)
5. 下单后您可以调用get_open_orders取得所有未完成的订单, 调用cancel_order取消订单
6. 您可以在handle_data里面调用record()函数记录某些数据, 我们会以图表的方式显示在回测结果页面
7. 您可以在任何时候调用log.info/debug/warn/error函数来打印一些日志
8. 回测结束后我们会画出您的收益和基准(参见[set_benchmark])收益的曲线,  列出每日持仓,每日交易和一系列风险数据。

### 数据
1. 股票数据：我们拥有所有A股上市公司2005年以来的股票行情数据、[市值数据](/data/dict/fundamentals#市值数据)、[财务数据](/data/dict/fundamentals)、[上市公司基本信息](/help/data/data?name=jy)、[融资融券信息](/api#getmtss)等。为了避免[幸存者偏差](http://zh.wikipedia.org/zh/%E5%80%96%E5%AD%98%E8%80%85%E5%81%8F%E5%B7%AE)，我们包括了已经退市的股票数据。
2. 商品期货：我们支持从2005年以来上海国际能源交易中心、上期所、郑商所、大商所的行情数据，并包含历史产品的数据。
3. 基金数据：我们目前提供了600多种在交易所上市的基金的行情、净值等数据，包含[ETF](/data/dict/fundData#etf列表)、[LOF](/data/dict/fundData#lof列表)、[分级A/B基金](/data/dict/fundData#分级基金列表)以及[货币基金](/data/dict/fundData#mmf列表)的完整的行情、净值数据等，请点击[基金数据](/data/dict/fundData)查看。
4. 金融期货数据：我们提供中金所推出的所有[金融期货产品](/data/dict/sfData)的行情数据，并包含历史产品的数据。
5. 股票指数：我们支持近600种[股票指数](/data/dict/indexData)数据，包括指数的行情数据以及成分股数据。为了避免未来函数，我们支持获取历史任意时刻的指数成分股信息，具体见[get_index_stocks]。注意：指数不能买卖
6. 行业板块：我们支持按行业、按板块选股，具体见[get_industry_stocks]
7. 概念板块：我们支持按概念板块选股，具体见[get_concept_stocks]
8. 宏观数据：我们提供全方位的[宏观数据](/data/dict/macroData)，为投资者决策提供有力数据支持。
9. 所有的行情数据我们均已处理好前复权信息。
10. 我们当日的回测数据会在收盘后通过多数据源进行校验，并在T+1（第二天）的00:01更新。

### 安全
1. 保证您的策略安全是我们的第一要务
2. 在您使用我们网站的过程中, 我们全程使用https传输
3. 策略会加密存储在数据库
4. 回测时您的策略会在一个安全的进程中执行, 我们使用了进程隔离的方案来确保系统不会被任何用户的代码攻击, 每个用户的代码都运行在一个有很强限制的进程中:
- 只能读指定的一些python库文件
- 不能写和执行任何文件, 如果您需要保存和读取私有文件, 请看[write_file]/[read_file]
- 不能创建进程或者线程
- 限制了cpu和内存, 堆栈的使用
- 可以访问网络, 但是对带宽做了限制, 下载最大带宽为500KB/s, 上传带宽为10KB/s
- 有严格的超时机制, 如果handle_data超过30分钟则立即停止运行
对于读取回测所需要的数据, 和输出回测结果, 我们使用一个辅助进程来帮它完成, 两者之间通过管道连接.

    我们使用了linux内核级别的apparmer技术来实现这一点.
    有了这些限制我们确保了任何用户不能侵入我们的系统, 更别提盗取他人的策略了.

###  运行频率
**1. Bar 的概念**

在一定时间段内的时间序列就构成了一根 K 线（日本蜡烛图），单根 K 线被称为 Bar。

如果是一分钟内的 Tick 序列，即构成一根分钟 K 线，又称分钟 Bar;
如果是一天内的分钟序列，即构成一根日线 K 线，又称日线 Bar;

Bar 的示意图如下所示：

![Bar 的示意图](http://img0.ph.126.net/QVCQxL-2IUy6NYG9MD-oRQ==/6631975962305988684.png)

Bar 就是时间维度上，价格在空间维度上变化构成的数据单元。如下图所示，多个数据单元 Bar 构成的一个时间序列。

![K线序列](http://img1.ph.126.net/HAVUVlSog-bkwxWOm5UJWw==/6632518021535809699.png)


**2. 频率详解**

下列图片中齿轮为 handle_data(context, data) 的运行时间，before_trading_start(context) 等其他函数运行时间详见[相关API](/api#handledata-可选)。

**频率：天**

当选择天频率时， 算法在每根日线 Bar 都会运行一次，即每天运行一次。

在算法中，可以获取任何粒度的数据。

![日K线](http://img1.ph.126.net/bBL80Rm3WkBCYZ331cQ-tA==/6632136491003641585.png)


**频率：分钟**

当选择分钟频率时， 算法在每根分钟 Bar 都会运行一次，即每分钟运行一次。

在算法中，可以获取任何粒度的数据。

![分钟K线](http://img2.ph.126.net/NnyZ_w-rEbBOOdFa0HR0jg==/6632093610050159615.png)


**频率：Tick**

当选择 Tick 频率时，每当新来一个 Tick，算法都会被执行一次。

执行示意图如下图所示：

![Tick序列](http://img0.ph.126.net/C7zTMuhM2ququLeNGSuMAQ==/6632092510538531835.png)

### 订单处理
对于您在某个单位时间下的单, 我们会做如下处理:

#### 回测

使用Bar撮合

- 市价单:
    - 按天回测
        - 当 “最新价+滑点” 在涨跌停范围内，则进行撮合，反之撤销
        - 交易价格: 最新价 + [滑点]，如果在开盘时刻运行， 最新价格为开盘价。 其他情况下， 为上一分钟的最后一个价格。
        - 最大成交量: 每次下单成交量不会超过该股票当天的总成交量.  可通过选项 [order_volume_ratio] 设置每天最大的成交量, 例如: 0.25 表示下单成交量不会超过当天成交量的 25%
        - 注意:
            - context.portfolio 中的持仓价格会使用上一分钟的最后一个价格更新。
            - data 是昨天的按天数据, 要想拿到当天开盘价, 请使用 [get_current_data] 拿取 day_open 字段

    - 分钟回测
        - 当 “最新价+滑点” 在涨跌停范围内，则进行撮合，反之撤销
        - 交易价格: 因为我们是在每分钟的第一秒钟执行代码, 所以价格是上一分钟的最后一个价格 + [滑点]
        - 同按天回测规则, 每次下单成交量不会超过该股票当天的总成交量, [order_volume_ratio] 同样有效.注意: 这是限制了每个订单的成交量, 当然, 你可以通过一天多次下单来超过一天成交量, 但是, 为了对你的回测负责, 请不要这么做.

    - 所有市价单下单之后同步完成(也即 order_XXX 系列函数返回时完成), context.portfolio 会同步变化

- 限价单
    - 回测(天、分钟):
	    - 当 委托价 > 最新价+滑点，按市价单模式撮合
		- 当 委托价 <= 最新价+滑点，则挂单，在Bar结束时按照Bar信息进行撮合：
			- 当 委托价 > bar 的最低价，则成交价为委托价，成交量不超过 Bar 成交量 * order_volume_ratio
			- 天、分钟相同
    - 不是立即完成, 下单之后 context.portfolio.cash 和 context.portfolio.positions 不会同步变化.
    - 按天模拟交易暂时不支持限价单

- 上述过程中, 如果实际价格已经涨停或者跌停, 则相对应的买单或卖单不成交, 市价单直接取消(log中有警告信息), 限价单会挂单直到可以成交.
- 一天结束后, 所有未完成的订单会被取消
- 每次订单完成(完全成交)或者取消后, 我们会根据成交量计算手续费(参见[set_order_cost]), 减少您的现金
- 更多细节, 请看[order](#order-method)函数

#### 模拟交易

模拟交易默认开启盘口撮合模式。可通过[设置是否开启盘口撮合模式]进行设定，决定您定的模拟交易使用盘口还是 Bar 进行撮合。

##### **1.使用盘口撮合**

- 市价单
	- 买单：
		- 根据卖单盘口进行撮合
		- 优先从卖一档开始撮合，根据成交量算出 加权均价
		- 成交价： 加权均价
		- 5档成交剩余撤销：买入时从“卖一"到“卖五"价格依次成交，卖出时从“买一”到“买五"价格依次成交。若无法全部成交，则剩余未匹配量自动撤销
		- 当前没有盘口时，按照 使用 Bar 处理 处理：
			- 当成交量不为零为，使用最新价+滑点成交
			- 当成交量为零，则取消该订单
	- 卖单：
		- 根据买单盘口进行撮合
		- 优先从买一档开始撮合，根据成交量算出 加权均价
		- 成交价 ： 加权均价
		- 5档成交剩余撤销：买入时从“卖一"到“卖五"价格依次成交，卖出时从“买一”到“买五"价格依次成交。若无法全部成交，则剩余未匹配量自动撤销
		- 当前没有盘口时，按照 使用 Bar 处理 处理：
			- 当成交量不为零为，使用最新价+滑点成交
			- 当成交量为零，则取消该订单

- 限价单
	- 买单：
		- 根据卖单盘口进行撮合
		- 优先从卖一档开始撮合，直至盘口价格>委托价的档位，根据成交量算出 加权均价
		- 成交价 ： 加权均价
		- 当限价单根据盘口撮合完，为部分成交时，剩余委托数量会在 Bar 结束时根据 Bar 信息进行撮合，详情见Bar撮合方式。

	- 卖单：
		- 根据买单盘口进行撮合
		- 优先从买一档开始撮合，直至盘口价格<委托价的档位，根据成交量算出 加权均价
		- 成交价 ： 加权均价
		- 当限价单根据盘口撮合完，为部分成交时，剩余委托数量会在 Bar 结束时根据 Bar 信息进行撮合，详情见 Bar 撮合方式。
	- 注意：如模拟盘下限价委托单，并且根据盘口撮合后，订单为部分成交，则该订单完整信息在收盘时更新。


##### **2.使用Bar撮合**

- 市价单:
    - 模拟交易
        - 当 “最新价+滑点” 在涨跌停范围内，则进行撮合，反之撤销
        - 交易价格: 最新价 + 滑点
        - 最大交易量: 不管是按天, 按分钟, 还是按tick, 由于市价单都是同步完成, 下单那一刻无法知道当天成交量, 所以市价单都不考虑成交量, 全量成交.

    - 所有市价单下单之后同步完成(也即 order_XXX 系列函数返回时完成), context.portfolio 会同步变化

- 限价单
	- 模拟交易(天、分钟、Tick):
		- 当 委托价 > 最新价+滑点，按市价单模式撮合
		- 当 委托价 <= 最新价+滑点，则挂单，在Bar结束时按照Bar信息进行撮合：
			- 当 委托价 > bar 的最低价，则成交价为委托价，成交量不超过 Bar 成交量 * order_volume_ratio
			- 天、分钟、tick 相同
			- 按tick, 不是立即完成, 而是下单之后每个tick根据这个tick的分价表撮合一次, 直到完全成交或者当天收盘为止. 同样考虑 [order_volume_ratio] 选项.
		- 注意：如模拟盘下限价委托单，订单信息会在收盘后更新。

    - 不是立即完成, 下单之后 context.portfolio.cash 和 context.portfolio.positions 不会同步变化.

    - 按天模拟交易暂时不支持限价单

- 上述过程中, 如果实际价格已经涨停或者跌停, 则相对应的买单或卖单不成交, 市价单直接取消(log中有警告信息), 限价单会挂单直到可以成交.
- 一天结束后, 所有未完成的订单会被取消
- 每次订单完成(完全成交)或者取消后, 我们会根据成交量计算手续费(参见[set_order_cost]), 减少您的现金
- 更多细节, 请看[order](#order-method)函数



### 拆分合并和分红
- **传统前复权回测模式：**当股票发生拆分，合并或者分红时，股票价格会受到影响，为了保证价格的连续性, 我们使用前复权来处理之前的股票价格，给您的所有股票价格已经是前复权的价格。
- **真实价格（动态复权）回测模式：**当股票发生拆分，合并或者分红时，会按照历史情况，对账户进行处理，会在账户账户中增加现金或持股数量发生变化，并会有日志提示。

    **注：**传统前复权回测模式 与 真实价格（动态复权）回测模式 区别[见这里](/post/1629)

### 股息红利税的计算

真实的税率计算方式如下：

- 分红派息的时候，不扣税；
- 等你卖出该只股票时，会根据你的股票持有时间（自你买进之日，算到你卖出之日的前一天，下同）超过一年的免税。2015年9月之前的政策是，满一年的收5%。现在执行的是,2015年9月份的新优惠政策：满一年的免税；
- 等你卖出股票时，你的持有时间在1个月以内（含1个月）的，补交红利的20%税款，券商会在你卖出股票当日清算时直接扣收；
- 等你卖出股票时，你的持有时间在1个月至1年间（含1年）的，补交红利的10%税款，券商直接扣；
- 分次买入的股票，一律按照“先进先出”原则，对应计算持股时间；
- 当日有买进卖出的（即所谓做盘中T+0），收盘后系统计算你当日净额，净额为买入，则记录为今日新买入。净额为卖出，则按照先进先出原则，算成你卖出了你最早买入的对应数量持股，并考虑是否扣税和税率问题。

在回测及模拟交易中，由于需要在分红当天将扣税后的分红现金发放到账户，因此无法准确计算用户的持仓时间（不知道股票卖出时间），我们的计算方式是，统一按照 20% 的税率计算的。


### 滑点
在实战交易中，往往最终成交价和预期价格有一定偏差，因此我们加入了滑点模式来帮助您更好地模拟真实市场的表现。

您可以通过[set_slippage]来设置回测具体的滑点参数。

### 交易税费
交易税费包含券商手续费和印花税。您可以通过[set_order_cost]来设置具体的交易税费的参数。
##### 券商手续费
中国A股市场目前为双边收费，券商手续费系默认值为万分之三，即0.03%，最少5元。
##### 印花税
印花税对卖方单边征收，对买方不再征收，系统默认为千分之一，即0.1%。

### 风险指标
风险指标数据有利于您对策略进行一个客观的评价。

**注意**: 无论是回测还是模拟, 所有风险指标(年化收益/alpha/beta/sharpe/max_drawdown等指标)都只会**每天更新一次, 也只根据每天收盘后的收益计算, 并不考虑每天盘中的收益情况**. 例外:

- 分钟和TICK模拟盘每分钟会更新策略收益和基准收益
- 按天模拟盘每天开盘后和收盘后会更新策略收益和基准收益

那么可能会造成这种现象: 模拟时收益曲线中有回撤, 但是 max_drawdown 可能为0.

##### Total Returns（策略收益）

$$Total\space Returns=(P_{end}-P_{start})/P_{start}*100\%$$
$$P_{end}=策略最终股票和现金的总价值$$
$$P_{start}=策略开始股票和现金的总价值$$

##### Total Annualized Returns（策略年化收益）
$$Total\space Annualized\space Returns=R_p=((1+P)^\frac{250}{n}-1)*100\%$$$$P=策略收益$$$$n=策略执行天数$$

##### Benchmark Returns（基准收益）
$$Benchmark\space Returns=(M_{end}-M_{start})/M_{start}*100\%$$$$M_{end}=基准最终价值$$$$M_{start}=基准开始价值$$

##### Benchmark Annualized Returns（基准年化收益）
$$Benchmark\space Annualized\space Returns=R_m=((1+M)^\frac{250}{n}-1)*100\%$$$$M=基准收益$$$$n=策略执行天数$$

##### Alpha（阿尔法）
投资中面临着系统性风险（即Beta）和非系统性风险（即Alpha），Alpha是投资者获得与市场波动无关的回报。比如投资者获得了15%的回报，其基准获得了10%的回报，那么Alpha或者价值增值的部分就是5%。

$$Alpha=\alpha=R_p- [R_f+\beta_p(R_m-R_f)]$$$$R_p=策略年化收益率$$$$R_m=基准年化收益率$$$$R_f=无风险利率（默认0.04）$$$$\beta_p=策略beta值$$
Alpha值 | 解释
--------|--------
α>0|策略相对于风险，获得了超额收益
α=0|策略相对于风险，获得了适当收益
α<0|策略相对于风险，获得了较少收益

##### Beta（贝塔）
表示投资的系统性风险，反映了策略对大盘变化的敏感性。例如一个策略的Beta为1.5，则大盘涨1%的时候，策略可能涨1.5%，反之亦然；如果一个策略的Beta为-1.5，说明大盘涨1%的时候，策略可能跌1.5%，反之亦然。
$$Beta=\beta_p=\frac{Cov(D_p,D_m)}{Var(D_m)}$$$$D_p=策略每日收益$$$$D_m=基准每日收益$$$$Cov(D_p,D_m)=策略每日收益与基准每日收益的协方差$$$$Var(D_m)=基准每日收益的方差$$
Beta值 | 解释
--------|--------
β<0|投资组合和基准的走向通常反方向，如空头头寸类
β=0|投资组合和基准的走向没有相关性，如固定收益类
0<β<1|投资组合和基准的走向相同，但是比基准的移动幅度更小
β=1|投资组合和基准的走向相同，并且和基准的移动幅度贴近
β>1|投资组合和基准的走向相同，但是比基准的移动幅度更大

##### Sharpe（夏普比率）
表示每承受一单位总风险，会产生多少的超额报酬，可以同时对策略的收益与风险进行综合考虑。
$$Sharpe\space Ratio=\frac{R_p - R_f}{\sigma_p}$$$$R_p=策略年化收益率$$$$R_f=无风险利率（默认0.04）$$$$\sigma_p=策略收益波动率$$

##### Sortino（索提诺比率）
表示每承担一单位的下行风险，将会获得多少超额回报。
$$Sortino\space Ratio=\frac{R_p - R_f}{\sigma_{pd}}$$$$R_p=策略年化收益率$$$$R_f=无风险利率（默认0.04）$$$$\sigma_{pd}=策略下行波动率$$

##### Information Ratio（信息比率）
衡量单位超额风险带来的超额收益。信息比率越大，说明该策略单位跟踪误差所获得的超额收益越高，因此，信息比率较大的策略的表现要优于信息比率较低的基准。合理的投资目标应该是在承担适度风险下，尽可能追求高信息比率。
$$Information\space Ratio=\frac{R_p - R_m}{\sigma_t}$$$$R_p=策略年化收益率$$$$R_m=基准年化收益率$$$$\sigma_t=策略与基准每日收益差值的年化标准差$$

##### Algorithm Volatility（策略波动率）
用来测量策略的风险性，波动越大代表策略风险越高。
$$Algorithm\space Volatility=\sigma_p=\sqrt{\frac{250}{n-1} \sum_{i=1}^{n}(r_p-\bar{r_p})^2}$$$$r_p=策略每日收益率$$$$\bar{r_p}=策略每日收益率的平均值=\frac{1}{n} \sum_{i=1}^{n}r_p$$$$n=策略执行天数$$

##### Benchmark Volatility（基准波动率）
用来测量基准的风险性，波动越大代表基准风险越高。
$$Benchmark\space Volatility=\sigma_m=\sqrt{\frac{250}{n-1} \sum_{i=1}^{n}(r_m-\bar{r_m})^2}$$$$r_m=基准每日收益率$$$$\bar{r_m}=基准每日收益率的平均值=\frac{1}{n} \sum_{i=1}^{n}r_m$$$$n=基准执行天数$$

##### Max Drawdown（最大回撤）
描述策略可能出现的最糟糕的情况，最极端可能的亏损情况。
$$Max\space Drawdown=Max(P_x-P_y)/P_x$$$$P_x,P_y=策略某日股票和现金的总价值，y>x$$

##### Downside Risk（下行波动率）
策略收益下行波动率。和普通收益波动率相比，下行标准差区分了好的和坏的波动。
$$Downside\space Risk=\sigma_{pd}=\sqrt{\frac{250}{n} \sum_{i=1}^{n}(r_p - \bar{r_{pi}})^2 f(t)}$$$$r_p=策略每日收益率$$$$\bar{r_{pi}}=策略至第i日平均收益率=\frac{1}{i} \sum_{j=1}^{i}r_j$$$$n=策略执行天数$$$$f(t)=1\space if \space r_p< \bar{r_{pi}}$$$$f(t)=0\space if \space r_p>=\bar{r_{pi}}$$

##### 胜率(%)
盈利次数在总交易次数中的占比。
$$胜率 = \frac{盈利交易次数}{总交易次数}$$

##### 日胜率(%)
策略盈利超过基准盈利的天数在总交易数中的占比。
$$日胜率 = \frac{当日策略收益跑赢当日基准收益的天数}{总交易日数}$$

##### 盈亏比
周期盈利亏损的比例。
$$盈亏比 = \frac{总盈利额}{总亏损额}$$

### 运行时间

- 开盘前(9:00)运行:
    - [run_monthly]/[run_weekly]/[run_daily]中指定time='before_open'运行的函数
    - [before_trading_start]

- 盘中运行:
    - [run_monthly]/[run_weekly]/[run_daily]中在指定交易时间执行的函数, 执行时间为这分钟的第一秒. 例如: `run_daily(func, '14:50')` 会在每天的14:50:00(精确到秒)执行
    - [handle_data]
        - 按日回测/模拟, 在9:30:00(精确到秒)运行, [data]为昨天的天数据
        - 按分钟回测/模拟, 在每分钟的第一秒运行, 每天执行240次, 不包括11:30和15:00这两分钟, [data]是上一分钟的分钟数据. 例如: 当天第一次执行是在9:30:00, [data]是昨天14:59这一分钟的分钟数据, 当天最后一次执行是在14:59:00, [data]是14:58这一分钟的分钟数据.

- 收盘后(15:00后半小时内)运行:
    - [run_monthly]/[run_weekly]/[run_daily]中指定time='after_close'运行的函数
    - [after_trading_end]

- 同一个时间点, 总是先运行 run_XXX 指定的函数, 然后是 [before_trading_start], [handle_data] 和 [after_trading_end]

- 注意:
    - run_XXX 指定的函数只能有一个参数 [context],  [data] 不再提供, 请使用 [history]/[attribute_history] 获取
    - [initialize] / [before_trading_start] / [after_trading_end] / [handle_data] 都是可选的, 如果不是必须的, 不要实现这些函数, 一个空函数会降低运行速度.


