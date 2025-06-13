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
# Request
# Response
