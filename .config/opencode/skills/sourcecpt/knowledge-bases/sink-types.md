# Sink 类型清单（sink-types）

> Phase 0 引用此文件做 Grep 工具/系统 shell grep 识别 28 类敏感操作 sink。
> 约束力：**强制**。每条命中必引 grep 原始输出，禁凭记忆判 sink（反幻觉#1）。
> 与entry-types.md 配套: 入口面 + sink 索引决定 Phase 2a 污点追踪的 source-sink 对。
> 组织: 按 OWASP Top10 双轨 × CWE 编号, 每 sink 必挂 1 OWASP + 1 CWE。
> **跨平台**：本文件所有 grep 模式用双兼容正则子集（SRC_ACCESS §1.3），在 bash `grep -E` 与 PowerShell `Select-String` 下通用。

---

## 28 sink 类（按 OWASP × CWE 双轨）

| sink_type | owasp | cwe | grep 模式（Java/Go/Python/JS） | Source 示例 | Sink 示例 |
|-----------|-------|-----|------------------------------|-----------|---------|
| path_traversal | A01 | CWE-22 | Java: `new File(\|Files.read\|Files.newInputStream\|FileInputStream\|Files.probeContentType\|RandomAccessFile\|Paths.get(\|FileUtils.*File` Go: `os.Open\|ioutil.ReadFile\|os.OpenFile\|filepath.Join\|bufio.NewReader` Python: `open(\|os.path.join\|Path(.*:\|fileinput.input` JS: `fs.readFile\|fs.writeFileSync\|fs.open\|path.join` | @RequestParam fileName | `new FileInputStream("/data/"+fileName)` |
| reflection_load | A01 | CWE-470 | Java: `Class.forName(\|ClassLoader.loadClass\|URLClassLoader\|Reflection.*invoke\|MethodHandle.invoke` Go: `reflect.PtrTo\|reflect.TypeOf\|Value.Call` Python: `importlib.import_module\|__import__\|getattr(\|setattr(` JS: `eval(\|Function(\|require(` | @RequestParam className | `Class.forName(className).newInstance()` |
| weak_hash | A02 | CWE-328 | Java: `MessageDigest.getInstance("MD5"\|MessageDigest.getInstance("SHA-1"\|MessageDigest.getInstance("SHA1"\|DigestUtils.md5\|md5Hex\|sha1Hex` Go: `crypto/md5\|crypto/sha1\|md5.Sum\|sha1.Sum` Python: `hashlib.md5(\|hashlib.sha1(` JS: `crypto.createHash("md5"\|crypto.createHash("sha1"` | (无 source, 检测存在) | `MessageDigest.getInstance("MD5")` |
| weak_crypto | A02 | CWE-327 | Java: `Cipher.getInstance("DES"\|Cipher.getInstance("RC4"\|Cipher.getInstance("3DES"\|Cipher.getInstance("AES/ECB"` Go: `crypto/des\|crypto/rc4\|crypto.NewCBCEncrypter` Python: `Crypto.Cipher.DES\|Crypto.Cipher.ARC4\|Crypto.Cipher.DES3` JS: `crypto.createCipher("des\|crypto.createCipheriv("aes-128-ecb"` | (检测存在) | `Cipher.getInstance("DES")` |
| hardcoded_cred | A02 | CWE-798 | Java/Go/Python/JS: `"password"\s*=\s*['"][^'"]{4,}\|pwd\s*=\s*['"]\|password\s*=\s*['"]\|secret\s*=\s*['"]\|api[Kk]ey\s*=\s*['"]\|token\s*=\s*['"][^'"]{8,}\|connection.*password.*=\|jdbc.*password.*=\|private.*key.*=.*['"]` | (检测存在) | `private String dbPassword = "admin123";` |
| weak_random | A02 | CWE-330 | Java: `new Random(\|Math.random(\|ThreadLocalRandom` Go: `math/rand\|rand.Int\|rand.Intn` Python: `random.Random\|random.randint(` JS: `Math.random(` | (检测存在) | `new Random().nextInt(1000);` |
| sql_splice | A03 | CWE-89 | Java: `prepareStatement(".*" *+\|prepareStatement(".*concat\|createNativeQuery(\|EntityManager.*createNative\|String.format("SELECT` Go: `db.Exec(.*)fmt.Sprintf\|db.QueryContext.*+.*sql` Python: `cursor.execute(.*)%.*\|cursor.execute(.*)format\|cursor.execute(.*)+` JS: `connection.query(.*)+.*inject` | @RequestParam `name\|id\|query` | `prepareStatement("SELECT * FROM users WHERE id=" + userId)` |
| cmd_exec | A03 | CWE-78 | Java: `Runtime.getRuntime().exec\|ProcessBuilder\|Process.*start(` Go: `exec.Command\|exec.CommandContext\|os/exec.Command` Python: `os.system(\|subprocess.run\|subprocess.Popen\|os.popen\|commands.*\|shlex.split` JS: `child_process.exec\|child_process.spawn\|exec(\|execSync(` | @RequestParam `cmd\|command\|shell` | `Runtime.getRuntime().exec(cmd);` |
| ldap_injection | A03 | CWE-90 | Java: `LdapTemplate\|DirContext.search\|Context.search\|InitialDirContext.search` Go: `go-ldap.*Filter\|ldap.NewFilter\|SearchRequest` Python: `ldap3.*search\|filter=.*format\|filter=.*+` JS: `ldapjs.*search\|filter=.*\|searchRequest` | @RequestParam uid | `DirContext.search("uid="+uid, ...)` |
| nosql_injection | A03 | CWE-943 | Java: `MongoCollection.find(\|BsonDocument.*parse\|BasicDBObject.*put\|Filters.*text\|MongoTemplate.find` Go: `bson.M{.*:\|collection.Find.*ctx\|bson.D{{` Python: `MongoClient.*find\|collection.find\|db.command(` JS: `collection.find(\|db.collection(\|\$where\|eval(` | @RequestParam `name\|keyword\|query` | `collection.find({name: req.query.name})` |
| xpath_injection | A03 | CWE-643 | Java: `XPath.evaluate\|xpath.compile\|XPathExpression.evaluate\|XPathFactory` Go: `xpath.*Compile\|xmlquery.*Parse\|XPath.` Python: `lxml.etree.XPath\|etree.XPath(\|XPath.*evaluate` JS: `xpath.*select\|evaluateXpath\|xpath.*parse\|DOMParser.*parseString` | @RequestParam `expr\|filter` | `xpath.evaluate("//user[name='"+name+"']")` |
| expression_injection | A03 | CWE-917 | Java: `SpelExpressionParser\|Ognl.getValue\|OgnlUtil.getValue\|Expression.getValue\|MVEL.eval\|AviatorEvaluator.execute\|JexlEngine.*create` | @RequestParam `formula\|expr\|expression` | `Ognl.getValue(userInput, context);` |
| ssti | A03 | CWE-1336 | Java: `freemarker.template.*process\|Template.process\|velocity.*VelocityContext\|VelocityEngine.evaluate\|Thymeleaf.*processTemplate\|StringTemplate` Go: `text/template.*Execute\|html/template.*Execute` Python: `render_template_string\|Template(\|render_template(\|jinja2.*Template\|TemplateResponse` JS: `ejs.render\|handlebars.compile\|mustache.render\|pug.render` | @RequestParam `name\|template\|html` | `render_template_string("<h1>"+name+"</h1>")` |
| xslt | A03 | CWE-91 | Java: `TransformerFactory.newTransformer\|XSLTC.*compile\|Transformer.transform\|StreamSource` | @RequestParam `xsl\|xml` | `transformerFactory.newTransformer(new StreamSource(userXsl));` |
| crlf_header | A03 | CWE-93 | Java: `response.setHeader(\|response.addHeader(\| HttpServletResponse.*setHeader\|sendRedirect(` Python: `Response.*headers\|response["*\|set_header(` JS: `res.setHeader(\|res.set(\|response.setHeader(` | @RequestParam `redirect\|header\|target` | `response.setHeader("Location", "/redirect?to=" + url);` |
| mail_header | A03 | CWE-93 | Java: `MimeMessage\|InternetAddress\|setHeader(\|Transport.send\|setFrom(\|setRecipient(` Python: `smtplib.*sendmail\|email.message.*EmailMessage\|sendmail.*message` JS: `nodemailer.*sendMail\|mailcomposer` | @RequestParam `to\|cc\|bcc\|subject` | `message.setHeader("To", userToInput);` |
| log4shell | A03 | CWE-502 | Java: `logger.info(\|logger.error(\|logger.warn(\|logger.debug(\|slf4j.*log\|LogFactory.*log` | @RequestParam 任意字段（jndi 字符串） | `logger.info("User-Agent: " + userAgent);` |
| debug_endpoint | A05 | CWE-489 | Java: `management.endpoints.web\|management.endpoint\|@ActuatorEndpoint\|@RestController.*actuator\|@Endpoint.*@ReadOperation` + 配置文件 `management.endpoints.web.exposure.include=*` Python: `django.*DEBUG\|settings.*DEBUG = True\|app.debug = True` JS: `app.use(express.static\|NODE_ENV.*development` | (检测存在) | `management.endpoints.web.exposure.include=*` |
| info_leak | A05 | CWE-209 | Java: `e.printStackTrace(\|response.getWriter().write(e.toString\|exception.printStackTrace\|stream.*print.*exception\|Throwable.*getMessage.*response` Python: `traceback.format_exc\|raise.*response\|except.*Exception.*as e.*return str(e)` JS: `console.log(e\|err.stack\|err.message.*response\|res.send(err.stack)` | (检测存在) | `e.printStackTrace();` |
| sbom_cve | A06 | CWE-1035 | pom.xml / build.gradle / package.json 中 version 范围命中已知 CVE 库（Phase 2 SBOM 子模块审，Phase 0 此处仅识别 manifest 同名类依赖）| (Phase 0 不执行, 二阶段审) | （略） |
| missing_authz | A07 | CWE-862 | Phase 1b 标 authz_state=unsecured 的 entry 节点 + 4b LLM 推理引用 | (无 sink，检测鉴权缺失) | Phase 1b 已标 |
| idor | A07 | CWE-639 | Phase 4 LLM 推理看到 entry_param 含资源 ID(id/userId/orderId)但 Phase 1b 显示无归属校验 | (无 sink，越权逻辑) | Phase 4b 处理 |
| deser_native | A08 | CWE-502 | Java: `ObjectInputStream.*readObject(\|XMLDecoder\|readObject()(\|*Object\|.*register.*ValidatingObjectInputStream` | @RequestBody / @XmlElement | `ObjectInputStream.readObject()` |
| deser_fastjson | A08 | CWE-502 | Java: `JSON.parseObject(\|JSONObject.parseObject\|JSON.parse(\|XStream.fromXML\|Jackson.*readValue.*withoutFeature` | @RequestBody / @FormField | `JSON.parseObject(userInput);` |
| jndi_injection | A08 | CWE-74 | Java: `Context.lookup(\|InitialContext.doLookup\|Context.*lookup(\|InitialContext.*lookup\|RmiInitialContextFactory` | @RequestParam `rmiUrl\|jndiName` | `context.lookup("rmi://"+userUrl);` |
| ssrf_http | A10 | CWE-918 | Java: `HttpClient\|RestTemplate.*exchange\|RestTemplate.*getForObject\|WebClient.*retrieve\|HttpURLConnection\|OkHttpClient.*execute\|URL.openConnection` Go: `http.Get\|http.Post\|http.NewRequest\|http.DefaultClient.Do` Python: `requests.get\|requests.post\|urllib.request.urlopen\|urllib2.urlopen\|httpx.` JS: `fetch(\|axios(\|request(\|needle.*get\|` | @RequestParam `url\|redirect\|target` | `restTemplate.getForObject(userUrl, String.class);` |
| ssrf_dns | A10 | CWE-918 | Java: `InetAddress.getByName\|InetAddress.getAllByName\|dns.Client.*lookup\|NameService` Go: `net.LookupHost\|net.LookupIP\|net.ResolveIPAddr` Python: `socket.gethostbyname\|dns.resolver.*resolve\|dnspython.*query\|` JS: `dns.lookup\|dns.resolve\|dns.resolve4` | @RequestParam `host\|domain\|hostname` | `InetAddress.getByName(userHost);` |
| xxe | A04 | CWE-611 | Java: `DocumentBuilderFactory\|SAXParserFactory\|XMLReader\|XMLInputFactory\|JAXBContext.*unmarshal\|SAXReader\|SAXBuilder\|Digester.*parse` Python: `xml.dom.*parseString\|xml.sax.parse\|lxml.etree.*parse` | @RequestBody xml | `DocumentBuilderFactory.newDocumentBuilder().parse(userXml);` |
| file_upload | A01 | CWE-434 | Java: `MultipartFile.transferTo\|Part.write(\|Files.copy.*MultipartFile\|FileItem.*write\|FileUpload.*parseRequest` Go: `fileHeader.Filename\|c.SaveUploadedFile` Python: `werkzeug.*secure_filename\|file.save(\|request.files[.*].save` JS: `multer.*storage\|formidable.*parse\|busboy.*on.*file` | @RequestParam file | `file.transferTo(new File(uploadDir + "/" + file.getOriginalFilename()));` |
| open_redirect | A01 | CWE-601 | Java: `response.sendRedirect(\|sendRedirect(\|response.sendRedirect(\|Spring.*redirect:\|RedirectView` Go: `http.Redirect\|c.Redirect(` Python: `redirect(\|HttpResponseRedirect\|flask.redirect(` JS: `res.redirect(\|location.href =` | @RequestParam `url\|redirect` | `response.sendRedirect(userUrl);` |
| xss_output | A03 | CWE-79 | Java: `response.getWriter().write(\|PrintWriter.*print.*user\|HttpServletResponse.*write` Python: `render_template_string(.*user\|HttpResponse(.*input\|.*return\|.*user_input.*response\|return Response(.*user` JS: `res.send(.*user\|element.innerHTML = .*user\|document.write(.*input` | @RequestParam `name\|html\|msg` | `response.getWriter().write("<h1>"+userName+"</h1>");` |

---

## 引用方式

Phase 0 SKILL.md 必读此文件执行 Bash grep 识别 sink。

每条命中立即写知识图谱 sink 节点：

```json
{
  "node_type": "sink",
  "sink_type": "(取本表 sink_type 列)",
  "owasp": "(取本表 owasp 列)",
  "cwe": "(取本表 cwe 列)",
  "path": "(命中文件路径)",
  "line": "(命中行号)",
  "code_snippet": "(截 sink 行原文，≤200 字)",
  "audit_state": "[ ]"
}
```

**不许**凭记忆判 sink——必须按本文件 grep 模式执行 bash（反幻觉#1）。

**不许**自行扩展 sink_type 命名——必须用本表 28 类之一。新 sink 模式需在 Phase 9 模式进化评估晋升后才能加入（详见子项目 F）。

0 命中（某类 sink 在全项目 0 命中）必须 QA 报警——可能 grep 模式漏覆盖或漏面（反幻觉#1）。

被反幻觉#1 引用。