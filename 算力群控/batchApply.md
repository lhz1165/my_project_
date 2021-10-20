batchApply

步骤：校验参数，业务订单号是否存在，准备批量申请算力，异步批量申请算力





## 校验参数

bussOrderNo，notifyUrl，notifySignKey必填的业务订单号是否存在：做幂等，重复订单直接返回

## 准备批量申请算力

 检查电脑数量，通过传递进来的算力包id(包含配置信息),去查询该机房有多少机器，如果缺少是否满足最小数量否则返回。

保存算力群控订单（状态申请中）

保存算力群控订单明细记录(applyId由雪花算法生成，状态申请中)

## 批量申请算力

使用spring@Async，异步申请，

```
dispatchAdapter.apply(applyId, calcOrder);
```

根据applyIds 循环申请算力，(网络参数,查询默认串流参数)

为了防止申请到同一台客户机,使用客户机id和ip作为联合唯一键插入客户机使用表，结束订单后删除这条数据



# kafka监听开机事件

查询调度订单

关机成功就，结束订单，发送消息到kafka资产服务，并且 调度订单状态变更通知（关机）

开机成功，订单开始计费，发送消息到kafka资产服务，并且  调度订单状态变更通知(开机)

## 调度订单状态变更通知

DispatchOrderNotify

通知表入库

```
/**
 * 回调通知频率：0 -> 3 -> 9 -> 27 -> 81
 */
@Retryable(value = NotifyException.class, maxAttempts = 5,
    backoff = @Backoff(delay = 3000, multiplier = 3, maxDelay = 300000))
public void retryNotify(QueryDispatchOrderResp order, CalcOrder calcOrder, Integer notifyStatus) {
    // 查询通知表
    LambdaQueryWrapper<DispatchOrderNotify> queryWrapper = new LambdaQueryWrapper<>();
    queryWrapper.eq(DispatchOrderNotify::getApplyId, order.getApplyId());
    queryWrapper.eq(DispatchOrderNotify::getOrderStatus, notifyStatus);
    DispatchOrderNotify statusNotify = dispatchOrderNotifyMapper.selectOne(queryWrapper);

    // 封装回调通知参数
    Map<String, Object> params = buildNotifyParams(order, calcOrder, notifyStatus);
    // 第1次通知
    if (statusNotify == null) {
        // 通知表入库
        statusNotify = new DispatchOrderNotify();
        statusNotify.setApplyId(order.getApplyId());
        statusNotify.setOrderStatus(notifyStatus);
        statusNotify.setNotifyParams(JSON.toJSONString(params));
        statusNotify.setNotifyTime(LocalDateTime.now());
        statusNotify.setNotifyTimes(1);
        statusNotify.setResponseStatus(NotifyResponseStatusEnum.NO.getStatus());
        try {
            dispatchOrderNotifyMapper.insert(statusNotify);
        } catch (DuplicateKeyException e) {
            LoggerContext.getLogger().info("订单 {} 已触发过通知, params = {}", order.getApplyId(), params);
            return;
        }
        LoggerContext.getLogger().info("订单 {} 第 1 次通知, params = {}", order.getApplyId(), params);
    } else {
        // 已响应不再进行通知
        if (statusNotify.getResponseStatus().equals(NotifyResponseStatusEnum.YES.getStatus())) {
            LoggerContext.getLogger().info("订单 {} 回调已响应, 不再发送通知, params = {}", order.getApplyId(),
                     params);
            return;
        }
        // 只有前5次采用衰减重试
        if (statusNotify.getNotifyTimes() >= 5) {
            LoggerContext.getLogger().info("订单 {} 已触发过 {} 次通知, params = {}", order.getApplyId(),
                statusNotify.getNotifyTimes(), params);
            return;
        }
        // 第2、3、4、5次通知
        int times = statusNotify.getNotifyTimes() + 1;
        DispatchOrderNotify statusNotifyUpdate = new DispatchOrderNotify();
        statusNotifyUpdate.setId(statusNotify.getId());
        statusNotifyUpdate.setNotifyTime(LocalDateTime.now());
        statusNotifyUpdate.setNotifyTimes(times);
        dispatchOrderNotifyMapper.updateById(statusNotifyUpdate);
        LoggerContext.getLogger().info("订单 {} 第 {} 次通知, params = {}", order.getApplyId(), times, params);
    }

    // 签名，发送请求
    String response = doPost(order, calcOrder, params);
    if (!StringUtils.equalsIgnoreCase(response, RESPONSE_SUCCESS)) {
        throw new NotifyException("调用方未响应success, params = " + params);
    }

    // 通知成功响应
    DispatchOrderNotify notifySuccess = new DispatchOrderNotify();
    notifySuccess.setId(statusNotify.getId());
    notifySuccess.setResponseStatus(NotifyResponseStatusEnum.YES.getStatus());
    notifySuccess.setResponseTime(LocalDateTime.now());
    dispatchOrderNotifyMapper.updateById(notifySuccess);
}
```

