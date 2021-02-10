# notebook
知识梳理，便于记忆

classLoader整理

有两种方式可以打破classLoader的双亲委派机制：
1，使用Thread.currentThread().getContextClassLoader()，比如SPI机制，具体实现可参考JDBC的driver;

2，继承ClassLoader并覆写loadClass方法，ClassLoader的loadClass方法里面有双亲委派机制的实现（注意findClass是loadClass里面的一个方法，一般是实现从哪个地方加载类，最后一般调用defineClass方法将流转换成class），tomcat打破双亲委派就是以此种方式实现的。

下面扩展一下tomcat的具体实现方式：

tomcat中具体的实现是WebappClassLoader，实现方法在其父类WebappClassLoaderBase的loadClass里面，主要逻辑是先用ExtClassLoader加载，
再用自定义的classLoader加载，最后让AppClassLoader加载，代码如下
```java
public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
 		......
        // (0) Check our previously loaded local class cache
        clazz = findLoadedClass0(name);

        // (0.1) Check our previously loaded class cache
        clazz = findLoadedClass(name);
        // (0.2) Try loading the class with the system class loader, to prevent the webapp from overriding J2SE classes
        //这里的j2seClassLoader是指extClassLoader
        clazz = j2seClassLoader.loadClass(name);

        //delegated是否委托父类加载
        boolean delegateLoad = delegate || filter(name);

        // (1) Delegate to our parent if requested
        if (delegateLoad) {
              ...
        }

        // (2) Search local repositories
        if (log.isDebugEnabled())
            log.debug("  Searching local repositories");
        try {
            //主要实现在findClass里面
            clazz = findClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Loading class from local repository");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }

        // (3) Delegate to parent unconditionally
        if (!delegateLoad) {
            try {
                clazz = Class.forName(name, false, parent);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from parent");
                    if (resolve)
                        resolveClass(clazz);
                    return (clazz);
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
        }
    }
        throw new ClassNotFoundException(name);
```
findClass方法里面会调用findClassInternal方法，主要实现是查找webapps下面具体项目的WEB-INF包下面的classes文件夹和**lib文件夹**。

tomcat会为每个项目创建一个WebAppClassLoader，然后调用其loadClass方法，这样就可以实现不同版本的jar包可以用不同的类加载加载，做到不同项目之间jar的隔离，从而打破双亲委派模型。
