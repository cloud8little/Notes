
#### Chainlink VRF
有两种获取随机数的方式，一种利用订阅模式，通过subscription manager进行管理，另外一种则是直接获取

##### 订阅者模式

1. suscription manager 方式
vrf.chain.link进入管理页面，可以createSubscription/cancel, addConsumer/removeConsumer, addFunds/removeFunds
核心是通过VRFCoordinatorV2合约进行操作

关于订阅者的概念： (不用每次都支付LINK，只需要预付费就行)
- subscription id: 64位无符号整数
- subscription accounts: 用来支付LINK的账户
- subscription owner: 创建管理订阅者账户的钱包地址，任何能够添加LINK到订阅余额的账户，添加授权消费合约哈希或者取钱
- consumers: 消费合约  需要获取随机数的那本合约哈希
- subscription balance: 订阅账户的LINk余额  

总体GAS的消耗可以分为三部分
- Gas price 当前以太坊网络gas情况
- Callback gas: 回调函数使用的gas
- Verification gas 验证随机数的gas

所有消耗的gas最终只能由执行完交易才能确认，但是可以定义最大使用的gas限额
- Gas lane: 你愿意为一个Oracle请求支付的最大的GAS  可以在网络拥堵的时候快速处理请求 指定恰当的keyHash来指定。
- Callback gas limit: 你愿意在callback请求函数中花费的最大的gas callbackGasLimit
有两种账户
- EOA 外部拥有账户， 可以控制智能合约，交易只能由EOA发起  即普通账户
- 智能合约：没有private key， 作为去中心化应用来执行设计好的指令

步骤

消费合约必须继承自VRFConsumerBaseV2， 并且实现fulfillRandomWords 函数，这是标准的callback VRF函数，通过调用VRF Coordinator requestRandomWords来提交VRF请求
- keyHash
- s_subscriptionId
- requestConfirmations
- callbackGasLimit
- numWords

VRF coordinator释放一个事件 event
VRF service选中event, 等待特定个块确认，然后给VRF coordinator回复N个随机数以及一个证明
VRF coordinator验证这个证明，并且调用消费合约的fulfillRandomWords函数，执行完后，最终的gas消耗记录下来

(Gas price * (Verification gas + Callback gas)) = total gas cost

total gas转成LINK，用ETH/LINK数据种子转换，如果data feed不可用，则用
fallbackWeiPerUnitLink 转换

(total gas cost + LINK premium) = total request cost

最小LINK余额，用下列公式估算

<code>
(((Gas lane maximum * (Max verification gas + Callback gas limit)) / (1,000,000,000 Gwei/ETH)) / (ETH/LINK price)) + LINK premium = Minimum LINK
</code>

测试网实验

创建subscription  调用VRFCoordinatorV2: 0x2ca8e0c643bde4c2e08ab1fa0da3401adad7734d  （测试网）
https://goerli.etherscan.io/tx/0x08ebee8cc80ba4760f12023b61f20b8cb5d58067050952093804d414b49ecb9e
1. AddFunds 
给LINK合约打LINK 调用transferAndCall
https://goerli.etherscan.io/tx/0x2454779ed11de0055a1ee9acb68ed799fc05f93546fe3fe8b3cd1aa0c7681e72
1. Add Consumer
https://goerli.etherscan.io/tx/0x2965e591512998e6d751e82f790a228dcf0bd35b3f92d0c6a03aa5164d5f3d65
addConsumer 
记住subscriptionId  在我的账号测试网中是 8817
参数： 想要获取随机数的合约哈希

关于Coordinator合约：
getConfig 可以获取到如下这些参数
  s_config.minimumRequestConfirmations,
  s_config.maxGasLimit,
  s_config.stalenessSeconds,
  s_config.gasAfterPaymentCalculation
keyHash的配置： 
https://docs.chain.link/docs/vrf-contracts/#configurations
https://docs.chain.link/vrf/v2/security  【待定】
在合约中动态创建subscription
利用VRFCoordinatorV2Interface的createSubscription()方法

##### 直接获取模式

Gas总体消耗有以下几个变量
- Gas Price 当前网络情况
- Callback gas  回调需要的gas
- Verification gas: 链上验证随机数的gas
- Wrapper overhead gas  VRF Wrapper 合约花费的gas VRF v2 Wrapper合约设计下次再讲

这种方案有两部分
- VRF v2 Wrapper 链上部分：  VRF Coordinator的wrapper，提供给消费合约的接口
- VRF v2 Coordinator 链上：用来和VRF service 交互的合约，当请求随机数时，释放一个event,然后验证随机数，证明是如何被VRF service产生的
- VRF service 链下：通过监听订阅Coordinator产生的event logs,基于block hash和 nonce计算一个随机数，而后给VRFCoordinator发送一个交易，交易包括随机数，以及如何产生的一个证明

步骤
- consuming contract 需要继承VRFV2WrapperConsumerBase， 实现fulfillRandomWords 方法，调用requestRandomness获取随机数，传入参数
  - requestConfirmations VRF service需要等待的块数
  - callbackGasLimit: 最大愿意支付的回调函数使用gas
  - numWords: 请求的随机数的个数

-  consuming contract调用VRFV2Wrapper的calculateRequestPrice 函数，估计使用的所有交易费，调用LINK的transferAndCall, 
   -  (Gas price * (Verification gas + Callback gas limit + Wrapper gas Overhead)) = total gas cost
   -  总的费用转换成所需的LINK个数，根据ETH/LINK价格，获取不到则调 fallbackWeiPerUnitLink
      -  除此之外还有LINK的抽成
         -  Wrapper抽成： 有一个比例，在supported network里面可以查到  https://docs.chain.link/vrf/v2/direct-funding/supported-networks/#goerli-testnet  目前都是0
         -  Coordinator抽成： flat fee. 在fulfillmentFlatFeeLinkPPMTier1 参数中固定了
    - VRF Coordinator 释放事件event
    - VRF service拿到event,等待确认区块数，回复给VRF coordinator随机数和证明
    - VRF coordinator 链上验证这个随机数，调用wrapper的fulfillRandomWords 回调函数
    - VRF Wrapper回调consuming contract.

注意： callbackGasLimit <=  maxGasLimit - wrapperGasOverhead.

实验：
1. 创建subscription,获取subscriptionId, add Funds 2LINK, 然后requestRandomWords 有pending的请求提示funds不足 明天看一下是不是能成功
2. 


   


