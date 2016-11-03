# Libreoffice extension development with C++ - Part 8 - Singleton Component.

### Audience selection

1. Audience type **[AUD-C](README.md)** can jump directly to **[Test the prebuilt extension](#prebuilt)**
2. Audience type **[AUD-B](README.md)** can jump directly to **[Download, build and test the extension](#buildsec)**

___

A *singleton component* is one which is instantiated just once during the whole lifetime of LO process. This may be useful for holding resources that are common to all other component instances. For example if we want to maintain a single long running TCP connection to external server process, we could let a singleton component hold and maintain the network connection.

In this part we create a singleton component that simulates a single bit global storage that implements a custom interface called `inco::niocs::test::XBoolDataStore` which has the following methods :
1. `getBool()` returns the boolean state stored by the implementation.
2. `setBool()` sets the boolean state of the implementation to a user supplied value.
3. `getAdddress()` returns the memory address of the singleton component class.

We define the custom interface `inco::niocs::test::XBoolDataStore` and singleton named `theBoolDataStore` in the idl file called `SimpleDataStore.idl` using the `singleton` keyword.

```
#ifndef __inco_niocs_test_simpledatastore_idl__
#define __inco_niocs_test_simpledatastore_idl__

module inco { module niocs { module test {

    interface XBoolDataStore
    {
        boolean getBool();
        void    setBool( [in]boolean bSet );
	string  getAddress();
    };

    singleton theBoolDataStore : XBoolDataStore;
    service BoolDataStore : XBoolDataStore;
	
}; }; };

#endif

```

Lets now define the class `BoolDataStoreImpl` that implements the singleton `theBoolDataStore` in singleton.hxx.

```cpp
#define IMPLEMENTATION_NAME "inco.niocs.test.BoolDataStoreImpl"

class BoolDataStoreImpl : public cppu::WeakImplHelper2
<
    css::lang::XServiceInfo,
    inco::niocs::test::XBoolDataStore
>
{
    sal_Bool bData;  // boolean state 
public:
    BoolDataStoreImpl() throw ()
            : bData(false)
    {}
    virtual ~BoolDataStoreImpl() {}

    // XBoolDataStore methods
    virtual sal_Bool SAL_CALL getBool() { return bData; }
    virtual void     SAL_CALL setBool( const sal_Bool bSet ) { bData = bSet; }
    virtual ::rtl::OUString SAL_CALL getAddress();
    
    // XServiceInfo methods
    virtual ::rtl::OUString SAL_CALL getImplementationName()
        throw (css::uno::RuntimeException);
    virtual sal_Bool SAL_CALL supportsService( const ::rtl::OUString& aServiceName )
        throw (css::uno::RuntimeException);
    virtual css::uno::Sequence< ::rtl::OUString > SAL_CALL getSupportedServiceNames()
        throw (css::uno::RuntimeException);
};


css::uno::Sequence< ::rtl::OUString > SAL_CALL BoolDataStoreImpl_getSupportedServiceNames()
    throw (css::uno::RuntimeException);

::com::sun::star::uno::Reference< ::com::sun::star::uno::XInterface > SAL_CALL BoolDataStoreImpl_createInstance(
    const css::uno::Reference< css::uno::XComponentContext > & rContext )
    throw( css::uno::Exception );
```

We now define the methods of `BoolDataStoreImpl` in singleton.cxx

```cpp
#define SERVICE_NAME "inco.niocs.test.BoolDataStore"

OUString SAL_CALL BoolDataStoreImpl::getAddress()
{
    return OUString::number(reinterpret_cast<std::uintptr_t>(this));
}

// DataStoreImpl : XServiceInfo
OUString SAL_CALL BoolDataStoreImpl::getImplementationName()
    throw (RuntimeException)
{
    return IMPLEMENTATION_NAME;
}

sal_Bool SAL_CALL BoolDataStoreImpl::supportsService( const OUString& rServiceName )
    throw (RuntimeException)
{
    return cppu::supportsService(this, rServiceName);
}

Sequence< OUString > SAL_CALL BoolDataStoreImpl::getSupportedServiceNames(  )
    throw (RuntimeException)
{
    return BoolDataStoreImpl_getSupportedServiceNames();
}

```

We need to ensure that only one instance of our component is to be made ever, we ensure this by doing :
```cpp
struct Instance {
    explicit Instance(
	Reference<XComponentContext> const & /*rxContext*/):
	instance(
	    static_cast<cppu::OWeakObject *>(new BoolDataStoreImpl()))
    {}
    Reference<XInterface> instance;
};


struct Singleton:
    public rtl::StaticWithArg<
    Instance, Reference<XComponentContext>, Singleton>
{};

```

The speciality of the `Singleton` struct is that we can if we call Singleton::get() it will create an instance of `BoolDataStoreImpl` if not already created and return it. If there is already an instance of `BoolDataStoreImpl` is created before, then it will just return that single instance.

Now we just use this struct to define the BoolDataStoreImpl_createInstance() helper function.

```cpp
Sequence< OUString > SAL_CALL BoolDataStoreImpl_getSupportedServiceNames()
    throw (RuntimeException)
{
    Sequence < OUString > aRet(1);
    OUString* pArray = aRet.getArray();
    pArray[0] =  OUString ( SERVICE_NAME );
    return aRet;    
}

Reference< XInterface > SAL_CALL BoolDataStoreImpl_createInstance( const Reference< XComponentContext > & rxContext )
    throw( Exception )
{
    return cppu::acquire(static_cast<cppu::OWeakObject *>(
			     Singleton::get(rxContext).instance.get()));
}
```

We now define the component operation functions that registers our singleton to LO using the helper function we created above.

```cpp
extern "C" SAL_DLLPUBLIC_EXPORT void * SAL_CALL component_getFactory(const sal_Char * pImplName, void * /*pServiceManager*/, void * pRegistryKey)
{
    void * pRet = 0;

    if (rtl_str_compare( pImplName, IMPLEMENTATION_NAME ) == 0)
    {
        Reference< XSingleComponentFactory > xFactory( createSingleComponentFactory(
            BoolDataStoreImpl_createInstance,
            OUString( IMPLEMENTATION_NAME ),
            BoolDataStoreImpl_getSupportedServiceNames() ) );

        if (xFactory.is())
        {
            xFactory->acquire();
            pRet = xFactory.get();
        }
    }

    return pRet;
}

extern "C" SAL_DLLPUBLIC_EXPORT void SAL_CALL
component_getImplementationEnvironment(
    char const ** ppEnvTypeName, uno_Environment **)
{
    *ppEnvTypeName = CPPU_CURRENT_LANGUAGE_BINDING_NAME;
}
```

We test the single component by using a macro. This gets the singleton using the implicit `get()` methods of our singleton twice to store to two different variables and checks if changing state in one affects change in other.

```vbscript
Sub TestSingleton

    Dim oServiceInst1
    Dim oServiceInst2
    Dim bad as Boolean
    On Error Goto ErrorHandler

    oServiceInst1 = inco.niocs.test.theBoolDataStore.get( GetDefaultContext() )
    addr1 = oServiceInst1.getAddress()
    If oServiceInst1.getBool() Then
        bad = True
    End If
    oServiceInst1.setBool(True)
    If oServiceInst1.getBool() = False Then
        bad = True
    End If

    oServiceInst2 = inco.niocs.test.theBoolDataStore.get( GetDefaultContext() )
    addr2 = oServiceInst2.getAddress()
    If oServiceInst2.getBool() = False Then
        bad = True
    End If
    oServiceInst2.setBool(False)
    If oServiceInst1.getBool() = True Then
        bad = True
    End If

    If bad Then
        msgbox "Bug in singleton component !"
    Else
        msgbox "singleton component works as expected. addr1 = " & addr1 & " and addr2 = " & addr2
    End If

    Exit Sub

    ErrorHandler:
    MsgBox "Error " & Err & ": " & Error$ & " (line : " & Erl & ")"
    'msgbox "singleton component not installed properly"

End Sub
```

##<a name="buildsec"></a>Download, build and test the extension

The whole project can be downloaded, built and installed by doing :

```
$ git clone https://github.com/niocs/SingletonComponent.git
$ cd SingletonComponent
$ make
$ $LOROOT/instdir/program/unopkg add /home/$username/libreoffice5.3_sdk/LINUXexample.out/bin/SingletonComponent.oxt
```

To remove the extension, do :
```
$ $LOROOT/instdir/program/unopkg remove SingletonComponent.oxt
```

To test this component we need to enable macros in Calc. Go to Tools > Options > Security > Click "Macro Security" > Select the radio button "Medium - Confirmation required before executing macros from untrusted source". Restart Calc now.
Open calc and load the document `TestSingleton.ods` and click on the button in the sheet "Push button", if everything went well you should see a message box with text `singleton component works as expected. addr1 = <number1> and addr2 = <number2>`, where `number1` and `number2` must be same the number.

## <a name="prebuilt"></a>Test the prebuilt extension

Download the extension as :
```
$ git clone https://github.com/niocs/SingletonComponent.git
$ cd SingletonComponent
```

In `SingletonComponent` directory you will see a file called `SingletonComponent.oxt`. This is the prebuilt extension. Install this extension using

```
$ unopkg add /path/to/SingletonComponent.oxt
```

To remove this extension do :

```
$ unopkg remove SingletonComponent.oxt
```

To test this component we need to enable macros in Calc. Go to Tools > Options > Security > Click "Macro Security" > Select the radio button "Medium - Confirmation required before executing macros from untrusted source". Restart Calc now.
Open calc and load the document `TestSingleton.ods` and click on the button in the sheet "Push button", if everything went well you should see a message box with text `singleton component works as expected. addr1 = <number1> and addr2 = <number2>`, where `number1` and `number2` must be the same number.