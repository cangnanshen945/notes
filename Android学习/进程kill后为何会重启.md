# AMS 处理进程死亡
## 如何判断进程死亡
- AMS与client之间通过binder通信，而binder拥有死亡通知的能力，即通信的一端结束，另一端能够收到通知
- 启动activity时，两次失败会判断为进程死亡
- 启动servicer时，失败判断为进程死亡
- unstableProviderDied会判断为进程死亡
## 判断重启
判断是否有活跃的provider，涉及到providerMap等，不太理解，但是从注释看，是有活跃的组件就会导致进程relaunch