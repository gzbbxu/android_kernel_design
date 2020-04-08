#### android 底层开发

目录结构

| 目录       | 说明                       |
| ---------- | -------------------------- |
| bionic     | c库                        |
| build      | 编译系统规则基础开发包配置 |
| cts        | 兼容性测试                 |
| dalvik     | java虚拟机                 |
| external   | android 引入的第三方模块   |
| frameworks | android 核心框架           |
| hardware   | 硬件适配层                 |
| system     | 底层文件系统库，应用和组建 |
| device     | 产品目标目录               |
| out        | 编程生成目标文件目录       |

 			

android 原生应用 packages/apps/下







#### android 编译过程脚本分析

android 编译过程

1. 初始化参数设置
2. 检查环境变量与目标环境
3. 选择lunch 并读取目标配置和平台信息
4. 清空输出目录
5. 编译
6. 生成升级包



#### 编写自己的android.mk 

build/core/definitions.mk 

```
define my-dir
$(strip \
  $(eval LOCAL_MODULE_MAKEFILE := $$(lastword $$(MAKEFILE_LIST))) \
  $(eval LOCAL_MODULE_MAKEFILE_DEP := $(if $(BUILDING_WITH_NINJA),,$$(LOCAL_MODULE_MAKEFILE))) \
  $(if $(filter $(BUILD_SYSTEM)/% $(OUT_DIR)/%,$(LOCAL_MODULE_MAKEFILE)), \
    $(error my-dir must be called before including any other makefile.) \
   , \
    $(patsubst %/,%,$(dir $(LOCAL_MODULE_MAKEFILE))) \
   ) \
 )
endef
```

获取makefile 列表的最后一行。

```
LOCAL_PATH:=$(call my-dir) //最后一个makefile 的路径
include $(CLEAR_VARS)
LOCAL_MODULE:=test
LOCAL_SRC_FILES :=test.c
LOCAL_MODULE_PATH:=$(LOCAL_PATH) // 目标文件生成的路径
include $(BUILD_EXECUTABLE) //生成二进制文件
```



多文件编译

```
LOCAL_PATH:=$(call my-dir) //最后一个makefile 的路径
include $(CLEAR_VARS)  //请空当前环境变量
LOCAL_MODULE:=test   //编译生成的目标名称
LOCAL_SRC_FILES :=test.c \          # 编译改模块需要的源文件
				test1.c
				
LOCAL_MODULE_PATH:=$(LOCAL_PATH) // 目标文件生成的路径
include $(BUILD_EXECUTABLE) //生成二进制文件  #编译生成的目标文件格式
```



**使用系统提供的函数处理**

```
define all-cpp-files-under
$(sort $(patsubst ./%,%, \
  $(shell cd $(LOCAL_PATH) ; \
          find -L $(1) -name "*$(or $(LOCAL_CPP_EXTENSION),.cpp)" -and -not -name ".*") \
 ))
endef

define all-c-files-under
$(call all-named-files-under,*.c,$(1))
endef

define find-files-in-subdirs
$(sort $(patsubst ./%,%, \
  $(shell cd $(1) ; \
          find -L $(3) -name $(2) -and -not -name ".*") \
 ))
endef


```



```
LOCAL_PATH:=$(call my-dir) //最后一个makefile 的路径
include $(CLEAR_VARS)  //请空当前环境变量
LOCAL_MODULE:=test   //编译生成的目标名称
LOCAL_C_ALL_FILES:=$(call all-c-files-under)
LOCAL_SRC_FILES :=$(LOCAL_SRC_FILES)    #系统函数 所有c文件
				
				
LOCAL_MODULE_PATH:=$(LOCAL_PATH) // 目标文件生成的路径
include $(BUILD_EXECUTABLE) //生成二进制文件  #编译生成的目标文件格式
```

**一个android.mk 生成多个目标文件**



```
LOCAL_PATH:=$(call my-dir) //最后一个makefile 的路径
include $(CLEAR_VARS)  //请空当前环境变量
LOCAL_MODULE:=test1   //编译生成的目标名称
LOCAL_C_ALL_FILES:=$(call all-c-files-under)
LOCAL_SRC_FILES :=$(LOCAL_SRC_FILES)    #系统函数 所有c文件				
LOCAL_MODULE_PATH:=$(LOCAL_PATH) // 目标文件生成的路径
include $(BUILD_EXECUTABLE)/bin //生成二进制文件  #编译生成的目标文件格式




LOCAL_PATH:=$(call my-dir) //最后一个makefile 的路径
include $(CLEAR_VARS)  //请空当前环境变量
LOCAL_MODULE:=test2   //编译生成的目标名称
LOCAL_C_ALL_FILES:=$(call all-c-files-under)
LOCAL_SRC_FILES :=$(LOCAL_SRC_FILES)    #系统函数 所有c文件				
LOCAL_MODULE_PATH:=$(LOCAL_PATH) // 目标文件生成的路径
include $(BUILD_EXECUTABLE)/bin //生成二进制文件  #编译生成的目标文件格式
```



编译动态库 

编译类型修改为：BUILD_SHARED_LIBRARY

```
LOCAL_PATH:=$(call my-dir) //最后一个makefile 的路径
include $(CLEAR_VARS)  //请空当前环境变量
LOCAL_MODULE:=libtest   //编译生成的目标名称
LOCAL_C_ALL_FILES:=$(call all-c-files-under)
LOCAL_SRC_FILES :=$(LOCAL_SRC_FILES)    #系统函数 所有c文件				
LOCAL_MODULE_PATH:=$(LOCAL_PATH) // 目标文件生成的路径
include $(BUILD_SHARED_LIBRARY)/bin  #编译生成的目标文件格式
```

编译静态态库 

编译类型修改为：BUILD_STATIC_LIBRARY



**项目种引入系统的库**

LOCAL_SHARED_LIBRARIES+=libxxxx:

 将系统库文件名添加到Android.mk 中。

**项目引入第三方库**

LOCAL_LDFLAGS:=-L/Path -lxxx

引入第三方头文件

LOCAL_C_INCLUDES:=path

**编译生成apk**

 