---
title: 计算java对象大小的两种方法
date: 2017/09/13 19:22:11
toc: true
list_number: false
categories:
- Java
tags:
- Java
---


**前言：** 主要介绍下精确计算java对象大小的两种方法：Instrumentation和Unsafe方式。
关于`Instrumentation`的使用和原理Google一大推，这里说说如何在IDEA中，一步一步的操作，计算对象的大小！
关于Unsafe方式，这个使用方法很简单，直接复制代码使用即可！可以在运行时动态计算，推荐这种方式。


# 一. Instrumentation计算对象大小

# 1. 步骤
1. 使用IDEA中原有的和新建的项目都可以，这里我直接在已有项目中新建了一个package。
如：新建一个`sizeof`，绝对路径为：`/home/hzz/github/helper/common/src/main/java/org/helper/common/objsize`
2. 新建一个类`ObjectSizeOf`，并写入代码，代码见源码部分。
这里包名就是：`package org.helper.common.objsize;`
3. 打开IDEA的`Terminal`后其他终端，`cd到objsize目录下`。编译`ObjectSizeOf.java`：`javac ObjectSizeOf.java`
此时，会在`objsize`目录下生成一个`ObjectSizeOf.class`文件。
包括后面的操作，生成的文件都在此目录下！
4. 打包成jar包：`jar -cvf agent.jar ObjectSizeOf.class`
此时，查看jar包中的内容：`jar -tf agent.jar`或到jar包目录下双击点开jar包，会有`META-INF/MANIFEST.MF`和`ObjectSizeOf.class`两个文件！
5. 修改`MANIFEST.MF`
我们不能直接修改`agent.jar`包中的文件内容，所以我们可以修改后重新打包。
`jar -xvf agent.jar`或`jar -xf agent.jar META-INF/MANIFEST.MF` 解压，获得`MANIFEST.MF`文件，并在该文件中加入一行：`Premain-Class: org.helper.common.objsize.ObjectSizeOf`
注意冒号后面的空格。然后重新打包：`jar -cvfm agent.jar ./META-INF/MANIFEST.MF ObjectSizeOf.class`，并检查文件都打包进jar包了。
OK，`agent.jar`包制作完成，位置：`/home/hzz/github/helper/common/src/main/java/org/helper/common/objsize/agent.jar`
6. 新建一个`MainSizeOf`类，源码style见源码部分。
7. 运行一遍`MainSizeOf`类中`main`方法，生成一个执行的配置文件，然后点击IDEA中的`Edit Configurations ...`编辑`MainSizeOf`配置文件，在`VM options`中加入以下内容：
`-javaagent:/home/hzz/github/helper/common/src/main/java/org/helper/common/objsize/agent.jar`
8. 再次运行`MainSizeOf`类，得出结果。
这里我在命令行中`java -javaagent:agent.jar MainSizeOf`的方式，多次尝试未成功，主要可能是java运行环境引入问题，不纠结此，尝试借助IDEA能完成。
如果按照上述步骤配置，仍然有问题，比如：
`Exception in thread "main" java.lang.ClassNotFoundException: sizeof.ObjectSizeOf`
`Error occurred during initialization of VM`，`agent library failed to init: instrument`
等，都可能是java运行环境问题，详细检查！

