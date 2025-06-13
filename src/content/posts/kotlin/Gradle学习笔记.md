---
title: Gradle框架学习笔记
published: 2025-06-12 17:01:00
description: ''
tags: [Java, Kotlin, Build]
category: 学习笔记
draft: false
---
# 版本管理
## 使用gradle.properties + by project方式
```property
# gradle.properties
koin_version=3.2.0
retrofit_version=2.9.0
```
```kotlin
// build.gradle.kts
dependencies {
    implementation("io.insert-koin:koin-core:$koin_version")
    implementation("com.squareup.retrofit2:retrofit:$retrofit_version")
}
```
## 使用libs.versions.toml方式
```toml
# gradle/libs.versions.toml
[versions]
koin = "3.2.0"
retrofit = "2.9.0"

[libraries]
koin-core = { module = "io.insert-koin:koin-core", version.ref = "koin" }
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
```
```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.koin.core)
    implementation(libs.retrofit)
}
```