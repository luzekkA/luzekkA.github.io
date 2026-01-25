+++
title = "终极FreeRTOS学习清单"
date = "2026-01-21T1:48:25+08:00"
toc = true
author = "LuzekkA"
tags = ["FreeRTOS", "并发"]
description = "一份FreeRTOS 终极能力清单"
+++

## 一、FreeRTOS 基础认知（必须 100% 勾完）

  ### 1. 内核模型与调度机制

  -  能清楚说出 FreeRTOS 是 **抢占式实时内核**
    **判定标准**：
    - 能解释什么是抢占
    - 能说明 `configUSE_PREEMPTION` 的作用
  -  能解释 **任务切换发生的三种条件**
    - Tick 中断
    - API 主动触发（如 `vTaskDelay`）
    - 高优先级任务就绪
      **判定标准**：
    - 不看资料，能口述完整流程
  -  能解释 **优先级调度 + 时间片轮转**
    **判定标准**：
    - 能解释“同优先级任务为什么都会运行”

------

  ### 2. Tick、中断与时间概念

  -  明确 Tick 的来源（SysTick / esp_timer / SYSTIMER）
    **判定标准**：
    - 能说清 ESP32 的 Tick 不是传统 SysTick
  -  能计算 Tick → 实际时间
    **判定标准**：
    - 知道 `configTICK_RATE_HZ`
    - 能算 `vTaskDelay(100)` 是多少 ms
  -  明确 Tick 中断 **不能做什么**
    **判定标准**：
    - 能说出为什么 ISR 不能阻塞

------

  ## 二、Task（任务）系统（核心能力）

  ### 3. Task 创建与生命周期

  -  能区分以下 API 使用场景
    - `xTaskCreate`
    - `xTaskCreatePinnedToCore`
    - `xTaskCreateStatic`
      **判定标准**：
    - 知道什么时候必须 static（如无 heap）
  -  能画出 **任务生命周期状态图**
    - Ready
    - Running
    - Blocked
    - Suspended
      **判定标准**：
    - 能解释每种状态如何进入/退出
  -  能解释任务函数 **为什么不能 return**
    **判定标准**：
    - 能说明返回后的栈与 TCB 问题

------

  ### 4. Task 栈（高频 bug 来源）

  -  明确任务栈存放内容
    - 局部变量
    - 返回地址
    - 保存的寄存器
      **判定标准**：
    - 能解释“栈溢出为什么是随机死机”
  -  会检测栈水位
    - `uxTaskGetStackHighWaterMark()`
      **判定标准**：
    - 实际打印过水位值
  -  配置并触发过栈溢出 hook
    **判定标准**：
    - `configCHECK_FOR_STACK_OVERFLOW != 0`
    - 实际进入过 `vApplicationStackOverflowHook`

------

  ## 三、任务间通信（工程重点）

  ### 5. Queue（最基础）

  -  明确 Queue 是 **拷贝数据**
    **判定标准**：
    - 知道不能直接传指针省事
  -  会区分
    - `xQueueSend`
    - `xQueueSendFromISR`
      **判定标准**：
    - 知道 FromISR 版本不能阻塞
  -  处理过 Queue 满 / 空 的阻塞逻辑
    **判定标准**：
    - 使用过 timeout ≠ 0 的 Queue API

------

  ### 6. Semaphore（同步）

  -  明确 Binary Semaphore ≠ Mutex
    **判定标准**：
    - 能说出优先级继承只存在于 Mutex
  -  正确使用 ISR → Task 信号
    **判定标准**：
    - 使用过 `xSemaphoreGiveFromISR`
  -  用 Semaphore 解决过资源访问竞争
    **判定标准**：
    - 真实工程中保护过 SPI / I2C / UART

------

  ### 7. Event Group（高级同步）

  -  能解释 EventGroup 的 bit 含义
    **判定标准**：
    - 能设计一个“多条件唤醒”模型
  -  使用过 `xEventGroupWaitBits`
    **判定标准**：
    - 能解释 clear-on-exit 行为

------

  ### 8. Task Notification（高性能必备）

  -  知道 Task Notification 是 **轻量级信号**
    **判定标准**：
    - 能说出它比 Queue 快的原因
  -  用 Notification 替换过 Queue / Semaphore
    **判定标准**：
    - 在代码中真实用过

------

  ## 四、ISR 与 RTOS 的边界（高级分水岭）

  ### 9. ISR 规则

  -  严格遵守 ISR 可用 API 清单
    **判定标准**：
    - 从未在 ISR 中调用阻塞 API
  -  正确处理 `BaseType_t xHigherPriorityTaskWoken`
    **判定标准**：
    - 能解释为什么要 `portYIELD_FROM_ISR`

------

  ## 五、内存管理（高级工程师必会）

  ### 10. Heap 管理模型

  -  知道 FreeRTOS 至少 5 种 heap 实现
    **判定标准**：
    - 能说出 heap_4 / heap_5 区别
  -  能解释碎片是如何产生的
    **判定标准**：
    - 知道反复创建/删除任务的风险
  -  在工程中刻意避免动态分配
    **判定标准**：
    - 使用 static Task / Queue

------

  ## 六、系统级 Debug 能力（你已经在做的事）

  ### 11. 死锁 / 卡死定位

  -  能区分
    - 死锁
    - 优先级反转
    - 饥饿
      **判定标准**：
    - 能从日志判断是哪一种
  -  使用过以下至少一种
    - 打印 task list
    - 打印 stack 水位
    - 打印 heap 剩余
      **判定标准**：
    - 有真实 debug 经验

------

  ### 12. 性能与实时性分析

  -  理解 CPU 占用 ≠ 实时性
    **判定标准**：
    - 能解释“低 CPU 占用仍然卡”的原因
  -  使用过运行时间统计
    **判定标准**：
    - 开启 `configGENERATE_RUN_TIME_STATS`

------

  ## 七、工程架构能力（高级工程师标志）

  ### 13. 任务划分能力

  -  不以“功能”为唯一划分标准
    **判定标准**：
    - 能按“数据流 / 事件流”拆任务
  -  明确哪些东西 **不该成为 Task**
    **判定标准**：
    - 能说出 ISR / Timer / Worker 模式

------

  ### 14. 可维护性设计

  -  能重构一个混乱 RTOS 项目
    **判定标准**：
    - 能画任务通信拓扑图
  -  能写博客解释 RTOS bug 根因
    **判定标准**：
    - 你已经做到（FreeRTOS 那篇）

------

  ## 八、终极自检（是否“高级”）

  如果以下 **80% 你能打勾**：

  -  我能不看资料写一个稳定 RTOS 架构
  -  我 debug RTOS bug 时是兴奋而不是恐慌
  -  我知道每一个 Task 为什么存在
  -  我能向别人解释 FreeRTOS 的“本质”

  👉 **你已经不是“会用 FreeRTOS”**
  👉 **你是“驾驭 FreeRTOS 的人”**

------

  ## 接下来我可以继续帮你做的三件事

  1. **把这份清单变成「30 天上班打勾计划」**
  2. **结合 ESP32，标注哪些你已经隐含掌握了**
  3. **下一步：RTOS × SoC × 总线 的进阶路线**

  你选一个，我继续。