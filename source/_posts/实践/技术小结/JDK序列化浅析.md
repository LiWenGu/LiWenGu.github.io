---
title: JDK序列化浅析
date: 2020-01-08 00:10:00
updated: 2020-01-08 00:10:00
tags:
  - serializable
categories: 
  - 实践
  - 技术小结
comments: true
permalink: tech/jdk_seriazable.html    
---

# 0. 背景

一直在用 RPC，但是对非常重要的序列化原理一直模糊不懂，尤其是其中的 serialVersionUID 也模糊不清，虽然 Dubbo 已经帮我们使用了经过优化过的 hession2，但是还是想一探究竟，为什么原生的 JDK 序列化就不行呢？原生 JDK 序列化到底是怎么样的。

主要参考《分布式Java应用》的**4.3序列化/反序列化**章节

# 1. 序列化之 writeObject

## 1.1 获取类信息

序列化时，需要获取到当前序列化对象的类信息，获取类信息主要是调用的 `ObjectStreamClass.lookup(cl, true)`，ObjectStreamClass 会通过 ConcurrentMap 缓存类信息优化序列化速度：`static final ConcurrentMap<WeakClassKey,Reference<?>> localDescs`，其中 key 使用了弱引用（在 ThreadLocal 也有使用弱引用）。

```java
private void writeObject0(Object obj, boolean unshared)
        throws IOException
{
    // ...
        for (;;) {
            // REMIND: skip this check for strings/arrays?
            Class<?> repCl;
            // 获取类信息
            desc = ObjectStreamClass.lookup(cl, true);
            if (!desc.hasWriteReplaceMethod() ||
                    (obj = desc.invokeWriteReplace(obj)) == null ||
                    (repCl = obj.getClass()) == cl)
            {
                break;
            }
            cl = repCl;
        }
        if (enableReplace) {
            Object rep = replaceObject(obj);
            if (rep != obj && rep != null) {
                cl = rep.getClass();
                desc = ObjectStreamClass.lookup(cl, true);
            }
            obj = rep;
        }

        // if object replaced, run through original checks a second time
        if (obj != orig) {
            subs.assign(orig, obj);
            if (obj == null) {
                writeNull();
                return;
            } else if (!unshared && (h = handles.lookup(obj)) != -1) {
                writeHandle(h);
                return;
            } else if (obj instanceof Class) {
                writeClass((Class) obj, unshared);
                return;
            } else if (obj instanceof ObjectStreamClass) {
                writeClassDesc((ObjectStreamClass) obj, unshared);
                return;
            }
        }

        // remaining cases
        if (obj instanceof String) {
            writeString((String) obj, unshared);
        } else if (cl.isArray()) {
            writeArray(obj, desc, unshared);
        } else if (obj instanceof Enum) {
            writeEnum((Enum<?>) obj, desc, unshared);
        } else if (obj instanceof Serializable) {
            writeOrdinaryObject(obj, desc, unshared);
        } else {
            if (extendedDebugInfo) {
                throw new NotSerializableException(
                        cl.getName() + "\n" + debugInfoStack.toString());
            } else {
                throw new NotSerializableException(cl.getName());
            }
        }
    } finally {
        depth--;
        bout.setBlockDataMode(oldMode);
    }
}
```

```java
private static class Caches {
    /** cache mapping local classes -> descriptors */
    static final ConcurrentMap<ObjectStreamClass.WeakClassKey,Reference<?>> localDescs =
            new ConcurrentHashMap<>();

    /** cache mapping field group/local desc pairs -> field reflectors */
    static final ConcurrentMap<ObjectStreamClass.FieldReflectorKey,Reference<?>> reflectors =
            new ConcurrentHashMap<>();

    /** queue for WeakReferences to local classes */
    private static final ReferenceQueue<Class<?>> localDescsQueue =
            new ReferenceQueue<>();
    /** queue for WeakReferences to field reflectors keys */
    private static final ReferenceQueue<Class<?>> reflectorsQueue =
            new ReferenceQueue<>();
}

static ObjectStreamClass lookup(Class<?> cl, boolean all) {
    if (!(all || Serializable.class.isAssignableFrom(cl))) {
        return null;
    }
    processQueue(ObjectStreamClass.Caches.localDescsQueue, ObjectStreamClass.Caches.localDescs);
    ObjectStreamClass.WeakClassKey key = new ObjectStreamClass.WeakClassKey(cl, ObjectStreamClass.Caches.localDescsQueue);
    Reference<?> ref = ObjectStreamClass.Caches.localDescs.get(key);
    Object entry = null;
    if (ref != null) {
        entry = ref.get();
    }
    ObjectStreamClass.EntryFuture future = null;
    if (entry == null) {
        ObjectStreamClass.EntryFuture newEntry = new ObjectStreamClass.EntryFuture();
        Reference<?> newRef = new SoftReference<>(newEntry);
        do {
            if (ref != null) {
                ObjectStreamClass.Caches.localDescs.remove(key, ref);
            }
            ref = ObjectStreamClass.Caches.localDescs.putIfAbsent(key, newRef);
            if (ref != null) {
                entry = ref.get();
            }
        } while (ref != null && entry == null);
        if (entry == null) {
            future = newEntry;
        }
    }

    if (entry instanceof ObjectStreamClass) {  // check common case first
        return (ObjectStreamClass) entry;
    }
    if (entry instanceof ObjectStreamClass.EntryFuture) {
        future = (ObjectStreamClass.EntryFuture) entry;
        if (future.getOwner() == Thread.currentThread()) {
            /*
             * Handle nested call situation described by 4803747: waiting
             * for future value to be set by a lookup() call further up the
             * stack will result in deadlock, so calculate and set the
             * future value here instead.
             */
            entry = null;
        } else {
            entry = future.get();
        }
    }
    if (entry == null) {
        try {
            entry = new ObjectStreamClass(cl);
        } catch (Throwable th) {
            entry = th;
        }
        if (future.set(entry)) {
            ObjectStreamClass.Caches.localDescs.put(key, new SoftReference<Object>(entry));
        } else {
            // nested lookup call already set future
            entry = future.get();
        }
    }

    if (entry instanceof ObjectStreamClass) {
        return (ObjectStreamClass) entry;
    } else if (entry instanceof RuntimeException) {
        throw (RuntimeException) entry;
    } else if (entry instanceof Error) {
        throw (Error) entry;
    } else {
        throw new InternalError("unexpected entry: " + entry);
    }
}
```

