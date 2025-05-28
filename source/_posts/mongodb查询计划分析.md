---
title: mongodb查询计划分析
date: 2020-05-19 16:29:28
tags: mongodb
---

## 背景

从今年 1 月份线上 db 开始出现慢查询，14w 数据响应时间大概在 12s-14s 浮动，这种数据量确实不应该产生这样的问题，所以记录一下排查过程，防止以后踩坑。顺便可以更好的在以后的业务中建立有效的索引以及写更优雅的 sql。

出现问题的 sql 语句如下：

```js
db.getCollection("labs")
  .find(
    {
      User: ObjectId("59acf82b3f7ad7006d3bf336"),
      "OrganizationInfo.Organization": null,
      Deleted: false,
    },
    {
      Title: 1,
      Language: 1,
      UpdateDate: 1,
      ViewsCount: 1,
      Version: 1,
      Versions: 1,
      ShortDescription: 1,
      OrganizationInfo: 1,
      Columns: 1,
      IsCooperateLab: 1,
      User: 1,
      Private: 1,
      CooperateMembers: 1,
      AuthorizedMembers: 1,
      Parent: 1,
    }
  )
  .sort({ UpdateDate: -1 })
  .skip(13)
  .limit(13);
```

为了分析问题方便，我们简化为如下

```js
db.labs.find({
    "User": ObjectId("59acf82b3f7ad7006d3bf336"),
    "OrganizationInfo.Organization": null,
    "Deleted": false
},{
    "Title":1,
    ...
    "Parent":1,
}).sort({ UpdateDate: -1 }).skip(12).limit(12);
```

当时尝试了以下步骤

1. 只查 user 字段 -> 正常 -> 数据量 不到 100 条
2. 添加或删除 projection 的字段 即改变查询 -> 正常 -> 仅当前 sql 存在问题
3. 热备 Db -> 正常 -> 排除 sql 语句问题

同样的 sql 和数据在生产 Db 和热备 db 产生了不同的结果，至此猜猜是同类型的 sql 改变了缓存的查询计划-> 改变了执行索引 -> 影响了引擎执行的结果。

当天清除了该 SQL 的 plan cache， 短暂的恢复了正常，具体的清除 cache 文档如下
https://docs.mongodb.com/manual/reference/method/PlanCache.getPlansByQuery/index.html

但是截止本文撰写时，该 sql 语句仍然会发生慢查询。说明清除查询计划的缓存并没有解决问题，查询计划又被缓存了我们不期望的结果，因此花了点时间去研究了一下背后的机制。

因此，本文会分析的内容主要有

- Mongodb 如何解析查询
- 查询计划的生成以及执行
- 查询计划的缓存-> plan cache 运行机制
- 造成生产 db 慢查询的原因
- 如何 fix 此类问题

不会涉及的内容有

- 索引的匹配原理 -> 单字段完全匹配 && 联合索引最左匹配 && 交叉索引
- mongodb 查询计划 explain()的使用

## mongodb 是怎么解析查询的?

例如查询\_id = 1 的数据，我们都知道先去根据\_id 的值拿到数据的磁盘地址，然和去根据磁盘地址去拿数据。但是业务中 条件查询很多时候都不会只用\_id。

那么思考这个问题，如果一条 sql 查询中的多个字段都有索引，那么我们选择哪个索引去扫描数据呢？ 如何确定某个索引可以最快的拿到我们想要的结果？

### 如何解析查询语句

mongodb 会根据查询的谓词来进行解析 sql 语句，例如以下语句

```js
db.files.find({ name: "test", size: 1024 });
```

首先 mongodb engine 会将该 sql 解析成为标准 query -> `CanonicalQuery`

```js
{"name": {$eq: somevalue}, "size":{$eq: somevalue}}
```

它不关心具体的值，即上述 sql 等价于

```js
db.files.find({ name: "test123", size: 128 });
```

在 mongodb 中称为`queryShape`, 即他们有相同的“形状”。

![图片](./shape.png)

可以看到，`query`,`projection`,`sort`都会影响 shape 的构成。

也就是说不同的 queryShape 会生成不同的查询计划，即 projection 里面任意添加改变 key 都会造成查询计划的改变。这也就是为什么先前在 1 月份的给该 sql 的 projection 添加了\_id 字段，该 sql 恢复了正常。另外要注意 sort 也会影响 shape 的构成。

例如 sort -> {a: 1} && { b: 1} 是不同的 shape。 但是 sort -> {a: 1} && { a: -1} 是相同的 shape,改变的无非是索引扫描的方向(forward or backward)。

