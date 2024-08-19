# 电源管理

# autosleep

autosleep是Linux系统的一个功能，它可以让系统进入休眠状态，实现代码位于<kernel/power/autosleep.c\>。在编译内核时，必须开启CONFIG_PM_AUTOSLEEP选项才能使用该功能。  

/sys/power/autosleep可以用来配置autosleep，不同的状态有：

1. freeze：系统进入休眠状态，但不关闭电源，系统可以被唤醒。
2. standby：系统进入休眠状态，并关闭电源，系统可以被唤醒。
3. mem：系统进入休眠状态，并关闭内存，系统可以被唤醒。
4. disk：系统进入休眠状态，并关闭磁盘，系统可以被唤醒。
5. off：系统进入休眠状态，并关闭电源，内存和磁盘，系统不能被唤醒。




