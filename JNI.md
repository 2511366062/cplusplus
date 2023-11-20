# java native Interface

## 1.静态注册

静态注册主要用于NDK开发

+ 创建java类

```java
package com.tinnove.mylibrary;

public class MediaRecord {

    static {
        //对于java世界来说，加载对应的jni库，声明方法就行
        System.loadLibrary("media_jni");
        native_init();

    }

    private static native final void native_init();
    public native void start() throws IllegalStateException;
}
```

+ 生成MediaRecord.h

```c++
//进入java文件夹路径执行以下两条命令
javac com/tinnove/mylibrary/MediaRecord.java //生成class文件
javah com.tinnove.mylibrary.MediaRecord    //生成头文件
```

```c

#include <jni.h>


#ifndef _Included_com_tinnove_mylibrary_MediaRecord
#define _Included_com_tinnove_mylibrary_MediaRecord
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_tinnove_mylibrary_MediaRecord
 * Method:    native_init
 * Signature: ()V
 */
JNIEXPORT void JNICALL Java_com_tinnove_mylibrary_MediaRecord_native_1init //因为java中出现了_,而java的'.'替换成了'_'
  (JNIEnv *, jclass);

/*
 * Class:     com_tinnove_mylibrary_MediaRecord
 * Method:    start
 * Signature: ()V
 */
JNIEXPORT void JNICALL Java_com_tinnove_mylibrary_MediaRecord_start
  (JNIEnv *, jobject);

#ifdef __cplusplus
}
#endif
#endif
```

+ 说明
  + JNIEnv *指针：java世界的代表，通过它访问java世界代码
  + 初次调用会建立连接保存函数指针，这样会影响效率

	## 2.动态注册

### 1. 移植c++工程

+ 在模块的build.gradle下添加

```
externalNativeBuild {
        cmake {
            path file('src/main/cpp/CMakeLists.txt')
            version '3.18.1'
        }
    }
    
    
 //c++标准库
 externalNativeBuild{
            cmake{
                cppFlags "-std=c++11"
            }
        }
    }
```



+ 赋值cpp目录

### 2.cmake基础语法

```cmake

# 定义变量并赋值
set(name lixiaokang) 

# 打印编译日志
message(${name})
message(${CMAKE_CURRENT_LIST_FILE})
message(${CMAKE_CURRENT_LIST_DIR})

#条件语句
if(TRUE)
    message(name)
endif()


# 将自定.cpp文件加载为动态库
add_library(
        shit_lib #库名
        SHARED   # SHARED动态库、STATIC静态库
        Twss.cpp # 文件

)


# 查找库
find_library( 
        log-lib # 将查找到的库名给赋值给该变量
        
        log # 查找的库
        
        )
        
# 链接库        
target_link_libraries( 
        test_lib

        shit_lib
        
        ${log-lib})
```

