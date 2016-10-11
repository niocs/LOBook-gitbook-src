# Libreoffice extension development with C++ - Part 4 - Build a simple Component with two services.

Note that reading previous posts are a prerequisite for following this part unless you know how to setup LO SDK env and have a good understanding of UNO concepts.

## What we are about to do ?

1. Create an .idl file with
  1. a simple interface XSomething that has two methods.
  2. two services MyService1 and MyService2 which supports XSomething interface.

2. Write an implementation for the two services in the files service1_impl.cxx and service2_impl.cxx.

3. Write service operations for the services in service1_impl.cxx and service2_impl.cxx.

4. Write component operations in service2_impl.cxx.

5. Use UNO SDK tools to create C++ header files from .idl files (ours and the dependent LO buildin .idl files).

6. Build the component "SimpleComponent" from service implementation cxx files with the hdl files generated in step 5,
   and package it into an extension file named as "SimpleComponent.oxt".

7. Install our extension in LO.

8. Create an ods file with embedded macro to test the two services in our extension.

9. Use a C++ main program that accesses remote LO process to test the two services in our extension.


Let's now go into these 9 steps in detail.


## Step 1

Lets define an interface called `XSomething` that has two methods -
* `methodOne()` that accepts a string and returns a string.
* `methodTwo()` that has no parameters and returns a string.

```
interface XSomething
{
    string methodOne( [in] string val );
    string methodTwo();
};
```

Next we define a service called `MyService1` that supports the interface `XSomething`.
This is just a one liner.

```
service MyService1 : XSomething;
```

We now define second service called `MyService2` that again supports the interface `XSomething`.
All services come with an implicit object creator function called `create()` that **implicitly** accepts the component context object , but we want `MyService2`'s `create()` function to accept a string parameter *in addition*. We also want this function to throw an `IllegalArgumentException` if the user did not give the string argument. We would code that logic while implementation the service, but we need to specify the exception it may throw if any. One exception is `RuntimeException` which can be thrown by any methods without explicitly specifying it.

```
service MyService2 : XSomething
{
    create([in]string sArgument)
         raises (::com::sun::star::lang::IllegalArgumentException);
};
```

We next need to scope all these definitions under a heirarchy of modules. Lets scope the definitions under the module scope `inco::niocs::test` as an example.

```
module inco { module niocs { module test {

// Put the interface and service definitions here.

}; }; };

```

Since all interfaces implicitly derive from the interface `XInterface` we need to include its corresponding idl file. Also since we need to use the exception `IllegalArgumentException`, we need to include its idl file as well.

```cpp
#include <com/sun/star/uno/XInterface.idl>
#include <com/sun/star/lang/IllegalArgumentException.idl>

```

Like in C++ header files we can avoid namespace conflicts while sourcing header files. This is done be putting the whole content inside the following :

```cpp
#ifndef __inco_niocs_test_some_idl__
#define __inco_niocs_test_some_idl__

//  The idl file contents goes here.


#endif
```


## Step 2

Although in the service definition in some.idl file our services only support `XSomething` interface explicitly, `XSomething` derives from `XInterface` interface implicitly we need to implement its methods too.
As a exercise have a look at `XInterface` idl file from api.libreoffice.org to refresh your memory.

Any service **needs to** implement the following Interfaces. We don't need to specify these interfaces in the service definition idl file though.
* `XInterface`
* `XTypeProvider` - This interface is used by scripting languages such as OpenOffice.org Basic to get type information. OpenOffice.org Basic cannot use the component without it
* `XServiceInfo` - This interface lets other objects get information about which all services an implementation supports.
* `XWeak` - This interface lets anyone to keep a **weak** reference to your object. The difference between a weak reference and a normal reference is that former lets anyone to access your object without changing its *reference count*. As a result a weak reference cannot prevent the destruction of the object unless someone else keeps atleast one hard reference to it. XWeak interface also lets the user to retrieve a hard reference.

There exists helper classes for easy implementation of `XInterface`, `XTypeProvider`, `XWeak` interfaces. `XServiceInfo` interface implementation needs to be implemented manually without helper classes.


Following are the interfaces a service may optionally implement. Infact a service may implement any of the LO's pre-defined interfaces, but these just happens to be the most commonly implemented ones as they are very useful.

* `XComponent` - This interface is used if there is possibility of occurance of cyclic references. It helps in resolving the cyclic references.
* `XInitialization` - This interface lets anyone to use `createInstanceWithArguments()` and `createInstanceWithArgumentsAndContext()` with your object. This lets users to pass in any number of arguments while creating our object.

There exist a helper class for `XComponent` interface.


Now we are ready to implement the two services we defined in step 1.

