Compound 是一个开放性金融借贷协议，其与点对点借贷市场的根本区别在于，它让拥有一种资产的人可以借入另一种资产，一定程度上可以理解为区块链去中心化银行。

Compound 基于流动性的资金池（和Uniswap一样），借贷利率由资金池供需决定。



二、Compound核心逻辑
根据用户存储到Compound智能合约的货币市场（Money Market）的底层资产（Underlying Asset），智能合约按照兑换率发放对应数量的由Compound铸造（Mint）的cToken到用户账户。持有cToken用户可以收取利息，也可以随时赎回（Redeem）Token，也可以设置作为抵押物（Collateral）借出（Borrow）其他cToken。

而用户的存款利息和贷款利息之间差额的利息，构成了Compound的收入。举例，用户在Compound上存入100USDT，可以获得8%的年收益；在平台上借出100USDT，则需要支付13%的利息。差额的5%的利息，就为Compound的收入。

Compound的盈利为手续费和API组件。



三、底层资产与预言机
BAT/DAI/ETH/USDC/USDT/WBTC/ZRX 每个底层资产都会有对应的独立的资金池，当发生一笔抵押时，资金池会增加；当发生一笔借款时，资金池会减少。比如抵押ETH借出USDC时，ETH资金池会增加，而USDC资金池会减少。

由于存在实时计算借款人抵押物的资产价值、放贷人存入底层资产兑换为代币的数量等需求，因此Compound需要预言机实时提供各种底层资产的价格信息。



四、借贷利率
借贷利率模型是Compound 最核心的元素，涉及兑换率（Exchange Rate）、使用率（ Utilization Rate）、存款利率（Supply Rate）、借款利率（Borrow Rate）、抵押率(Collateral Factor)等概念。

Compound按照15秒一个区块作为利息计算时间单位。年化利率=每个区块利率*2102400。

totalCash：放贷人存入智能合約，但尚未被借走的Token的数量
totalBorrows：所有借款人应偿还Token的数量（借款本金+利息)
totalReserves：平台储备金总数量（借款人利息的部分作为平台储备金保留)
totalSupply：所有放贷人获得的cToken的总数量


1.兑换率 Exchange Rate
exchangeRate = (totalCash + totalBorrows – totalReserves) / totalSupply

2.使用率 Utilization Rate
utilizationRate = totalBorrows / (totalCash + totalBorrows – totalReserves)

3.备用金 Reserves
totalReservesNew = interestAccumulated * reserveFactor + totalReserves

备用金因子 （reserveFactor ）的作用是确保抵押收益率小于借款利率。

4.借款利率 Borrow Rate
Compound借款利率有两个版本，1.0版本为线性利率模型，2.0版本为分段利率模型，1.0版本是2.0版本的特殊情况。

2.0版本：

2.0版本的核心思路：如果使用率超过一定比例（kink），则使用分段利率，否则则采用1.0版本的利率模型。

当 utilizationRate<= kink时候

borrowRate = baseRate + utilizationRate*multiplier

当 utilizationRate>kink时候

borrowRate = baseRate + utilizationRate*multiplier + (utilizationRate-kink)*jumpMultiplier

可以将kink理解为边际利率，utilizationRate-kink理解为溢出利率

上述公式用中文理解，可以翻译为：

当使用率 <= 边际利率 时候：

借款利率=基础利率+使用率*使用率乘数

当 使用率 > 边际利率 时候：

借款利率=基础利率+使用率*使用率乘数+边际利率*边际利率乘数

其中：

baseRate 基础利率。

对DAI baseRate 依赖于DAI的Dai Savings Rate (DSR)

对其他Token，baseRate = baseRatePerYear/blocksPerYear

multiplier、jumpMultiplier为对应的乘数，算法有点复杂，具体可以参考其代码。

1.0版本

borrowRate = baseRate + utilizationRate*multiplier

1.0版本就是 当 utilizationRate<= kink时候 特殊情况



5.存款利率 Supply Rate
存款利率的计算需先得到借款利率，和借款利率一样，都是每个区块计算一次，同一个区块的放贷人对于相同的资产获得相同的放贷利率。

supplyRate=borrowRate * (1 – reserveFactor) * utilizationRate

即：

放贷利率=借款利率*(1-备用金因子)*使用率



6.抵押率/抵押因子 Collateral Factor
针对不同的资产，Compound内有着不同的抵押因子，抵押因子范围是0-1，代表用户抵押的资产价值对应可得到的借款的比率，为零时表示该类资产不能作为抵押品去借贷其他资产。小市值资产，抵押因子小，同样的抵押资产能借出的资金少。





五、抵押与贷款额度
Compound采取超额抵押，抵押品价值是贷款额度的1.5倍。抵押即可获得利息，与获得的贷款额度是否被使用无关。这样可以鼓励不需要贷款的用户将空闲的代币抵押到资金池，扩大市场的供给。

可借出数量=贷款额度/代币当前价格。由于当借款额度为负时，会导致抵押物被清算，所以一般要确保借贷之后保持一定数量的贷款额度。平台推荐安全最大借出数量是不要超过可借出数量的80%。

Compound账本遵循复式记账法（Compound Interest）的基本公式：资产=负债+所有者权益，以下途径会增加贷款额度：

增加抵押物
抵押物价格上涨
归还借贷
抵押获得利息（直接增加到抵押物余额里，和1类似）


以下途径会减少贷款额度：

减少抵押物
抵押物价格下跌
新增借贷
借款支付利息（直接增加贷款额，和3类似）


六、清算与清算激励
假设1 REP的价格等同于1 BAT，你存入1.5 BAT 作为抵押并借出1 REP，那么当BAT价格下跌到 1.3 BAT = 1 REP时，对于平台就有风险了，因为如果BAT继续下跌的话，抵押物就无法偿债，因此这时智能合约就会允许其他人买走你的抵押物即1.3 BAT。

清算（Liquidation）：Compound中的清算是指当用户抵押资产价值小于借款价值时，清算人可以代替被清算人还款一部分，目前可以最多一次帮还50%，同时清算人可以得到被清算人抵押物的一定百分比奖励。

关闭因子（Close Factor）：在清算过程中，清算人可以帮助贷款人还掉的债务最大比例，在0~1之间。这个因子可以被连续调用，直到用户借款订单处于安全状态。

清算激励（Liquidation Incentive）：目前清算人得到有8%奖励。



七、COMP代币
2020年2月，Compound宣布将发行1000万枚COMP，其中50.05%预留给使用协议的用户，23.96%分配给Compound Labs的股东，22.26%分配给团队，3.73%分配给未来的团队成员。423万枚将在大约四年内被分发给所有用户。每天会分发2880个COMP以供进行借贷挖矿，借贷金额越大，获得的COMP也越多。借贷双方平分奖励的COMP。



COMP是治理代币，它的功能和另一个去中心化借贷平台MakerDAO的治理代币MKR一样，主要有：

拥有1%的委托代币就可以发起治理提议，包括增加新资产、改变利率模型等各种协议的参数或变量
采用一币一票的机制对Compound重大提议进行投票，即一个COMP对应一个选票；
在Compound借贷平台中，借款人可以用COMP去偿还利息部分。