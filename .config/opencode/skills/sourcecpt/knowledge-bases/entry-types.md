# 入口面类型清单（entry-types）

> Phase 0 引用此文件做 bash grep 识别 13 入口面通道。
> 约束力：**强制**。每条命中必引 grep 原始输出，禁凭记忆判入口（反幻觉#1）。
> 配套sink-types.md：sink 索引的 28 类清单。

---

## 13 入口面通道

| # | entry_type | 鉴别说明 | grep 模式（Java/Go/Python/JS） | 正例 | 反例 |
|---|-----------|---------|------------------------------|------|------|
| 1 | rest | REST/Web 框架入口 | Java: `@RequestMapping\|@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping\|@RestController` Go: `router.GET\|router.POST\|gin.HandlerFunc\|http.HandleFunc` Python: `app.route\|@app.route\|flask.Flask\|@api.route` JS: `app.get\|app.post\|express.Router\|router.get` | `@GetMapping("/api/users/{id}")` | 普通 `@Bean` 注解（不是入口） |
| 2 | rpc | RPC 服务入口 | Java: `@DubboService\|@Service.*impl\|@GrpcService` + 文件名 `*.proto` Go: `grpcRegisterService\|RegisterServiceServer` Python: `grpc.server\|add_.*Servicer_to_server` 跨语言: `service .* {` in *.proto | `@DubboService(version="1.0")` | `@Service`(Spring 普通 Bean 不是 RPC) |
| 3 | mq | 消息队列消费者入口 | Java: `@RabbitListener\|@KafkaListener\|@StreamListener\|@JmsListener` Go: `consumer.Subscribe\|Subscribe` in🐇 context Python: `consumer.subscribe\|on_message=\|@celery.task` JS: `channel.consume\|amqplib.*consume` | `@RabbitListener(queues="orders")` | 普通 subscribe 调用不是入口 |
| 4 | ws | WebSocket/SSE 入口 | Java: `@ServerEndpoint\|@MessageMapping\|WebSocketHandler\|TextMessage` Go: `websocket.Upgrader\|websocket.Handler` Python: `@socketio.on\|sio.on` JS: `socket.on\|socketio.on\|wsServer` | `@ServerEndpoint("/ws/chat")` | 普通 HTTP 不带 WS 升级 |
| 5 | graphql | GraphQL 入口 | Java: `@QueryMapping\|@MutationMapping\|@SchemaMapping\|graphql.schema.*DataFetcher` 跨语言: `type Query\|type Mutation` in *.graphqls | `@QueryMapping("users")` | `Query` entity JPA 名（不是 GraphQL） |
| 6 | cron | 定时任务入口 | Java: `@Scheduled\|@Cron\|SchedulingConfigurer\|@EnableScheduling` Go: `cron.New\|c.AddFunc\|time.Tick` Python: `@celery.task.*schedule\|apscheduler.schedulers.background\|@scheduler.scheduled_job` JS: `node-cron\|cron.schedule\|setInterval` | `@Scheduled(fixedRate=300000)` | 普通 java.util.Timer（不规范不识别） |
| 7 | cli | 命令行入口 | Java: `public static void main(\|@CommandLineRunner\|@SpringApplication\|picocli.Command` Go: `func main()\|cobra.Command` Python: `if __name__ == "__main__"\|argparse.ArgumentParser\|click.command` JS: `process.argv\|commander\|yargs\|process.mainModule` | `public static void main(String[] args)` | `main` 字段（非函数不是入口） |
| 8 | script | 脚本入口 | 文件名 `*.sh\|*.py\|*.tool\|*.bundle`；bash 含 `#!/bin/bash\|#!/usr/bin/env`；python 含 `if __name__` | `cleanup.sh: #!/bin/bash` | 普通 shell 函数文件无 shebang |
| 9 | deser | 反序列化入口 | Java: `@JsonView\|@XmlElement\|@JsonTypeInfo\|ObjectInputStream.*readObject\|XMLDecoder\|XStream.fromXML\|fastjson.*parse\|@JsonCreator` Python: `pickle.loads\|yaml.unsafe_load\|marshal.loads` JS: `JSON.parse\|eval`警示eval | `ObjectInputStream.readObject()` | 普通 toString 不是反序列化 |
| 10 | file | 文件类入口(用户控制文件/路径) | Java: `MultipartFile\|@RequestPart\|Part.*write\|Commons FileUpload\|@RequestParam.*MultipartFile` Go: `c.FormFile\|r.ParseMultipartForm` Python: `request.files\|werkzeug.secure_filename` JS: `multer\|formidable\|busboy` | `@PostMapping("/upload") @RequestParam MultipartFile file` | 内部代码读文件不是入口 |
| 11 | webservice | WebService/SOAP 入口 | Java: `@WebService\|@SOAPBinding\|@WebMethod\|@WebParam\|Endpoint.publish\|JAX-WS` + 文件名 `*.wsdl` Go: `soap.NewServer\|gosoap\|@webservice` Python: `spyne.*Application\|zeep` JS: `soap.listen\|strong-soap` | `@WebService(serviceName="UserService")` | 普通 RestController 不是 SOAP |
| 12 | custom_proto | 自定义协议入口 | Java: `ChannelInitializer\|Netty.*ChannelHandler\|Mina.*IoHandler\|Decoder\|Encoder\|ChannelPipeline\|@ChannelHandler` Go: `net.Listen\|bufio.Reader\|tcpMux\|custom.*encode\|decoder.*Decode` Python: `asyncio.*protocol\|socketserver\|protocols.*connection` JS: `net.createServer\|dgram.*Socket` | `public class MyDecoder extends ByteToMessageDecoder` | 普通 TCP 客户端不是入口 |
| 13 | event | 事件监听入口 | Java: `@EventListener\|ApplicationListener\|@TransactionEventListener\|@Subscribe\|@PostConstruct\|EventBus` Go: `chan\|select.*case.*<-.*channel\|errgroup\|signal.Notify` Python: `@blick\|sender.send\|@signal.EventHandler\|@dispatcher.connect` JS: `EventEmitter.*on\|emitter.on\|process.on` | `@EventListener public void onUserCreated(UserEvent e)` | 普通回调函数不是事件总线 |

---

## 引用方式

Phase 0 SKILL.md 必读此文件执行 Bash grep 识别入口面。

每条命中立即写知识图谱 entry 节点：

```json
{
  "node_type": "entry",
  "entry_type": "(取本表 entry_type 列)",
  "path": "(命中文件路径)",
  "line": "(命中行号)",
  "signature": "unknown",
  "param_tree": [],
  "module": "unknown",
  "authz_state": "unknown",
  "authz_ref": null,
  "audit_state": "[ ]"
}
```

签名/参数树/模块/authz_state 由 Phase 1a/1b 补齐, Phase 0 仅留 unknown 占位 (反幻觉#6: 不许残留占位符只能写成 "unknown", 不许"待定")。

经典非法模式: 见反例列, 防误识。

0 命中（某整通道）必须 QA 报警——可能漏面 (反幻觉#1)。

被反幻觉#1 引用 — 不准凭记忆出命令，必须按本文件 grep 模式执行 bash。