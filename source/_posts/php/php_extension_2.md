---
title: php扩展扩展开发之函数
date: 2020-01-02
categories: 
  - php
tags:
  - php
---
# 函数入口
## PHP_FUNCTION宏

宏替换流程
---
PHP_FUNCTION(name) 
-> ZEND_FUNCTION(name)
-> ZEND_NAMED_FUNCTION(zif_##name)
-> void ZEND_FASTCALL name(INTERNAL_FUNCTION_PARAMETERS)
-> void zif_name(zend_execute_data *execute_data, zval *return_value)