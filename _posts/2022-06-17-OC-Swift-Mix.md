---
layout:     post
title:      "OC+Swift 混编知多少"
subtitle:   "简单记录自己在混编时遇到的困惑"
date:       2022-06-17 15:30:00
author:     "Self"
header-style: text
catalog: true
tags:
    - OC
    - Swift
    - iOS
---

## framework与modular
我的理解是在cocoapods在1.5.0版本之前如果要混编只能采用动态库管理，就是要在podfile里面加上 use_frameworks! 在引用OC库时要 #import<xxx/xxx.h> 使用尖括号这种形式。在1.5.0版本之后支持swift静态库了，所以可以使用 #use_modular_headers! 来管理所有的pod，modular是可以直接在Swift中进行import ，所以不需要再经过 bridging-header 的桥接。但是开启 use_modular_headers! 之后，会启用更严格的搜索路径和生成模块映射，历史项目可能会出现重复引用等问题，因为在一些老项目里CocoaPods是利用Header Search Paths 来完成引入编译，当然使用 use_modular_headers!可以提高加载性能和减少体积[^1]。<br>
OC库开启modular模式，build setting->define module->YES

OC主工程新建一个swift文件就可以，会提示是否自动创建一个 ProjectName-Bridging-Header.h的桥接文件，创建成功后就是混编模式了。没有自动创建的，自己查一下怎么手动创建吧。

## @objc @objcMembers open public
swift要想被oc访问到的前提是需要使用关键字 @objc @objcMembers,这2个关键字的区别是声明了@objcMembers会为内部的属性和方法都自动加上@objc。

默认访问权限是internal，模块内部访问，如果需要暴露给外部，至少需要是public，如果需要继承则应该是open

## 不同的混编场景

### Pod内部混编的互相调用

| 调用方 | 被调用方 | 方式 | 
| OC | OC | #import "xxx.h" |
| OC | swift | #import <PodName/PodName-Swift.h> | 
| swift | OC | 直接使用 |
| swift | swift | 直接使用 |

### 主工程中调用主工程中的OC Swift

| 调用方 | 被调用方 | 方式 | 
| OC | OC | #import "xxx.h" |
| OC | swift | #import "ProjectName-Swift.h" | 
| swift | OC | 在ProjectName-Bridging-Header.h 添加 import "xxx.h" |
| swift | swift | 直接使用 |

### 主工程调用pod的OC Swift

| 调用方 | 被调用方 | 方式 | 
| OC | OC | framework: #import<PodName/xxx.h>  module: @import PodName; |
| OC | swift | #import "PodName-Swift.h" | 
| swift | OC | 在ProjectName-Bridging-Header.h 添加 import "xxx.h" |
| swift | swift | import PodName |

### Pod调用pod

| 调用方 | 被调用方 | 方式 | 
| OC | OC | framework: #import<PodName/xxx.h>  module: @import PodName; |
| OC | swift | #import "PodName-Swift.h" | 
| swift | OC | 对应的OC库必须是modular，使用@import引入 |
| swift | swift | import PodName |

### 相关资料

[Swift与OC混编](https://blog.csdn.net/u012477117/article/details/122727567)

[Swift Static Libraries迁移实践](https://juejin.cn/post/6844903710909267976)

[^1]:[Flutter iOS OC 混编 Swift 遭遇动态库和静态库问题填坑](https://www.agora.io/cn/community/blog-120-category-24163)


