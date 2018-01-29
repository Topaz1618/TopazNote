# Publish\Subscribe(消息发布\订阅)： 
    1)生产者只能将信息发送到exchange
    2)exchange接收来自生产者的消息，并将它们推送到队列
    3)echange必须准确知道接收到的消息如何处理，其规则由exchange类型定义（应该附加到特定queue吗？应该附加到很多queue吗？或者应该丢弃）
    PS:为哈前半段没有exchange都能成功将消息发送到queue呢，因为exchange=''实际用了默认的exchange
# exchange类型
## 1.fanout(重点):广播收到的消息到所有queue，和默认的区别就是有名字广播
    1)首先创建随机queue，不给queue_declare提供参数就ok（连接rabbit时需要创建一个新的空队列）
    	result = channel.queue_declare()
    2)其次，一旦断开消费者，queue应该被删除：
    	result = channel.queue_declare(exclusive=True)	
    3)绑定exchange和queue，实现exchange发送消息到queue
    	channel.queue_bind(exchange='topaz',queue=result.method.queue)
    4)名为topaz的exchange开始增加消息到queue，fanout exchange 忽略提供 routing_key的值
### 实例：
    生产者：
    import pika
    import sys
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
    channel = connection.channel()
    channel.exchange_declare(exchange='logs',type='fanout')
    message = ' '.join(sys.argv[1:]) or "info: Hello World!"
    channel.basic_publish(exchange='logs',routing_key='',body=message)
    print(" [x] Sent %r" % message)
    connection.close()
    消费者：
    import pika
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
    channel = connection.channel()
    channel.exchange_declare(exchange='logs',type='fanout')
    result = channel.queue_declare(exclusive=True)
    queue_name = result.method.queue
    channel.queue_bind(exchange='logs',queue=queue_name)	#bind是关联exchange和queue的
    print(' [*] Waiting for logs. To exit press CTRL+C')
    def callback(ch, method, properties, body):
    	print(" [x] %r" % body)
    channel.basic_consume(callback,queue=queue_name,no_ack=True)
    channel.start_consuming()
		参考：https://www.rabbitmq.com/tutorials/tutorial-three-python.html
## 2.direct: 根据routingKey(关键字)和exchange判定发到哪个queue
    我们可以仅订阅一部分内容，将关键的错误引导到日志文件，例如：可以只将关键的错误消息引导到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有日志消息
    direct类型的exchange背后的交换路由算法很简单 ==> 把消息给绑定routing_key的queue
    绑定方法：
    	a.exchange绑定queue1 queue2， q1 routing_key为red，q2有两个绑定 routing_key分别为black和yellow #不符合key的数据都丢弃
    	b.exchange绑定queue1 queue2，q1 key为red，q2 key为red		#发送到所有queue key为black的
### 实例：
    生产者：
    import pika
    import sys
    print(len(sys.argv))
    
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
    channel = connection.channel()
    channel.exchange_declare(exchange='topaz',type='direct')
    severity = sys.argv[1] if len(sys.argv) > 2 else 'info'
    message = ' '.join(sys.argv[2:]) or 'Hello World!'
    channel.basic_publish(exchange='topaz',
    					routing_key=severity,
    					body=message)
    print(" [x] Sent %r:%r" % (severity, message))
    connection.close()
    消费者：
    import pika
    import sys
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
    channel = connection.channel()
    channel.exchange_declare(exchange='topaz',type='direct')
    result = channel.queue_declare(exclusive=True)
    queue_name = result.method.queue
    severities = sys.argv[1:]
    if not severities:
    	sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    	sys.exit(1)
    for severity in severities:
    	channel.queue_bind(exchange='topaz',
    					queue=queue_name,
    					routing_key=severity)
    print(' [*] Waiting for logs. To exit press CTRL+C')
    def callback(ch, method, properties, body):
    	print(" [x] %r:%r" % (method.routing_key, body))
    channel.basic_consume(callback,
    					queue=queue_name,
    					no_ack=True)
    channel.start_consuming()
    记录log打开命令行(只记录warning，error)
    	python receive_logs_direct.py warning error > logs_from_rabbit.log
    在screen上查看log（只看warning，error）
    	python receive_logs_direct.py info warning error
    参考：https://www.rabbitmq.com/tutorials/tutorial-four-python.html
