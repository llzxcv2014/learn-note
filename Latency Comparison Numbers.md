# 延时参考

|名称|时间(ns)|时间(微秒)|时间(毫秒)|备注|
---|---|---|---|---
L1 cache reference|0.5|
Branch mispredict|5|
L2 cache reference|7|||14x L1 cache
Mutex lock/unlock|25|
Main memory reference|100|||20x L2 cache, 200x L1 cache
Compress 1K bytes with Zippy|3000|3|
Send 1K bytes over 1 Gbps network|10000|10|
Read 4K randomly from SSD*|150000|150||~1GB/sec SSD
Read 1 MB sequentially from memory|250000|250|
Round trip within same datacenter|500000|500|
Read 1 MB sequentially from SSD*|1000000|1000|1|~1GB/sec SSD, 4X memory
Disk seek|10000000|10000|10|20x datacenter roundtrip
Read 1 MB sequentially from disk|20000000|20000|20|80x memory, 20X SSD
Send packet CA->Netherlands->CA|150000000|150000|150|


----
## Note
1 ns = 10^{-9} seconds   
1 us = 10^-6 seconds = 1,000 ns    
1 ms = 10^-3 seconds = 1,000 us = 1,000,000 ns

[Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832)    
[Teach Yourself Programming in Ten Years](http://norvig.com/21-days.html#answers)