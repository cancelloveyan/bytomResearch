参考资料：

 [比原链DAPP开发流程](https://github.com/oysheng/bufferserver/blob/master/docs/frame.md )

[储蓄分红合约Demo访问地址](http://app.bycoin.io:9000)

[储蓄分红合约后端bufferserver源码](https://github.com/oysheng/bufferserver)

[储蓄分红合约前端源代码](https://github.com/Bytom/Bytom-Dapp-Demo)

[储蓄分红合约详细说明](https://github.com/oysheng/bufferserver/blob/master/docs/instance.md)

[equity](https://github.com/Bytom/equity)

[智能合约学习文档](https://bytom.github.io/mydoc_smart_contract_overview.cn.html)

[插件钱包API](https://bytom.github.io/dapp-sdk-doc/#/common?id=bytomsend_advanced_transaction)





#### 简介

​	这章的内容详细分析一下涉及智能合约Dapp的整个开发流程，注意是涉及只能合约，如果你只要一些基本转BTM功能没有太大意义，本内容补充一下官方提供的 [比原链DAPP开发流程](https://github.com/oysheng/bufferserver/blob/master/docs/frame.md )，详细实践过好踩到的一些坑，还有一些真正具体的技巧还有经验，个人认为非常有用，起码让开发者可以更快速地去操作。

​	资料说的储蓄分红合约太复杂了，简单说说逻辑，银行发了一笔股份资产，用合约锁定，用户去触发这个合约的方法，付出了钱兑换了对应份额的股份资产，当达到一定的高度，就可以通过用股份资产兑换回本金与分红（钱+利息）。 里面包含了两个合约~~



整体流程

​	开发流程分为，1）编写智能合约；2）发合约交易；3）测试解锁合约方法；4）基于插件钱包开发Dapp前端；5）开发后端；

​	流程貌似非常简单，本人在1，2,3 步浪费了很多时间。其中有些坑踩过接下来介绍一下；

​	1）**编写智能合约**，上面提供的 [比原链DAPP开发流程](https://github.com/oysheng/bufferserver/blob/master/docs/frame.md )，写得很清楚，使用的是[equity](https://github.com/Bytom/equity)非常简单，直接下载最新版 用命令 【./equity TradeOffer --instance  】 就能得到一串编译后的合约程序代码，简称智能合约程序。

​       `E:\GoWorks\src\github.com\equity\equity>equity.exe jiedai_6.txt --instance ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff 260374 260474 260574 260674 260774 260874 260874 00141ccef16d2ac1ab22baa8acfa1633fdc32df
d55aa b1f38553d95177c53755996baf523da006da977008f069792bb6a2c3b6a253fb

======= PartLoanCollateral =======
Instantiated program:

20b1f38553d95177c53755996baf523da006da977008f069792bb6a2c3b6a253fb1600141ccef16d2ac1ab22baa8acfa1633fdc32dfd55aa030afb03030afb0303a6fa030342fa0303def903037af9030316f90320ffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
ffffff4d2b015b7a76529c641d010000640c0100005c7900a0695c790500f2052a01947600a0695379cd9f5579cda09a916164380000005a95639a00000054798ccd9f5679cda09a916164500000005895639a00000055798ccd9f5779cda09a916164680000005695639a00000056798ccd
9f5879cda09a916164800000005495639a00000057798ccd9f5979cda09a916164980000005295639a0000005195c3787ca169c3787c9f916164f5000000005e795479515e79c1695178c2516079c16952c3527994c251006079895f79895e79895d79895c79895b79895a79895979895879
895779895679890274787e008901c07ec1696307010000005e795479515e79c16951c3c2516079c169632b010000587acd9f6900c3c2515c7ac1632b010000755b7aaa5b7a8800c3c2515d7ac1747800c0`

2）**发合约交易**， 先解释一下合约的逻辑，储蓄分红合约太复杂，所以我们用**币币交易**合约去举例子，

```go
contract TradeOffer(assetRequested: Asset,

                    amountRequested: Amount,

                    seller: Program,

                    cancelKey: PublicKey) locks valueAmount of valueAsset {

  clause trade() {

    lock amountRequested of assetRequested with seller

    unlock valueAmount of valueAsset

  }

  clause cancel(sellerSig: Signature) {

    verify checkTxSig(cancelKey, sellerSig)

    unlock valueAmount of valueAsset

  }

}
```



看看智能合约的交易图，方便小白理解:

![img](https://github.com/cancelloveyan/bytomResearch-/blob/master/image/2_p1.jpg?raw=true)

所以储蓄分红合约一开始肯定要锁定一部分资产，所以必须部署合约交易。那么如何触发呢？

本人通过PC钱包的接口方式去部署合约，具体很多例子可以在[智能合约学习文档](https://bytom.github.io/mydoc_smart_contract_overview.cn.html)看到。

PC钱包方式，所有交易都必须三部，build-transaction，sign-transaction，submit-transaction，三个接口。



**踩过的坑**：   

1. 调试智能合约很慢，要等到交易确认才能知道是否成功，而且报错不明显，不知道哪里出问题；

     解决方案：  

​             本地PC钱包solonet模式调试，更改源码，快速出块  difficulty/difficulty.go

```go
func CheckProofOfWork(hash, seed *bc.Hash, bits uint64) bool { 
    compareHash := tensority.AIHash.Hash(hash, seed)   
    return HashToBig(compareHash).Cmp(CompactToBig(bits)) <= 0
}         
```

​          里面那句添加  ||true 如下

```go
return HashToBig(compareHash).Cmp(CompactToBig(bits)) <= 0 || true
```

   一开始没想到这样做，以为很快调试好，搞了三天晚上10点才调试完。

   2.智能合约对于**除法的支持**很不友好，尽量不要用除法，一开始写了一个很复杂的合约，不知道错误，智能逐步改代码快速调试去定位，最后发现 A/B，如果A=B没问题，否则就直接报错，问过官方没有得到合适的回答,我尝试过是存证这种问题，非常坑。

   3.程序必须计算好对应结果utxo 流转action的 input、ouput ；如下

```go
{

 "base_transaction": null,
 "actions": [
   {
     "output_id": "13fbd1e5df196a1488e85e3b5983e51444c49ef3695df789c9473abb636e0f5c",
     "arguments": [
       {
         "type": "integer",
         "raw_data": {
           "value": 5500000000
         }
       },   {
   			"type": "data",
   			"raw_data": {
   				"value": "00141ccef16d2ac1ab22baa8acfa1633fdc32dfd55aa"
   			}
   		},
           {
             "type": "integer",
             "raw_data": {
               "value": 0
             }
           }
         ],
         "type": "spend_account_unspent_output"
       },
       {
         "amount": 5500000000,
         "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
         "control_program": "0014d470cdd1970b58b32c52ecc9e71d795b02c79a65",
         "type": "control_program"
       },
       {
         "amount": 5000000000,
         "asset_id": "80013f81a66cb99977879e31639bb4fe4b12b4c7050fe518585d3f7f159d26a9",
         "control_program": "00141ccef16d2ac1ab22baa8acfa1633fdc32dfd55aa",
         "type": "control_program"
       },
       {
         "amount": 9999995000000000,
         "asset_id": "80013f81a66cb99977879e31639bb4fe4b12b4c7050fe518585d3f7f159d26a9",
         "control_program": "20b1f38553d95177c53755996baf523da006da977008f069792bb6a2c3b6a253fb160014d470cdd1970b58b32c52ecc9e71d795b02c79a6503e1830403e1830403e256040322350403a21e0403e20e040307fb0320ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff4d2b015b7a76529c641d010000640c0100005c7900a0695c790500f2052a01947600a0695379cd9f5579cda09a916164380000005a95639a00000054798ccd9f5679cda09a916164500000005895639a00000055798ccd9f5779cda09a916164680000005695639a00000056798ccd9f5879cda09a916164800000005495639a00000057798ccd9f5979cda09a916164980000005295639a0000005195c3787ca169c3787c9f916164f5000000005e795479515e79c1695178c2516079c16952c3527994c251006079895f79895e79895d79895c79895b79895a79895979895879895779895679890274787e008901c07ec1696307010000005e795479515e79c16951c3c2516079c169632b010000587acd9f6900c3c2515c7ac1632b010000755b7aaa5b7a8800c3c2515d7ac1747800c0",
         "type": "control_program"
       },
       {
         "account_id": "0U374V0300A02",
         "amount": 5500000000,
         "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
         "type": "spend_account"
       },
       {
         "account_id": "0U374V0300A02",
         "amount": 20000000,
         "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
         "type": "spend_account"
       }
     ],
     "ttl": 10000
   }

```



   一个解锁合约交易要包含action类型有，spend_account_unspent_output (合约的参数)，spend_account (输入的资产描述)，    control_program或者control_address （接收者资产描述），可以理解成**质量守恒**。

 如上面例子 

   spend_account_unspent_output 的action里面有个output_id =13fbd1e5df196a1488e85e3b5983e51444c49ef3695df789c9473abb636e0f5c，这个资产的小数位为8（这里没有体现），代表我要解锁这个utxo，他的值为  100000000.00000000 就是1亿。

拆分成两个action，一个 50.00000000，一个 99999950.00000000

只有btm = ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff 需要用来等手续费，所以允许不守恒，最后由旷工挖矿拿到手续费。



**总结**：那么程序相当于要把合约里面的逻辑整合进去，才能计算好真正的input、output~~我理解是交易确认的时候，解锁合约的程序验证现在的input、ouput是否跟合约一样。



**3）测试解锁合约方法**，2）里面采坑已经说清楚这个问题了，补充一下就是最好一下子不要写太复杂的合约，从简单来开发调试。一定要注意质量守恒定律，只要懂了这个原理其实非常简单。



**4）基于插件钱包开发Dapp前端**， 这块具体可以看[插件钱包API](https://bytom.github.io/dapp-sdk-doc/#/common?id=bytomsend_advanced_transaction)，[储蓄分红合约前端源代码](https://github.com/Bytom/Bytom-Dapp-Demo)，里面说的非常清楚， 涉及到的接口，暂时他们API文档还没有整理出来，来自上一章说的blockcenter的接口

url地址 ：testnet: 'http://app.bycoin.io:3020/', mainnet: 'https://api.bycoin.im:8000/'

核心用到的接口有：

### **根据合约与资产ID查询UTXO接口**

/api/v1/btm/q/list-utxos   

##### 参数：

```go
{    
    "filter": {                    "script":"20b1f38553d95177c53755996baf523da006da977008f069792bb6a2c3b6a253fb160014d470cdd1970b58b32c52ecc9e71d795b02c79a6503e1830403e1830403e256040322350403a21e0403e20e040307fb0320ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff4d2b015b7a76529c641d010000640c0100005c7900a0695c790500f2052a01947600a0695379cd9f5579cda09a916164380000005a95639a00000054798ccd9f5679cda09a916164500000005895639a0000798ccd9f5779cda09a916164680000005695639a00000056798ccd9f5879cda09a916164800000005495639a00000057798ccd9f5979cda09a916164980000005295639a0000005195c3787ca169c3787c9f916164f5000000005e795479515e79c1695178c2516079c16952c3527994c251006079895f79895e79895d79895c79895b79895a79895979895879895779895679890274787e008901c07ec1696307010000005e795479515e79c16951c3c2516079c169632b010000587acd9f6900c3c2515c7ac1632b010000755b7aaa5b7a8800c3c2515d7ac1747800c0", 
        "asset":"80013f81a66cb99977879e31639bb4fe4b12b4c7058585d3f7f159d26a9" ,
        "unconfirmed":false
    },    
    "sort":  {
        "by":"amount", 
        "order":"desc" 
        }
}
```

unconfirmed ，代表是否确认的，这个对后期的并发问题非常有用，第三章我会详细说明。

##### 结果

```go
{
    "code": 200,
    "msg": "",
    "result": {
        "_links": {},
        "data": [
            {
                "hash": "16749b694a9f1bc6a7759cf66baefed4c864b65985e7488e8721184ecc4d6965",
                "asset": "80013f81a66cb99977879e31639bb4fe4b12b4c7058585d3f7f159d26a9",
                "amount": 3000000000
            },
            {
                "hash": "e5f75036b6f662ff705378b55dd29dc1a43acb23d701dd44a068cdab2c43ad0c",
                "asset": "80013f81a66cb99977879e31639bb4fe4b12b4c7058585d3f7f159d26a9",
                "amount": 15000000000
            }
        ],
        "limit": 10,
        "start": 0
    }
}

```

（自己准备参数调用一下，以上是例子而已）



### 查询用户地址信息与余额接口

/api/v1/btm/account/list-addresses 

#### 参数

`{"guid":"b414005b-b501-4a0e-8b0f-e1cd762272f4"} `



#### 结果

```go
{

	"code": 200,

	"msg": "",

	"result": {

		"_links": {},

		"data": [{

			"guid": "b414005b-b501-4a0e-8b0f-e1cd762272f4",

			"address": "bm1qp4t6thlyktt6sh02scs8dqcpnk3ufk9e9pmq9s",

			"label": "",

			"balances": [{

				"asset": "80013f81a66cb99977879e31639bb4fe4b12b4c7050fe518585d3f7f159d26a9",

				"balance": "68900000000",

				"total_received": "69000000000",

				"total_sent": "100000000",

				"decimals": 8,

				"alias": "",

				"in_usd": "0.00",

				"in_cny": "0.00",

				"in_btc": "0.000000"

			}, {

				"asset": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",

				"balance": "1329551000",

				"total_received": "53790000000",

				"total_sent": "52460449000",

				"decimals": 8,

				"alias": "btm",

				"in_usd": "1.45",

				"in_cny": "10.10",

				"in_btc": "0.000142"

			}]

		}],

		"limit": 10,

		"start": 0

	}

}

```

**ps:**

​	guid是专门插件钱包提供的，是唯一的，这个非常有用，第三章我会详细说。





### 查询交易信息

/api/v1/btm/account/list-transactions 

#### 参数

 `{"address":"bm1qp4t6thlyktt6sh02scs8dqcpnk3ufk9e9pmq9s","start":0,"limit":100} `

#### 结果

```go
{
	"code": 200,
	"msg": "",
	"result": {
		"data": [{
			"ID": 111,
			"Hash": "471e5b267f646546be33505773186ee9d8dde2180a515df67a90d1a5f9d17bd2",
			"AssetID": "80013f81a66cb99977879e31639bb4fe4b12b4c7050fe518585d3f7f159d26a9",
			"Amount": 7000000000,
			"Address": "bm1qp4t6thlyktt6sh02scs8dqcpnk3ufk9e9pmq9s",
			"BaseID": 5,
			"Timestamp": "2019-07-08T09:23:12+08:00",
			"Height": 263728,
			"TransactionID": "471e5b267f646546be33505773186ee9d8dde2180a515df67a90d1a5f9d17bd2",
			"InputAmount": 5700000000
		}, {
			"ID": 64,
			"Hash": "e69631a8d6321d738793646399ffe022ac177a5732f562970e706ee76d49de82",
			"AssetID": "80013f81a66cb99977879e31639bb4fe4b12b4c7050fe518585d3f7f159d26a9",
			"Amount": 5000000000,
			"Address": "bm1qp4t6thlyktt6sh02scs8dqcpnk3ufk9e9pmq9s",
			"BaseID": 5,
			"Timestamp": "2019-07-05T16:37:07+08:00",
			"Height": 262170,
			"TransactionID": "e69631a8d6321d738793646399ffe022ac177a5732f562970e706ee76d49de82",
			"InputAmount": 5500000000
		}, {
			"ID": 56,
			"Hash": "cf74906808a1a6bc6a056c148510d542a10d2cbc350a4d830c670aa5ba973873",
			"AssetID": "80013f81a66cb99977879e31639bb4fe4b12b4c7050fe518585d3f7f159d26a9",
			"Amount": 39000000000,
			"Address": "bm1qp4t6thlyktt6sh02scs8dqcpnk3ufk9e9pmq9s",
			"BaseID": 5,
			"Timestamp": "2019-07-03T14:59:22+08:00",
			"Height": 261006,
			"TransactionID": "cf74906808a1a6bc6a056c148510d542a10d2cbc350a4d830c670aa5ba973873",
			"InputAmount": 8900000000
		}, {
			"ID": 54,
			"Hash": "6aedf609d47b3c06de2ce7dc9f2c99895124c80074573cd29407ac3b34ef8d40",
			"AssetID": "80013f81a66cb99977879e31639bb4fe4b12b4c7050fe518585d3f7f159d26a9",
			"Amount": 2000000000,
			"Address": "bm1qp4t6thlyktt6sh02scs8dqcpnk3ufk9e9pmq9s",
			"BaseID": 5,
			"Timestamp": "2019-07-03T12:11:12+08:00",
			"Height": 260936,
			"TransactionID": "6aedf609d47b3c06de2ce7dc9f2c99895124c80074573cd29407ac3b34ef8d40",
			"InputAmount": 5200000000
		}]
	}
}

```

5）开发后端，相当于bufferserver，第三章详细说明顺便我解析一下bufferserver的源码内容，还有里面踩过的坑。



总结：

​	这一章内容主要比较繁琐强调是调试合约方面，就是最核心的问题，这里抛出一个问题，就是UTXO问题，调试过程中非常繁琐，本来区块链不是做高并发，但是也存在并发问题，应该如何解决？  有使用过PC钱包的朋友肯定知道，里面PC钱包的UTXO，在交易过程中锁定了，没办法操作下一个，有些很多UTXO还好，如果只有一个，基本上调试跟实用都很麻烦~~~第三章我们基于原有**bufferserver**基础上根据官方的方案改一下，一定程度解决并发问题，大家期待一下。