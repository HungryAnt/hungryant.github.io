---
layout: post
title:  "C#桌面程序单进程实例控制"
date:   2015-11-09 00:00:00
categories: csharp
---

## 说两句

最近开发私人游戏项目，需要一个客户端启动器，包含账号注册、登录、启动ruby游戏等功能

重拾c# WPF技术

这几天会写几篇博客，将一些技术细节记载下来，未来遇到相同问题，将解决问题的时间消耗控制在数分钟内

_本人重点关注如何将程序快速优雅实现出来，不空谈设计_

## 目标——程序不允许同时存在多个进程实例

当程序已启动，在此启动该程序时，提示“程序已在运行”，同时弹出运行进程的主窗口

##实现方案

在程序入口处，添加如下代码

若为WPF程序，则在App.xaml.cs中间中重载方法`OnStartup()`在其中添加对`CheckRunning()`的调用


- 检测是否存在相同程序进程

    `CheckRunning()`实现如下

        private static void CheckRunning()
        {
            bool createdNew;
            var mutex = new System.Threading.Mutex(true, "ant_yecaibuluo_startup", out createdNew);
            if (!createdNew)
            {
                MessageBoxUtil.ShowError("本程序已经在运行了，请不要重复运行！");
                ShowRunningAppWindow();
                Environment.Exit(1);
            }
        }

    这里使用互斥量控制单一进程，out参数将返回该互斥量是否为新创建，若非新创建则说明该程序进程已存在

- 展示已有进程主窗口

    获取已存在的进程并将其主窗口展示到前台

        private static void ShowRunningAppWindow()
        {
            Process currentProcess = Process.GetCurrentProcess();
            foreach (Process process in Process.GetProcessesByName(currentProcess.ProcessName))
            {
                if (process.Id != currentProcess.Id)
                {
                    NativeWindowApiHelper.ShowWindowAsync(process.MainWindowHandle, NativeWindowApiHelper.SW_RESTORE);
                    NativeWindowApiHelper.SetForegroundWindow(process.MainWindowHandle);
                    break;
                }
            }
        }

- Native API

    其中使用到Native API `ShowWindowAsync`与`SetForegroundWindow`

    `ShowWindowAsync`对窗体执行“恢复”操作，若窗口被隐藏或者被最小化，此方法可以将窗口恢复到面板中

    `SetForegroundWindow`顾名思义将创建置于最上层

    封装在NativeWindowApiHelper静态类型中，实现如下

        public static class NativeWindowApiHelper
        {
            public const int SW_SHOWNOMAL = 1;
            public const int SW_RESTORE = 9;

            [DllImport("User32.dll")]
            public static extern bool ShowWindowAsync(IntPtr hWnd, int cmdShow);

            [DllImport("User32.dll")]
            public static extern bool SetForegroundWindow(IntPtr hWnd);
        }