# 2. 源码
#### ObjectSizeOf
```
package org.helper.common.objsize;

import java.lang.instrument.Instrumentation;
import java.lang.reflect.Array;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.IdentityHashMap;
import java.util.Map;
import java.util.Stack;

public class ObjectSizeOf {
    private static Instrumentation inst;

    public static void premain(String agentArgs, Instrumentation instP) {
        inst = instP;
    }

    /**
     * 简单计算对象的大小，不包含对象引用的对象大小
     */
    public static long sizeOf(Object o) {
        if (inst == null) {
            throw new IllegalStateException("Can not access instrumentation environment.");
        }
        return inst.getObjectSize(o);
    }

    /**
     * 计算对象的总大小,包含对象引用的对象的大小
     */
    public static long fullSizeOf(Object obj) {
        Map<Object, Object> visited = new IdentityHashMap<>();
        Stack<Object> stack = new Stack<>();
        long result = internalSizeOf(obj, stack, visited);
        //通过栈进行遍历
        while (!stack.isEmpty()) {
            result += internalSizeOf(stack.pop(), stack, visited);
        }
        visited.clear();
        return result;
    }

    /**
     * 判定哪些是需要跳过的
     */
    private static boolean skipObject(Object obj, Map<Object, Object> visited) {
        if (obj instanceof String) {
            if (obj == ((String) obj).intern()) {
                return true;
            }
        }
        return (obj == null) || visited.containsKey(obj);
    }

    private static long internalSizeOf(Object obj, Stack<Object> stack, Map<Object, Object> visited) {
        //跳过常量池对象、跳过已经访问过的对象
        if (skipObject(obj, visited)) {
            return 0;
        }
        //将当前对象放入栈中
        visited.put(obj, null);
        long result = 0;
        result += sizeOf(obj);
        Class<?> clazz = obj.getClass();
        //如果是数组
        if (clazz.isArray()) {
            // skip primitive type array
            if (clazz.getName().length() != 2) {
                int length = Array.getLength(obj);
                for (int i = 0; i < length; i++) {
                    stack.add(Array.get(obj, i));
                }
            }
            return result;
        }
        //计算非数组对象的大小
        return getNodeSize(clazz, result, obj, stack);
    }

    /**
     * 获取非数组对象自身的大小，并且可以向父类进行向上搜索
     */
    private static long getNodeSize(Class<?> clazz, long result, Object obj, Stack<Object> stack) {
        while (clazz != null) {
            Field[] fields = clazz.getDeclaredFields();
            for (Field field : fields) {
                //这里抛开静态属性,抛开基本关键字（因为基本关键字在调用java默认提供的方法就已经计算过了）
                if (!Modifier.isStatic(field.getModifiers()) && !field.getType().isPrimitive()) {
                    field.setAccessible(true);
                    try {
                        Object objectToAdd = field.get(obj);
                        if (objectToAdd != null) {
                            //将对象放入栈中，一遍弹出后继续检索
                            stack.add(objectToAdd);
                        }
                    } catch (IllegalAccessException ex) {
                        assert false;
                    }
                }
            }
            //找父类class，直到没有父类
            clazz = clazz.getSuperclass();
        }
        return result;
    }
}
```

#### MainSizeOf
```
package org.helper.common.objsize;
public class MainSizeOf {
    public static void main(String[] args) {
        long objSize = ObjectSizeOf.fullSizeOf(new Object());
        long intObjSize = ObjectSizeOf.fullSizeOf(new Integer(1));
        long strObjSize = ObjectSizeOf.fullSizeOf(new String());
        long strObjSize2 = ObjectSizeOf.fullSizeOf(new String("a"));
        long charSize = ObjectSizeOf.fullSizeOf(new char[4]);
        String format = String.format("full size:obj:%s intObjSize:%s,strObjSize:%s:%s,charSize:%s",
                objSize, intObjSize, strObjSize, strObjSize2, charSize);
        System.out.println(format);

        //仅计算当前对象大小,不递归到引用对象,做参考
        objSize = ObjectSizeOf.sizeOf(new Object());
        intObjSize = ObjectSizeOf.sizeOf(new Integer(1));
        strObjSize = ObjectSizeOf.sizeOf(new String());
        strObjSize2 = ObjectSizeOf.sizeOf(new String("a"));
        charSize = ObjectSizeOf.sizeOf(new char[0]);
        format = String.format("size:obj:%s intObjSize:%s,strObjSize:%s:%s,charSize:%s",
                objSize, intObjSize, strObjSize, strObjSize2, charSize);
        System.out.println(format);

        long a = ObjectSizeOf.fullSizeOf(new A());
        long b = ObjectSizeOf.fullSizeOf(new B());
        long c = ObjectSizeOf.fullSizeOf(new C());
        format = String.format("A:%s B:%s,C:%s:", a, b, c);
        System.out.println(format);

    }

    public static class A { byte b;}

    public static class B extends A { byte b;}

    public static class C extends B { byte b;}
}

```
**附结果：**

