#   ADB 常用命令
原文:

http://gityuan.com/2015/06/28/adb-notes/
```
一. 基本指令
adb -s serialNumber shell //进入指定设备
adb version //查看版本
adb logcat //查看日志
adb devices //查看设备
adb get-state //连接状态
adb start-server //启动ADB服务
adb kill-server //停止ADB服务
adb push local remote //电脑推送到手机
adb pull remote local //手机拉取到电脑
二. am 与pm
am start -n {packageName}/.{activityName} 启动app
am kill <packageName> 杀app的进程
am force-stop <packageName> 强制停止一切
am startservice 启动服务
am stopservice 停止服务
am start -a android.intent.action.VIEW -d http://www.12306.cn/ 打开12306网站
am start -a android.intent.action.CALL -d tel:10086 拨打10086
pm list packages 列出手机所有的包名
pm install/uninstall 安装/卸载
三. 模拟用户事件
文本输入: adb shell input text <string> 例手机端输出demo字符串，相应指令：adb shell input "demo".
键盘事件： input keyevent <KEYCODE>，其中KEYCODE见本文结尾的附表 例点击返回键，相应指令： input keyevent 4.
点击事件： input tap <x> <y> 例点击坐标（500，500），相应指令： input tap 500 500.
滑动事件： input swipe <x1> <y1> <x2> <y2> <time> 例从坐标(300，500)滑动到(100，500)，相应指令： input swipe 300 500 100 500. 例200ms时间从坐标(300，500)滑动到(100，500)，相应指令： input swipe 300 500 100 500 200.
四. logcat
logcat \| grep <str> 显示包含的logcat
logcat \| grep -i <str> 显示包含，并忽略大小写的logcat
logcat -s “ActivityManager” 显示该标签的log
logcat -d 读完所有log后返回，而不会一直等待
logcat -c 清空log并退出
logcat -t <count> 打印最近的count
logcat -v <format>， 格式化输出Log，其中format有如下可选值：
brief — 显示优先级/标记和原始进程的PID (默认格式)
process — 仅显示进程PID
tag — 仅显示优先级/标记
thread — 仅显示进程：线程和优先级/标记
raw — 显示原始的日志信息，没有其他的元数据字段
time — 显示日期，调用时间，优先级/标记，PID
long —显示所有的元数据字段并且用空行分隔消息内容
```