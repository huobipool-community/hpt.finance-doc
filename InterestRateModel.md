# InterestRateModel

## 说明

计算收益率的模型接口，每个币可以有一个单独的实现，主要提供了下面的两个方法：

> 借款利息

```
function getBorrowRate(uint cash, uint borrows, uint reserves) external view returns (uint)
```

> 存款利息

```
function getSupplyRate(uint cash, uint borrows, uint reserves, uint reserveFactorMantissa) external view returns (uint);
```

源码中有三种实现：

1. JumpRateModel
2. WhitePaperInterestRateModel
3. DAIInterestRateModelV2

## JumpRateModel

提供了一个基本的费率计算模型，使用初始设置的值加上运行过程中的借款率综合计算得出

> 名词解释

1. baseRatePerBlock 每个块的借款利率的一个基数
2. multiplierPerBlock 每个块的借款利率的一个加数
3. kink 对借款利率设定的一个阀值，大于阀值时使用一个计算公式，小于阀值时使用一个计算公式
4. jumpMultiplierPerBlock 当借款利率大于阀值是加权的借款利率
5. utilization 市场的借款利率可以认为是池子中的资金用于借款的比例

> 计算公式

>> utilization

```
utilization = borrows / (cash + borrows - reserves)
```

>>> borrowRate

>>>> 当当前的 **借款利率 < 阀值** 时

```
borrowRate = utilization * multiplierPerBlock + baseRatePerBlock
```

>>>> 当当前的 **借款利率 > 阀值** 时

```
borrowRate = (utilization - kind) * jumpMultiplierPerBlock + kind * multiplierPerBlock + baseRatePerBlock
```

>>> supplyRate

```
supplyRate = utilization * borrowRate * (1 - reserveFactor)
```

## WhitePaperInterestRateModel

白皮书中定义的费率模型，和 **JumpRateModel** 相比，少了对不同借款利用率的阀值处理，其他的计算规则一样

## DAIInterestRateModelV2

The parameterized model described in section 2.4 of the original Compound Protocol whitepaper.
   Version 2 modifies the original interest rate model by increasing the "gap" or slope of the model prior
   to the "kink" from 0.05% to 2% with the goal of "smoothing out" interest rate changes as the utilization
   rate increases.
   
在**JumpRateModel**的基础上的实现，不同于**JumpRateModel**在开始设定一个baseRate和multiplyRate，这个模型从**PotLike**和**JugLike**两个合约里获取数据，根据获取到的数据计算baseRate和multiplyRate。这两个合约的原理还有待研究

> 名词解释

1. dsrPerBlock: 从**PotLike**合约中获取到的数据
2. stabilityFeePerBlock: 从**JugLike**合约获取到的数据
3. gapPerBlock: 每个块从baseRate中保留的比例，目前是恒定值 **2%**

> 计算公式

>> dsrPerBlock

通过调用**PotLike.dsr**方法得到每秒的产生量，这是一个1^27次方的数，减去1^27得到比例,再除以1^9将结果转换为1^18，这是每秒的产生量，按eth出块时间为15秒，将该值再乘以15。因此最终的计算公式为：

```
dsrPerBlock = (PotLike.dsr() - 1^27) / 1^9 * 15
```

>> stabilityFeePerBlock

通过调用**jug.ilks("ETH-A")**得到每秒的额外产量和**jug.base**的值相加得到每秒的固定产量，在减去1^27乘以1^18除以1^27转换单位，再乘以15得到每个块的产量。因此计算公式为：

```
stabilityFeePerBlock = (jug.ilks("ETH-A").duty + jug.base() - 1^27) * 1^18 / 1^27 * 15
```

>> baseRatePerBlock

```
baseRatePerBlock = (dsrPerBlock * 1 ^ 18) / (1 - reserveFactor)
```

>> multiplierPerBlock
>>> 如果**baseRatePerBlock < stabilityFeePerBlock**

```
multiplierPerBlock = (stabilityFeePerBlock - baseRatePerBlock + gapPerBlock) * 1 ^ 18 / kind
```

>>> 如果**baseRatePerBlock >= stabilityFeePerBlock

```
multiplierPerBlock = gapPerBlock * 1 ^ 18 / kind
```

>> supplyRate
>>> 如果当前池子总的资金量为0,那么使用和**JumpRateModel**一样的计算规则
>>> 如果当前池子总的资金量不为0

```
supplyRate = 可用资金 * dsrPerBlock / (可用资金 + 借出资金 - 保留资金) + supplyRate
```

## 结论

新的链上目前还支持不了**DAIInterestRateModelV2**，**JumpRateModel**本身的计算模型对于目前已经足够使用，建议所有币种直接使用**JumpRateModel**，不同的币种，提供不同的初始化参数。之后需要更新参数或模型，再从新设置新的模型。