## 3.topic:相比于direct灵活性更大，能基于多个标准进行路由选择（举俩例子：来自设备cron的严重错误，来自auth的所有日志），可以用表达式
    规则：
    1）topaic类型的exchange不能有任意routing_key  	
    2）必须是由点划分的字符列表
    3）可以是任意字符，但通常指定与消息相关的（例子：“stock.usd.nyse”，“nyse.vmw”，“quick.orange.rabbit”）
    4）绑定key还是用熟悉的姿势
    5）违反contract，发送一个或四个words的key（如“orange”或“quick.orange.male.rabbit”）消息将不会匹配任何绑定，并丢失
    	如果是"lazy.orange.male.rabbit"则匹配最后的绑定，发送到指定queue
    6）queue绑定"#"为RoutingKey，topic Exchange 接受所有消息，相当于fanout
    7）bind不使用"*"和"#"时，topic Exchange 表现得像 direct Exchange
    特殊字符：
    	topic exchange背后的逻辑和direct相似，都是发送包含指定key的消息到绑定的queue，但是，绑定key有两个特殊情况
    	* (star) 匹配一个字符
    	"#" (hash) 匹配0个或更多字符	(使用的时候不需要双引号，为了不注释)
### 实例：
    生产者：
    import pika
    import sys
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
    channel = connection.channel()
    channel.exchange_declare(exchange='topic_logs',type='topic')
    routing_key = sys.argv[1] if len(sys.argv) > 2 else 'anonymous.info'
    message = ' '.join(sys.argv[2:]) or 'Hello World!'
    channel.basic_publish(exchange='topic_logs',
    					routing_key=routing_key,
    					body=message)
    print(" [x] Sent %r:%r" % (routing_key, message))
    connection.close()
    消费者：
    import pika
    import sys
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
    channel = connection.channel()
    channel.exchange_declare(exchange='topic_logs',type='topic')
    result = channel.queue_declare(exclusive=True)
    queue_name = result.method.queue
    binding_keys = sys.argv[1:]
    if not binding_keys:
    	sys.stderr.write("Usage: %s [binding_key]...\n" % sys.argv[0])
    	sys.exit(1)
    for binding_key in binding_keys:
    	channel.queue_bind(exchange='topic_logs',
    					queue=queue_name,
    					routing_key=binding_key)
    print(' [*] Waiting for logs. To exit press CTRL+C')
    def callback(ch, method, properties, body):
    	print(" [x] %r:%r" % (method.routing_key, body))
    channel.basic_consume(callback,
    					queue=queue_name,
    					no_ack=True)
    channel.start_consuming()
    log：
    	To receive all the logs run >>> python receive_logs_topic.py "#"
    	To receive all logs from the facility "kern" >>> python receive_logs_topic.py "kern.*"
    	You can create multiple bindings >>> python receive_logs_topic.py "kern.*" "*.critical"
		参考：https://www.rabbitmq.com/tutorials/tutorial-five-python.html						
## exchange参数：
    exchange = '[name]'		#exchange的名称，为空就用默认exchange
## 操作：
    rabbitmqctl list_exchanges	#查看所有exchange，amq.*和default (unnamed) exchange 是自动创建的，没啥用
    rabbitmqctl list_bindings	#列出已存在的exchange和queue的绑定
## 记录日志：
    logs_from_rabbit.log	#命令行运行
    python receive_logs.py	#要是想在screen上看就执行这条
    python emit_log.py		#发射日志类型？？？
    源码：https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/receive_logs.py	
# Remote procedure call (RPC):远程
## 使用rabbitmq构造的rpc system：
    客户端和一个可以扩展的RPC服务器
    客户端发送rpc请求消息并阻塞，直到服务器回复响应消息。为了收到响应，客户端要设定reply_to = "随机queue名称"
