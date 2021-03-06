## 根节点

既然是树，肯定有它的根节点，我们就来找一找Android编译系统的根节点。
如果make命令没有指定文件的话默认会在当前目录寻找Makefile这个文件，所以先看看Android源码根目录的那个Makefile文件：


```MakeFile
### DO NOT EDIT THIS FILE ###
include build/core/main.mk
### DO NOT EDIT THIS FILE ###
原来这里只是指路牌，真正的文件是build/core/main.mk。
```


```
在 Makefile 使用 include 关键字可以把别的 Makefile 包含进来,这很像 C 语言的
#include,被包含的文件会原模原样的放在当前文件的包含位置。
```

打开main.mk发现它有上千行代码，并且在其中又包含了很多其他makefile脚本，所以整个文件的内部结构让人感觉杂乱无章，无法入手。这时候我们就不能一行行的去读这个文件，这样很可能会事倍功半。我们需要找出它其中依赖树的根节点，以此为突破口。但往往大型的工程中不止一棵依赖树，这时候如果没有指定依赖树的话make命令会以从上至下第一个target作为默认依赖树的根节点。其实大家非常熟悉的make clean中的clean就是这些依赖树中的一棵。


```
.PHONY: clean
clean:
    @rm -rf $(OUT_DIR)/*
    @echo "Entire build directory removed."
```

    
clean是一个伪目标，伪目标一般没有目标文件。且使用.PHONY显示声明。
从上面clean的COMMANDS知道它的作用是删除(OUTDIR)下的所有目录和文件，而(OUTDIR)下的所有目录和文件，而(OUT_DIR)就是out目录。


OK，那么我们就跟着上面的思路找寻那棵默认依赖树，对照main.mk代码很快就发现了它的默认依赖树根节点：


```
# This is the default target.  It must be the first declared target.
.PHONY: droid
DEFAULT_GOAL := droid
$(DEFAULT_GOAL):
```

从注释中可以看出来droid就是我们找的依赖树的根节点，不过这里只是定义了一下，没有给出真正的规则。

在分析droidcore和dist_files两个先决条件之前先来看一下main.mk的一个文件结构，它除了构建droid等依赖树外，有一大半内容是在做一下这些事情。

- 对编译环境的检测

比如java环境是否符合要求，当前是linux系统还是mac系统。如果这些检测中有任何一项不符合要求，则会终止编译。
- 进行一些必要的前期处理

比如整个项目工程是否要进行清理操作，部分工具的安装等。
- 引用其他Makefile文件

比如引用config.mk,cleanbuild.mk等。
- 设置全局变量

- 各种函数的实现

Android编译系统中定义了很多实用的函数，它们提供了整个编译系统的统一解决方案。比如my-dir这个在Android.mk中出镜率最高的函数，就是用来获得当前的路径。

下表是对Android编译系统中涉及的主要Makefile文件的解释，可供参考。
|Name|Description|
|----|-----------|
|main.mk|整个编译系统的主导文件|
|config.mk|产品配置的主导文件|
|base_rules.mk|编译系统需要遵循的基础规则定义|
|build_id.mk|版本id号的定义|
|cleanbuild.mk|clean操作的定义|
|clear_vars.mk|LOCAL开头的相关系统变量|
|definitions.mk|提供了大量实用的函数定义|
|envsetup.mk|配置编译时的环境变量|
|executable.mk|负责BUILD_EXECUTABLE的具体实现|
|host_executable.mk|负责BUILD_HOST_EXECUTABLE的具体实现|
|host_static_library.mk|负责BUILD_HOST_STATIC_LIBRARY的具体实现，其他类型的BUILD_XX这里不再赘述|
|product_config.mk|产品级别的配置，属于config的一部分|
|version_defaults.mk|负责生成版本信息，如版本号BUILD_NUMBER := eng.(USER).(USER).(shell date +%Y%m%d.%H%M%S)|


## droidcore节点
droid规则的定义如下：