## 1.2 初始化 ObjectStreamClass

此时从缓存获取的为 null，因此需要初始化：`new ObjectStreamClass(cl);`，该初始化方法会有一系列的判断：
1. 是否是代理类（JDK动态代理都会继承 Proxy）
2. 是否为 Enum
3. 是否为 serializable（通过 native）
4. 是否为 externalizable
5. 获取父类信息（父类如果没有实现 Serializable 则必须有空构造方法）

当为 serializable 类型，继续如下步骤：
1. 如果为 enum 则生成一个值为 0 的 suid，并将 fields 设置为空的 ObjectStreamFiled 数组。
2. 如果为 Array 类型，将 fields 设置为空的 ObjectStreamField 数组。
3. 如果非以上两种类型则获取定义的 serialVersionUID 的值
4. 最后写入类信息若发现没有 suid，则根据类签名信息（`computeDefaultSUID`）来生成
5. 如果类实现了 Externalizable 接口则调用 `writeExternal`。
6. 写入流时，直接采用全类名写入。

```java
private ObjectStreamClass(final Class<?> cl) {
    this.cl = cl;
    name = cl.getName();
    isProxy = Proxy.isProxyClass(cl);
    isEnum = Enum.class.isAssignableFrom(cl);
    serializable = Serializable.class.isAssignableFrom(cl);
    externalizable = Externalizable.class.isAssignableFrom(cl);
    // 获取父类信息
    Class<?> superCl = cl.getSuperclass();
    superDesc = (superCl != null) ? lookup(superCl, false) : null;
    localDesc = this;

    if (serializable) {
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                if (isEnum) {
                    suid = Long.valueOf(0);
                    fields = NO_FIELDS;
                    return null;
                }
                if (cl.isArray()) {
                    fields = NO_FIELDS;
                    return null;
                }
                // 生成 suid
                suid = getDeclaredSUID(cl);
                try {
                    fields = getSerialFields(cl);
                    // 计算Filed空间，引用属性（非基本类型）个数
                    computeFieldOffsets();
                } catch (InvalidClassException e) {
                    serializeEx = deserializeEx =
                            new ObjectStreamClass.ExceptionInfo(e.classname, e.getMessage());
                    fields = NO_FIELDS;
                }

                if (externalizable) {
                    cons = getExternalizableConstructor(cl);
                } else {
                    cons = getSerializableConstructor(cl);
                    writeObjectMethod = getPrivateMethod(cl, "writeObject",
                            new Class<?>[] { ObjectOutputStream.class },
                            Void.TYPE);
                    readObjectMethod = getPrivateMethod(cl, "readObject",
                            new Class<?>[] { ObjectInputStream.class },
                            Void.TYPE);
                    readObjectNoDataMethod = getPrivateMethod(
                            cl, "readObjectNoData", null, Void.TYPE);
                    hasWriteObjectData = (writeObjectMethod != null);
                }
                domains = getProtectionDomains(cons, cl);
                writeReplaceMethod = getInheritableMethod(
                        cl, "writeReplace", null, Object.class);
                readResolveMethod = getInheritableMethod(
                        cl, "readResolve", null, Object.class);
                return null;
            }
        });
    } else {
        suid = Long.valueOf(0);
        fields = NO_FIELDS;
    }

    try {
        fieldRefl = getReflector(fields, this);
    } catch (InvalidClassException ex) {
        // field mismatches impossible when matching local fields vs. self
        throw new InternalError(ex);
    }

    if (deserializeEx == null) {
        if (isEnum) {
            deserializeEx = new ObjectStreamClass.ExceptionInfo(name, "enum type");
        } else if (cons == null) {
            deserializeEx = new ObjectStreamClass.ExceptionInfo(name, "no valid constructor");
        }
    }
    for (int i = 0; i < fields.length; i++) {
        if (fields[i].getField() == null) {
            defaultSerializeEx = new ObjectStreamClass.ExceptionInfo(
                    name, "unmatched serializable field(s) declared");
        }
    }
    initialized = true;
}
```

# 2. 反序列化之 readObject

在最开始校验：
```java
final static short STREAM_MAGIC = (short)0xaced;

/**
 * Version number that is written to the stream header.
 */
final static short STREAM_VERSION = 5;
```
![][1]

1. 读取流中类名、suid，是否有 writeObject、是否是 enum 类型，Filed 个数属性等。

[1]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/somephoto/2020-01-08%E5%BA%8F%E5%88%97%E5%8C%96%E5%A4%B4.png