环境：`jdk1_8 64bit`,`8G RAM Mark Word:8 byte`; `Class:4 byte`; `数组头部:4 byte` 
`new Object()`: `8+4+字节对其 = 16`
`new Integer(1)`: `8+4+4(int变量) = 16 `
`new String()`:`8+4+4(int hash变量) + 4(数组引用) + 字节对其 + {8+4+4(数组头部)}(char数组对象)+对其=40 `
`new String("a")`:`24 + {8+4+4(数组头部)}(char数组对象)+2(java一个字符占2字节)+对其=48 `
`new char[0]`: `8(Mark word)+4(class指针)+4(数组头)=16 `
`new char[4]`: `8+4+4 + 8(数组空间,每个cha占2byte)=24 `
`new char[5]`: `8+4+4 + 10(数组空间,每个cha占2byte) + 对其=32 `
`new char[8]`: `8+4+4 + 16(数组空间,每个cha占2byte)=32 `
`A:16 B:24,C:24`：得出结论:`C={8+4+8字节对其} + 1+1+1+对其=24`;(与很多有出入,得考究)


# 二. Unsafe方式计算对象大小

#### 测试类
```
public class MainObjSize {
    public static void main(String[] args) throws IllegalAccessException {
        final ClassIntrospector ci = new ClassIntrospector();
        System.out.println("当前系统引用大小:" + ClassIntrospector.getObjectRefSize());

        ClassIntrospector.ObjectInfo res = ci.introspect(new Integer(1));
        System.out.println("new Integer(1):" + res.getDeepSize());

        res = ci.introspect(new String());
        System.out.println("new String():" + res.getDeepSize());

        res = ci.introspect(new String("a"));
        System.out.println("new String(\"a\"):" + res.getDeepSize());

        res = ci.introspect(new char[0]);
        System.out.println("new char[0]:" + res.getDeepSize());

        res = ci.introspect(new char[4]);
        System.out.println("new char[4]:" + res.getDeepSize());

        res = ci.introspect(new char[8]);
        System.out.println("new char[8]:" + res.getDeepSize());

        res = ci.introspect(new A());
        System.out.println("new A():" + res.getDeepSize());

        res = ci.introspect(new B());
        System.out.println("new B():" + res.getDeepSize());

        res = ci.introspect(new C());
        System.out.println("new C():" + res.getDeepSize());

        res = ci.introspect(new ObjectA());
        System.out.println("new ObjectA():" + res.getDeepSize());
    }


    public static class A {
        byte b;
    }

    public static class B extends MainSizeOf.A {
        byte b;
    }

    public static class C extends MainSizeOf.B {
        byte b;
    }

    private static class ObjectA {
        String str;  // 4 引用字节
        int i1; // 4
        byte b1; // 1
        byte b2; // 1
        int i2;  // 4
        ObjectB obj; //4 引用字节
        byte b3;  // 1
    }

    private static class ObjectB {

    }

}
```
**结果：**
```
当前系统引用大小:4
new Integer(1):16
new String():40
new String("a"):48
new char[0]:16
new char[4]:24
new char[8]:32
new A():16
new B():24
new C():24
new ObjectA():32
```