```
# Build files and then package it into the rom formats
.PHONY: droidcore
droidcore: files \
    systemimage \
    $(INSTALLED_BOOTIMAGE_TARGET) \
    $(INSTALLED_RECOVERYIMAGE_TARGET) \
    $(INSTALLED_USERDATAIMAGE_TARGET) \
    $(INSTALLED_CACHEIMAGE_TARGET) \
    $(INSTALLED_VENDORIMAGE_TARGET) \
    $(INSTALLED_FILES_FILE)
```

可以看出来droidcore有如下几个先决条件，

|Prerequisite|Description|
|------------|-----------|
|files|代表其所依赖的先决条件的集合，没有实际意义|
|systemimage|将生成system.img|
|INSTALLED_BOOTIMAGE_TARGET|将生成boot.img|
|INSTALLED_RECOVERYIMAGE_TARGET|将生成recovery.img|
|INSTALLED_USERDATAIMAGE_TARGET|将生成userdata.img|
|INSTALLED_CACHEIMAGE_TARGET|将生成cache.img|
|INSTALLED_VENDORIMAGE_TARGET|将生成vendor.img|
|INSTALLED_FILES_FILE|将生成install-files.txt,用于记录当前系统中预装的程序、库等模块|


分析一下Android.mk文件在整个编译系统中的地位和作用。

一棵大树的繁茂和枝叶的多少息息相关。一方面只有枝干足够茁壮才能托起枝叶，另一方面枝叶的光合作用也能促进枝干的生长。那么在Android编译系统中，droid就是这棵树中强有里的枝干，而Android.mk则是一片片的叶子，纵观整个Android平台Android.mk的数量在一千个以上。那么如此多的makefile文件又是在何时被整合进整个编译系统的呢？其实答案还是在main.mk中。

ONE_SHOT_MAKEFILE变量和编译选项有关，当选择默认make命令进行整编的时候ONE_SHOT_MAKEFILE值为空，这是就会走下面这个分支


```
subdir_makefiles := \
    $(shell build/tools/findleaves.py $(FIND_LEAVES_EXCLUDES) $(subdirs) Android.mk)

$(foreach mk, $(subdir_makefiles), $(info including $(mk) ...)$(eval include $(mk)))
```

