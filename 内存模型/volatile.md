# volatile

## [volatile的官方专业解释][1]

 * Volatile Loads 等价 MonitorEnter
 * Volatile Stores 等价 MonitorExit
 
 
# Volatile Happen-Before
 对一个volatile变量的写操作happen-before对此变量的任意操作(当然也包括写操作了)
 
 * Thread A 写一个volatile变量v，Thread B 读写这个volatile v，则 Thread B 被保证可以看到 <u><font color="#FF0000">**Thread A 写v之前所有的改变**</font></u>，包括Thread A对no-volatile变量的改变。
 * Thread C 如果没有读写这个volatile v，则Thread C仍有可能看到Thread A操作过的no-volatile变量的过期数据，直到Thread C也对v进行操作。
  



[1]: http://gee.cs.oswego.edu/dl/jmm/cookbook.html