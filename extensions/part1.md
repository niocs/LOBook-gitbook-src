# Libreoffice extension development with C++ - Part 1 - Getting to know UNO

UNO stands for Universal Network Objects and is the base component technology for LO. For a great coverage on UNO refer to 
https://wiki.openoffice.org/wiki/Documentation/DevGuide/OpenOffice.org_Developers_Guide. This is mostly compatible with LO as per
LO dev wiki. This blog is heavily based on this wiki.

> UNO is used to access OpenOffice.org, using its Application Programming Interface (API). The OpenOffice.org API is the comprehensive specification that describes the programmable features of OpenOffice.org

The LO api doc can be found at api.libreoffice.org

UNO can be used to :
* Access to a locally/remotely running LO from C++/Java program (Out-process)
* Develop UNO components in C++/Java/Python/LO Basic that can be instantiated by LO process to add new features to it.
  **Example** : *Calc Addins, Chart Addins, new file filters, database drivers*.
* Write complete applications using LO like a groupware client.

## Interfaces

An interface specifies a set of attributes and methods. An interface can inherit from a set of other interfaces.

Example 1 :  com.sun.star.resource.XResourceBundle ( interface defined at http://api.libreoffice.org/docs/idl/ref/XResourceBundle_8idl_source.html )

```
module com { module sun { module star { module resource {

published interface XResourceBundle: com::sun::star::container::XNameAccess
{
   [attribute] XResourceBundle Parent;

   com::sun::star::lang::Locale getLocale();

   any getDirectElement( [in] string key );
   
};

}; }; }; };
```

`XResourceBundle` interface is scoped inside `com::sun::star::resource` module.
This interface inherits the attributes/methods from `com::sun::star::container::XNameAccess` interface.
In addition it has "Parent" attribute and the methods `getLocale()` and `getDirectElement()`.

## Services and Service Managers

Service managers are factories that create services and services can be thought of as UNO objects that can be used to do specific tasks.

Examples of services :

* `com::sun::star::frame::Desktop`  (load/access all loaded docs)
* `com::sun::star::configuration::ConfigurationProvider`  (provides access to LO config)
* `com::sun::star::system::SystemShellExecute` (can exec system commands)
    
A service always holds on to a **component context** which provides access to the *service manager* that created the *service* and other data to be used by the *service*.

There are two types of services :

1. New-style service ( single interface based services ) :

   Example :
   
   ```
   module com { module sun { module star { module bridge {   
   	     service UnoUrlResolver: XUnoUrlResolver;
   }; }; }; };
   ```
   
   In this example `UnoUrlResolver` service is scoped under the module `com::sun::star::bridge` and it supports `XUnoUrlResolver` interface.

2. Old-style service/accumalation based service : (not to be used for new additions but support present for backward compatibility)

   Example :
   
   ```
   module com { module sun { module star { module frame { service DesktopOld {
      service FrameOld;
      interface XDesktop;
      interface XComponentLoader;
      interface com::sun::star::document::XEventBroadcaster;
  };
  }; }; }; };
  ```

  The service `DesktopOld` supports all interfaces exported (`XDesktop` `XComponentLoader` `XEventBroadcaster`) and inherited services (`FrameOld`)
  Additionally an old-style service may specify one or more *properties* like :

  ```
  module com { module sun { module star { module frame { service FrameOld {   
      interface com::sun::star::frame::XFrame;
      interface com::sun::star::frame::XDispatchProvider;
      // ...
      [property] string Title;
      [property, optional] XDispatchRecorderSupplier RecorderSupplier;
      // ...
  };
  }; }; }; };
  ```
  
  **Properties** are like *attributes* of an interface except that they can't be accessed directly. They can be accessed via generic interfaces like
  `com.sun.star.beans.XPropertySet`. `XPropertySet` is always supported when properties are present in service.
  Properites are used to represent *additional* features provided by the service.
  `XPropertySet` has the following methods :
  
  a.
    ```
     any getPropertyValue( [in] string PropertyName )
     	  raises( com::sun::star::beans::UnknownPropertyException,
	  	  com::sun::star::lang::WrappedTargetException );
    ```
  b.
     ```
     void setPropertyValue( [in] string aPropertyName,
     	  		    [in] any aValue )
   	  raises( com::sun::star::beans::UnknownPropertyException,
                  com::sun::star::beans::PropertyVetoException,
                  com::sun::star::lang::IllegalArgumentException,
		  com::sun::star::lang::WrappedTargetException );
     ```

  `any` in interface definition language corresponds the class `com::sun::star::uno::Any`
  See [Any.hxx](http://opengrok.libreoffice.org/xref/core/include/com/sun/star/uno/Any.hxx).
  Also see [api-reference](http://api.libreoffice.org/docs/cpp/ref/a00264.html).


>  The concepts of interfaces and services were introduced for the following reasons :

>  1)  Interfaces and services separate specification from implementation.

>  2)  Service names allow to create instances by specification name, not by class names.


## Setup LO SDK after building LO

Go to $LOROOT/instdir/sdk and run :
```
$ ./setsdkenv_unix
```
hit enter for all questions, letting it use the defaults except for "Automatic deployment of UNO components (YES/NO) [YES]: " type NO.

This will show something like the following after it finishes.

```
 ************************************************************************
 * ... your SDK environment has been prepared.
 * For each time you want to use this configured SDK environment, you
 * have to run the "setsdkenv_unix" script file!
 * Alternatively can you source one of the scripts
 *   "/home/dennis/libreoffice5.3_sdk/dennis-work/setsdkenv_unix.sh"
 * to get an environment without starting a new shell.
 ************************************************************************


 ************************************************************************
 *
 * SDK environment is prepared for Linux
 *
 * SDK = /ssd1/work/dennis/core/instdir/sdk
 * Office = /ssd1/work/dennis/core/instdir/sdk/..
 * Make = /usr/bin
 * Zip = /usr/bin
 * cat = /usr/bin
 * sed = /usr/bin
 * C++ Compiler = /usr/bin
 * Java = /usr
 * SDK Output directory = /home/dennis/libreoffice5.3_sdk
 * Auto deployment = NO
 *
 ************************************************************************
```

First section says that **we need to**
```bash
$ source /home/dennis/libreoffice5.3_sdk/dennis-work/setsdkenv_unix.sh
```
in shell **before** trying to build our own extensions/UNO standalone programs.

## Example C++ program that shows how to access LO from a standalone program

```cpp
// This program is a simplified version of
// http://api.libreoffice.org/examples/DevelopersGuide/ProfUNO/SimpleBootstrap_cpp/SimpleBootstrap_cpp.cxx

#include <stdio.h>
#include <sal/main.h>
#include <cppuhelper/bootstrap.hxx>
#include <com/sun/star/lang/XMultiComponentFactory.hpp>

using namespace com::sun::star::uno;
using namespace com::sun::star::lang;

using ::rtl::OString;
using ::rtl::OUString;
using ::rtl::OUStringToOString;

SAL_IMPLEMENT_MAIN()
{
    try
    {
        // get the remote office component context
        Reference< XComponentContext > xContext( ::cppu::bootstrap() );
	if ( !xContext.is() )
	{
	    fprintf(stdout, "\nError getting context from running LO instance...\n");
	    return -1;
	}
	
	// retrieve the service-manager from the context
        Reference< XMultiComponentFactory > rServiceManager = xContext->getServiceManager();
	if ( rServiceManager.is() )
	    fprintf(stdout, "\nremote ServiceManager is available\n");
	else
	    fprintf(stdout, "\nremote ServiceManager is not available\n");
	fflush(stdout);
    }
    catch ( ::cppu::BootstrapException& e )
    {
        fprintf(stderr, "caught BootstrapException: %s\n",
                OUStringToOString( e.getMessage(), RTL_TEXTENCODING_ASCII_US ).getStr());
        return 1;
    }
    catch ( Exception& e )
    {
        fprintf(stderr, "caught UNO exception: %s\n",
                OUStringToOString( e.Message, RTL_TEXTENCODING_ASCII_US ).getStr());
        return 1;
    }

    return 0;

}
```

How to build and run this program ?

```bash
$ git clone https://github.com/niocs/SimpleBootstrapCppUNO.git
$ make
$ make SimpleBootstrap.run
```

## Objects in LO

> In UNO, an object is a software artifact that has methods that you can call and attributes that you can get and set.

An **object** is a result of implementation of *services* which follows a set of *interfaces*.

*Objects* are created by *service managers*. Following shows how to create a `Desktop` *service* object :

```cpp
Reference< XInterface > xDesktop = xServiceManager->createInstanceWithContext( OUString("com.sun.star.frame.Desktop"), xContext );
```

The returned object is casted as `XInterface` interface even though the `Desktop` service implements `XDesktop2` interface.
See [Desktop api reference](http://api.libreoffice.org/docs/idl/ref/servicecom_1_1sun_1_1star_1_1frame_1_1Desktop.html)

`Document` objects are ones that represent the docs opened with LO. These are created by `Desktop` service object using `loadComponentFromURL()`

```cpp
Reference< XDesktop2 > xDesktop2( xDesktop, UNO_QUERY ); // Upcast previously created XInterface type
Reference< XComponent > xComponent = xDesktop2->loadComponentFromURL(
                                      OUString( "private:factory/scalc" ), // URL to the ods file
                                      OUString( "_blank" ), 0,
                                      Sequence < ::com::sun::star::beans::PropertyValue >() );
```

The complete C++ program that creates a Desktop object and loads an empty calc document can be built and run as :

```bash
$ git clone https://github.com/niocs/UNOCreateUseObject.git
$ make
$ make CreateUseObject.run
```

The above is an example of getting an object from another object. There are two different cases for getting objects from other objects :

1. Objects(special ones) that are related to features that are integral part of an object can be obtained by get*() methods.
   For example `getSheets()` gives sheets object for a calc document, `getText()` for a writer document, `getDrawpages()` for Draw document.
   After loading a document these methods can be used to get Sheets/Text/Drawpages objects of corresponding document.

2. Objects that are not considered integral part of an object are accessible through a set of "universal" methods such as
   `Any getPropertyValue( OUString aPropertyName )` (these objects are considered as properties of the main object)
   The type `Any` is explained in next part.


