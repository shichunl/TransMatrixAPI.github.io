### Quick Start

Transmatrix是一个高度自定义化的量化投研框架，可以帮助用户进行因子研究、策略研究和回测分析等任务。本文将简要介绍如何使用Transmatrix进行因子研究和策略研究，从而帮助用户快速入门。



#### 因子研究全流程

在Transmatrix中，因子研究涉及因子策略逻辑的编写、因子评价逻辑的编写、数据的订阅，以及其他参数的配置。

以下是一个简单的因子研究例子，展示了如何使用Transmatrix进行因子计算和分析。

- strategy.py
    ```python
    # 导入Transmatrix中的策略模块
    from transmatrix.strategy import SignalStrategy
    from transmatrix.data_api import create_data_view, NdarrayData, DataView3d, DataView2d
    from scipy.stats import zscore
    
    # 创建一个简单的均线策略
    class Factor(SignalStrategy):
        def init(self):
            # 订阅数据
            ## 通过这一步操作，数据将会被储存在self.pv中，其数据结构为DataView3d
            self.subscribe_data(
                'pv', ['default','stock_bar_1day',self.codes,'open,high,low,close', 10]
            )
            # 设定回测发生的时间
            self.add_clock(milestones='09:25:00')
    
        def pre_transform(self):
            # 通过to_dataframe()将pv转成了一个Dict(field, dataframe)
            pv = self.pv.to_dataframe()
    
            # 计算收益率ret和反转因子reverse
            ret = (pv['close'] / pv['close'].shift(1) - 1).fillna(0)
            reverse = -ret.rolling(window = 3, min_periods = 5).mean().fillna(0)
            reverse = zscore(reverse, axis = 1, nan_policy='omit')
    
            # 保存反转信号到self.reverse中，此时self.reverse和self.pv拥有同样的数据结构 
            self.reverse: DataView3d = create_data_view(
                NdarrayData.from_dataframes({'reverse':reverse})
            )
            # 以self.pv的时间戳为标准，将self.reverse和self.pv对齐时间戳
            self.reverse.align_with(self.pv)
    
        def on_clock(self):
            # 通过get方法获取在pre_transform中计算好的因子数据
            mysignal = self.reverse.get(fields='reverse')
            # 更新因子值, 因子值会被存在self.signal中
            self.update_signal(mysignal)
    
        def post_transform(self):
            # 由于self.signal的数据结构是DataView2D, to_dataframe()会将其直接转成dataframe
            signal = self.signal.to_dataframe()
            # 因子后处理: 去inf
            signal = signal.replace([np.inf,-np.inf], np.nan)
            # 再次储存到self.signal中
            self.set_signal(signal)
    ```
    
    在上面代码中，我们计算了一个简单的反转因子，通过继承SignalStrategy来编写逻辑。
    
    具体而言，
    
    - 首先该策略在初始化时需要传入一个窗口大小参数，在后面的函数中可以通过`self.window_size`使用该参数；
    - 在`init`方法中，订阅了股价数据，并设置了回测发生时间；
    - 在`pre_transform`方法中，计算股票收益率和反转因子，并将其保存到`self.reverse`中；
    - 在`on_clock`方法中，根据当前时刻，从`self.reverse`中获取计算好的反转信号数据，并将其更新到`self.signal`中；
    - 最后，在`post_transform`方法中对因子进行了后处理，去除了inf值，然后将其保存到`self.signal`中。
    
