描述 kafka 中的 i/o model。

# 服务端的 i/o 模型

简单来说就是 一个监听套接字对应一个 acceptor，一个 acceptor 分发连接给多个 processor，每个 processor 对应一个 selector 负责读写套接字，<br>
每个 processor 对应多个真正处理逻辑的 handler thread，processor 通过一个 queue 分发 requests 给 handler thread。响应发送到 processor 对应的 response queue，<br>
processor 的 selector 负责把这些 response 发出去。

# 客户端的 i/o 模型

客户端的 i/o 模型简单多了，就一个 selector。


kafka 的 `NetworkClient` 的 i/o 模型是多路复用，多路指的是多个连接/套接字，复用指的是复用一个线程读写所有的套接字。

kafka 中一个连接可以同时发送多个请求，即 inflightRequests。那么当接收到 resp 的时候，我们怎么知道这个 resp 对应的是哪个 req？<br>
kafka 中每个 req 都有 coorelationID 字段，每当创建请求的时候，都会给 coorelationID 字段分配一个值，在响应该请求的时候也会带上该值，<br>
通过比较 req 与 resp 的 coorelationID 是不是一样，来判断响应是否符合预期。
