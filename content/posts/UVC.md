---
title: "UVC使用"
date: 2026-03-18T16:20:42+08:00
draft: true
toc : true
tags: ["C语言", "编译器", "链接", "操作系统"]
categories: ["底层原理"]
summary: "本文深度拆解 C 语言运行时的‘动态秩序’，揭示栈帧回溯、堆分配算法与 sbrk 机制的物理本质，透视硬件 ABI 与启动代码如何共同构建出从局部变量到动态内存的逻辑分区。"
---

# UVC使用

```
esp_err_t uvc_start(void)
{
    rx_frames_queue = xQueueCreate(3, sizeof(uvc_host_frame_t *));
    assert(rx_frames_queue);

    // Install USB Host driver. Should only be called once in entire application
    WS_LOGI("Installing USB Host");
    const usb_host_config_t host_config = {
        .skip_phy_setup = false,
        .intr_flags = ESP_INTR_FLAG_LEVEL1,
    };
    ESP_ERROR_CHECK(usb_host_install(&host_config));

    // Create a task that will handle USB library events
    BaseType_t task_created = xTaskCreatePinnedToCore(usb_lib_task, "usb_lib", 4096, NULL, EXAMPLE_USB_HOST_PRIORITY, NULL, tskNO_AFFINITY);
    assert(task_created == pdTRUE);

    WS_LOGI("Installing UVC driver");
    const uvc_host_driver_config_t uvc_driver_config = {
        .driver_task_stack_size = 4 * 1024,
        .driver_task_priority = EXAMPLE_USB_HOST_PRIORITY + 1,
        .xCoreID = tskNO_AFFINITY,
        .create_background_task = true,
    };
    ESP_ERROR_CHECK(uvc_host_install(&uvc_driver_config));

    task_created = xTaskCreatePinnedToCore(frame_handling_task, "frame_hdl", 4096, (void *)&stream_config, EXAMPLE_USB_HOST_PRIORITY - 2, NULL, tskNO_AFFINITY);
    assert(task_created == pdTRUE);
    return ESP_OK;
}
```

## 创建队列

## 安装USB驱动











frame_callback（帧回调）

触发时机：每收到一帧视频数据时调用（高频）。
参数内容：带有帧指针 uvc_host_frame_t（frame->data/len 是 MJPEG/JPEG 原始字节）。
典型用途：把帧转交给应用层处理（排队、推流、显示、解码等）。
返回值语义：返回 false 表示“我先留着，稍后自己调用 uvc_host_frame_return 归还”；返回 true 表示“驱动现在就可以回收”。
注意事项：必须非常快，不要阻塞；若返回 false，就一定要在处理完后调用 uvc_host_frame_return，否则内存耗尽。
stream_callback（流事件回调）

触发时机：流状态变化或错误时调用（低频）。
参数内容：没有帧数据，只有事件类型和相关信息，例如：
UVC_HOST_TRANSFER_ERROR：传输错误
UVC_HOST_DEVICE_DISCONNECTED：设备拔出（带 stream 句柄）
UVC_HOST_FRAME_BUFFER_OVERFLOW/UNDERFLOW：帧缓冲溢出/欠载
典型用途：管理状态与资源（打印日志、关闭流、准备重连、调整参数如 frame_size/buffer 数量/URB 大小）。
注意事项：也要快，不要做耗时逻辑；真正的恢复/重试可以交给其他任务做。