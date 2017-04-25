---
title: 【Android】BaseDexClassLoader类加载分析小结.md
date: 2017-1-11 17:22
tag: ClassLoader
category: 热修复
---
## 参考资料

* [热修复入门：Android中的ClassLoader](http://jaeger.itscoder.com/android/2016/08/27/android-classloader.html)

* [AndroidXRef - BaseDexClassLoader.java](http://androidxref.com/6.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java)

* [AndroidXRef - PathClassLoader.java](http://androidxref.com/6.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/PathClassLoader.java)

* [AndroidXRef - DexClassLoader.java](http://androidxref.com/6.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java)

* [AndroidXRef - DexPathList](http://androidxref.com/6.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexPathList.java)

* [AndroidXRef - DexFile](http://androidxref.com/6.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexFile.java#defineClassNative)

<!-- more -->

## 概述

`PathClassLoader`和`DexClassLoader`是BaseDexClassLoader的子类，只是做了一些简单的封装



## PathClassLoader与DexClassLoader的源码分析

* PathClassLoader源代码:

```

public class PathClassLoader extends BaseDexClassLoader {

    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }

    public PathClassLoader(String dexPath, String libraryPath,
            ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}
```

* DexClassLoader源代码

```

public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```



## BaseDexClassLoader源代码分析

```

public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;

   
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }

    @Override
    protected URL findResource(String name) {
        return pathList.findResource(name);
    }

    @Override
    protected Enumeration<URL> findResources(String name) {
        return pathList.findResources(name);
    }

    @Override
    public String findLibrary(String name) {
        return pathList.findLibrary(name);
    }
....

```
通过源代码发现，对操作进行了包装，具体操作都是通过`DexPathList`来实现的。



## DexPathList

完整源代码：[AndroidXRef - DexPathList](http://androidxref.com/6.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexPathList.java)

```

final class DexPathList {
    private static final String DEX_SUFFIX = ".dex";
    private static final String zipSeparator = "!/";

    /** class definition context */
    private final ClassLoader definingContext;

    /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private final Element[] dexElements;

    /** List of native library path elements. */
    private final Element[] nativeLibraryPathElements;

    /** List of application native library directories. */
    private final List<File> nativeLibraryDirectories;

    /** List of system native library directories. */
    private final List<File> systemNativeLibraryDirectories;

    /**
     * Exceptions thrown during creation of the dexElements list.
     */
    private final IOException[] dexElementsSuppressedExceptions;

    
    public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {

        //...一些校验
        // save dexPath for BaseDexClassLoader
        this.dexElements = makePathElements(splitDexPath(dexPath), optimizedDirectory,
                                            suppressedExceptions);
        //....省略代码
        }
    
    private static Element[] makePathElements(List<File> files, File optimizedDirectory,
                                              List<IOException> suppressedExceptions) {
        List<Element> elements = new ArrayList<>();
        /*
         * Open all files and load the (direct or contained) dex files
         * up front.
         */
        for (File file : files) {
            File zip = null;
            File dir = new File("");
            DexFile dex = null;
            String path = file.getPath();
            String name = file.getName();

            if (path.contains(zipSeparator)) {
                String split[] = path.split(zipSeparator, 2);
                zip = new File(split[0]);
                dir = new File(split[1]);
            } else if (file.isDirectory()) {
                // We support directories for looking up resources and native libraries.
                // Looking up resources in directories is useful for running libcore tests.
                elements.add(new Element(file, true, null, null));
            } else if (file.isFile()) {
                if (name.endsWith(DEX_SUFFIX)) {
                    // Raw dex file (not inside a zip/jar).
                    try {
                        dex = loadDexFile(file, optimizedDirectory);
                    } catch (IOException ex) {
                        System.logE("Unable to load dex file: " + file, ex);
                    }
                } else {
                    zip = file;

                    try {
                        dex = loadDexFile(file, optimizedDirectory);
                    } catch (IOException suppressed) {
                        /*
                         * IOException might get thrown "legitimately" by the DexFile constructor if
                         * the zip file turns out to be resource-only (that is, no classes.dex file
                         * in it).
                         * Let dex == null and hang on to the exception to add to the tea-leaves for
                         * when findClass returns null.
                         */
                        suppressedExceptions.add(suppressed);
                    }
                }
            } else {
                System.logW("ClassLoader referenced unknown path: " + file);
            }

            if ((zip != null) || (dex != null)) {
                elements.add(new Element(dir, false, zip, dex));
            }
        }

        return elements.toArray(new Element[elements.size()]);
    }
    
        public Class findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;

            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
      //...省略代码
```

* 省略了大部分代码

* 贴出`findClass`的代码

* 贴出创建`Element`数组`dexElements`的代码



上述过程可以简述为：

* `makePathElements`方法将外部传入的dex文件全部以Element数据结构存入

* `findClass`时，通过`Element`中`DexFile`（存储dex文件的数据结构）的`findClassBinaryName`方法找到对应的`Class`。（即最终的类加载又交由`DexFile`去加载)



## DexFile源码

完整源代码：[AndroidXRef - DexFile](http://androidxref.com/6.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexFile.java#defineClassNative)

` DexPathList`最终会调用`DexFile` 的 `dex.loadClassBinaryName(name, definingContext, suppressed)` 完成类加载

通过DexFile源码可知，其流程是：

* `DexFile` 的 `loadClassBinaryName(name, definingContext, suppressed)`

* 然后调用 `DexFile` 的 `defineClass(name, loader, mCookie, suppressed)`

* 然后调用`DexFile`的`defineClassNative(name, loader, cookie)`

* 调用Native方法



## 小结


至此，BaseDexClassLoader加载Class的思路为：

* 当传入一个完整的类名，调用 `BaseDexClassLader` 的 `findClass(String name) `方法

* `BaseDexClassLader` 的 `findClass` 方法会交给 `DexPathList` 的 `findClass(String name, List<Throwable> suppressed` 方法处理

* 在 `DexPathList` 方法的内部，会遍历 `dexFile` ，通过 `DexFile` 的 `dex.loadClassBinaryName(name, definingContext, suppressed)` 来完成类的加载

* 最终调用`DexFile`的`defineClassNative(name, loader, cookie)`Native方法进行类加载


**-Hans**