在 mongodb 中，真正会影响怎样执行查询的只有 query 和 sort，如果查询谓语中

- query 无索引 -> 则会进行全表扫描 -> collscan
- query 有单索引 -> ixscan -> 会生成查询计划 -> 如果 query 的多个字段都有索引，则会生成 multiPlans, 也就是根据索引的数量生成一个数组的查询计划，它的最大值为 internalQueryPlannerMaxIndexedSolutions = 64
- sort 的字段有索引，那么 sort 的字段也会参与查询计划，也就是影响索引的扫描方向。

### 如何生成查询计划

假如如上的语句，我们给 name 和 size 分别建立了索引

```js
indexes: {
  _id_, name_1, size_1;
}
```

在 mongodb 中每种索引都会生成一条查询计划，执行 explain 查看

```js
db.files.find({name:'test',size:1024}).explain()
{
        "queryPlanner" : {
                ...
                "winningPlan" : {
                        "stage" : "FETCH",
                        ...
                        "inputStage" : {
                                "stage" : "IXSCAN",
                                "indexName" : "size_1",
                        }
                },
                "rejectedPlans" : {
                        "stage" : "FETCH",
                        "inputStage" : {
                                "stage" : "IXSCAN",
                                "indexName" : "name_1",
                        }
                },
        }
```

可以看到每个查询计划都会生成 stage, 这些 stage 是 MongoDB 用来执行一条 sql 的最小单位。
比如这条 sql ->

```js
find({ a: 1 }).project("a b c").sort({ d: -1 }).skip(10).limit(10);
```

其实和我们文章开头的 sql 是一样的，简单的 stages 会有

1. ixscan
2. fetch
3. sort
4. skip
5. limit
6. projection

每个阶段会依次执行。但是有的 sql 会没有 sort 阶段，例如假设我们创建了联合索引 `a_1_d_-1`, 还需要排序吗？ 并不需要，我们只需要索引正序扫描，d 就是排好序的。

### 如何选择查询计划

上面可以看到根据各自的索引生成了 winningPlan 和 rejectPlan，即获胜计划就是我们认为的最优计划，那么 Mongodb 是如何确定哪个 plan 是最优解的呢？

#### 如何执行查询计划

plan 的初衷衡量每条计划索引的优异程度，也就是哪条索引能更快的返回数据。即它只是跑一部分数据，根据计算 score 来衡量 plan 的合适度。
那么查询计划需要考虑的问题就是

- 一共需要扫描多少条数据？
- 整个计划扫描到什么时候停止？
- 怎么计算单个 plan 的 score?

在 Mongodb 中，查询计划是并行去执行的，设计的思想就是优胜劣汰，扫描同样的次数，谁的分最高谁就是最优计划。

```c++
for (size_t ix = 0; ix < numWorks; ++ix) {
        bool moreToDo = workAllPlans(numResults, yieldPolicy);
        if (!moreToDo) {
            break;
        }
 }
```

即查询计划里面的所有 plan 一起执行，numWorks 即是需要执行的次数，numResults 即是达到我们需要的结果，那么就可以停了。详细的计算代码如下:

执行 numWorks 次

```c++
numWorks = std::max(
static_cast<size_t>(internalQueryPlanEvaluationWorks),
static_cast<size_t>(fraction _ collection->numRecords(txn)));
```

生产 Db 的版本为 3.4.6，在当前版本下

```js
internalQueryPlanEvaluationWorks = 10000;
fraction = 0.3;
```

即取(集合的数量 \* 0.3 )和 1W 的最大值， 也就是说，colletions 大于 1w 条数据那就最多执行集合数目的 30%，如果小于 1w 条那就扫描最多 1w 次。我们找个例子

![图片](./15d618f8-cc8e-44f8-9925-29e3654c1199.png)

这里是当时 dump 下来生产环境的 PlanCache，可以看到该计划扫描了 43200 次，那么可以得知当时 collections 的总数约为 43200/0.3 = 14W，与我们生产环境的数据相符。

> works 指的是扫描的次数，advanced 指的是符合当前阶段的数量,needTime 就是不匹配的数量。即 works = advanced + needTime;

获得的结果数满足 numResults || 达到 EOF 状态 => 则 stop

```c++
  numResults = std::min(
  static_cast<size_t>(_query.getQueryRequest().getLimit()),
  internalQueryPlanEvaluationMaxResults);
  internalQueryPlanEvaluationMaxResults = 101；
```