## A note of rpc：当不知道函数调用是本地函数还是RPC时，会出现问题，滥用RPC可能导致代码不可维护，so：
    确保显而易见哪个函数调用是本地的，哪个是远程的
    记录系统，清除组件之间的依赖关系
    错误事件处理：RPC服务器长时间停机时，客户端应该如何反应
    尽量使用异步管道，而不是类似RPC的阻塞，结果被异步推送到下一个计算阶段
## message属性：AMQP 0-9-1 协议预先定义了14个消息属性，常用的只有以下几个
    delivery_mode：标记为持久消息（value为2）或transient（任意值）
    content_type：用于描述mime类型的编码 （例如：使用的JSON编码，将此属性设置为：application / json）
    reply_to：命名callback队列
    correlation_id：用于将RPC响应与请求相关联，为每个rpc请求建立queue很低效，可以为每个客户端建立queue
    这引发一个问题，在queue收到响应，会不清楚响应所属的请求，correlation_id可以解决这个问题，遇到未知的
    correlation_id值会丢弃消息
## 工作流程：
    当客户端启动时，它创建一个匿名callback队列
    客户端发送消息包含：reply_to(callback 队列)和correlation_id(每个请求唯一值)，请求被发送到rpc_queue队列
    server端等待queue上的请求，请求出现时就工作，并使用reply_to字段中的队列将结果发回给客户端
    客户端等待callback队列中的数据，收到消息检查correlation_id 值，与请求中的值correlation_id 相同，就返回对应用程序的响应
## 实例：斐波那契数列
    server端：
    #_*_coding:utf-8_*_
    # Author:Topaz
    import pika
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
    channel = connection.channel()
    channel.queue_declare(queue='rpc_queue')
    def fib(n):
    	'''生成斐波那契'''
    	if n == 0:
    		return 0
    	elif n == 1:
    		return 1
    	else:
    		return fib(n-1) + fib(n-2)
    def on_request(ch, method, props, body):
    	'''回调函数'''
    	n = int(body)
    	print(" [.] fib(%s)" % n)
    	response = fib(n)
    	ch.basic_publish(exchange='',
    					routing_key=props.reply_to,
    					properties=pika.BasicProperties(correlation_id = \
    														props.correlation_id),
    					body=str(response))
    	ch.basic_ack(delivery_tag = method.delivery_tag)
    channel.basic_qos(prefetch_count=1) #多个rpc服务负载均衡
    channel.basic_consume(on_request, queue='rpc_queue')
    print(" [x] Awaiting RPC requests")
    channel.start_consuming()
    client端：
    import pika
    import uuid
    class FibonacciRpcClient(object):
    	def __init__(self):
    		self.connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))  #创建连接
    		self.channel = self.connection.channel()    #创建通道
    		result = self.channel.queue_declare(exclusive=True)
    		self.callback_queue = result.method.queue   #声明一个回调对列用来回复
    		self.channel.basic_consume(self.on_response, no_ack=True,queue=self.callback_queue) # 订阅回调队列
    	def on_response(self, ch, method, props, body):
    		'''对每个响应执行'''
    		if self.corr_id == props.correlation_id:    #检查correlation_id
    			self.response = body    #是这样，它会将响应保存在self.response中并打破消费循环
    	def call(self, n):
    		'''执行实际的rpc请求'''
    		self.response = None
    		self.corr_id = str(uuid.uuid4())
    		self.channel.basic_publish(exchange='',     #发送包含correlation_id 和  reply_to的请求消息
    								routing_key='rpc_queue',
    								properties=pika.BasicProperties(
    										reply_to = self.callback_queue,
    										correlation_id = self.corr_id,
    										),
    								body=str(n))
    		while self.response is None:        #循环等数据
    			self.connection.process_data_events()
    		return int(self.response)
    fibonacci_rpc = FibonacciRpcClient()
    print(" [x] Requesting fib(30)")
    response = fibonacci_rpc.call(30)
    print(" [.] Got %r" % response)

	参考：https://www.rabbitmq.com/tutorials/tutorial-six-python.html	
