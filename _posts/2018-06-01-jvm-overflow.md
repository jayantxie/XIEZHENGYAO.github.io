---
layout: post
title: jvm内存溢出
categories: Java
description: JVM
keywords: JVM
---

- Java堆溢出

```Java
    //反复创建对象
    while(true) list.add(new OOMObject());
```

- Java栈溢出

```Java
    //虚拟机栈和本地方法栈溢出
    //反复调用递归创建函数栈帧结构
    public void stackLeak(){
        stackLeak();
    }
```
```Java
    //反复创建线程，每个线程分配的栈容量越大，可建立的线程数量越少，越容易产生栈溢出
    public void stackLeakByThread(){
        while(true){
            Thread thread = new Thread(new Runnable(){
                @Override
                public void run(){
                    dontStop();
                }
            });
            thread.start();
        }
    }
```

- 方法区和运行时常量池溢出

```Java
    //String.intern()方法将string存储常量池
    while(true){
        list.add(String.valueOf(i++).intern());
    }
```
```Java
    //大量加载class类信息，如代理类，使用CGLib字节码增强，动态语言等
    //使用CGLib使方法区出现内存溢出异常
    public class JavaMethodAreaOOM{
        public static void main(String[] args){
            while(true){
                Enhancer enhancer = new Enhancer();
                enhancer.setSuperclass(OOMObject.class);
                enhancer.setUseCache(false);
                enhancer.setCallback(new MethodInterceptor(){
                    public Object intercept(Object obj, Method method
                    Object[] args, MethodProxy proxy) throws Throwable{
                        return proxy.invokeSuper(obj, args);
                    }
                });
                enhancer.create();
            }
        }
        public class OOMObject{
        }
    }
```

- 本机直接内存溢出

//rt.jar中的类可以使用Unsafe类
//DirectByteBuffer 分配内存出现异常是通过程序计算抛出的，而真正申请分配内存的方法是unsafe.allocateMemory()
```Java
    public class DirectMemoryOOM{
        private static final int _1MB = 1024 * 1024;
        
        public static void main(String[] args) throws Exception {
            Field unsafeField = Unsafe.class.getDeclaredFields()[0];
            unsafeField.setAccessible(true);
            Unsafe unsafe = (Unsafe) unsafeField.get(null);
            while(true){
                unsafe.allocateMemory(_1MB);
            }
        }
    }
```
