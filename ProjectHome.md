# What is this? #
This code is intended to solve a small, but annoying problem that occurs when loading native libraries (specifically DLLs) within a web container (Tomcat in this case).

## Loading a DLL into a Web Container ##
Intuitively, the following code would be written to load a DLL within a web container.
```
System.loadLibrary("name-of-dll");
```

It is important to only call this method once, as calling it multiple times will result in the following error:
```
java.lang.UnsatisfiedLinkError: Native Library "name-of-dll" already loaded ...
```

## Loading a DLL once in a Web Container ##
To correct for this issue, you can load the DLL in a static initializer block, which guarantees that it is only loaded once for the lifetime of the classloader assigned to that web container.
```
static {
   System.loadLibrary("name-of-dll");
}
```

However, if the webapp is redeployed within a running web container such as Tomcat, the code above will try to reload the DLL and throw the following error:
```
java.lang.UnsatisfiedLinkError: Native Library "name-of-dll" already loaded in another classloader
   at java.lang.ClassLoader.loadLibrary0(ClassLoader.java:1715)
   at java.lang.ClassLoader.loadLibrary(ClassLoader.java:1646)
   at java.lang.Runtime.load0(Runtime.java:787)
   at java.lang.System.load(System.java:1022)
```

What does this mean?  According to [Tomcat's documentation](http://tomcat.apache.org/tomcat-7.0-doc/class-loader-howto.html), each webapp is loaded within it's own classloader.  Which means when a webapp is redeployed, it loses access to the original classloader and the JVM won't allow it to load the native library again.

The user can restart the web container OR load the library in the common Tomcat classloader.


## Loading DLLs in the Tomcat Common Classloader ##
Solving this problem with a running web container requires loading the DLL into the common/shared classloader within Tomcat.

In order to accomplish this, you must create a class with a static initializer that loads the required DLLs.  That class must be compiled and placed in CATALINA\_HOME/lib, then referenced in your webapp to ensure the class is referenced thus the static initializer block is run.

The class that needs to exist in CATALINA\_HOME/lib, which will load the DLL into the common classloader for all webapps to reference.
```
package foo;

public class Loader {
  static {
     System.loadLibrary("name-of-dll");
  }

  // required for JDK 6 and JDK 7
  public static void main(String[] args) {}
}
```

In your webapp, the following code will ensure the Loader class is referenced.
```
  Class.forName("foo.Loader");

  // now your DLL is guaranteed to be available for use.
```



## What is this project? ##
I've written a small java class to use as an example OR can be used directly.  The dll-bootstrapper.jar file can be put directly into CATALINA\_HOME/lib, with a dll-bootstrapper.properties file that contains a list of DLLs to load.

dll-bootstrapper.properties file example:
```
# Example properties file.
# Specify up to 100 DLLs (or SOs) to load.
# Each will be loaded via System.loadLibrary(<name of library>);

# Example - load the libsndfile java wrapper dlls in order
dll.0=libsndfile-1
dll.1=libsndfile_wrap
```

Which would require only the following code in your application:
```
// triggers the loading of all required DLLs, as specified in dll-bootstrapper.properties
// loaded only once per JVM instance, shared across classloaders
Class.forName("msm.DLLBootstrapper");
```
