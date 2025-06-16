---
title: Ktor框架学习笔记
published: 2025-06-13 14:47:00
description: ''
tags: [WebServer, Kotlin]
category: 学习笔记
draft: false
---
# Introduce
[官方文档](https://ktor.io/docs/welcome.html)
>Ktor is a framework for building asynchronous server-side and client-side applications with ease.
# 支持平台
- Kotlin/JVM
  - jvm
- IOS
  - iosArm32
  - iosArm64
  - iosX64 
  - iosSimulatorArm64
- watchOS
  - watchosArm32
  - watchosArm64
  - watchosX86
  - watchosX64
  - watchosSimulatorArm64
- tvOS
  - tvosArm64
  - tvosX64
  - tvosSimulatorArm64
- macOS
  - macosX64
  - macosArm64
- linux
  - linuxX64
# 引入
```gradle
plugins {
    // ...
    id("io.ktor.plugin") version "3.1.3"
}
dependencies {
    implementation("io.ktor:ktor-server-core")
    implementation("io.ktor:ktor-server-netty")
    // ...
}
```
# 配置
## Configuration in code
```kotlin
package com.example

import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*

fun main(args: Array<String>) {
    if (args.isEmpty()) {
        println("Running basic server...")
        println("Provide the 'configured' argument to run a configured server.")
        runBasicServer()
    }

    when (args[0]) {
        "basic" -> runBasicServer()
        "configured" -> runConfiguredServer()
        else -> runServerWithCommandLineConfig(args)
    }
}

fun runBasicServer() {
    embeddedServer(Netty, port = 8080) {
        routing {
            get("/") {
                call.respondText("Hello, world!")
            }
        }
    }.start(wait = true)
}

fun runConfiguredServer() {
    embeddedServer(Netty, configure = {
        connectors.add(EngineConnectorBuilder().apply {
            host = "127.0.0.1"
            port = 8080
        })
        connectionGroupSize = 2
        workerGroupSize = 5
        callGroupSize = 10
        shutdownGracePeriod = 2000
        shutdownTimeout = 3000
    }) {
        routing {
            get("/") {
                call.respondText("Hello, world!")
            }
        }
    }.start(wait = true)
}

fun runServerWithCommandLineConfig(args: Array<String>) {
    embeddedServer(
        factory = Netty,
        configure = {
            val cliConfig = CommandLineConfig(args)
            takeFrom(cliConfig.engineConfig)
            loadCommonConfiguration(cliConfig.rootConfig.environment.config)
        }
    ) {
        routing {
            get("/") {
                call.respondText("Hello, world!")
            }
        }
    }.start(wait = true)
}
```
[示例代码](https://github.com/ktorio/ktor-documentation/tree/3.1.3/codeSnippets/snippets/embedded-server)
## Configuration in a file
Application.kt
```kotlin
package com.example

import io.ktor.server.application.*
import io.ktor.server.response.*
import io.ktor.server.routing.*

fun main(args: Array<String>): Unit = io.ktor.server.netty.EngineMain.main(args)

fun Application.module() {
    routing {
        get("/") {
            call.respondText("Hello, world!")
        }
    }
}
```
application.conf
```
ktor {
    deployment {
        port = 8080
    }
    application {
        modules = [ com.example.ApplicationKt.module ]
    }
}
```
# 引擎
- Netty
- Jetty
- Tomcat
- CIO(Coroutine-based I/O)
- ServletApplicationEngine
# 模块
Ktor允许你通过模块来构建应用。
```kotlin
import io.ktor.server.application.*
import io.ktor.server.response.*
import io.ktor.server.routing.*

fun Application.module1() {
    routing {
        get("/module1") {
            call.respondText("Hello from 'module1'!")
        }
    }
}
```
```config
ktor {
    application {
        modules = [ com.example.ApplicationKt.module1,
                    com.example.ApplicationKt.module2,
                    org.sample.SampleKt.module3 ]
    }
}
```
# 插件
插件的设计提供了最大的灵活性，而且允许它们出现在管道的任何地方。
![a](https://ktor.io/docs/images/plugin-pipeline.png)
```kotlin
import io.ktor.server.application.*
import io.ktor.server.plugins.cors.*
import io.ktor.server.plugins.compression.*
// ...
fun main() {
    embeddedServer(Netty, port = 8080) {
        install(CORS)
        install(Compression)
        // ...
    }.start(wait = true)
}
```
# 开发模式
```conf
ktor {
    development = true
}
```
Ktor提供了针对开发的特殊模式，该模式提供了一下功能：
- Auto-reload
- Extended information for debugging pipelines
- Extended debugging information on a response page in a case of a 5xx server error
# Routing
## 定义
```kotlin
import io.ktor.server.routing.*
import io.ktor.http.*
import io.ktor.server.response.*

routing {
    route("/hello", HttpMethod.Get) {
        handle {
            call.respondText("Hello")
        }
    }
}
```
或者
```kotlin
import io.ktor.server.routing.*
import io.ktor.server.response.*

routing {
    get("/hello") {
        call.respondText("Hello")
    }
}
```
定义路由需要包含以下内容：
- HTTP verb
- Path pattern
- Handler
## 路径模式
- Wildcard `/user/*`
- Tailcard `/user/{...}`
- Path parameter `/user/{login}`和`/user/{login?}`
- Path parameter with tailcard `/user/{param...}`
- Regular expression `Reg(".+/hello")`
  - 命名组，使用`（？<name>pattern）`定义，在handler中通过`call.parameters`来获取
## 多个路由
### 按动词函数分组
```kotlin
routing {
    get("/customer/{id}") {

    }
    post("/customer") {

    }
    get("/order") {

    }
    get("/order/{id}") {

    }
}
```
### 按路径分组
```kotlin
routing {
    route("/customer") {
        get {

        }
        post {

        }
    }
    route("/order") {
        get {

        }
        get("/{id}") {

        }
    }
}
```
### 嵌套路由
```kotlin
routing {
    route("/order") {
        route("/shipment") {
            get {

            }
            post {

            }
        }
    }
}
```
## 扩展函数
```kotlin
routing {
    listOrdersRoute()
    getOrderRoute()
    totalizeOrderRoute()
}

fun Route.listOrdersRoute() {
    get("/order") {

    }
}

fun Route.getOrderRoute() {
    get("/order/{id}") {

    }
}

fun Route.totalizeOrderRoute() {
    get("/order/{id}/total") {

    }
}
```
## 路由跟踪
通过配置日志，Ktor会启用路由跟踪，帮助确定某些路由未执行的原因。
## Type-safe routing(Resource)
### 引入
```gradle
implementation("io.ktor:ktor-server-resources:$ktor_version")
```
### 定义resource
```kotlin
import io.ktor.resources.*

@Resource("/articles")
class Articles()

@Resource("/articles")
class Articles(val sort: String? = "new")

@Resource("/articles")
class Articles() {
    @Resource("new")
    class New(val parent: Articles = Articles())
}

@Resource("/articles")
class Articles() {
    @Resource("{id}")
    class Id(val parent: Articles = Articles(), val id: Long)
}
```
### 定义处理程序
```kotlin
fun Application.module() {
    install(Resources)
    routing {
        get<Articles> { article ->
            // Get all articles ...
            call.respondText("List of articles sorted starting from ${article.sort}")
        }
        get<Articles.New> {
            // Show a page with fields for creating a new article ...
            call.respondText("Create a new article")
        }
        post<Articles> {
            // Save an article ...
            call.respondText("An article is saved", status = HttpStatusCode.Created)
        }
        get<Articles.Id> { article ->
            // Show an article with id ${article.id} ...
            call.respondText("An article with id ${article.id}", status = HttpStatusCode.OK)
        }
        get<Articles.Id.Edit> { article ->
            // Show a page with fields for editing an article ...
            call.respondText("Edit an article with id ${article.parent.id}", status = HttpStatusCode.OK)
        }
        put<Articles.Id> { article ->
            // Update an article ...
            call.respondText("An article with id ${article.id} updated", status = HttpStatusCode.OK)
        }
        delete<Articles.Id> { article ->
            // Delete an article ...
            call.respondText("An article with id ${article.id} deleted", status = HttpStatusCode.OK)
        }
    }
}
```
### HTML DSL
```kotlin
get {
    call.respondHtml {
        body {
            this@module.apply {
                p {
                    val link: String = href(Articles())
                    a(link) { +"Get all articles" }
                }
                p {
                    val link: String = href(Articles.New())
                    a(link) { +"Create a new article" }
                }
                p {
                    val link: String = href(Articles.Id.Edit(Articles.Id(id = 123)))
                    a(link) { +"Edit an exising article" }
                }
                p {
                    val urlBuilder = URLBuilder(URLProtocol.HTTPS, "ktor.io", parameters = parametersOf("token", "123"))
                    href(Articles(sort = null), urlBuilder)
                    val link: String = urlBuilder.buildString()
                    i { a(link) { +link } }
                }
            }
        }
    }
}
```
## Application structure
### 按文件分组
CustomerRoutes.kt
```kotlin
fun Route.customerByIdRoute() {
    get("/customer/{id}") {

    }
}

fun Route.createCustomerRoute() {
    post("/customer") {

    }
}
```
OrderRoutes.kt
```kotlin
fun Route.getOrderRoute() {
    get("/order/{id}") {

    }
}

fun Route.totalizeOrderRoute() {
    get("/order/{id}/total") {

    }
}
```
Application.kt
```kotlin
routing {
    customerRouting()
    listOrdersRoute()
    getOrderRoute()
    totalizeOrderRoute()
}
```
### 按定义分组
CustomerRoutes.kt
```kotlin
fun Application.customerRoutes() {
    routing {
        listCustomersRoute()
        customerByIdRoute()
        createCustomerRoute()
        deleteCustomerRoute()
    }
}
```
OrderRoutes.kt
```kotlin
fun Application.orderRoutes() {
    routing {
        listOrdersRoute()
        getOrderRoute()
        totalizeOrderRoute()
    }
}
```
Application.kt
```kotlin
fun Application.module() {
    // Init....
    customerRoutes()
    orderRoutes()
}
```
### 按文件夹分组
![](https://ktor.io/docs/images/ktor-routing-1.png)
### 按特征分组
![](https://ktor.io/docs/images/ktor-routing-2.png)
![](https://ktor.io/docs/images/ktor-routing-3.png)
## AutoHeadResponse插件
引用次插件，可以实现对于定义的`GET`，可以自动响应`HEAD`请求。
```kotlin
implementation("io.ktor:ktor-server-auto-head-response:$ktor_version")
```
```kotlin
import io.ktor.server.application.*
import io.ktor.server.plugins.autohead.*
import io.ktor.server.response.*
import io.ktor.server.routing.*

fun Application.main() {
    install(AutoHeadResponse)
    routing {
        get("/home") {
            call.respondText("This is a response to a GET, but HEAD also works")
        }
    }
}
```
# Request
## Handling requests
### 一般请求信息
- url `call.request.uri`
- headers `ApplicationRequest.headers`
- cookies `ApplicationRequest.cookies`
- connection details `APplicationRequest.local`
- X-Forwarded- headers `ApplicationRequest.origin`
### Path parameters
```kotlin
get("/user/{login}") {
    if (call.parameters["login"] == "admin") {
        // ...
    }
}
```
### Query parameters
```kotlin
get("/products") {
    if (call.request.queryParameters["price"] == "asc") {
        // Show products from the lowest price to the highest
    }
}
```
### Body contents
#### Raw payload
- String `call.receiveText()`
- ByteArray `call.receive<ByteArray>()`
- ByteReadChannel `call.receiveChannel().readText()`
#### Objects
```kotlin
post("/customer") {
    val customer = call.receive<Customer>()
    customerStorage.add(customer)
    call.respondText("Customer stored correctly", status = HttpStatusCode.Created)
```
#### Form parameters
```kotlin
post("/signup") {
    val formParameters = call.receiveParameters()
    val username = formParameters["username"].toString()
    call.respondText("The '$username' account is created")
}
```
#### Multipart form data
```kotlin
import io.ktor.server.application.*
import io.ktor.http.content.*
import io.ktor.server.request.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.util.cio.*
import io.ktor.utils.io.*
import java.io.File

fun Application.main() {
    routing {
        post("/upload") {
            var fileDescription = ""
            var fileName = ""
            val multipartData = call.receiveMultipart(formFieldLimit = 1024 * 1024 * 100)

            multipartData.forEachPart { part ->
                when (part) {
                    is PartData.FormItem -> {
                        fileDescription = part.value
                    }

                    is PartData.FileItem -> {
                        fileName = part.originalFileName as String
                        val file = File("uploads/$fileName")
                        part.provider().copyAndClose(file.writeChannel())
                    }

                    else -> {}
                }
                part.dispose()
            }

            call.respondText("$fileDescription is uploaded to 'uploads/$fileName'")
        }
    }
}
```
## Request validation
### 引入
```kotlin
implementation("io.ktor:ktor-server-request-validation:$ktor_version")
```
### 配置
Receive body
```kotlin
routing {
    post("/text") {
        val body = call.receive<String>()
        call.respond(body)
    }
}
```
Validation function
```kotlin
install(RequestValidation) {
    validate<String> { bodyText ->
        if (!bodyText.startsWith("Hello"))
            ValidationResult.Invalid("Body text should start with 'Hello'")
        else ValidationResult.Valid
    }
}
```
Handle validation exceptions
```kotlin
install(StatusPages) {
    exception<RequestValidationException> { call, cause ->
        call.respond(HttpStatusCode.BadRequest, cause.reasons.joinToString())
    }
}
```
## Rate limiting
### 引入
```kotlin
implementation("io.ktor:ktor-server-rate-limit:$ktor_version")
```
### 例子
```kotlin
package com.example

import io.ktor.http.*
import io.ktor.server.application.*
import io.ktor.server.plugins.ratelimit.*
import io.ktor.server.plugins.statuspages.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import kotlin.time.Duration.Companion.seconds

fun main(args: Array<String>): Unit = io.ktor.server.netty.EngineMain.main(args)

fun Application.module() {
    install(RateLimit) {
        register {
            rateLimiter(limit = 5, refillPeriod = 60.seconds)
        }
        register(RateLimitName("public")) {
            rateLimiter(limit = 10, refillPeriod = 60.seconds)
        }
        register(RateLimitName("protected")) {
            rateLimiter(limit = 30, refillPeriod = 60.seconds)
            requestKey { applicationCall ->
                applicationCall.request.queryParameters["login"]!!
            }
            requestWeight { applicationCall, key ->
                when(key) {
                    "jetbrains" -> 1
                    else -> 2
                }
            }
        }
    }
    install(StatusPages) {
        status(HttpStatusCode.TooManyRequests) { call, status ->
            val retryAfter = call.response.headers["Retry-After"]
            call.respondText(text = "429: Too many requests. Wait for $retryAfter seconds.", status = status)
        }
    }
    routing {
        rateLimit {
            get("/") {
                val requestsLeft = call.response.headers["X-RateLimit-Remaining"]
                call.respondText("Welcome to the home page! $requestsLeft requests left.")
            }
        }
        rateLimit(RateLimitName("public")) {
            get("/public-api") {
                val requestsLeft = call.response.headers["X-RateLimit-Remaining"]
                call.respondText("Welcome to public API! $requestsLeft requests left.")
            }
        }
        rateLimit(RateLimitName("protected")) {
            get("/protected-api") {
                val requestsLeft = call.response.headers["X-RateLimit-Remaining"]
                val login = call.request.queryParameters["login"]
                call.respondText("Welcome to protected API, $login! $requestsLeft requests left.")
            }
        }
    }
}
```
# Response
## Set response payload
### Plain text
```kotlin
get("/") {
    call.respondText("Hello, world!")
}
```
### HTML
```kotlin
routing {
    get("/") {
        val name = "Ktor"
        call.respondHtml(HttpStatusCode.OK) {
            head {
                title {
                    +name
                }
            }
            body {
                h1 {
                    +"Hello from $name!"
                }
            }
        }
    }
}
```
```kotlin
get("/index") {
    val sampleUser = User(1, "John")
    call.respond(FreeMarkerContent("index.ftl", mapOf("user" to sampleUser)))
}
```
```kotlin
get("/index") {
    val sampleUser = User(1, "John")
    call.respondTemplate("index.ftl", mapOf("user" to sampleUser))
}
```
### Object
```kotlin
routing {
    get("/customer/{id}") {
        val id: Int by call.parameters
        val customer: Customer = customerStorage.find { it.id == id }!!
        call.respond(customer)
```
### File
```kotlin
import io.ktor.http.*
import io.ktor.server.application.*
import io.ktor.server.http.content.*
import io.ktor.server.plugins.autohead.*
import io.ktor.server.plugins.partialcontent.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import java.io.File
import java.nio.file.Path

fun Application.main() {
    install(PartialContent)
    install(AutoHeadResponse)
    routing {
        get("/download") {
            val file = File("files/ktor_logo.png")
            call.response.header(
                HttpHeaders.ContentDisposition,
                ContentDisposition.Attachment.withParameter(ContentDisposition.Parameters.FileName, "ktor_logo.png")
                    .toString()
            )
            call.respondFile(file)
        }
        get("/downloadFromPath") {
            val filePath = Path.of("files/file.txt")
            call.response.header(
                HttpHeaders.ContentDisposition,
                ContentDisposition.Attachment.withParameter(ContentDisposition.Parameters.FileName, "file.txt")
                    .toString()
            )
            call.respond(LocalPathContent(filePath))
        }
    }
```
### Raw payload
`call.respondBytes`
## Set response parameters
### Status code
```kotlin
get("/") {
    call.response.status(HttpStatusCode.OK)
}
get("/") {
    call.response.status(HttpStatusCode(418, "I'm a tea pot"))
}
```
### Content type
```kotlin
get("/") {
    call.respondText("Hello, world!", ContentType.Text.Plain, HttpStatusCode.OK)
}
```
### Headers
```kotlin
get("/") {
    call.response.headers.append(HttpHeaders.ETag, "7c876b7e")
}
get("/") {
    call.response.header(HttpHeaders.ETag, "7c876b7e")
}
get("/") {
    call.response.etag("7c876b7e")
}
get("/") {
    call.response.header("Custom-Header", "Some value")
}
```
### Coolies
```kotlin
get("/") {
    call.response.cookies.append("yummy_cookie", "choco")
}
```
### Redirects
```kotlin
get("/") {
    call.respondRedirect("/moved", permanent = true)
}

get("/moved") {
    call.respondText("Moved content")
}
```
## Status pages
### 引入 
```kotlin
implementation("io.ktor:ktor-server-status-pages:$ktor_version")
```
### 配置
#### Exceptions
```kotlin
install(StatusPages) {
    exception<Throwable> { call, cause ->
        call.respondText(text = "500: $cause" , status = HttpStatusCode.InternalServerError)
    }
}
```
```kotlin
install(StatusPages) {
    exception<Throwable> { call, cause ->
        if(cause is AuthorizationException) {
            call.respondText(text = "403: $cause" , status = HttpStatusCode.Forbidden)
        } else {
            call.respondText(text = "500: $cause" , status = HttpStatusCode.InternalServerError)
        }
    }
}
```
#### Status
```kotlin
install(StatusPages) {
    status(HttpStatusCode.NotFound) { call, status ->
        call.respondText(text = "404: Page Not Found", status = status)
    }
}
```
#### Status file
```kotlin
install(StatusPages) {
    statusFile(HttpStatusCode.Unauthorized, HttpStatusCode.PaymentRequired, filePattern = "error#.html")
}
```
# Content negotiation and serialization
## Dependencies
```gradle
implementation("io.ktor:ktor-server-content-negotiation:$ktor_version")
implementation("io.ktor:ktor-serialization-kotlinx-json:$ktor_version")
implementation("io.ktor:ktor-serialization-kotlinx-xml:$ktor_version")
implementation("io.ktor:ktor-serialization-kotlinx-cbor:$ktor_version")
implementation("io.ktor:ktor-serialization-kotlinx-protobuf:$ktor_version")
```
## Config
### JSON
```kotlin
install(ContentNegotiation) {
    json(Json {
        prettyPrint = true
        isLenient = true
    })
```
### XML
```kotlin
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.xml.*
import nl.adaptivity.xmlutil.*
import nl.adaptivity.xmlutil.serialization.*

install(ContentNegotiation) {
    xml(format = XML {
        xmlDeclMode = XmlDeclMode.Charset
    })
}
```
### CBOR
```kotlin
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.cbor.*
import kotlinx.serialization.cbor.*

install(ContentNegotiation) {
    cbor(Cbor {
        ignoreUnknownKeys = true
    })
}
```
### ProtoBuf
```kotlin
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.protobuf.*
import kotlinx.serialization.protobuf.*

install(ContentNegotiation) {
    protobuf(ProtoBuf {
        encodeDefaults = true
    })
}
```
### 自定义
```kotlin
install(ContentNegotiation) {
    register(ContentType.Application.Json, CustomJsonConverter())
    register(ContentType.Application.Xml, CustomXmlConverter())
}
```
## 接收和发送数据
### 接收
```kotlin
post("/customer") {
    val customer = call.receive<Customer>()
    customerStorage.add(customer)
    call.respondText("Customer stored correctly", status = HttpStatusCode.Created)
```
### 发送
```kotlin
routing {
    get("/customer/{id}") {
        val id: Int by call.parameters
        val customer: Customer = customerStorage.find { it.id == id }!!
        call.respond(customer)
```
# 静态内容
## Folders
```kotlin
routing {
    staticFiles("/resources", File("files"))
}
```
## ZIP
```kotlin
routing {
    staticZip("/", "", Paths.get("files/text-files.zip"))
}
```
## Resources
```kotlin
routing {
    staticResources("/resources", "static")
}
```
## 其他配置
### Index file
```kotlin
staticResources("/custom", "static", index = "custom_index.html")
```
### 预压缩文件
```kotlin
staticFiles("/", File("files")) {
    preCompressed(CompressedFileType.BROTLI, CompressedFileType.GZIP)
}
```
### HEAD requests
```kotlin
staticResources("/", "static"){
    enableAutoHeadResponse()
}
```
### Default file response
```kotlin
staticFiles("/", File("files")) {
    default("index.html")
}
```
### Content type
```kotlin
staticFiles("/files", File("textFiles")) {
    contentType { file ->
        when (file.name) {
            "html-file.txt" -> ContentType.Text.Html
            else -> null
        }
    }
}
```
### Caching
```kotlin
fun Application.module() {
    routing {
        staticFiles("/files", File("textFiles")) {
            cacheControl { file ->
                when (file.name) {
                    "file.txt" -> listOf(Immutable, CacheControl.MaxAge(10000))
                    else -> emptyList()
                }
            }
        }
    }
}
object Immutable : CacheControl(null) {
    override fun toString(): String = "immutable"
}
```
### Excluded files
```kotlin
staticFiles("/files", File("textFiles")) {
    exclude { file -> file.path.contains("excluded") }
}
```
### File extensions fallbacks
```kotlin
staticResources("/", "static"){
    extensions("html", "htm")
}
```
### Custom modifications
```kotlin
staticFiles("/", File("files")) {
    modify { file, call ->
        call.response.headers.append(HttpHeaders.ETag, file.name.toString())
    }
}
```
# Single-page服务
## 特定框架
```text
react-app
├── index.html
├── ...
└── static
    └── ...
```
```kotlin
import io.ktor.server.application.*
import io.ktor.server.http.content.*
import io.ktor.server.routing.*

fun Application.module() {
    routing {
        singlePageApplication {
            react("react-app")
        }
    }
}
```
## 自定义
```text
sample-web-app
├── main.html
├── ktor_logo.png
├── css
│   └──styles.css
└── js
    └── script.js
```
```kotlin
import io.ktor.server.application.*
import io.ktor.server.http.content.*
import io.ktor.server.routing.*

fun Application.module() {
    routing {
        singlePageApplication {
            useResources = true
            filesPath = "sample-web-app"
            defaultPage = "main.html"
            ignoreFiles { it.endsWith(".txt") }
        }
    }
}
```
# 模版
- HTML DSL
- CSS DSL
- FreeMarker
- Velocity
- Mustache
- Thymeleaf
- Pebble
- JTE
# 身份验证和授权
## 支持的验证类型
- HTTP
- Form-based
- JSON Web Tokens(JWT)
- LDAP
- OAuth
- Session
- Custom

[详细配置](https://ktor.io/docs/server-auth.html)
## 使用
```kotlin
implementation("io.ktor:ktor-server-auth:$ktor_version")
```
### Step 1: 选择provider
```kotlin
import io.ktor.server.application.*
import io.ktor.server.auth.*
// ...
install(Authentication) {
    basic {
        // Configure basic authentication
    }
}
```
### Step 2: 指定provider名称
```kotlin
install(Authentication) {
    basic("auth-basic") {
        // Configure basic authentication
    }
    form("auth-form") {
        // Configure form authentication
    }
    // ...
}
```
### Step 3: 配置provider
```kotlin
install(Authentication) {
    basic("auth-basic") {
        realm = "Access to the '/' path"
        validate { credentials ->
            if (credentials.name == "jetbrains" && credentials.password == "foobar") {
                UserIdPrincipal(credentials.name)
            } else {
                null
            }
        }
    }
}
```
### Step 4: 保护特定资源
```kotlin
routing {
    authenticate("auth-basic") {
        get("/login") {
            // ...
        }
        get("/orders") {
            // ...
        }
    }
    get("/") {
        // ...
    }
}
```
```kotlin
routing {
    authenticate("auth-session", strategy = AuthenticationStrategy.Required) {
        get("/hello") {
            // ...
        }
        authenticate("auth-basic", strategy = AuthenticationStrategy.Required) {
            get("/admin") {
                // ...
            }
        }
    }
}
```
### Step 5: 在路由处理程序中获取principal
```kotlin
routing {
    authenticate("auth-basic") {
        get("/") {
            call.respondText("Hello, ${call.principal<UserIdPrincipal>()?.name}!")
        }
    }
}
```
# Session
`implementation("io.ktor:ktor-server-sessions:$ktor_version")`
[详情](https://ktor.io/docs/server-sessions.html)
# HTTP
## Swagger
`implementation("io.ktor:ktor-server-swagger:$ktor_version")`
```kotlin
import io.ktor.server.plugins.swagger.*

// ...
routing {
    swaggerUI(path = "swagger", swaggerFile = "openapi/documentation.yaml")
}
```
## OpenAPI
`implementation("io.ktor:ktor-server-openapi:$ktor_version")`
```kotlin
import io.ktor.server.plugins.openapi.*

// ...
routing {
    openAPI(path="openapi", swaggerFile = "openapi/documentation.yaml")
}
```
## Default headers
`implementation("io.ktor:ktor-server-default-headers:$ktor_version")`
## 压缩
`implementation("io.ktor:ktor-server-compression:$ktor_version")`
```kotlin
// specific encoders
install(Compression) {
    gzip()
    deflate()
}
// content type
install(Compression) {
    gzip {
        matchContentType(
            ContentType.Application.JavaScript
        )
    }
    deflate {
        matchContentType(
            ContentType.Text.Any
        )
    }
}
// response size
install(Compression) {
    deflate {
        minimumSize(1024)
    }
}
// custom conditions
install(Compression) {
    gzip {
        condition {
            request.uri == "/orders"
        }
    }
}
```
## CORS
`implementation("io.ktor:ktor-server-cors:$ktor_version")`
### 配置
#### Hosts
```kotlin
install(CORS) {
    allowHost("client-host")
    allowHost("client-host:8081")
    allowHost("client-host", subDomains = listOf("en", "de", "es"))
    allowHost("client-host", schemes = listOf("http", "https"))
}
install(CORS) {
    anyHost()
}
```
#### HTTP methods
```kotlin
install(CORS) {
    allowMethod(HttpMethod.Options)
    allowMethod(HttpMethod.Put)
    allowMethod(HttpMethod.Patch)
    allowMethod(HttpMethod.Delete)
}
```
#### Allow headers
```kotlin
install(CORS) {
    allowHeader(HttpHeaders.ContentType)
    allowHeader(HttpHeaders.Authorization)
}
```
#### Expose headers
```kotlin
install(CORS) {
    // ...
    exposeHeader("X-My-Custom-Header")
    exposeHeader("X-Another-Custom-Header")
}
```
#### Credentials
```kotlin
install(CORS) {
    allowCredentials = true
}
```
## Data conversion
```kotlin
install(DataConversion) {
    convert<LocalDate> { // this: DelegatingConversionService
        val formatter = DateTimeFormatterBuilder()
            .appendValue(ChronoField.YEAR, 4, 4, SignStyle.NEVER)
            .appendValue(ChronoField.MONTH_OF_YEAR, 2)
            .appendValue(ChronoField.DAY_OF_MONTH, 2)
            .toFormatter(Locale.ROOT)

        decode { values -> // converter: (values: List<String>) -> Any?
            LocalDate.from(formatter.parse(values.single()))
        }

        encode { value -> // converter: (value: Any?) -> List<String>
            listOf(SimpleDateFormat.getInstance().format(value))
        }
    }
}
```
# Websockets
## 引入
`implementation("io.ktor:ktor-server-websockets:$ktor_version")`
## 配置
```kotlin
install(WebSockets) {
    pingPeriod = 15.seconds // ping的间隔时间
    timeout = 15.seconds // 关闭连接的超时时间
    maxFrameSize = Long.MAX_VALUE // 接收或发送的最大Frame
    masking = false // 是否启用masking
}
```
## Handling
```kotlin
routing {
    webSocket("/echo") {
       // Handle a WebSocket session
    }
}
```
```kotlin
routing {
    webSocket("/echo") {
        send("Please enter your name")
        for (frame in incoming) {
            frame as? Frame.Text ?: continue
            val receivedText = frame.readText()
            if (receivedText.equals("bye", ignoreCase = true)) {
                close(CloseReason(CloseReason.Codes.NORMAL, "Client said BYE"))
            } else {
                send(Frame.Text("Hi, $receivedText!"))
            }
        }
    }
}
```
```kotlin
val messageResponseFlow = MutableSharedFlow<MessageResponse>()
val sharedFlow = messageResponseFlow.asSharedFlow()

webSocket("/ws") {
    send("You are connected to WebSocket!")

    val job = launch {
        sharedFlow.collect { message ->
            send(message.message)
        }
    }

    runCatching {
        incoming.consumeEach { frame ->
            if (frame is Frame.Text) {
                val receivedText = frame.readText()
                val messageResponse = MessageResponse(receivedText)
                messageResponseFlow.emit(messageResponse)
            }
        }
    }.onFailure { exception ->
        println("WebSocket exception: ${exception.localizedMessage}")
    }.also {
        job.cancel()
    }
}
```
## Websocket API
- onConnect
- onMessage
- onClose
- onError
# Server-Sent Events(SSE)
`implementation("io.ktor:ktor-server-sse:$ktor_version")`
```kotlin
routing {
    sse("/events") {
        // send events to clients
    }
}
```
# Sockets
## Server
```kotlin
package com.example

import io.ktor.network.selector.*
import io.ktor.network.sockets.*
import io.ktor.utils.io.*
import kotlinx.coroutines.*

fun main(args: Array<String>) {
    runBlocking {
        val selectorManager = SelectorManager(Dispatchers.IO)
        val serverSocket = aSocket(selectorManager).tcp().bind("127.0.0.1", 9002)
        println("Server is listening at ${serverSocket.localAddress}")
        while (true) {
            val socket = serverSocket.accept()
            println("Accepted $socket")
            launch {
                val receiveChannel = socket.openReadChannel()
                val sendChannel = socket.openWriteChannel(autoFlush = true)
                sendChannel.writeStringUtf8("Please enter your name\n")
                try {
                    while (true) {
                        val name = receiveChannel.readUTF8Line()
                        sendChannel.writeStringUtf8("Hello, $name!\n")
                    }
                } catch (e: Throwable) {
                    socket.close()
                }
            }
        }
    }
}
```
## Client
```kotlin
package com.example

import io.ktor.network.selector.*
import io.ktor.network.sockets.*
import io.ktor.utils.io.*
import kotlinx.coroutines.*
import kotlin.system.*

fun main(args: Array<String>) {
    runBlocking {
        val selectorManager = SelectorManager(Dispatchers.IO)
        val socket = aSocket(selectorManager).tcp().connect("127.0.0.1", 9002)

        val receiveChannel = socket.openReadChannel()
        val sendChannel = socket.openWriteChannel(autoFlush = true)

        launch(Dispatchers.IO) {
            while (true) {
                val greeting = receiveChannel.readUTF8Line()
                if (greeting != null) {
                    println(greeting)
                } else {
                    println("Server closed a connection")
                    socket.close()
                    selectorManager.close()
                    exitProcess(0)
                }
            }
        }

        while (true) {
            val myMessage = readln()
            sendChannel.writeStringUtf8("$myMessage\n")
        }
    }
}
```
# Monitoring
## Call logging
`implementation("io.ktor:ktor-server-call-logging:$ktor_version")`
```kotlin
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.application.*
import io.ktor.server.plugins.calllogging.*

fun main() {
    embeddedServer(Netty, port = 8080) {
        install(CallLogging) {
            // logging level
            level = Level.INFO
            // filter
            filter { call ->
                call.request.path().startsWith("/api/v1")
            }
            // customize format
            format { call ->
                val status = call.response.status()
                val httpMethod = call.request.httpMethod.value
                val userAgent = call.request.headers["User-Agent"]
                "Status: $status, HTTP method: $httpMethod, User agent: $userAgent"
            }
            // MDC
            mdc("name-parameter") { call ->
                call.request.queryParameters["name"]
            }
        }
    }.start(wait = true)
}
```
## Tracing requests
`implementation("io.ktor:ktor-server-call-id:$ktor_version")`
```kotlin
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.application.*
import io.ktor.server.plugins.callid.*

fun main() {
    embeddedServer(Netty, port = 8080) {
        install(CallId) {
            // Retrieve a call ID
            retrieveFromHeader(HttpHeaders.XRequestId)
            retrieve { call ->
                call.request.header(HttpHeaders.XRequestId)
            }
            // Generate a call ID
            generate(10, "abcde12345")
            val counter = atomic(0)
            generate {
                "generated-call-id-${counter.getAndIncrement()}"
            }
            // Verify a call ID
            verify { callId: String ->
                callId.isNotEmpty()
            }
            // Send a call ID to the client
            header(HttpHeaders.XRequestId)
            replyToHeader(HttpHeaders.XRequestId)
            call.response.header(HttpHeaders.XRequestId, callId)
            // Put a call ID into MDC
            callIdMdc("call-id")
        }
    }.start(wait = true)
}
```
## Micrometer metrics
### 引入
`implementation("io.ktor:ktor-server-metrics-micrometer:$ktor_version")`
`implementation("io.micrometer:micrometer-registry-prometheus:$prometheus_version")`
```kotlin
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.application.*
import io.ktor.server.metrics.micrometer.*

fun main() {
    embeddedServer(Netty, port = 8080) {
        install(MicrometerMetrics)
        // ...
    }.start(wait = true)
}
```
### 创建注册表
```kotlin
import io.ktor.server.application.*
import io.ktor.server.metrics.micrometer.*
// ...
fun Application.module() {
    // Prometheus: expose a scrape endpoint
    val appMicrometerRegistry = PrometheusMeterRegistry(PrometheusConfig.DEFAULT)
    install(MicrometerMetrics) {
        registry = appMicrometerRegistry
        // Timers
        timers { call, exception ->
            tag("region", call.request.headers["regionId"])
        }
        // Distribution statistics
        distributionStatisticConfig = DistributionStatisticConfig.Builder()
            .percentilesHistogram(true)
            .maximumExpectedValue(Duration.ofSeconds(20).toNanos().toDouble())
            .serviceLevelObjectives(
                Duration.ofMillis(100).toNanos().toDouble(),
                Duration.ofMillis(500).toNanos().toDouble()
            )
            .build()
        // JVM and system metrics
        meterBinders = listOf(
            JvmMemoryMetrics(),
            JvmGcMetrics(),
            ProcessorMetrics()
        )
    }
    // Prometheus: expose a scrape endpoint
    routing {
        get("/metrics") {
            call.respond(appMicrometerRegistry.scrape())
        }
    }
}
```
## Dropwizard metrics
`implementation("io.ktor:ktor-server-metrics:$ktor_version")`
```kotlin
import io.ktor.server.application.*
import io.ktor.server.metrics.dropwizard.*
// ...
fun Application.module() {
    install(DropwizardMetrics) {
        // SLF4J reporter
        Slf4jReporter.forRegistry(registry)
            .outputTo(this@module.log)
            .convertRatesTo(TimeUnit.SECONDS)
            .convertDurationsTo(TimeUnit.MILLISECONDS)
            .build()
            .start(10, TimeUnit.SECONDS)
        // JMX reporter
        JmxReporter.forRegistry(registry)
            .convertRatesTo(TimeUnit.SECONDS)
            .convertDurationsTo(TimeUnit.MILLISECONDS)
            .build()
            .start()
    }
}
```
# Testing
`testImplementation("io.ktor:ktor-server-test-host:$ktor_version")`
`testImplementation("org.jetbrains.kotlin:kotlin-test:$kotlin_version")`
```kotlin
package com.example

import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.server.application.*
import io.ktor.server.testing.*
import kotlin.test.*

class ApplicationTest {
    @Test
    fun testRoot() = testApplication {
        application {
            module()
        }
        val response = client.get("/")
        assertEquals(HttpStatusCode.OK, response.status)
        assertEquals("Hello, world!", response.bodyAsText())
    }
}
```
## 步骤
1. 创建JUnit测试类和测试方法
2. 使用testApplication函数，配置本地测试应用
3. 使用Ktor HTTP client向服务器发送请求、接收响应并进行断言
