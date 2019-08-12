# 简介

​	本章内容会针对比原官方提供的dapp-demo，分析里面的前端源码与bufferserver源码，分析清楚整个demo的流程，然后针对里面开发过程遇到的坑，添加一下个人的见解还有解决的方案。



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



进入**submitContract**方法

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

4）执行window.bytom.send_advanced_transaction方法，是高级交易方法，这个是bycoin插件的原生方法，调起 提交交易的页面，让用户输入密码；

5）交易确认后，调用 updateDatatbaseBalance 提交数据到后端；



