##简介

​	本章内容主要直接分析bufferserver源码，也就是比原链官方Dapp-demo的后端接口，里面包含了UTXO的托管逻辑、账单逻辑等，还会介绍一些改进的源码内容。



[储蓄分红合约后端bufferserver源码](https://github.com/oysheng/bufferserver)

本次源码分析主要根据bufferserver，2019年5月13号的版本，到此3个月没有更新了。

### 源码分析

​	我们来看看bufferserver的源码，项目是用golang语言开发的web服务端，内容比较简单也就几个接口。先看看源码的结构：

![img](https://raw.githubusercontent.com/cancelloveyan/bytomResearch/master/image/4_p1.png)

所有的golang项目首先都要看一下main.go，但是本项目有两个，因为一个是负责web的http接口的，另外一个是负责后端同步数据的。

先看看表结构，dump.sql

```go
##基础配置表
CREATE TABLE `bases` (
  `id` int(11) NOT NULL AUTO_INCREMENT,	
  `asset_id` char(64) NOT NULL,			##合约锁定的资产ID
  `control_program` text NOT NULL,		##合约的代码，在第二章里面提及通过equit工具生成
  PRIMARY KEY (`id`),
  KEY `asset_id` (`asset_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

#账单表
CREATE TABLE `balances` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `address` varchar(256) NOT NULL,			##地址
  `asset_id` char(64) NOT NULL,				##涉及的asset_id
  `amount` bigint(20) DEFAULT '0',			##交易的金额
  `tx_id` char(64) NOT NULL,				##交易ID
  `status_fail` tinyint(1) DEFAULT '0',		##状态
  `is_confirmed` tinyint(1) DEFAULT '0',	##交易是否确认
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,	##创建时间
  PRIMARY KEY (`id`),
  KEY `address` (`address`),
  KEY `asset_id` (`asset_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

##UTXO表，最重要最核心这个表
CREATE TABLE `utxos` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `hash` char(64) NOT NULL,						###UTXO的哈希，其实就是钱包里面UTXO的id
  `asset_id` char(64) NOT NULL,					##资产ID
  `amount` bigint(20) unsigned DEFAULT '0',		 ##UTXO	的额度	
  `control_program` text NOT NULL,				##该UTXO对应的锁定合约	
  `is_spend` tinyint(1) DEFAULT '0',			##是否已经使用
  `is_locked` tinyint(1) DEFAULT '0',			##是否锁定
  `submit_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,		##提交时间
  `duration` bigint(20) unsigned DEFAULT '0',		##锁定时间
  PRIMARY KEY (`id`),
  UNIQUE KEY `hash` (`hash`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

看到表结构，估计各位已经懂了核心逻辑，基本上就是同步UTXO过来，前端使用的时候锁定，然后如果使用了就更改状态，如果没有使用就解放放开锁，如图。

![img](https://github.com/cancelloveyan/bytomResearch-/blob/master/image/1_p3.png?raw=true)

我们来看看同步数据的main.go。

cmd/updater/mian.go

```go
func main() {
	cfg := config.NewConfig()     //####### 1.
	db, err := database.NewMySQLDB(cfg.MySQL, cfg.Updater.BlockCenter.MySQLConnCfg)
	if err != nil {
		log.WithField("err", err).Panic("initialize mysql db error")
	}

	go synchron.NewBlockCenterKeeper(cfg, db.Master()).Run()   //####### 2.
	go synchron.NewBrowserKeeper(cfg, db.Master()).Run()	   //####### 3.

	// keep the main func running in case of terminating goroutines
	var wg sync.WaitGroup
	wg.Add(1)
	wg.Wait()
}
```

1）启动的时候读取配置文件，config_local.json, 读取相关配置；

2）定时任务，定时在blockcenter里面同步对应的utxo数据，以及UTXO的状态；

3）定时任务，定时判断锁定的UTXO，超过时间恢复状态；



先来看看NewBlockCenterKeeper，blockcenter.go

```go

func (b *blockCenterKeeper) Run() {
	ticker := time.NewTicker(time.Duration(b.cfg.Updater.BlockCenter.SyncSeconds) * time.Second) //####### 1.
	for ; true; <-ticker.C {
		if err := b.syncBlockCenter(); err != nil { //####### 2.
			log.WithField("err", err).Errorf("fail on bytom blockcenter")
		}
	}
}
```

1）非常简单，初始化一个定时器，定时时间是b.cfg.Updater.BlockCenter.SyncSeconds = 60 秒；

2）定期调用syncBlockCenter()方法；



blockcenter.go 的syncBlockCenter 方法

```go
func (b *blockCenterKeeper) syncBlockCenter() error {
	var bases []*orm.Base
	if err := b.db.Find(&bases).Error; err != nil {
		return errors.Wrap(err, "query bases")
	}

	filter := make(map[string]interface{})
	for _, base := range bases {
		filter["asset"] = base.AssetID
		filter["script"] = base.ControlProgram
		filter["unconfirmed"] = true
		req := &common.Display{Filter: filter}
		resUTXOs, err := b.service.ListBlockCenterUTXOs(req)  //####### 1.
		if err != nil {
			return errors.Wrap(err, "list blockcenter utxos")
		}
		//####### 2.
		if err := b.updateOrSaveUTXO(base.AssetID, base.ControlProgram, resUTXOs); err != nil {
			return err
		}
		//####### 3.
		if err := b.updateUTXOStatus(base.AssetID, base.ControlProgram, resUTXOs); err != nil {
			return err
		}
	}

	if err := b.delIrrelevantUTXO(); err != nil {//####### 4.
		return err
	}

	return nil
}
```

1）调用blockcenter接口，查询UTXO列表；

2）updateOrSaveUTXO方法，插入或者更新UTXO锁定状态；‘

2）updateUTXOStatus方法，更新UTXO的使用状态；



调用**blockcenter接口**，非常简单，不过要注意这里程序里面unconfirmed = true方式去调用，

![img](https://raw.githubusercontent.com/cancelloveyan/bytomResearch/master/image/4_p2.png)

当unconfirmed=false的时候，返回的是所有已经确定交易的UTXO；

当unconfirmed=true的时候，返回的是包含已确认的、未确认的交易衍生出来的UTXO；

**PS：这里有个大坑，我搞了一笔肯定会失败的交易，衍生出来的UTXO，一样会返回过来，容易产生链式错误，所以我们应该尽可能保证我们的DAPP对应合约交易是一定会成功的，这个很容易，最怕恶意攻击，具体在第三章内容已经提及过了。**



blockcenter.go 的**updateOrSaveUTXO**

```GO
func (b *blockCenterKeeper) updateOrSaveUTXO(asset string, program string, bcUTXOs []*service.AttachUtxo) error {
	for _, butxo := range bcUTXOs {
		utxo := orm.Utxo{Hash: butxo.Hash}
        //####### 1.
		if err := b.db.Where(utxo).First(&utxo).Error; err != nil && err != gorm.ErrRecordNotFound {
			return errors.Wrap(err, "query utxo")
		} else if err == gorm.ErrRecordNotFound {
            //####### 2.
			utxo := &orm.Utxo{
				Hash:           butxo.Hash,
				AssetID:        butxo.Asset,
				Amount:         butxo.Amount,
				ControlProgram: program,
				IsSpend:        false,
				IsLocked:       false,
				Duration:       uint64(600),
			}
			if err := b.db.Save(utxo).Error; err != nil {
				return errors.Wrap(err, "save utxo")
			}
			continue
		}
        //####### 3.
		if time.Now().Unix()-utxo.SubmitTime.Unix() < int64(utxo.Duration) {
			continue
		}
        //####### 4.
		if err := b.db.Model(&orm.Utxo{}).Where(&orm.Utxo{Hash: butxo.Hash}).Where("is_locked = true").Update("is_locked", false).Error; err != nil {
			return errors.Wrap(err, "update utxo unlocked")
		}
	}
	return nil
}
```

1）通过utxo的hash，查询自己的数据库，如果查到就赋值给utxo；

2）如果查不到就会报错gorm.ErrRecordNotFound，就定义一个utxo，插入数据库表；

3）判断里面表里面锁定的时间是否超过了，因为有可能有些utxo数据被锁定了；

4）如果超过时间，该utxo还依然存在，那么代表UTXO没有被消耗掉，那么直接解锁；



blockcenter.go 的**updateUTXOStatus**

```go
func (b *blockCenterKeeper) updateUTXOStatus(asset string, program string, bcUTXOs []*service.AttachUtxo) error {
	utxoMap := make(map[string]bool)
	for _, butxo := range bcUTXOs {
		utxoMap[butxo.Hash] = true
	}

	var utxos []*orm.Utxo
    //####### 1.
	if err := b.db.Model(&orm.Utxo{}).Where(&orm.Utxo{AssetID: asset, ControlProgram: program}).Where("is_spend = false").Find(&utxos).Error; err != nil {
		return errors.Wrap(err, "list unspent utxos")
	}

	for _, u := range utxos {
		if _, ok := utxoMap[u.Hash]; ok {
			continue
		}
		//####### 2.
		if err := b.db.Model(&orm.Utxo{}).Where(&orm.Utxo{Hash: u.Hash}).Update("is_spend", true).Error; err != nil {
			return errors.Wrap(err, "update utxo spent")
		}
	}

	return nil
}
```

1)查询所有的未消耗的UTXO列表；’

2）循环数据库查出来未消耗的UTXO列表，如果不在blockcenter查询回来的UTXO列表里面，代表已经消耗掉了，更改状态is_spend = true，非常简单；

#### 总结：现在已经讲完了其中一个定时任务，比较简单，就是同步一下数据而已；

看看另外一个定时任务

browser.go

```go
func NewBrowserKeeper(cfg *config.Config, db *gorm.DB) *browserKeeper {
	service := service.NewService(cfg.Updater.Browser.URL)
	return &browserKeeper{
		cfg:     cfg,
		db:      db,
		service: service,
	}
}

func (b *browserKeeper) Run() {
	ticker := time.NewTicker(time.Duration(b.cfg.Updater.Browser.SyncSeconds) * time.Second)
	for ; true; <-ticker.C {
		if err := b.syncBrowser(); err != nil {
			log.WithField("err", err).Errorf("fail on bytom browser")
		}
	}
}

func (b *browserKeeper) syncBrowser() error {
	var balances []*orm.Balance
	if err := b.db.Model(&orm.Balance{}).Where("status_fail = false").Where("is_confirmed = false").Find(&balances).Error; err != nil {
		return errors.Wrap(err, "query balances")
	}

	expireTime := time.Duration(b.cfg.Updater.Browser.ExpirationHours) * time.Hour
	for _, balance := range balances {
		if balance.TxID == "" {
			if err := b.db.Delete(&orm.Balance{ID: balance.ID}).Error; err != nil {
				return errors.Wrap(err, "delete without TxID balance record")
			}
			continue
		}

		res, err := b.service.GetTransactionStatus(balance.TxID)
		if err != nil {
			log.WithField("err", err).Errorf("fail on query transaction [%s] from bytom browser", balance.TxID)
			continue
		}

		if res.Height == 0 {
			if time.Now().Unix()-balance.CreatedAt.Unix() > int64(expireTime) {
				if err := b.db.Delete(&orm.Balance{ID: balance.ID}).Error; err != nil {
					return errors.Wrap(err, "delete expiration balance record")
				}
			}
			continue
		}

		if err := b.db.Model(&orm.Balance{}).Where(&orm.Balance{ID: balance.ID}).Update("status_fail", res.StatusFail).Update("is_confirmed", true).Error; err != nil {
			return errors.Wrap(err, "update balance")
		}
	}
	
```

这里不直接深入讲解，因为经历上面的讲解，已经非常容易理解，就是同步交易的状态，更新本地的库，但是balance的数据是前端接口同步过来的，这样设计上就有问题，应该所有的交易同步都从后端自己去同步。

## **分享一下自己的源码：**

新建两个表

```go
#累积表，记录当前同步的高度
CREATE TABLE `stats` (
  `start_height` int(11) NOT NULL,		#开始统计的高度
  `current_height` int(11) NOT NULL,	#当前高度
  `base_id` int(11) NOT NULL
)
#交易表
CREATE TABLE `transactions` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `hash` char(64) NOT NULL,						#当前交易涉及的合约utxo的哈希
  `asset_id` char(64) NOT NULL,					#合约涉及的utxo的资产ID
  `amount` bigint(20) unsigned DEFAULT '0',			#涉及的资产数目
  `address` varchar(256) NOT NULL,						#获取的
  `timestamp` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,	#交易的时间戳
  `base_id` int(11) NOT NULL,						#关联累积表配置ID	
  `height` int(11) DEFAULT NULL,					#当前高度
  `transaction_id` varchar(255) NOT NULL,			#交易ID
  `input_amount` bigint(20) unsigned DEFAULT '0',		#用户支付出去的输入数目，我们这个dapp例子是以BTM为基准
  PRIMARY KEY (`id`),
  UNIQUE KEY `hash` (`hash`)
)
```



参考一下源码

```go
//爬取统计逻辑-------------------------------start
	//查询统计配置
	var stats []*orm.Stat
	if err := b.db.Find(&stats).Error; err != nil {
		return errors.Wrap(err, "query stats")
	}

	for _, stat := range stats {
		height := stat.StartHeight
		if(stat.CurrentHeight >= stat.StartHeight){
			height = stat.CurrentHeight
		}

		for i := 1; i < 100; i++ {
			height = height + 1
			//查找对应的合约&orm.Utxo{Hash: u.Hash}
			var bases []*orm.Base
			if err := b.db.Model(&orm.Base{}).Where(&orm.Base{ID: stat.BaseID}).Find(&bases).Error; err != nil {
				return errors.Wrap(err, "query bases")
			}

			res, err := b.service.GetBlock(height);
			if err != nil {
				log.WithField("err", err).Errorf("fail on query block [%s] from bytom browser", height)
				return nil
			}

			//只要请求有对象就更新stat
			if err := b.db.Model(&orm.Stat{}).Where(&orm.Stat{BaseID: stat.BaseID}).Update("current_height", height).Error; err != nil {
				return errors.Wrap(err, "update bases")
			}

			for _, tran := range res.Transactions {
				var cond1,cond2 bool
				var d2 service.Detail
				var spendDetail service.Detail
				for _, detail := range tran.Details {
					//来源有一个符合我们的合约还有资产id
					if detail.Type == "spend" && detail.AssetID == bases[0].AssetID && detail.ControlProgram == bases[0].ControlProgram {
						cond1 =true
					}
					//输出有一个非锁定的输出
					if detail.Type == "control" && detail.AssetID == bases[0].AssetID && detail.ControlProgram != bases[0].ControlProgram{
						cond2 =true
						d2 = detail
					}
					if detail.Type == "control" && detail.ControlProgram == "0014d470cdd1970b58b32c52ecc9e71d795b02c79a65" {
						spendDetail = detail
					}
				}
				if cond1 && cond2 {
					//保存saveTransaction
					if err!=nil {
						log.WithField("err", err).Errorf("fail on strconv.ParseUint([%s], 10, 64)", d2.Amount)
					}
					transaction := &orm.Transaction{
						Hash:           	tran.Id,
						AssetID:        	d2.AssetID,
						Amount:         	uint64(d2.Amount),
						Address:        	d2.Address,
						BaseID:	   			stat.BaseID,
						Timestamp:     		time.Unix(tran.Timestamp, 0),
						TransactionID:		d2.TransactionID,
						Height:          	height,
						InputAmount:		uint64(spendDetail.Amount),
					}

					if err := b.db.Save(transaction).Error; err != nil {
						return errors.Wrap(err, "save transaction")
					}

				}
			}
		}



	}
```

里面定时调用区块链浏览器的接口，不断获取最新的交易信息，解析里面的input与output的数据，然后再保存到库里面。例子中的合约是嵌套合约（第二章提过），就是只要总数够大就一定会衍生出新的UTXO，也就是重新被**同样的control_program**合约代码锁定的UTXO,  通过 这样的方式，监控所有最新交易，爬取入本地库，这样比较好的设计。

到此我们再看看另外一个api/main.go代码

```go
func main() {
	cfg := config.NewConfig()
	if cfg.GinGonic.IsReleaseMode {
		gin.SetMode(gin.ReleaseMode)
	}

	apiServer := api.NewServer(cfg)
	apiServer.Run()
}
///NewServer的代码
func setupRouter(apiServer *Server) {
	r := gin.Default()
	r.Use(apiServer.Middleware())
	r.HEAD("/dapp", handlerMiddleware(apiServer.Head))
	v1 := r.Group("/dapp")
	
	v1.POST("/list-utxos", handlerMiddleware(apiServer.ListUtxos))
	v1.POST("/list-balances", handlerMiddleware(apiServer.ListBalances))
	v1.POST("/update-base", handlerMiddleware(apiServer.UpdateBase))
	v1.POST("/update-utxo", handlerMiddleware(apiServer.UpdateUtxo))
	v1.POST("/update-balance", handlerMiddleware(apiServer.UpdateBalance))

	apiServer.engine = r
}
```

看到里面几个接口，各位都比较熟悉了

/list-utxos，查询可用的UTXO

/list-balances，查询交易信息

/update-base， 更新配置，这里基本上没有什么用，可以忽略。

/update-utxo，这里是锁定UTXO的接口，因为UTXO只能用一次，所以需要用的时候要锁定；

/update-balance，这里是添加一个账单信息，可以忽略，单纯是demo简单，但是落地不可能这样设计。



### 总结：

​	到此分析完bufferserver的源码，也非常简单，容易理解，里面涉及的内容单纯是调用http接口，存储或更新数据库而已，没有什么太复杂的逻辑，前提要对比原链比较熟悉，接下来我说说一些痛点经验与改进的方案。

1）开发过程中可以考虑用自己本地的网络，因为测试网络挖矿很慢，现在UTXO默认是锁定600秒=10分钟，很容易超过了10分钟交易都没有提交，数据就会大乱，链式错误，所以建议用本地网络更改一下秒出区块（第二章提及。）

2）随着dapp的业务扩展，有可能在这里基础上添加其他业务，这用用数据库锁的方式不大好，提倡用redis分布式锁，现在代码里面已经有接入redis的代码。最佳方案不是这样，是blockcenter可以通过utxo查询交易，包括已经提交却暂时没有确认的交易，这样可以实时监控到utxo最新的状态，期待blockcenter的发展。

3）本项目只有单机状态，如果分布式的话就有问题，定时任务要加分布式锁；



最后关于dapp-demo的内容分了四章已经全部讲完了，期待之后有更好的解决方案再继续更新。