也就是说，如果有 Limit 的条件下，只要返回 limit+skip 和 101 的最小值就会停止。
这里要注意，这个值是最终需要结果的值，而不是某一阶段的 advanced。
假设出现这样一种情况，如果某个查询计划出现异常怎么办？

```c++
for (size_t ix = 0; ix < _candidates.size(); ++ix) {
CandidatePlan& candidate = _candidates[ix];
if (candidate.failed) {
continue;
}
...
```

也就是说，如果某个查询计划失败，那么它就不再在接下来的扫描中执行。不会影响其他的查询计划的执行。

这里还有一种情况会停止扫描 -> EOF 状态，也就是说扫描完了，没数据了。下面会详细讲。

#### 如何计算 plan 的分数

```c++
double score = baseScore + productivity + tieBreakers;
// baseScore = 1.0 基础分
double productivity =static*cast<double>(stats->common.advanced)/ static_cast<double>(workUnits);
// productivity = advanced/works 也就是有效查询率
double tieBreakers = noFetchBonus + noSortBonus + noIxisectBonus;
// 决胜者在于三个值,是否存在 fetch,sort,index intersection(交叉索引)
// 如果存在那么该值为 0 否则为默认值 也就是说最好情况
// tieBreakers = 3 * epsilon
// 他们的值都默认为常量 epsilon
const double epsilon = std::min(1.0 / static*cast<double>(10 * workUnits), 1e-4);
// 这里的常数 epsilon 是取一个很小的数 10 \_ works 和 万分之一的最小值
```

决胜者中的 bonus -> 这三个阶段分别代表的阶段为

- STAGE_PROJECTION || STAGE_FETCH // fetch 或者 projection
- STAGE_SORT // default sort 会将文档加载到内存中，和索引排序是有区别的
- STAGE_AND_HASH || STAGE_AND_SORTED // 交叉索引
  这三个阶段都是比较影响性能的，所以不存在就会被默认置为 0

计算分数的时候，当然还有一种情况就是达到 eof 状态

```c++
if (statTrees[i]->common.isEOF) {
LOG(5) << "Adding +" << eofBonus << " EOF bonus to score.";
score += 1;
}
```

10w 条数据的集合，如果按照设计我们需要扫描 33000 次，结果 name 为 test 的数据只有 10 条，其实我们只需要扫描 10 次拿到所有值，第 11 次就没有这条索引了，那么就会进入 eof 状态，不再继续执行了。为什么要这样设计？ 我们在选查询计划的时候，其实是在找最优解而不是真正的 query，所以扫描次数越少越好，尽快去执行计划。

#### 选择执行计划

分数已经计算出来了，其实就是以分数进行了排序，最高分就是 best plan;

```C++
vector<std::pair<double, size_t>> scoresAndCandidateindices;
// 这里对查询计划遍历 push 分数和 index 的 Pair
for(;;){
..
 scoresAndCandidateindices.push_back(std::make_pair(score, i));
..
}
```

// 进行排序

```C++
std::stable_sort(
scoresAndCandidateindices.begin(), scoresAndCandidateindices.end(), scoreComparator);
// 返回最优计划的 index
size_t bestChild = scoresAndCandidateindices[0].second;
return bestChild;
```

至此，我们通过扫描和计算得到了最优的查询计划。

#### cache 最优计划

以上的查询计划分析都是我们第一次进行查询的时候，得到最终的 best plan。那么接下来就是缓存这条 shape,为以后的 sql 直接选择这条 cache,再也不用执行第二次计算过程。

```c++
\_collection->infoCache()->getPlanCache()->add(\*\_query, solutions, ranking.release());
```

