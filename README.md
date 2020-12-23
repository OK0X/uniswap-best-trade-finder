# 给出两种token，假设为TokenA和tokenB，寻找最优交易路径。

## 最优交易计算方法bestTradeExactIn

位于uniswap-sdk里面的，trade.ts中，该方法对于给定的pair数组，输入的币种数量（TokenAmount），输出币种（Token），计算出输出数量最大的前几位trades。其中可以指定输出结果（trade）数量（maxNumResults 参数控制，默认为3，降序排列）和最大允许跳转次数（maxHops参数控制，默认为3）。

## bestTradeExactIn内部是如何实现的？

主要做了两个方面的事情：
1. 找到tokenA到tokenB的兑换路径
- for循环所有的交易对，如果交易对里面两种token中没有一个是输入的token，则去除该交易对。
- 如果pair中两种代币，其中一种为输入的token，则调用pair的getOutputAmount方法。获取输出的token信息。
- 如果输出的token即为目标token，则加入bestTades。
- 如果不是，则将getOutputAmount的获得的token做为输入token递归调用bestTradeExactIn方法，继续寻找到达目标token的路径，当然不能超过最大跳转次数maxHops（默认为3跳）。

2. 排序确定最佳的trades
sortedInsert里面有一个策略函数tradeComparator(a: Trade, b: Trade)，tradeComparator的结果小于0代表a优于b，反之则b优于a，其中有以下几种比较策略：
- 首先是比较输入输出数量，输出越大越好，输出数量相等时，输入越少越好。
- priceImpact越小越好，因为越小交易越不容易失败，因为越小越不容易超出用户设定的slippageTolerance，即允许的滑点.
- route.path越小越好，因为越小gas费越少。

## 在自己的dex中进行测试的步骤

1. 在ropsten网络发布多种token,其中包含要演示的两种Token。

2. 在自己的dex中创建多种pair，并添加流动性。

3. 通过bestTradeExactIn方法获取best trades。

4. 使用best trade，调用router合约的swapExactTokensForTokens执行swap。

## More

1. 关于uniswap-interface中的Trades.ts
- 使用useTradeExactIn()包装了sdk里面的bestTradeExactIn方法，useTradeExactIn方法主要作用是调用useAllCommonPairs()根据已有token，组合出所有的pair。其中还包括重复的pairs,无效的pairs，然后再一 一过滤去除。
- 执行useAllCommonPairs时候有获取链上合约数据，如：token的储备量。而每个pair是一个单独的合约，所以useAllCommonPairs应该是一个相对耗时的操作。

2. 可能更好的方案
- 把pair合约里面的event Sync(uint112 reserve0, uint112 reserve1)移到factory中来记录（当然还要记录下相应的token address），让pair合约调用factory的方法来触发事件，这样一个事件就可以监听所有pair合约中两种token储备量的变动。
- create pair本身是有事件记录的。
- 可以在nodejs后台监听以上两个事件。
- app端只需一次请求便可以获取所有有效pair及其储备量信息，无需通过useAllCommonPairs进行复杂计算和多次请求。甚至可以将bestTradeExactIn放在后端计算，这样一次请求便可快速获取best trade,然后app端直接用beast trade去swap即可。

该方案已发布实现示例：
- nodejs代码见https://github.com/OK0X/uniswap-q1
- 合约代码见https://github.com/OK0X/uniswapv2-factory和https://github.com/OK0X/uniswapv2-router2
- 示例中用到的合约和交易对信息
factory: 0x74073279A2809fAE01C63838061E58Ea19faB76A

routerV2: 0xcF90da2D7134d4001FDE1094AAFE95593A9B5A22

WETH：0x24564639ef1615887f23fefb2265289220894139

DAI : 0x2c3af037312ab82a367799c27e3d4e7263c0f04d   

USDT: 0x3c5A535D0bda6F11884e178c3AfA268154957e75  

CRO: 0xbe5F7BC290Cb4A98cbdFFa868F1Ab5CA68BaDFea  

WETH/DAI    

WETH/USDT 

WETH/CRO

DAI/CRO

USDT/CRO