其中就是通过findleaves.py这个脚本来查找所有的Android.mk文件但可能并不是所有的Android.mk都会被包好进来比如.repo .git下的就会被排除在外。这些排除选项由FIND_LEAVES_EXCLUDES决定。所有被包含进来的Android.mk的路径都会被追加到subdir_makefiles变量，接着通过一个foreach函数将所有的Android.mk文件都include进来。其中(infoincluding(infoincluding(mk) ...)负责打印这些文件信息，如下

OK，到这里所有的Android.mk文件都被包含进来了，等整个大树被构建完成后make会从依赖树最外层的叶子开始往上执行所有的COMMANDS。


接下来我们选取Settings模块作为例子，详细的解释一下Android.mk的编写规则和一些注意事项。Settings模块的Android.mk内容如下


```
#LOCAL_PATH表示当前目录的地址，一般位于include $(CLEAR_VARS)之前
LOCAL_PATH:= $(call my-dir)

#CLEAR_VARS对应的是clean_vars.mk，用于清除除了LOCAL_PATH以外的所有LOCAL_打头的变量
include $(CLEAR_VARS)

#重定向java库文件
LOCAL_JAVA_LIBRARIES := bouncycastle conscrypt telephony-common ims-common

#重定向java静态库文件
LOCAL_STATIC_JAVA_LIBRARIES := android-support-v4 android-support-v13 jsr305

#模块tag为optional，表示不管是选择了什么模式都会编译该模块
LOCAL_MODULE_TAGS := optional

#重定向本地源码
LOCAL_SRC_FILES := \
        $(call all-java-files-under, src) \
        src/com/android/settings/EventLogTags.logtags

#重定向本地资源文件
LOCAL_RESOURCE_DIR := $(LOCAL_PATH)/res

#模块名
LOCAL_PACKAGE_NAME := Settings

#模块证书签名
LOCAL_CERTIFICATE := platform

#是否是特权文件
LOCAL_PRIVILEGED_MODULE := true

#使用代码混淆
LOCAL_PROGUARD_FLAG_FILES := proguard.flags

#判断是否进行增量编译
ifneq ($(INCREMENTAL_BUILDS),)
    LOCAL_PROGUARD_ENABLED := disabled
    LOCAL_JACK_ENABLED := incremental
endif

#include三个makefile文件，进项相关变量赋值
include frameworks/opt/setupwizard/navigationbar/common.mk
include frameworks/opt/setupwizard/library/common.mk
include frameworks/base/packages/SettingsLib/common.mk

#开始编译Settings模块，对应package.mk文件。感兴趣的可以进一步研究apk是怎么被编译出来的,里面还是很复杂的
include $(BUILD_PACKAGE)


# 如果使用的是mm或mmm命令单编Settings模块的话会额外include test目录下的Android.mk,用于编译测试模块。
ifeq (,$(ONE_SHOT_MAKEFILE))
include $(call all-makefiles-under,$(LOCAL_PATH))
endif
```

通过上述对Android.mk文件的分析，我们可以看到需要编译一个模块要做的工作还是很少的，只要指定几个变量就可以了，这也得益与google的用心良苦，它把所有的公共操作都抽取了出来，做好了各种模板，如BUILD_PACKAGE等，我们要做的只是调用适当的模板就行了。

下面介绍一些Android.mk中常用的变量，以供读者参考。


变量名	| 说明
---|---
LOCAL_PATH |	用于确定源码所在的目录，一般把它放在CLEAR_VARS变量引用的前面，因为它不会背清除，每个Android.mk只需要定义一次就行
CLAER_VARS |	清空很多LOCAL_开头的变量（LOCAL_PATH除外）。因为所有的Makefile都是在一个编译环境下执行，因此变量的定义理论上都是全局的，每个模块开始编译前进行清理工作是必不可少的
LOCAL_MODULE|	模块名，需要保证唯一存在且中间不能又空格
LOCAL_MODULE_PATH |	模块的输出路径
LOCAL_SRC_FILES |	模块编译所涉及的源文件。如果是java程序，可以考虑调用all-java-files-under添加java代码。因为有LOCAL_PATH，所以这里只需要给出文件名即可，如src
LOCAL_CC |	用于指定C编译器
LOCAL_CXX |	用于指定C++编译器
LOCAL_CPP_EXTENSION |	用于指定特殊的C++文件后缀名
LOCAL_CFLAGS |	C语言编译时的额外选项
LOCAL_CXXFLOAGS |	C++编译时的额外选项
LOCAL_C_INCLUDES |	编译C和C++时需要的额外头文件
LOCAL_STATIC_LIBRARIES |	编译所需的静态库列表
LOCAL_SHARED_LIBRARIES |	编译所需的共享库列表
LOCAL_JAVA_LIBRARIES |	编译时所需的JAVA类库
LOCAL_LDLIBS |	编译时所需的链接选项
LOCAL_COPY_HEADERS |	安装应用程序时需要复制的头文件列表，需要和LOCAL_COPY_HEADERS_TO变量配合使用
LOCAL_COPY_HEADERS_TO |	上述头文件列表的复制目的地
BUILD_XX_XX |	各种形式的编译模板，如生成静态、动态库文件，可执行文件，文档等



在Android编译系统的学习中，我们先从最基础的makefile语法规则入手，导出了依赖树的概念，然后按照依赖树的结构逐步梳理出一个完整的Android版本编译所设计的几个重要节点。Android编译系统是非常庞大的，不过经过这次的学习希望大家能够对它的结构和基本原理有一个初步的认识。那么接下来的各种编译细节也能通过代码的研读和分析变得明朗起来。


## 参考链接

[Android编译系统入门（一）](https://www.cnblogs.com/zqlxtt/p/5016654.html)

[Android编译系统入门（二）](https://www.cnblogs.com/zqlxtt/p/5018956.html)

[深入理解：Android 编译系统](https://blog.csdn.net/huangyabin001/article/details/36383031/)