那么问题来了，缓存是计算机技术的伟大发明，如何保证缓存是有效的则是另一个需要考虑的问题，这里 mongodb 做了以下措施。(文档找不到了。。 凭记忆写吧

- 写操作达到 1000 次 ->重新计算 best plan
- collection 创建或移除索引
- 上文提到的 api -> clearQueryCache
- 终极大法 -> 重启 db

接下来我们就来看看第二次查询的时候 mongodb 是如何用 plan cache 的。

## Plan Cache 是怎样工作的？

在具有该 queryShape 的时候，会直接从 plan cache 中拿到该计划。
拿到之后直接执行吗？ 并不是。 首先它会计算一个扫描次数来衡量 best plan 是不是最优解。

```C++
size_t maxWorksBeforeReplan =
static_cast<size_t>(internalQueryCacheEvictionRatio \* \_decisionWorks);
// internalQueryCacheEvictionRatio = 10
// \_decisionWorks 是 best plan 的扫描次数
```

也就是说，放大该扫描计划的 10 倍，可以理解为置信区间为它的 10 倍。然后它会去扫描索引。如果在这个次数内

- 拿到了需要的结果的数量
- 达到了 EOF 状态

那么就去更新 cache 的 feedback，that's why we chosen it.认为这个 Plan 是最优的。

```c++
// We store up to a constant number of feedback entries.
if (entry->feedback.size() < size_t(internalQueryCacheFeedbacksStored)) {
entry->feedback.push_back(autoFeedback.release());
}
// internalQueryCacheFeedbacksStored = 20
```

什么时候会认为该计划有问题呢？

1. 如果超过置信区间 -> 说明第一次缓存样本差太大 -> replan
2. 或者执行该计划失败了 -> 计划有问题 -> replan

### Replan 机制

replan 其实就是重新生成查询计划，再计算每个计划的 plan score，然后选择 best plan 进行缓存。
如果 replan 的时候存在该 sql 的缓存，那就删掉再重新写入。

```c++
stdx::lock_guard<stdx::mutex> cacheLock(\_cacheMutex);
std::unique_ptr<PlanCacheEntry> evictedEntry = \_cache.add(computeKey(query), entry);
...
std::unique_ptr<V> add(const K& key, V* entry) {
// If the key already exists, delete it first.
KVMapConstIt i = \_kvMap.find(key);
if (i != \_kvMap.end()) {
KVListIt found = i->second;
delete found->second;
\_kvMap.erase(i);
\_kvList.erase(found);
\_currentSize--;
}
...
}
```

至此，mongodb 的查询计划流程已经讲完了，下面复现问题。

## 问题回顾复现

首先，回顾一下当时出问题的 sql 语句。

```js
db.labs.find({
"User": ObjectId("59acf82b3f7ad7006d3bf336"),
"OrganizationInfo.Organization": null,
"Deleted": false
},{
"Title":1,
...
"Parent":1,
}).sort({ UpdateDate: -1 }).skip(12).limit(12);
```

根据查看 db 索引，我们可以看到相关的索引有这些：

```js
"indexes" : {
    "User_1" : 1, // <100
    "OrganizationInfo.Organization_1_OrganizationInfo.OrganizationTag_1" : 1,
    "OrganizationInfo.Organization_1_UpdateDate_1" : 1,
    "User_1_Private_1_UpdateDate_1" : 1,
}
```

那么根据上文就会生成 4 条查询计划。这是 dump 的 plan cache json。

![图片](./37458616-c419-43c9-81b0-246c3410950f.png)

我们可以看到 score 最高的是索引 `OrganizationInfo.Organization_1_UpdateDate_1`，它的分数是 `1.000212962962963`。计算方法很简单，它一共扫描了 43200 次，而返回的结果只有 9 条，因为它没有 sort 阶段和交叉索引，因此

![图片](./50bcedc1-c546-41f2-873a-e5454aeb886c.png)

而我们期望可以执行的索引是 `User_1_Private_1_UpdateDate_1` 或者是 `User_1`，那么它的分数呢？

![图片](./82eb7ddf-6ba8-4aa7-840e-63e1ad2a883c.png)

计算分数 -> 1 + 0/32 + 0.0001 = 1.0001 没问题啊？

那是不是说用 Org.org 这个索引就最好呢？ 其实不是的，我们来计算一下：

![图片](./95691ba3-c78d-4da7-b107-f4faf4681888.png)

![图片](./4cf05c92-628d-43f5-8d46-2f6ddb8535f3.png)

一个需要扫描 11W 次，而 user 索引只需要扫描 32 次。

按照逻辑，我们肯定要走 user 的索引，因为只需要 32 条就可以拿到所有数据了。而且最主要的一点，在计算查询转化率的时候，也就是查询的数据/扫描的次数，在我们需要固定数量数据的时候，那么 works 越少肯定分数会越高。那为什么却去扫描更多的 org 索引呢？

其实仔细观察，会发现 user 的 advanced 是 0 -> 没有符合条件的数据。明明有数据怎么会是 0 呢？ 我们仔细看 `User_1_Private_1_UpdateDate_1` 这条索引的查询计划，它的 sort 阶段

![图片](./81cf3962-6412-4c22-9653-84c7e77c34a8.png)

在 `SORT_KEY_GENERATOR` 阶段都是有 26 条数据，为什么到了 SORT 阶段就为 0 了？ 看代码

```c++
const size_t maxBytes = static_cast<size_t>(internalQueryExecMaxBlockingSortBytes);
if (\_memUsage > maxBytes) {
mongoutils::str::stream ss;
ss << "Sort operation used more than the maximum " << maxBytes
<< " bytes of RAM. Add an index, or specify a smaller limit.";
Status status(ErrorCodes::OperationFailed, ss);
*out = WorkingSetCommon::allocateStatusMember(\_ws, status);
return PlanStage::FAILURE;
}
```

就是说内存使用超过了最大限制就会返回失败。到这里有两个问题

1.  maxBytes 即 internalQueryExecMaxBlockingSortBytes 是多少？
2.  \_memUsage 怎么来的？

翻代码看到

```c++
MONGO*EXPORT_SERVER_PARAMETER(internalQueryExecMaxBlockingSortBytes,
int, 32 \* 1024 \_ 1024);
// 32M
if (\_dataSet->size() < limit) {
member->makeObjOwnedIfNeeded();
\_dataSet->insert(item);
\_memUsage += member->getMemUsage();
return;
}
// 这里的 member 其实就是每个 doc
// getMemUsage 就是获得每个 doc 的 objSize
```

其实到这里问题就清楚了，获得的文档大于 32M 导致执行计划的失败，并且改变了查询计划。

那么为什么 org 索引没有触发文档 exceeded limit 这个错误-> 因为没有 sort 阶段 -> 为什么没有-> update 是它的联合索引,返回的数据天然有序->只需要执行 fetch 和 projection 操作即可

而且 sort 操作是内存排序 -> sort 操作应该尽量使用索引

> ps-> 在 MongoDB 4.2 版本将这个值上调到了 100M

### 回顾小结

原理和问题都清楚了，我们就来看下当时一系列操作背后的原理。

1. 清除了 cache 之后，一条正常的 sql 会正常查询(user 相关索引)，然后会将该查询计划缓存。
2. 当某个用户（总 doc 的大小大于 32M 时候）执行该查询计划的时候 -> 失败，那么就会触发 replan 机制，重新进行选举最优计划。而由于该 sql 中 user 相关的索引都需要 sort 阶段，因此 org 索引是唯一执行成功的， 所以在计算分数时它的 productivity 不为 0，另一方面它没有 sort 阶段，所以在计算 bonus 的时候是要比 user 索引多一个 epsilon。因此胜出。
3. 而接下来\_decisonworks 被设为了 4w， 在接下来的 cache_plan 进行执行评估的时候，需要最多 4w \* 10 次扫描才会进行 replan，而我们的总数据只有不到 20w，所以该计划在有问题的 user 上就会被认为是最优计划，导致了慢查询。
4. 因为 cache 的最优计划被改变，即使下一次正常的 user 进来，在一次 org 索引扫描之后就会达到 eof 状态，就会进行 feedback 操作，正常缓存了此查询计划。
5. 也就是说正常状态，使用 user 索引并没有慢查询，只有文档过大的用户查询会走 Org 索引并且导致慢查询。如此反复。

## 解决方案

问题清楚了其实解决方案很简单，即

- 添加索引即可解决 `user_1_update_1`
- 或者改掉 `user_1_private_1_update_1` 为 `user_1_update_1_private_1`

在本问题中避免使用 sort 阶段即可->那么该查询计划就不会存在失败。
在生产 db 做这些分析操作本来是一件不被允许的事情，因此建立可靠的监控和日志系统是接下来要推进的事情。

关于慢查询的日志监控有很多文章，操作也比较简单，这里就不再赘述。

在以后的业务开发中，可以优化的地方

- sort 操作尽量使用索引，避免内存排序。
- sql 中的多条件查询使用“最小匹配”原则，使匹配的文档最少
- 同名索引是一种浪费,例如 user_1 & user_1_private_1 ，只要查 user = 'some' 使用 user_1_private_1 是同样的效果。
- 在文档大小可控的情况，返回全文档可能更有效率。例如 find({user:'jack',private:true})在有索引 user_1_private_1 的情况下不用添加 Projection。
- 在某些可预计的多索引情况下，当我们知道哪条索引的查询计划会是比较稳定有效的时候，可以使用 hint()方法强制使用某个索引。
  - 例如 find({a:‘somevalue’,b:‘somevalue’,c:‘somevalue’)，假设 a,b,c 都有索引且我们知道 c = ‘somevalue’的数量是最少的，那么可以直接使用 find({a:‘somevalue’,b:‘somevalue’,c:‘somevalue’})**.hint('c_1') **
  - hint 会指定 mongodb 使用指定的索引进行查询
   