- evaluator.py

  ```python
  import pandas as pd
  from transmatrix.evaluator.signal import SignalEvaluator
  
  class MySignalEval(SignalEvaluator):
      def init(self):
          # 订阅因子评价所需要的数据
          self.subscribe_data(
              'pv', ['default','stock_bar_1day',self.codes,'open,high,low,close', 0]
          )
      
      def critic(self):
          # 通过to_dataframe()将pv转成了一个Dict(field, dataframe)
          # 此时, self.pv已根据self.strategy.signal进行时间对齐了
          critic_data = self.pv.to_dataframe()
          
          price = critic_data['close']
          factor = self.strategy.signal.to_dataframe()
          ret_1d = price.shift(-1) / price - 1
          ic = vec_rankIC(factor, ret_1d)
          print(ic)
          return ic
      
  def vec_rankIC(factor_panel: pd.DataFrame, ret_panel: pd.DataFrame):
      return factor_panel.T.corrwith(ret_panel.T, method = 'spearman').mean()
  ```

  在上面代码中，我们编写了一个简单的因子评价逻辑，通过继承SignalEvaluator来实现。

  首先，我们同样在int中订阅数据，然后在critic中计算IC值，其中因子数据通过self.strategy.signal获取。

- 关于如何使用配置文件将因子全流程串联和运行起来，详见[配置文件说明](path)。



#### 交易策略研究全流程

交易策略研究涉及交易策略逻辑的编写、策略评价逻辑的编写、数据的订阅，以及其他参数的配置。

以下是一个简单的交易策略例子，展示了如何使用Transmatrix进行交易下单和收益分析。

- strategy.py

  ```python
  from transmatrix import Strategy
  # 策略逻辑组件
  class TestStra(Strategy):
      def init(self):
          # 订阅策略所需数据
          self.subscribe_data(
              'macd', ['demo', 'factor_data__stock_cn__tech__1day__macd', self.codes, 'value', 10]
          )
          # 添加一个回调发生器，在每天14:00的时候执行self.callback_on_14函数
          self.self.add_scheduler(milestones = ['14:00:00'], handler = self.callback_on_14)
          self.max_pos = 300
      
      #回调执行逻辑： 行情更新时
      def on_market_data_update(self, market_data):
          # 获取最近三天样本空间内的macd值，并排序
          macd = self.macd.query(time=self.time, periods=3)['value'].mean().sort_values() 
          # macd值最小的两只股票作为买入股票
          buy_codes = macd.iloc[:2].index 
  
          for code in buy_codes:
              # 获取某只股票的仓位
              pos = self.account.get_netpos(code)
  
              if  pos < self.max_pos:
                  price = market_data.get('close', code)
                  self.buy(
                      price, 
                      volume=100, 
                      offset='open', 
                      code=code, 
                      market='stock'
                  )
                  
  	def callback_on_14(self):
          # 当前时间
          print(self.time)
  ```
  
  其中，在获得buy_codes的步骤也可以使用get_window来获得macd数据：
  
  ```python
  # 获取最近三天样本空间内的macd值，并排序
  macd = self.macd.get_window(field='value', length=3)
  macd = np.mean(macd, axis=0)
  # macd值最小的两只股票作为买入股票
  idx = np.argsort(macd)[:2]
  buy_codes = [self.codes[i] for i in idx]
  ```
  
  在上面代码中，我们实现了一个简单的交易策略，通过继承Strategy来编写逻辑。
  
  具体而言，
  
  - 首先该策略在初始化时需要传入一个窗口大小参数，在后面的函数中可以通过`self.window_size`使用该参数；
  - 在`init`方法中，订阅了因子macd数据，并添加了一个回调发生器；
  - 在`on_market_data_update`方法中，根据当前时刻（即market_data更新的时刻），对macd值最小的两只股票进行买入操作；
  - 在`callback_on_14`方法中，打印当前时间，即每天的14:00。
  
- evaluator.py

  ```python
  from transmatrix.evaluator.simulation import SimulationEvaluator
  
  import pandas as pd
  import numpy as np
  
  class MySimulationEval(SimulationEvaluator):
      
      def init(self):
          pass
  
      def critic(self):
          # 获得每日损益
          pnl = self.get_pnl()      
          print(pnl)
          return np.nansum(pnl)
  ```

  在上面代码中，我们编写了一个简单的因子评价逻辑，通过继承SimulationEvaluator来实现。

  首先，我们在然后在critic中通过self.get_pnl()获得策略运行结果的每日损益，并通过加和获得总pnl。

- 关于如何使用配置文件将因子全流程串联和运行起来，详见[配置文件说明](path)。
