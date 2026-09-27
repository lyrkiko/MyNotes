设计思路：
记录一个“卡内升舱”字段，其中包含经济舱第2、3运卡信息、高舱非限制类最低价运价数据，升舱tab的激活状态，点击跳转的来源（类似列表页点击学生票标签跳转的标识）

待确认：
1. 点击筛选项、不点击筛选项的第2、3运卡的选择
2. 如果左上角有标签如何展示
3. 运价选择按钮上的余票信息展示，点击 卡内tab 切换可能会展示不同的余票信息

方案：
	在 `requestCabinData` 中记一个标识位 flag，等都请求完接口，将其置为 `true`. 然后取 Economy 中第2、3个运价 和 高舱中的非限制类最低价做对比。


自测BUG：

1. 选取的运卡不是第2、3位，测试case 是2、4位
2. 点击卡内高舱tab，会联动。导致另一个展示卡内升舱的运卡也锚定到高舱tab

测试BUG：
1. 点击卡内高舱tab，点击另外的卡折扣无法更新页面所选取的折扣。
   根因：当卡片处于"高舱"态时，用户实际看到、要操作的对象是 `highGradeLowestFare`，但 `updateSelectedPaymentDiscount(index, ...)` 改的是 `dataSource[index]`——也就是宿主经济舱卡，跟 `highGradeLowestFare` 是两个不同的对象引用。

	1. 数据模型层面：`displayPolicy` 在高舱态下是 `highGradeLowestFare`，它不属于当前 tab 的 `dataSource` 数组，和宿主卡的 `index` 没有对应关系。
	2. 写回逻辑层面：`CreditPayment`/`CreditPaymentModal` 全程只传 `index`，从未把"当前展示的到底是宿主卡还是 `highGradeLowestFare`"这个信息带下去；`updateSelectedPaymentDiscount` 按 `index` 从 `dataSource` 里找卡，天然找不到 `highGradeLowestFare`，改的是宿主经济舱卡，导致高舱侧界面上看不到任何变化。
	3. 次要放大因素：`userClickPolicy` 用对象引用 + `isSelectedByBook` 做去重，没考虑 `index`，在 `highGradeLowestFare` 被多张卡共享引用的场景下会进一步错配 `index`。