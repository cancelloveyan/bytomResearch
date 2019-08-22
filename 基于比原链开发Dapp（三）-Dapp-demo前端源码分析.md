# 简介

​	本章内容会针对比原官方提供的dapp-demo，分析里面的前端源码，分析清楚整个demo的流程，然后针对里面开发过程遇到的坑，添加一下个人的见解还有解决的方案。



### 储蓄分红合约简述

为了方便理解，这里简单说说储蓄分红合约的内容，具体可以查看[储蓄分红合约详细说明](https://github.com/oysheng/bufferserver/blob/master/docs/instance.md)，储蓄分红，顾名思义就是储蓄之后，当达到一定的时间，按照比例返回本息这样的意思，所以demo中拆分成**saving**（储蓄）与**profit**（提现）两个页面，本章内容是针对合约交易的提交，所以只针对储蓄页面说明。



### 比原官方Dapp-demo使用说明

[比原官方demo地址](http://app.bycoin.io:9001/ )

![img](https://raw.githubusercontent.com/cancelloveyan/bytomResearch/master/image/3_p1.png)

1）访问的前提需要用chrome打开[比原官方demo地址](http://app.bycoin.io:9001/ )，同时安装bycoin插件，在应用商店搜索就行；

2）安装完bycoin，需要初始化用户信息，新建或者导入备份文件去恢复用户；

3）填写指定资产数量，点击确定；

![img](https://raw.githubusercontent.com/cancelloveyan/bytomResearch/master/image/3_p2.png)



4)弹出合约交易专用页面，填写密码，点击确认；

5)查看交易流水

![img](https://raw.githubusercontent.com/cancelloveyan/bytomResearch/master/image/3_p3.png)



### 前端源代码分析

源码 ： [储蓄分红合约前端源代码](https://github.com/Bytom/Bytom-Dapp-Demo) （本章内容讲解的是 2019年7月10号 最新版的代码）

![img](https://raw.githubusercontent.com/cancelloveyan/bytomResearch/master/image/3_p4.png)

前端代码是基于前端框架react去做的，很容易读懂，结构如上，我们来看看作为储蓄页面（saving）Bytom-Dapp-Demo1\src\components\layout\save\index.jsx

```go
//提交后的方法
    FixedLimitDeposit(amount, address) //####### 1.
      .then(()=> {
          //####### 2.
          this.refs.btn.removeAttribute("disabled");
          this.setState({
            error:'',
            msg:`Submit success!!! you spent ${amount} deposite asset,and gain ${amount} billasset.`
          })
        }).catch(err => {
          //####### 3.
          this.refs.btn.removeAttribute("disabled");
          this.setState({
            error:err,
            msg: ''
          })
        })
```

1)接收了输入框的金额，还有当前用户的地址；

2）成功后提示内容；

3）失败后提示内容；

接下来到**FixedLimitDeposit**方法

```go
export function FixedLimitDeposit(amount, address) {
  const object = { 
    address: address,
    amount: amount,
    parameter: [amount, address]
  }
  return submitContract(listDepositUTXO, createContractTransaction, updateDatatbaseBalance, object)  //####### 1.
}
```

1)  传入三个方法体分别是 listDepositUTXO（查找当前合约所有的UTXO）， createContractTransaction（创建提交前的合约参数），updateDatatbaseBalance（更新用户的提交列表）



进入Dapp-Demo1\src\components\layout\save\action.js 的 **submitContract**方法

```go
return new Promise((resolve, reject) => {
    //list available utxo
    return listDepositUTXO().then(resp => { //####### 1.

      //create the Contract Transaction
      return createContractTransaction(resp, amount, address).then(object =>{ //####### 2.
        const input = object.input
        const output = object.output
        const args = object.args

        const utxo = object.utxo

        //Lock UTXO
        return updateUtxo({"hash": utxo}) //####### 3.
          .then(()=>{

            //Transactions
            return window.bytom.send_advanced_transaction({input, output, gas: GetContractArgs().gas*100000000, args}) //####### 4.
              .then((resp) => {
                  //Update Balance
                  return updateDatatbaseBalance(resp, ...updateParameters).then(()=>{//####### 5.
                    resolve()
                  }).catch(err => {
                    throw err
                  })
              })
              .catch(err => {
                throw err.message
              })
          })
          .catch(err => {
            throw err
          })
      }).catch(err => {
        throw err
      })
    }).catch(err => {
      reject(err)
    })
  })
```

1) 首先调用listDepositUTXO 拿到当前节约锁定的所有UTXO的信息，待会详细说明；

2）调用 createContractTransaction 方法，组装好合约的对应信息参数；

3）选取要使用的 UTXO后，调用updateUtxo  告诉bufferserver ，该UTXO已经被使用，更改状态，防止其他人调用了；

4）执行window.bytom.send_advanced_transaction方法，参考[插件钱包API](https://bytom.github.io/dapp-sdk-doc/#/common?id=bytomsend_advanced_transaction)，是高级交易方法，这个是bycoin插件的原生方法，调起 提交交易的页面，让用户输入密码；

5）交易确认后，调用 updateDatatbaseBalance 提交数据到后端；

------------------------------------------------------------------------------------------------------------------------------------------------

再来看看api.js的**listDepositUTXO** 方法，所有与bufferserver交互的接口全部写到这个文件里面：

```go
function listDepositUTXO() {
  return listDappUTXO({//****** 1.
    "program": GetContractArgs().depositProgram,   
    "asset": GetContractArgs().assetBill,         
    "sort": {
      "by":"amount",
      "order":"desc"
    }
  })
}

//Api call from Buffer server
export function listDappUTXO(params)
{
  let url
  switch (window.bytom.net){
    case "testnet":
      url = "/dapptestnet/list-utxos"
      break
    default:
      url = "/dapp/list-utxos"
  }
  return post(url, params).then(resp => resp.data)
}
```

很明显最终是调用bufferserver的/list-utxos方法，非常简单，值得一提的是

1）里面的结构体根据***program***（合约代码）与***asset***（资产ID）去查找UTXO，这里其实底层是调用官方的blockcenter接口的，后面会细说；



继续看看Dapp-Demo1\src\components\layout\save\action.js 的createContractTransaction方法

```go
function createContractTransaction(resp, amount, address){
  return new Promise((resolve, reject) => {
    //utxo pre calculation
    const limit = GetContractArgs().radio * 100000000   //****** 1.
    if (resp.length === 0) {
      reject( 'Empty UTXO info, it might be that the utxo is locked. Please retry after 60s.')
    } else if (amount < limit) {
      reject( `Please enter an amount bigger or equal than ${limit}.`)
    }

    const result = matchesUTXO(resp, amount) //****** 2.
    const billAmount = result.amount
    const billAsset = result.asset
    const utxo = result.hash

    //contract calculation
    if (amount > billAmount) {
      reject('input amount must be smaller or equal to ' + billAmount + '.')
    } else {
      const input = []
      const output = []

      const args = contractArguments(amount, address) //****** 3.
	  
      input.push(spendUTXOAction(utxo)) //****** 4.
      input.push(spendWalletAction(amount, GetContractArgs().assetDeposited)) //****** 5.

      if (amount < billAmount) { //****** 6.
        output.push(controlProgramAction(amount, GetContractArgs().assetDeposited, GetContractArgs().profitProgram))
        output.push(controlAddressAction(amount, billAsset, address))
        output.push(controlProgramAction((BigNumber(billAmount).minus(BigNumber(amount))).toNumber(), billAsset, GetContractArgs().depositProgram))
      } else {
        output.push(controlProgramAction(amount, GetContractArgs().assetDeposited, GetContractArgs().profitProgram))
        output.push(controlAddressAction(billAmount, billAsset, address))
      }

      resolve({ //****** 7
        input,
        output,
        args,
        utxo
      })
    }
  })
}
```

这个方法比较复杂，我们一步一步来

**这里先看看** 7），最终返回的内容是 input、  output、  args、  utxo， 跟发送交易页面里面的input、  output、  args对应起来，如 图K

![img](https://raw.githubusercontent.com/cancelloveyan/bytomResearch/master/image/3_p5.png)

​										 (图K)

***上一章说过，所有比原链的交易都要存在质量守恒定理，input与output的总和要相对应，合约交易里面执行方法必须需要参数，这里的args就代表传入的参数，utxo是代表 utxo的id***



1）做一下限制，设置最少值

2）matchesUTXO ，根据上面的内容，刚刚已经通过**listDepositUTXO** 拿到所有可用的UTXO列表，这个时候要根据用户输入的数额amount，选择一个起码 大于或等于的 amount 的UTXO出来；

3）contractArguments ，构建args，就是合约里面方法的参数；

4）通常合约交易会有自己资产的input，合约UTXO的input，这里是要解锁的utxo的input；

5）这个是钱包资产的input；

6)  上一章内容说过，解锁合约的交易，必须根据合约里面的逻辑，计算出对应的结果，所以这里的逻辑跟合约里面逻辑是一样的，[储蓄分红合约详细说明](https://github.com/oysheng/bufferserver/blob/master/docs/instance.md) 如图；

![img](https://raw.githubusercontent.com/cancelloveyan/bytomResearch/master/image/3_p6.png)

判断逻辑一样，这里插件钱包跟上一章说的pc钱包接口的结构有所不同，但是原理一样。



最后我们看看src\components\layout\save\action.js 的**updateDatatbaseBalance** 方法

```go
function updateDatatbaseBalance(resp, amount, address){
 return updateBalances({
    "tx_id": resp.transaction_hash,
    address,
    "asset": GetContractArgs().assetDeposited,
    "amount": -amount
  }).then(()=>{
    return updateBalances({
      "tx_id": resp.transaction_hash,
      address,
      "asset": GetContractArgs().assetBill,
      "amount": amount
    })
  }).catch(err => {
    throw err
  })
}


export function updateBalances(params)
{
  let url
  switch (window.bytom.net) {
    case "testnet":
      url = "/dapptestnet/update-balance"
      break
    default:
      url = "/dapp/update-balance"
  }
  return post(url, params)
}
```

同样是调用bufferserver，这里是调用update-balance方法，把成功后的交易提交到后端。

### 小结

上面介绍了dapp-demo前端代码的内容，介绍了里面的方法，除了插件api的调用比较复杂外，其他都是普通的应用逻辑调用，主要理解了**质量守恒定理**，剩下的都是对数据审核数据的问题，非常简单。

----------------------------------------------------------------------------------------------------------------------------------------------------------

### 遇到的坑

有应用开发的读者应该一下子就能理解到问题核心吧，我现在在说说里面的坑；

1） UTXO锁定接口容易被刷； 假如我一个开发人员知道这个接口，狂刷你这个接口狂锁应用的UTXO，这样应用长期都会瘫痪状态；

**解决方案**：这个应该从应用方面去考虑，譬如接口加一些一次性的验证码，加refer的监控，加授权之类的方式，后端加上交易监控，去维持着各种情况UTXO的状态，比较抽象，而且比较复杂，不过这是必须的；



2）并发问题；跟1）一样，就算我是一个正常用户，选择了一个UTXO解锁后，居然通过http接口告诉后端去锁定，调起 **输入密码页面**  （图K），这个时候如果用户不输入密码不提交，在比原链上该UTXO是没有被解锁，但是bufferserver会却锁住了。

**解决方案**： 后端源码是锁定一段时间，如果没有使用，则定期解锁出来，这种情况也是需要应用的监控判断，维护所有utxo的状态，个人建议在发合约的时候，**发多笔UTXO锁定合约**，可用的UTXO就会变多，这个时候有些同学问，TPS岂不是也一样不高，如果用过火币的同学就知道了，区块链交易本来就不太注重TPS，而且火币的交易必须要超过60-100个区块，才确定一笔交易，这个看应用开发者如何去判断，取舍。



3）用户交易信息接口容易被刷；跟1）一样，交易完成后，直接通过http接口去提交数据，我狂刷，岂不是亿万富翁....；

**解决方案**：想要用户的交易信息，生成交易账单，可以直接用插件的接口，不过要通过合约编码去筛选出来，笔者是通过监控区块链浏览器所有交易，进入数据库交易表的方式，这样可以时时刻刻监控所以交易。



4）容易产生链式错误； 这里dapp-demo发的是一个合约的UTXO，假如用户提交交易之后会产生新的UTXO，但是这个UTXO还没有确认的，bufferserver的list-utxo接口会把还没有确认的UTXO，从而解决并发问题，但是我一个开发人员，知道合约的编码，随便写个交易提交了，虽然肯定会失败，但是需要时间，这个时候bufferserver也把这个肯定失败的UTXO返回过来前端，一直链式产生一堆交易，很容易产生链式失败。

**解决方案**：1）这里我跟比原官方的老大深深讨论过，最优方案当然是合约本身设置一个密码，输入参数必须要根据这个密码去加incode密过，传入合约交易参数，合约本身在解释的时候，再decode解密验证，保证出入的参数是官方的，这样就不会有人攻击.....不过结论是，暂时比原链的合约引擎不支持。

2）一定要隐藏好合约逻辑，其他人就没办法去恶意调用恶意占用，例如前端代码混淆，或者args参数是后端生成，另外建议比原的blockcenter的build-transaction接口参数可以加密这样，去掩盖合约逻辑。

### PS:这是笔者对于以上问题的思考，有更好的解决方案，欢迎一起讨论。



### 总结

​	这种内容主要说了前端代码的源码分析，还有设计上的逻辑坑，具体的解决方案应该跟官方的开发人员沟通还有讨论，区块链的交易本来不追求大并发，但是也需要一定的并发性，笔者在第四章才根据bufferserver内容，在针对上面的问题，做出一些个人见解还有建议。