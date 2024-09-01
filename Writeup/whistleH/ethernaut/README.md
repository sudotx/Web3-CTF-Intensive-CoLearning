## Usage

```shell
# on chain
forge script ./script/Level01.s.sol -vvvv --private-key $PRI_KEY --rpc-url https://rpc.ankr.com/eth_sepolia --broadcast
````

## Level 00 - Hello

### Target

1. 学习 Foundry 与链上合约的交互

### PoC

1. 请求 contract.password() 获取passkey
2. 请求 authenticate 解决solution

## Level 01 - Fallback

### Target

1. 学习合约如何接受/发送ETH
2. 理解修饰器的作用

### PoC

1. 发现receive函数修改了Owner，但是没有做权限控制
2. 为了满足receive函数中的修改Owner操作，因此请求 contribute 方法满足分支
3. 修改Owner后直接withdraw
4. 具体的PoC见 [Level 01-PoC](./script/Level01.s.sol)

## Level 02 - Fallout

### Target

1. 学习合约的构造函数

### Poc

1. 合约的构造函数必须为` constructor`，在solidity 0.4.22之前也使用类同名的构造函数

   ``` solidity
   contract TestConract {
   	constructor(address initialOwner) {
       	owner = initialOwner;
   	}
   	
   	// solidity 0.4.22之前的写法
       function TestConract(){
   		...
       }
   }
   ```

2. 所以那个constructor的注释主要是用于迷惑我们，直接调用Fal1out就可以完成了
3. 具体的PoC见 [Level 02-PoC](./script/Level02.s.sol)

## Level 03 - CoinFlip

### Target

1. 学习链上的随机数风险

### PoC

1. 采用了block.number和blockhash来进行伪随机数的生成，但是这些信息我们都可以直接获取
2. 部署对应的合约，进行相同的随机数生成算法计算，就能得到相同的结果
3. 为了保证block.number的不同，因此需要手动进行10次交易
4. 具体的PoC见 [Level 03-PoC](./script/Level03.s.sol)

## Level 04 - Telephone

### Target

1. 学习tx.origin和msg.sender的区别

### PoC

1. tx.origin是EOA账户的地址
2. msg.sender是调用合约的地址
3. 为了使tx.origin和msg.sender不一致，可以中间加入一个代理合约，利用代理合约去调用漏洞方法
4. 具体的PoC见 [Level 04-PoC](./script/Level04.s.sol)

## Level 05 - Token

### Target

1. 学习整数溢出漏洞

### PoC

1. 初始给予了20个token
2. 存在transfer交易，并且交易中直接扣除相应的数值，并没有使用safeMath或者其他校验
3. 对当前账户发起转账21token的交易，则直接发生整数溢出
4. 具体的PoC见 [Level 05-PoC](./script/Level05.s.sol)

## Level 06 - Delegation

### Target

1. 学习委托代理
2. fallback函数以及Call的用法

### PoC

1. 这里我们需要通过合约Delegation去调用Delegate的pwn方法
2. pwn方法中会修改Delegate合约的owner
3. 通过Delegatecall方法，并不会修改msg.sender，所以修改后的owner还是我们的玩家地址而不是Delegation的合约地址
4. 为了通过编译，我们在PoC要使用Call方法来调用Delegation中不存在的方法pwn
5. 具体的PoC见 [Level 06-PoC](./script/Level06.s.sol)

## Level 07 - Force

### Target

1. 学习selfdestruct

### PoC

1. Force合约中没有可以payable的函数
2. 查询Solidity官方文档发现存在selfdestruct，并且在selfdestruct的时候将合约剩余的eth转移到指定的账户中去
3. 构造一个新合约，存入1wei；在该合约上使用selfdestrcut，将challenge的地址作为转入地址实现转移eth
4. 具体的PoC见 [Level 07-PoC](./script/Level07.s.sol)

## Level 08 - Vault

### Target

1. 学习Solidity的内存模型

### PoC

1. 对于Private的数据，是无法直接通过合约进行访问。

   但是我们可以看到password存储在storage中，因此直接访问链上的Storage进行Leak

   参考[accessing-private-data-in-solidity](https://medium.com/@solidity101/100daysofsolidity-070-accessing-private-data-in-solidity-smart-contracts-unveiling-hacks-tests-7724485fe004)

   ```shell
   cast storage 0x72b3ea50eBACc33E49c079Be6559aF458cfc52e9 1 --rpc-url=https://rpc.ankr.com/eth_sepolia
   0x412076657279207374726f6e67207365637265742070617373776f7264203a29
   ```

2. 使用获取到的password进行解锁
3. 具体的PoC见 [Level 08-PoC](./script/Level08.s.sol)

## Level 09 - King

### Target

1. 学习Solidity Dos攻击原理

### PoC

1. 理解合约，合约实际完成的步骤如下
   1. 向合约转账，如果超过king的转价，则把转入的金额直接转给king。
   2. 转入者成为新的king。

2. 我们要实现保证不会再有新的king诞生，可以利用Solidity的Dos攻击，当有新的king产生时，在我们的`receive`函数中设置revert()操作，使得不会再有新的king产生。
3. 具体的PoC见 [Level 09-PoC](./script/Level09.s.sol)

## Level 10 - Reentrance

### Target

1. 学习Solidity的可重入攻击原理

### PoC

1. 这个是经典的可重入攻击模型，其实结合King来看，我们都可以发现主要在于调用call或者transfer时无法对接收方的receive操作做出过多限制。
2. 在receive函数中继续调用withdraw操作即可
3. 具体的PoC见[Level 10-PoC](./script/Level10.s.sol)



