
# 关于

**jqfactor_analyzer: 聚宽单因子分析工具**

---

## **安装**

```bash
pip install jqfactor_analyzer
```

## **升级**

```bash
pip install -U jqfactor_analyzer
```

## **使用示例**

* ### 示例一

```python
#载入函数库
import pandas as pd
import jqfactor_analyzer as ja

# 获取 jqdatasdk 授权
# 输入用户名、密码，申请地址：http://t.cn/EINDOxE
# 聚宽官网及金融终端，使用方法参见：http://t.cn/EINcS4j
import jqdatasdk
jqdatasdk.auth('username', 'password')

# 对因子进行分析
factor_analyzer = ja.analyze_factor(
    factor_data,  # factor_data 为因子值的 pandas.DataFrame
    quantiles=10,
    periods=(1, 10),
    industry='jq_l1',
    weight_method='avg',
    max_loss=0.1
)

# 生成统计图表
factor_analyzer.create_full_tear_sheet(
    demeaned=False, group_adjust=False, by_group=False,
    turnover_periods=None, avgretplot=(5, 15), std_bar=False
)
```

* ### 示例二

```python
# 载入函数库
import pandas as pd
import jqfactor_analyzer as ja

# 获取 jqdatasdk 授权
# 输入用户名、密码，申请地址：http://t.cn/EINDOxE
# 聚宽官网及金融终端，使用方法参见：http://t.cn/EINcS4j
import jqdatasdk
jqdatasdk.auth('username', 'password')

# 生成数据接口
dataapi = ja.DataApi(fq='post', industry='jq_l1', weight_method='avg')

# 对因子进行分析
factor_analyzer = ja.FactorAnalyzer(
    factor_data,  # factor_data 为因子值的 pandas.DataFrame
    price=dataapi.get_price,
    groupby=dataapi.get_groupby,
    weights=dataapi.get_weights,
    binning_by_group=False,
    quantiles=10,
    periods=(1, 10)
)

# 生成统计图表
factor_analyzer.create_full_tear_sheet(
    demeaned=False, group_adjust=False, by_group=False,
    turnover_periods=None, avgretplot=(5, 15), std_bar=False
)
```

## **函数介绍**

[`fqfactor_analyzer.analyze_factor`](docs/apis文档.md#因子分析函数): 因子分析函数

