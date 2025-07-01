---
title: 如何写一个自己的“半自动”交易程序
date: 2025-07-02 01:48:23
tags:
---

本来标题是很狂妄的，叫做"从0到1写量化"。但是考虑到本人时常偷懒的性格，因此改为`自动交易`。 但是为什么又加了个“半”呢？因为之前咨询券商，接口需要500W自有资金体量，另一方面，初生牛犊也不敢把垃圾代码直接丢上去跑。因此，目标越定越小，现在的目标是：
> 能跑起来就算成功。

目标虽小，但是也要有标准路径的去规划，所以先从deepseek开始，如何写量化？
> 一个完整的量化程序，核心是“数据-策略-回测-风控-执行-监控”全流程自动化闭环。

ok，那么我们分别看看这些都是什么东西。

###  数据获取与处理
- 行情数据获取：实时/历史K线数据、分笔数据、Tick数据、财务数据等。
- 数据清洗：去除异常数据、补齐缺失值、数据对齐等。
- 数据存储：本地数据库（如SQLite、MySQL）或内存存储。

###  策略开发与回测
- 策略编写接口：支持自定义编写策略，例如买入卖出逻辑、因子选股等。
- 回测系统：模拟历史行情下的策略表现，计算收益、夏普比率、最大回撤等指标。
- 多周期/多标的支持：可同时对多个股票、多个周期（如5分钟、日线）进行回测。

###  风控与资金管理
- 仓位管理：支持多标的多仓位、资金分配、最大持仓、杠杆等。
- 止盈止损：自动止损止盈、风控参数可配置。
- 风险监测：实时监控风险暴露、单日亏损限制等。

###  交易执行模块
- 模拟交易：虚拟下单、回测下单逻辑验证。
- 实盘交易接口：对接券商API（如华泰、银河等）实现实盘买卖。
- 订单管理：挂单、撤单、成交回报、异常处理。

###  监控与通知
- 实时监控：监控策略运行状态、资金变化、订单执行情况。
- 日志记录：详细记录策略执行流程、异常信息、交易明细。
- 消息推送：邮件、微信、短信等方式推送预警和交易结果。

###  可视化与报告
- 收益曲线、回撤曲线、持仓分布等图表展示。
- 策略绩效报告：自动生成策略表现分析报告。

###  扩展与自动化
- 定时任务调度：实现定时回测、定时实盘、自动更新数据等。
- 插件/模块化设计：策略、数据源、下单通道可灵活切换和扩展。



这里涵盖的很多部分对于我们入门级别来说是不需要的，列出来只是作为我们的roadmap。

对于我们来说，tick级别的数据做日内高频暂时也不需要，我们先拿日线级别数据跑起来。后续再进行优化。

首先，获取大A所有股票。
```typescript
async function initAllStockList() {
    const types = ["sha", "sza", "cyb", "kcb"];
    logger(`getAllStockList: ${types}`);
    for (let i = 0; i < types.length; i++) {
        const type = types[i];
        logger(`${type} start`);
        const data: any[] = [];
        // 计算总数
        const perPage = 200;
        const detactData = await getAllStockList(1, perPage, { type });
        const count = detactData.data.count;
        const batch = Math.ceil(count / perPage);
        data.push(...detactData.data.list);
        logger(`${type}: total ${count}, batch ${batch}`);
        for (let i = 1; i < batch; i++) {
            logger(`${type}: ${i + 1} start`);
            const acualData = await getAllStockList(i + 1, perPage, { type });
            data.push(...acualData.data.list);
        }
        logger(`data tracked done, start write db`);
        await Promise.all(
            data
                .map((d) => {
                    d.type = type;
                    return d;
                })
                .map(async (d) => {
                    await StockModel.findOneAndUpdate({ symbol: d.symbol }, d, { upsert: true });
                }),
        );
        logger(`${type} has finished`);
    }
}
```

其次再来一个获取所有股票日线数据，当然，这个函数需要满足每天都可以更新。

```js
async function initDailyLine() {
    const stocks = await StockModel.find({});
    for (let i = 0; i < stocks.length; i++) {
        const stock = stocks[i];
        const stockLogger = debug("koevas:db:initDailyLine:" + stock.name);
        stockLogger(`batch ${i + 1}/${stocks.length}`);
        let flag = true;
        let begin = +new Date();
        let count = 1;
        const latestDoc = await StockDailyLineModel.findOne({ stock: stock._id }).sort({ timestamp: -1 });
        let exists = +new Date();
        if (latestDoc) {
            exists = latestDoc.timestamp + 1;
        }
        while (flag) {
            stockLogger(`batch ${count} start`);
            const data = await getStockData(stock.symbol, PeriodEnum.day, begin);
            if (data.item.length >= 284) {
                begin = data.item[0][0] - 1;
                count++;
                if (latestDoc && begin < exists) {
                    flag = false;
                    data.item = data.item.filter((i: any) => i[0] >= exists);
                }
            } else {
                stockLogger(`last one!`);
                flag = false;
                latestDoc && (data.item = data.item.filter((i: any) => i[0] >= exists));
            }
            const transformed = await transformer(data.item);
            if (transformed.length) {
                await StockDailyLineModel.create(
                    transformed.map((t) => {
                        t.stock = stock._id;
                        return t;
                    }),
                );
                stockLogger(`insert ${transformed.length} done`);
            } else {
                flag = false;
                stockLogger(`no data for ${stock.symbol}, skip this one`);
            }
        }
        stockLogger(`done; \n`);
    }
}
```

函数中的获取行情数据，至于爬虫还是API，都是可以的。跑起来的样子如下

![周线数据](./week_run.png)

这里我额外添加了周线数据，因为我们肯定暂时不需要很精细的数据，日线+周线先跑起来。跑完后，日线数据大概是1000w+
![日线数据](./total_doc.png)

比我想象中的要少很多。到这里，历史数据就搞定了。接下来，就需要一个回测系统。

阅读下篇文章 -> [如何写一个自己的“半自动”交易程序（二）- 回测](../quantitative-trade-2)



