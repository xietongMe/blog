## 链家爬虫踩坑记录
最近在开发一个爬取链家在售及已售二手房数据的爬虫，在这个过程中遇到了一些问题，将其记录下来，以供学习。
### 1. MySQL踩到的坑
在使用goroutine并发爬取数据并存储到数据库的时候，会经常报错"commands out of sync"导致爬取的数据并不能全部存储到数据库中，经过查询官方文档发现，
This can happen, for example, if you are using mysql_use_result() and try to execute a new query before you have called mysql_free_result(). It can also happen if you try to execute two queries that return data without calling mysql_use_result() or mysql_store_result() in between.<br>
官方文档说产生这个错误的原因可能有两个：
1. 是在将MYSQL_RESULT所指对象释放(使用mysql_free_result()）之前，执行了一个新的query；
2. 是执行了两条会返回结果的query，两者之间没有调用mysql_use_result() 或者 mysql_store_result() 来将结果集取出来。

根据我对项目的了解判断原因就是在使用goroutine写入数据的时候，写入速度太快，对于插入不需要返回值的数据，没有读完整个resault集并释放，就开始新的更新或插入数据操作，导致了这个错误。

而其中的原理就是由于Mysql客户端和服务端通信方式是采用“半双工”的通信方式，（半双工：指数据传输时可以实现双向通信，但是同一个时刻只能允许数据在一个方向进行传输，通信时一方必须发送或者接收完才能执行一下步的操作）客户端的每次查询操作，都必须使mysql_use_result() 或 mysql_store_result() 从服务器端取回结果并释放，才算是完成了一次完整的交互查询。解决办法就是在爬取数据后针对不同page适当降低数据插入的速度，比如我直接用time.Sleep(time.Duration(page) * time.Millisecond)来降低插入的速度。



### 2. Colly踩到的坑
在对链家网站数据进行爬取的时候一直会随机的报错"net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers) "
通过对代码核心逻辑的分析：向指定地址发送GET请求，并处理响应内容。判断应该是页面内容过多，在还没收到响应的页面内容或者还没完全收到页面内容时就已经超过了默认设置的时间，然后就发出了报错信息，因此我们在本地新增加一个获取资源的接口来验证我们的结论，通过控制接口的返回resource文件，来控制响应内容大小，然后发现当我们向resource文件写入大量内容后，确实会复现该错误，然后解决办法是通过SetRequestTimeout函数去覆盖了默认的10 seconds设置