#### 实现类 ClassIntrospector
```
import java.lang.reflect.Array;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Comparator;
import java.util.HashMap;
import java.util.IdentityHashMap;
import java.util.List;
import java.util.Map;

import sun.misc.Unsafe;

/**
 * 实现计算对象大小的类
 */
public class ClassIntrospector {

    private static final Unsafe unsafe;

    /**
     * 引用大小
     */
    private static final int objectRefSize;

    public static int getObjectRefSize() {
        return objectRefSize;
    }

    /**
     * 获取unsafe对象
     */
    static {
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            unsafe = (Unsafe) field.get(null);

            objectRefSize = unsafe.arrayIndexScale(Object[].class);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * Sizes of all primitive values
     */
    private static final Map<Class, Integer> primitiveSizes;

    static {
        primitiveSizes = new HashMap<>(10);
        primitiveSizes.put(byte.class, 1);
        primitiveSizes.put(char.class, 2);
        primitiveSizes.put(int.class, 4);
        primitiveSizes.put(long.class, 8);
        primitiveSizes.put(float.class, 4);
        primitiveSizes.put(double.class, 8);
        primitiveSizes.put(boolean.class, 1);
    }

    public ObjectInfo introspect(final Object obj) throws IllegalAccessException {
        try {
            return introspect(obj, null);
        } finally {
            // 获得对象大小后，清空缓存，以便对象重用
            visited.clear();
        }
    }

    private IdentityHashMap<Object, Boolean> visited = new IdentityHashMap<>(100);

    private ObjectInfo introspect(final Object obj, final Field fld) throws IllegalAccessException {
        boolean isPrimitive = fld != null && fld.getType().isPrimitive();
        // will be set to true if we have already
        boolean isRecursive = false;
        // seen this object
        if (!isPrimitive) {
            if (visited.containsKey(obj)) {
                isRecursive = true;
            }
            visited.put(obj, true);
        }

        final Class type = (fld == null || (obj != null && !isPrimitive)) ? obj.getClass() : fld.getType();
        int arraySize = 0;
        int baseOffset = 0;
        int indexScale = 0;
        if (type.isArray() && obj != null) {
            baseOffset = unsafe.arrayBaseOffset(type);
            indexScale = unsafe.arrayIndexScale(type);
            arraySize = baseOffset + indexScale * Array.getLength(obj);
        }

        final ObjectInfo root;
        if (fld == null) {
            root = new ObjectInfo("", type.getCanonicalName(), getContents(obj,
                    type), 0, getShallowSize(type), arraySize, baseOffset,
                    indexScale);
        } else {
            final int offset = (int) unsafe.objectFieldOffset(fld);
            root = new ObjectInfo(fld.getName(), type.getCanonicalName(),
                    getContents(obj, type), offset, getShallowSize(type),
                    arraySize, baseOffset, indexScale);
        }

        if (!isRecursive && obj != null) {
            if (isObjectArray(type)) {
                // introspect object arrays
                final Object[] ar = (Object[]) obj;
                for (final Object item : ar) {
                    if (item != null) {
                        root.addChild(introspect(item, null));
                    }
                }
            } else {
                for (final Field field : getAllFields(type)) {
                    if ((field.getModifiers() & Modifier.STATIC) != 0) {
                        continue;
                    }
                    field.setAccessible(true);
                    root.addChild(introspect(field.get(obj), field));
                }
            }
        }

        // sort by offset
        root.sort();
        return root;
    }

    /**
     * get all fields for this class, including all superclasses fields
     */
    private static List<Field> getAllFields(final Class type) {
        if (type.isPrimitive()) {
            return Collections.emptyList();
        }
        Class cur = type;
        final List<Field> res = new ArrayList<Field>(10);
        while (true) {
            Collections.addAll(res, cur.getDeclaredFields());
            if (cur == Object.class) {
                break;
            }
            cur = cur.getSuperclass();
        }
        return res;
    }

    /**
     * check if it is an array of objects. I suspect there must be a more
     * API-friendly way to make this check.
     */
    private static boolean isObjectArray(final Class type) {
        if (!type.isArray()) {
            return false;
        }
        return type != byte[].class && type != boolean[].class
                && type != char[].class && type != short[].class
                && type != int[].class && type != long[].class
                && type != float[].class && type != double[].class;
    }

    private static String getContents(final Object val, final Class type) {
        //advanced toString logic
        if (val == null) {
            return "null";
        }
        if (type.isArray()) {
            if (type == byte[].class) {
                return Arrays.toString((byte[]) val);
            } else if (type == boolean[].class) {
                return Arrays.toString((boolean[]) val);
            } else if (type == char[].class) {
                return Arrays.toString((char[]) val);
            } else if (type == short[].class) {
                return Arrays.toString((short[]) val);
            } else if (type == int[].class) {
                return Arrays.toString((int[]) val);
            } else if (type == long[].class) {
                return Arrays.toString((long[]) val);
            } else if (type == float[].class) {
                return Arrays.toString((float[]) val);
            } else if (type == double[].class) {
                return Arrays.toString((double[]) val);
            } else {
                return Arrays.toString((Object[]) val);
            }
        }
        return val.toString();
    }

    /**
     * obtain a shallow size of a field of given class (primitive or object reference size)
     */
    private static int getShallowSize(final Class type) {
        if (type.isPrimitive()) {
            final Integer res = primitiveSizes.get(type);
            return res != null ? res : 0;
        } else {
            return objectRefSize;
        }
    }

    public static class ObjectInfo {

        public final String name;

        public final String type;

        public final String contents;

        public final int offset;

        public final int length;

        public final int arrayBase;

        public final int arrayElementSize;

        public final int arraySize;

        public final List<ObjectInfo> children;

        public ObjectInfo(String name, String type, String contents, int offset, int length, int arraySize,
                          int arrayBase, int arrayElementSize) {
            this.name = name;
            this.type = type;
            this.contents = contents;
            this.offset = offset;
            this.length = length;
            this.arraySize = arraySize;
            this.arrayBase = arrayBase;
            this.arrayElementSize = arrayElementSize;
            children = new ArrayList<ObjectInfo>(1);
        }

        public void addChild(final ObjectInfo info) {
            if (info != null) {
                children.add(info);
            }
        }


        public long getDeepSize() {
            return addPaddingSize(arraySize + getUnderlyingSize(arraySize != 0));
        }

        long size = 0;

        private long getUnderlyingSize(final boolean isArray) {
            //long size = 0;
            for (final ObjectInfo child : children) {
                size += child.arraySize + child.getUnderlyingSize(child.arraySize != 0);
            }
            if (!isArray && !children.isEmpty()) {
                int tempSize = children.get(children.size() - 1).offset + children.get(children.size() - 1).length;
                size += addPaddingSize(tempSize);
            }

            return size;
        }

        private static final class OffsetComparator implements Comparator<ObjectInfo> {
            @Override
            public int compare(final ObjectInfo o1, final ObjectInfo o2) {
                //safe because offsets are small non-negative numbers
                return o1.offset - o2.offset;
            }
        }

        public void sort() {
            //sort all children by their offset
            Collections.sort(children, new OffsetComparator());
        }

        @Override
        public String toString() {
            final StringBuilder sb = new StringBuilder();
            toStringHelper(sb, 0);
            return sb.toString();
        }

        private void toStringHelper(final StringBuilder sb, final int depth) {
            depth(sb, depth).append("name=").append(name).append(", type=").append(type)
                    .append(", contents=").append(contents).append(", offset=").append(offset)
                    .append(", length=").append(length);
            if (arraySize > 0) {
                sb.append(", arrayBase=").append(arrayBase);
                sb.append(", arrayElemSize=").append(arrayElementSize);
                sb.append(", arraySize=").append(arraySize);
            }
            for (final ObjectInfo child : children) {
                sb.append('\n');
                child.toStringHelper(sb, depth + 1);
            }
        }

        private StringBuilder depth(final StringBuilder sb, final int depth) {
            for (int i = 0; i < depth; ++i) {
                sb.append("\t");
            }
            return sb;
        }

        private long addPaddingSize(long size) {
            if (size % 8 != 0) {
                return (size / 8 + 1) * 8;
            }
            return size;
        }

    }
}
```
