---
layout: post
title: "Java 中的加载资源方法"
date: "2019-05-10"
description: "Java 中的加载资源的方法"
tag: [java]
---

### 获取资源
在 java 代码中常常会需要加载资源文件, 通常使用的方法是 `Class#getResource(String)` 和 `ClassLoader#getResource(String)` 方法, 这两个方法看起来很类似但有一些不同

#### Class#getResource(String)
```
/**
 * Finds a resource with a given name.  The rules for searching resources
 * associated with a given class are implemented by the defining
 * {@linkplain ClassLoader class loader} of the class.  This method
 * delegates to this object's class loader.  If this object was loaded by
 * the bootstrap class loader, the method delegates to {@link
 * ClassLoader#getSystemResource}.
 *
 * <p> Before delegation, an absolute resource name is constructed from the
 * given resource name using this algorithm:
 *
 * <ul>
 *
 * <li> If the {@code name} begins with a {@code '/'}
 * (<tt>'&#92;u002f'</tt>), then the absolute name of the resource is the
 * portion of the {@code name} following the {@code '/'}.
 *
 * <li> Otherwise, the absolute name is of the following form:
 *
 * <blockquote>
 *   {@code modified_package_name/name}
 * </blockquote>
 *
 * <p> Where the {@code modified_package_name} is the package name of this
 * object with {@code '/'} substituted for {@code '.'}
 * (<tt>'&#92;u002e'</tt>).
 *
 * </ul>
 *
 * @param  name name of the desired resource
 * @return      A  {@link java.net.URL} object or {@code null} if no
 *              resource with this name is found
 * @since  JDK1.1
 */
public java.net.URL getResource(String name) {
    name = resolveName(name);
    ClassLoader cl = getClassLoader0();
    if (cl==null) {
        // A system class.
        return ClassLoader.getSystemResource(name);
    }
    return cl.getResource(name);
}
```
- 加载资源 path 规则: 以 `/` 开头的认为是绝对路径, 去除开头的 '/'; 不以 `/` 开头的认为是相对路径 (相对于当前类的包路径), 在资源路径前加上包路径; 即最后都转换为非 `/` 开头的路径, 并使用 `ClassLoader#getResource(String)` 或 `ClassLoader#getSystemResource(String)` 方法来加载

#### ClassLoader#getResource(String)
```
/**
    * Finds the resource with the given name.  A resource is some data
    * (images, audio, text, etc) that can be accessed by class code in a way
    * that is independent of the location of the code.
    *
    * <p> The name of a resource is a '<tt>/</tt>'-separated path name that
    * identifies the resource.
    *
    * <p> This method will first search the parent class loader for the
    * resource; if the parent is <tt>null</tt> the path of the class loader
    * built-in to the virtual machine is searched.  That failing, this method
    * will invoke {@link #findResource(String)} to find the resource.  </p>
    *
    * @apiNote When overriding this method it is recommended that an
    * implementation ensures that any delegation is consistent with the {@link
    * #getResources(java.lang.String) getResources(String)} method.
    *
    * @param  name
    *         The resource name
    *
    * @return  A <tt>URL</tt> object for reading the resource, or
    *          <tt>null</tt> if the resource could not be found or the invoker
    *          doesn't have adequate  privileges to get the resource.
    *
    * @since  1.1
    */
   public URL getResource(String name) {
       URL url;
       if (parent != null) {
           url = parent.getResource(name);
       } else {
           url = getBootstrapResource(name);
       }
       if (url == null) {
           url = findResource(name);
       }
       return url;
   }
```
- 加载资源 path 规则: 默认从类路径顶层开始匹配 (即 classes 目录), 命令规则为非 `/` 开头, `/` 来分割目录的资源路径
- 加载资源的查找顺序: 先从父类 ClassLoader 查找 (即最先在 BootstrapClassLoader 中查找), 如未找到则调用自己的 `findResource(String)` 方法
-  ClassLoader.getSystemResource(String): 本质上与 ClassLoader.getResource(String) 方法没有什么不同, 只是使用委派的系统 ClassLoader (通常就是 AppClassLoader)

>**参考:**
[Class.getResource() 和 ClassLoader.getResource() 的区别](https://blog.csdn.net/walkerJong/article/details/13019671)
