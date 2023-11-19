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

​	