The implementation of `MyService1` is written in service1_impl.cxx. In this file we declare and define the class `MyService1Impl` which implements the service `MyService1`.
The SDK helps us by giving helpers for easy implementation of services. We could however implement without using them always, but here we choose to make use of them to keep things simple.
It provides the template classes `cppu::WeakImplHelper2`, `cppu::WeakImplHelper3`, `cppu::WeakImplHelper4` ... `cppu::WeakImplHelper13` that acts as base class for our implementations that by automatically implements the interfaces `XInterface`, `XTypeProvider`, `XWeak`. So we do not need to implement them on our own.

In our case `MyService1Impl`, the implementation class of the service `MyService1` will just need to derive from the class `::cppu::WeakImplHelper2< ::inco::niocs::test::XSomething, lang::XServiceInfo >`.
The number 2 in `::cppu::WeakImplHelper2` stands for the number of interfaces we need to implement **ourselves**. Here we need to implement two interfaces ourselves which are `XSomething` and `XServiceInfo`.

Our `MyService1Impl` class declaration would look like :

```cpp
class MyService1Impl : public ::cppu::WeakImplHelper2< ::inco::niocs::test::XSomething, 
                                                       lang::XServiceInfo > 
{

public:

    // no need to implement XInterface, XTypeProvider, XWeak

    // XServiceInfo methods
    virtual OUString SAL_CALL getImplementationName() throw( RuntimeException );
    virtual sal_Bool SAL_CALL supportsService( OUString const & serviceName ) throw (RuntimeException);
    virtual Sequence< OUString >  SAL_CALL getSupportedServiceNames() throw (RuntimeException);

    // XSomething methods
    virtual OUString SAL_CALL methodOne( OUString const & str ) throw (RuntimeException);
    virtual OUString SAL_CALL methodTwo() throw (RuntimeException);

};

```
Note that `SAL_CALL` macro is not really required for building extensions in Linux. In Windows/Mingw, this expands to `__cdecl`. See [http://opengrok.libreoffice.org/xref/core/include/sal/types.h#255](http://opengrok.libreoffice.org/xref/core/include/sal/types.h#255)


Now lets implement the `XServiceInfo`'s three methods for `MyService1Impl` class.

```cpp
// XServiceInfo implementation
OUString MyService1Impl::getImplementationName()
    throw (RuntimeException)
{
    // a unique implementation name
    return OUString("inco.niocs.test.my_sc_impl.MyService1");
}

sal_Bool MyService1Impl::supportsService( OUString const & serviceName )
    throw (RuntimeException)
{
    // We just use the following helper function to do the job
    return cppu::supportsService(this, serviceName);
}

Sequence< OUString > MyService1Impl::getSupportedServiceNames()
    throw (RuntimeException)
{
    Sequence<OUString> names(1);
    names[0] = OUString("inco.niocs.test.MyService1");
    return names;
}

```

Finally we implement `XSomething`, the interface we actually care about, though it does not do much here.

```cpp
// XSomething implementation
OUString MyService1Impl::methodOne( OUString const & str )
    throw (RuntimeException)
{
    return OUString("called methodOne() of MyService1 implementation: ") + str;
}

OUString MyService1Impl::methodTwo()
    throw (RuntimeException)
{
    return OUString("called methodTwo() of MyService1 implementation: ");
}
```

Next we write the class `MyService2Impl` which implements the service `MyService2`. Remember that we need to accept one string parameter for the service object creation function. We do this via `XInitialization` interface. We also store the accepted string argument as a private data member called `m_sArg` in the class `MyService2Impl`. The constructor of `MyService2Impl` accepts the component context object and stores in a private data member `m_xContext`. In our simple example this is not actually required, but in non-trivial applications the service may need the component context object for accessing other objects in may need.

The class declaration of `MyService2Impl` would now look like :

```cpp
class MyService2Impl : public ::cppu::WeakImplHelper3< ::inco::niocs::test::XSomething, 
                                                       lang::XServiceInfo,
                                                       lang::XInitialization > 
{
    OUString m_sArg;
    Reference<XComponentContext> m_xContext;

public:

    inline MyService2Impl(Reference< XComponentContext > const & xContext) throw ()
            : m_xContext(xContext)
    {}
    virtual ~MyService2Impl() {}

    // no need to implement XInterface, XTypeProvider, XWeak
    
    // XServiceInfo methods
    virtual OUString SAL_CALL getImplementationName() throw( RuntimeException );
    virtual sal_Bool SAL_CALL supportsService( OUString const & serviceName ) throw (RuntimeException);
    virtual Sequence< OUString >  SAL_CALL getSupportedServiceNames() throw (RuntimeException);

    // XSomething methods
    virtual OUString SAL_CALL methodOne( OUString const & str ) throw (RuntimeException);
    virtual OUString SAL_CALL methodTwo() throw (RuntimeException);

    // XInitialization methods
    virtual void SAL_CALL initialize( Sequence< Any > const & args ) throw (Exception);

};

```

Now lets implement the `XServiceInfo`'s three methods for `MyService2Impl` class.

```cpp
// XServiceInfo implementation
OUString MyService2Impl::getImplementationName()
    throw (RuntimeException)
{
    // a unique implementation name
    return OUString("inco.niocs.test.my_sc_impl.MyService2");
}

sal_Bool MyService2Impl::supportsService( OUString const & serviceName )
    throw (RuntimeException)
{
    // We just use the following helper function to do the job
    return cppu::supportsService(this, serviceName);
}

Sequence< OUString > MyService2Impl::getSupportedServiceNames()
    throw (RuntimeException)
{
    Sequence<OUString> names(1);
    names[0] = OUString("inco.niocs.test.MyService2");
    return names;
}

```

Finally we implement `XSomething`, the interface we actually care about.

```cpp
// XSomething implementation
OUString MyService2Impl::methodOne( OUString const & str )
    throw (RuntimeException)
{
    return OUString("called methodOne() of MyService2 implementation: ") + str;
}

OUString MyService2Impl::methodTwo()
    throw (RuntimeException)
{
    return OUString("called methodTwo() of MyService2 implementation: ");
}
```

We also need to implement `initialize` - the sole method of `XInitialization` interface.
In our case we accept only a single string argument else we throw an `IllegalArgumentException` exception.
The accepted string argument is stored in the `m_sArg` data member of `MyService2Impl`.

```cpp
// XInitialization implementation
void MyService2Impl::initialize( Sequence< Any > const & args )
    throw (Exception)
{
    if (args.getLength() != 1) {
        throw lang::IllegalArgumentException(
              OUString("give a string instanciating this component!"),
              static_cast< ::cppu::OWeakObject * >(this),
              0 ); // argument pos
    }

    if (! (args[ 0 ] >>= m_sArg)) {
        throw lang::IllegalArgumentException(
              OUString("no string given as argument!"),
              static_cast< ::cppu::OWeakObject * >(this),
              0 ); // argument pos
    }
}
```


## Step 3

We now need to write service operation functions for both the services. These are not exported functions but we will use them in writing component operations functions we define in Step 4.

Following are the service operation functions we define in service1_impl.cxx file.

```cpp
// service operations for MyService1Impl
Sequence< OUString > getSupportedServiceNames_MyService1Impl()
{
    Sequence<OUString> names(1);
    names[0] = OUString("inco.niocs.test.MyService1");
    return names;
}

OUString getImplementationName_MyService1Impl()
{
    return OUString("inco.niocs.test.my_sc_impl.MyService1");
}

// Service object contructor function.
Reference< XInterface > SAL_CALL create_MyService1Impl( Reference< XComponentContext > const & xContext )
{
    return static_cast< lang::XTypeProvider * >( new MyService1Impl() );
}

```

and we define the following service operations in service2_impl.cxx file.

```cpp
// service operations for MyService2Impl
Sequence< OUString > getSupportedServiceNames_MyService2Impl()
{
    Sequence<OUString> names(1);
    names[0] = OUString("inco.niocs.test.MyService2");
    return names;
}

OUString getImplementationName_MyService2Impl()
{
    return OUString("inco.niocs.test.my_sc_impl.MyService2");
}

// Service object contructor function.
Reference< XInterface > SAL_CALL create_MyService2Impl( Reference< XComponentContext > const & xContext )
{
    return static_cast< lang::XTypeProvider * >( new MyService2Impl( xContext ) );
}

```

## Step 4

NOTE: The code we write in this step is boilerplate and is not explained in detail because we can afford to skim through this. We need to do this for every component we write and is quite copy/pastable.

We choose to write **component operation** functions in service2_impl.cxx. The Component operation functions register the component along with its service implementations with LO so that our implementations are discoverable and can be used in LO.

These functions are exported functions with defined with the macro `SAL_DLLPUBLIC_EXPORT` within `extern "C" block` so that g++ compiler does not mangle/change these exported names.

Before we define these functions, we create an unexported array of `::cppu::ImplementationEntry`'s of which each element corresponds to the particulars of a service.

```cpp
// This struct makes it easy to implement the extern "C" exported component operations
// using helper functions.
static struct ::cppu::ImplementationEntry s_component_entries [] =  {
    {   // Service1
        create_MyService1Impl, getImplementationName_MyService1Impl,
        getSupportedServiceNames_MyService1Impl, ::cppu::createSingleComponentFactory,
        0, 0  // last two entries reserved for future use, can be zeros.
    },
    {
        // Service2
        create_MyService2Impl, getImplementationName_MyService2Impl,
        getSupportedServiceNames_MyService2Impl, ::cppu::createSingleComponentFactory,
        0, 0
    },
    
    // NULL termination.
    { 0, 0, 0, 0, 0, 0 }
};
```

In the component operation function `component_getFactory()` we make use of the helper function `::cppu::component_getFactoryHelper()` and the array we created above.

```cpp
// Exported component operations

extern "C" // To skip g++'s function name decorations
{

    SAL_DLLPUBLIC_EXPORT void * SAL_CALL component_getFactory(
                              sal_Char const * implName, lang::XMultiServiceFactory * xMgr,
                              registry::XRegistryKey * xRegistry )
    {
        return ::cppu::component_getFactoryHelper(
               implName, xMgr, xRegistry, ::my_sc_impl::s_component_entries );
    }

    SAL_DLLPUBLIC_EXPORT void SAL_CALL component_getImplementationEnvironment(
                              char const ** ppEnvTypeName, uno_Environment **)
    {
        *ppEnvTypeName = CPPU_CURRENT_LANGUAGE_BINDING_NAME;
    }

}
```

## Steps 5 and 6

In this section we use UNO SDK tools to create C++ header files from .idl files (ours and the dependent LO buildin .idl files) and build the component "SimpleComponent" from service implementation cxx files with the hdl files generated and package it into an extension file named as "SimpleComponent.oxt". Doing these operations manually is tedious hence we have a Makefile reused from the LO SDK's sample extensions.

The whole project can be downloaded and built using the following steps.

```
$ git clone https://github.com/niocs/SimpleUNOComponent.git
$ cd SimpleUNOComponent
$ make
```

The extension created (`SimpleComponent.oxt`) by the above will go to a path that looks like `/home/dennis/libreoffice5.3_sdk/LINUXexample.out/bin/` unless you used a non default path while setting up SDK env.
You can also see the generated include files from idl files in `/home/dennis/libreoffice5.3_sdk/LINUXexample.out/inc/`.


## Step 7

Now we need to install the extension file `SimpleComponent.oxt` to LO. Run Calc and go to Tools > Extension Manager > Click add button, browse and select `SimpleComponent.oxt` and click on "Only for me" button for the dialog "For whom do you want to install the extension ?". Now exit Calc.

*Note that the LO version and the SDK must match for the extension to work. So if you built LO from git master branch, then use the SDK in `$LOROOT/instdir/sdk/` to build the extension.*


## Step 8

Lets now test if the extension is working. One way is we can instantiate the services we implemented, via LibreOffice Basic macros. For this we need to enable macros in Calc.
Go to Tools > Options > Security > Click "Macro Security" > Select the radio button "Medium - Confirmation required before executing macros from unstrusted source". Restart Calc now.

You could either follow the below instruction to setup macro and controls to execute it or just open the `SimpleComponent.ods` provided in the git repo. While opening click enable macros when asked. Now click on the button "Click to test our extension". On clicking if everything went well, it should show the messages from the `MyService1`'s `methodOne()` and `MyService2`'s `methodTwo()`.

### Creating an ods file with macro to access our extension.
Save the sheet as SimpleComponent.ods . Go to Tools > Macros > Organize Macros > LibreofficeDev Basic > Select "Standard" module inside "SimpleComponent.ods" on the left section > Click "New" button on the right section. Now it will open a Macro editor. Now insert the following Basic funtion named `TestMyExtension` to instantiate and call the methods on the two services.

```vbscript
Sub TestMyExtension
    Dim oService1Inst
    Dim oService2Inst
    On Error Goto ErrorHandler
    
    oService1Inst = inco.niocs.test.MyService1.create()
    oService2Inst = inco.niocs.test.MyService2.create( ":SERVICE2:" )
    If IsNull( oService1Inst ) Or IsNull( oService2Inst ) Then
        msgbox "Could not instantiate one or more of our services in the extension."
    Else
	msgbox oService1Inst.methodOne(":SERVICE1:") & ". and " & oService2Inst.methodTwo()
    End If
    Exit Sub
    ErrorHandler:
    msgbox "SimpleComponent Extension not installed properly"
End Sub
```

Lets now create a push button control. Go to View > Toolbars > Form Controls > Click on Design Mode in the new toolbar > Click on pushbutton control and draw in an empty sheet. Right click on the Push button and click "Control..." and change Label to "Click to test our extension". Go to Events tab and set Execution action > Assign Macro > Browse and select `TestMyExtension` macro we created above.

## Step 9

One other method to test if our extension is working is to create a standalone test C++ main program like we did in [Part 1](part1.md) and [Part 2](part2.md) of this article.
The complete test program `TestSimpleComponent.cxx` is provided in the git repo. So you could test the extension by issuing

```
$ make TestSimpleComponent.run
```

