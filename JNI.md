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
	+ 静态注册是在第一次调用时会去so库查找对应的JNI函数，如果存在就会保存函数指针
	

## 2.androidstudio移植c++工程

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



+ 复制cpp目录

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



## 3. 动态注册

静态注册在首次调用函数时会在对应的so库中查找对应的JNI函数，如果找到就保存函数指针，下次调用直接使用函数指针，当然这份工作由虚拟机完成。如果一开始就告诉函数指针呢？这就是动态注册与静态注册的主要区别。

### 1.java层调用JNI层

#### 1.使用JNINativeMethod结构体指定函数指针

```java
typedef struct {
      const char* name;				//java中的函数名
      const char* signature;		//签名信息：由参数类型和返回值组成，因为c++是可以函数重载，所以需要指定具体的参数返回值类型才能找到指定的方法
      void* fnPtr;				   //JNI层对应的函数指针
  } JNINativeMethod;
```

#### 2.在JNI_OnLoad函数中通过JINEnv.RegisterNatives注册JNINativeMethod结构体数组

```c++
/*java层在调用System.loadLibrary("")加载so库时，jni层回去查找调用JNI_OnLoad函数*/

jint JNI_OnLoad(JavaVM* vm, void* /* reserved */)
  {
      JNIEnv* env = NULL;
      jint result = -1;
  		
    
    //vm虚拟机的代表，每个进程有一个虚拟机，通过虚拟机获取JNIENV，JNIEnv是线程私有的
      if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
          ALOGE("ERROR: GetEnv failed\n");
          goto bail;
      }
    
    
    jclass clazz;
    //通过jnienv获取java的class对象
    clazz = (*env) -> FindClass(env,className);
    
    //注册映射指针
    (*env)->RegisterNatives(env,clazz,gMethods,numMethods)
    
}
```

### 2. JNIEnv

JNIEnv是一个与线程相关的代表JNI环境的结构体，每个线程都有子的JNIEnv

通过JNIEnv能做：

+ 调用java的函数
+ 操作jobject对象很多事情

JNIEnv和JavaVM的关系：

+ 调用javaVM的AttachCurrentThread函数获取当前线程的JNIEnv
+ 在当前线程推出前调用DetchCurrentThread函数释放资源



### 3.JNI调用java层代码

#### 1.通过JNIEnv提供的接口获取java的字段ID、方法ID

```c++

FindClass(env,className);
GetFieldID(jclass, "字段名", "签名信息");
GetMethodID(jclass,"方法名","签名信息");
```

#### 2. 通过JNIEnv提供的接口，传入指定字段ID、方法ID，操作Java层代码

```c++

Call<type>Method();  //type是返回类型
CallStatic<type>Method(); //type是返回类型

Set<Type>Field(); //Type是字段类型
Get<Type>Field(); //Type是字段类型
```



### 4. 签名类型

```java
javap -s -p xxx   // -s:输出成员签名信息   -p:打印成员签名信息
```

### 5.垃圾回收

JNI提供了三种类型的引用（因为会传进来jobject）

+ 本地引用：当函数返回后，可能就会被回收
+ 全局引用：需要主动释放
+ 弱引用：随时都会被回收，使用时，使用JNIEnv的IsSameObject判断是否回收再使用

### 6.jstring类型
