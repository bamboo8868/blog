---
title: php扩展开发-hello world
date: 2020-01-01
categories:
 - php
tags:
 - php
---
PHP 扩展是提升应用性能、集成底层功能的重要方式。虽然大多数开发者通过 composer 安装现成扩展，但了解扩展开发能帮助你更深入理解 PHP 内核，甚至定制专属功能。本文将以 PHP 8.0 为例，从零开始开发一个简单的 `hello world` 扩展，带你掌握扩展开发的基本流程。

## 一、开发环境准备

### 1. 系统要求



*   操作系统：Linux（本文以 Ubuntu 20.04 为例）或 macOS

*   PHP 8.0 源码及开发工具：需要 PHP 源码用于生成扩展骨架，以及编译工具链

### 2. 安装依赖



```
\# 安装编译工具

sudo apt update

sudo apt install -y build-essential gcc make autoconf libtool

\# 安装 PHP 8.0 开发依赖

sudo apt install -y php8.0 php8.0-dev php8.0-cli

\# 验证 PHP 版本及开发文件

php -v  # 输出 PHP 8.0.x

php-config --version  # 确认 php-config 可用（扩展编译依赖）
```

### 3. 获取 PHP 源码（可选但推荐）

PHP 扩展开发需要参考内核头文件，建议下载对应版本的源码：



```
\# 下载 PHP 8.0.30 源码（版本需与系统安装的 PHP 一致）

wget https://www.php.net/distributions/php-8.0.30.tar.gz

tar -zxvf php-8.0.30.tar.gz

cd php-8.0.30/ext  # 扩展开发工具在 ext 目录下
```

## 二、生成扩展骨架

PHP 提供了 `ext_skel.php` 工具（位于 PHP 源码的 `ext/` 目录），可自动生成扩展的基础目录结构和配置文件。

### 1. 创建扩展目录



```
\# 进入 PHP 源码的 ext 目录（或任意工作目录）

cd php-8.0.30/ext

\# 生成扩展骨架，扩展名为 "hello"

./ext\_skel.php --ext hello
```

执行后会生成 `hello/` 目录，结构如下：



```
hello/

├── config.m4       # 编译配置文件（用于 ./configure）

├── CREDITS         # 扩展作者信息

├── hello.c         # 核心代码文件

├── hello.php       # 扩展测试脚本

├── php\_hello.h     # 头文件

└── README          # 说明文档
```

## 三、编写扩展代码

### 1. 修改配置文件 `config.m4`

`config.m4` 用于告诉 `configure` 脚本如何编译扩展。打开文件，删除注释符 `dnl` 启用以下配置：



```
\# 启用动态加载扩展（无需重新编译 PHP 内核）

PHP\_ARG\_ENABLE(hello, whether to enable hello support,

\[  --enable-hello           Enable hello support])

if test "\$PHP\_HELLO" = "yes"; then

&#x20; AC\_DEFINE(HAVE\_HELLO, 1, \[Whether you have hello])

&#x20; PHP\_NEW\_EXTENSION(hello, hello.c, \$ext\_shared)

fi
```

### 2. 实现 `hello_world()` 函数

打开核心代码文件 `hello.c`，在 `PHP_FUNCTION(confirm_hello_compiled)` 函数下方添加自定义函数：



```
// 声明函数（需在 php\_hello.h 中声明，或直接写在 hello.c 顶部）

PHP\_FUNCTION(hello\_world);

// 实现函数

PHP\_FUNCTION(hello\_world) {

&#x20;   // 输出 "Hello World from PHP 8.0 Extension!"

&#x20;   zend\_string \*str = zend\_string\_init("Hello World from PHP 8.0 Extension!", 0, 0);

&#x20;   RETURN\_STR(str);

}
```

### 3. 注册函数

在 `hello.c` 中找到 `zend_function_entry hello_functions[]` 数组，添加函数注册信息（让 PHP 识别 `hello_world()`）：



```
const zend\_function\_entry hello\_functions\[] = {

&#x20;   PHP\_FE(confirm\_hello\_compiled,  NULL)  /\* 自动生成的测试函数 \*/

&#x20;   PHP\_FE(hello\_world,             NULL)  /\* 新增：注册 hello\_world 函数 \*/

&#x20;   PHP\_FE\_END  /\* 结束标记 \*/

};
```

### 4. 头文件声明（可选）

为规范代码，可在 `php_hello.h` 中添加函数声明：



```
\#ifndef PHP\_HELLO\_H

\#define PHP\_HELLO\_H

extern zend\_module\_entry hello\_module\_entry;

\#define phpext\_hello\_ptr \&hello\_module\_entry

PHP\_FUNCTION(confirm\_hello\_compiled);

PHP\_FUNCTION(hello\_world);  /\* 新增：声明 hello\_world 函数 \*/

\#endif  /\* PHP\_HELLO\_H \*/
```

## 四、编译与安装扩展

### 1. 编译扩展

在 `hello/` 目录下执行以下命令，生成 `.so` 扩展文件：



```
\# 生成 configure 脚本

phpize

\# 配置编译选项（--enable-hello 对应 config.m4 中的设置）

./configure --enable-hello

\# 编译

make
```

编译成功后，会在 `modules/` 目录下生成 `hello.so`（扩展二进制文件）。

### 2. 安装扩展



```
\# 安装扩展到 PHP 扩展目录（需 root 权限）

sudo make install
```

执行后会显示扩展安装路径，类似：



```
Installing shared extensions:     /usr/lib/php/20200930/
```

## 五、配置 PHP 加载扩展

### 1. 添加扩展配置

创建扩展配置文件（或修改 `php.ini`）：



```
\# 创建配置文件

sudo touch /etc/php/8.0/cli/conf.d/20-hello.ini

\# 写入配置（扩展路径为上一步的安装路径）

echo "extension=hello.so" | sudo tee /etc/php/8.0/cli/conf.d/20-hello.ini
```

### 2. 验证扩展加载



```
php -m | grep hello  # 输出 "hello" 表示加载成功
```

## 六、测试扩展功能

创建测试脚本 `test_hello.php`：



```
\<?php

// 调用扩展中的函数

echo hello\_world() . "\n";

// 调用自动生成的测试函数（验证扩展编译成功）

echo confirm\_hello\_compiled("hello") . "\n";

?>
```

执行脚本：



```
php test\_hello.php
```

输出结果：



```
Hello World from PHP 8.0 Extension!

Congratulations! You have successfully modified ext/hello/config.m4. Module hello is now compiled into PHP.