定时任务回调通知

```
/**
 * 回调SDK通知补偿
 */
@Override
@LogContext(value = "dispatchOrderNotifyTask", group = LogGroup.TASK)
public ReturnT<String> execute(String param) {
    XxlJobLoggerWrapper.log("********************** 回调SDK通知补偿 start **********************");

    // 最近一次通知时间超过30分钟并且通知次数小于10次的未响应通知
    LambdaQueryWrapper<DispatchOrderNotify> queryWrapper = new LambdaQueryWrapper<>();
    queryWrapper.eq(DispatchOrderNotify::getResponseStatus, NotifyResponseStatusEnum.NO.getStatus());
    queryWrapper.lt(DispatchOrderNotify::getNotifyTimes, 10);
    queryWrapper.lt(DispatchOrderNotify::getNotifyTime, LocalDateTime.now().plusMinutes(-30));
    List<DispatchOrderNotify> statusNotifyList = dispatchOrderNotifyMapper.selectList(queryWrapper);
    if (CollectionUtils.isEmpty(statusNotifyList)) {
        XxlJobLoggerWrapper.log("********************** 暂无订单需要通知补偿 end **********************");
        return ReturnT.SUCCESS;
    }

    for (DispatchOrderNotify statusNotify : statusNotifyList) {
        XxlJobLoggerWrapper.log("需要回调SDK通知补偿, order = {}", JSON.toJSONString(statusNotify));
        try {
            // 查询调度订单
            QueryDispatchOrderResp order = dispatchAdapter.getOrder(statusNotify.getApplyId());
            if (order == null) {
                continue;
            }
            // 继续通知
            dispatchOrderStatusChangeNotifier.continueNotify(order, statusNotify);

            XxlJobLoggerWrapper.log("回调SDK通知补偿成功! applyId = {}", order.getApplyId());
        } catch (Exception e) {
            XxlJobLoggerWrapper.log("回调SDK通知补偿异常!", e);
        }
    }

    XxlJobLoggerWrapper.log("********************** 回调SDK通知补偿 end **********************");
    return ReturnT.SUCCESS;
}
```

如果申请数量大于可用机器数量，那么定时任务补偿申请剩下的机器，找出可用的机器继续上述异步申请流程

```
/**
 * 计算需要补偿申请的数量
 */
private long reApplyCount(CalcOrder calcOrder) {
    // 不需要进行补偿
    if (calcOrder.getReapplyTime() == null || calcOrder.getApplyDurationTime() == null
        || calcOrder.getApplyDurationTime() == 0) {
        return 0;
    }

    // 查询算力订单明细
    LambdaQueryWrapper<CalcOrderDetail> queryWrapper = new LambdaQueryWrapper<>();
    queryWrapper.eq(CalcOrderDetail::getBatchId, calcOrder.getBatchId());
    Long applyCount = calcOrderDetailMapper.selectCount(queryWrapper);
    // 申请数量 > 已申请数量，并且未到截止时间
    LocalDateTime applyFinishTime = calcOrder.getReapplyTime().plusMinutes(calcOrder.getApplyDurationTime());
    if (calcOrder.getCount() > applyCount && LocalDateTime.now().isBefore(applyFinishTime)) {
        return calcOrder.getCount() - applyCount;
    }
    return 0;
}
```

