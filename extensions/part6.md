# Libreoffice extension development with C++ - Part 6 - ProtocolHandler Component.

### Audience selection

1. Audience type **[AUD-C](README.md)** can jump directly to **[Test the prebuilt extension](#prebuilt)**
2. Audience type **[AUD-B](README.md)** can jump directly to **[Download, build and test the extension](#buildsec)**

___

In [Part5](part5.md) we saw how to add menu and toolbar entries in Calc. In this part we see how the command URLs emitted by these new toolbar buttons and menu entries can be handled according to our needs. On clicking the "Insert Date"/"Insert Time" button/menu, we insert current date/time to cell A1.

One way to make our component handle any commandURL, is to implement `com.sun.star.frame.ProtocolHandler` service. `ProtocolHandler` service supports the interface `com.sun.star.frame.XDispatchProvider` mandatorily and optionally support the interface `com.sun.star.lang.XInitialization`.

![ProtocolHandler service](https://wiki.openoffice.org/w/images/thumb/a/a0/ProtocolHandler.png/800px-ProtocolHandler.png)

The interface `XDispatchProvider` supports two methods :
```
XDispatch queryDispatch( [in] ::com::sun::star::util::URL URL, 
                         [in] string TargetFrameName,
                         [in] long SearchFlags )

sequence< XDispatch > queryDispatches( [in] sequence< DispatchDescriptor > Requests )
```

Our component implementing `ProtocolHandler` service will be asked for its agreement to execute a given URL by a call to interface methods `queryDispatch()`. It has to parse and validate the incoming URL. If the URL is valid and the protocol handler is able to handle it, it should return a dispatch object, thus indicating that it accepts the request.
The returned object must implement the interface `com.sun.star.frame.XDispatch` which has the following methods :
```
void dispatch(
     [in] com::sun::star::util::URL URL,
     [in] sequence<com::sun::star::beans::PropertyValue> Arguments);

void addStatusListener(
     [in] XStatusListener Control,
     [in] com::sun::star::util::URL URL);

void removeStatusListener(
     [in] XStatusListener Control,
     [in] com::sun::star::util::URL URL);
```

If our ProtocolHandler component supports `XInitialization` interface, its method `void initialize([in] sequence< any > aArguments)` will be passed with a `com.sun.star.frame.Frame` object. This `Frame` object implements `com.sun.star.frame.XFrame` interface which gives us access to the controller from which we can access the document model where we can manipulate the document.

Following diagram shows how to get the controller and document model from XFrame interface.

![Frame-controller-model organization](https://wiki.openoffice.org/w/images/thumb/d/d1/FCMNavigation.png/800px-FCMNavigation.png)

Note that our ProtocolHandler Component will be instantiated when LO starts for every UI element whose URL we support.

Lets start with declaring our ProtocolHandler component implementation class `Addon` in `addon.hxx`

```cpp

#define IMPLEMENTATION_NAME "inco.niocs.test.protocolhandler"

class Addon : public cppu::WeakImplHelper3
<
    com::sun::star::frame::XDispatchProvider,
    com::sun::star::lang::XInitialization,
    com::sun::star::lang::XServiceInfo
>
{
private:
    ::com::sun::star::uno::Reference< ::com::sun::star::uno::XComponentContext > mxContext;
    ::com::sun::star::uno::Reference< ::com::sun::star::frame::XFrame > mxFrame;

public:
    Addon( const ::com::sun::star::uno::Reference< ::com::sun::star::uno::XComponentContext > &rxContext)
        : mxContext( rxContext )
    {
	printf("DEBUG>>> Created Addon object : %p\n", this); fflush(stdout);
    }

    // XDispatchProvider
    virtual ::com::sun::star::uno::Reference< ::com::sun::star::frame::XDispatch >
            SAL_CALL queryDispatch( const ::com::sun::star::util::URL& aURL,
                const ::rtl::OUString& sTargetFrameName, sal_Int32 nSearchFlags )
                throw( ::com::sun::star::uno::RuntimeException );
    virtual ::com::sun::star::uno::Sequence < ::com::sun::star::uno::Reference< ::com::sun::star::frame::XDispatch > >
        SAL_CALL queryDispatches(
            const ::com::sun::star::uno::Sequence < ::com::sun::star::frame::DispatchDescriptor >& seqDescriptor )
            throw( ::com::sun::star::uno::RuntimeException );

    // XInitialization
    virtual void SAL_CALL initialize( const ::com::sun::star::uno::Sequence< ::com::sun::star::uno::Any >& aArguments )
        throw (::com::sun::star::uno::Exception, ::com::sun::star::uno::RuntimeException);

    // XServiceInfo
    virtual ::rtl::OUString SAL_CALL getImplementationName()
        throw (::com::sun::star::uno::RuntimeException);
    virtual sal_Bool SAL_CALL supportsService( const ::rtl::OUString& ServiceName )
        throw (::com::sun::star::uno::RuntimeException);
    virtual ::com::sun::star::uno::Sequence< ::rtl::OUString > SAL_CALL getSupportedServiceNames()
        throw (::com::sun::star::uno::RuntimeException);
};

// Standard service operations functions needed for every component
::rtl::OUString Addon_getImplementationName()
    throw ( ::com::sun::star::uno::RuntimeException );

sal_Bool SAL_CALL Addon_supportsService( const ::rtl::OUString& ServiceName )
    throw ( ::com::sun::star::uno::RuntimeException );

::com::sun::star::uno::Sequence< ::rtl::OUString > SAL_CALL Addon_getSupportedServiceNames()
    throw ( ::com::sun::star::uno::RuntimeException );

::com::sun::star::uno::Reference< ::com::sun::star::uno::XInterface >
SAL_CALL Addon_createInstance( const ::com::sun::star::uno::Reference< ::com::sun::star::uno::XComponentContext > & rContext)
    throw ( ::com::sun::star::uno::Exception );

```

Next we define the `XDispatch` implementing class `DateTimeWriterDispatchImpl` in `addon.hxx`.

```cpp
class DateTimeWriterDispatchImpl : public cppu::WeakImplHelper1<com::sun::star::frame::XDispatch>
{
private:
    ::com::sun::star::uno::Reference< ::com::sun::star::frame::XFrame > mxFrame;
public:
    DateTimeWriterDispatchImpl( const ::com::sun::star::uno::Reference< ::com::sun::star::frame::XFrame > &rxFrame )
	: mxFrame( rxFrame )
    {
	printf("DEBUG>>> Created DateTimeWriterDispatchImpl object : %p\n", this); fflush(stdout);
    }

    // XDispatch
    virtual void SAL_CALL dispatch( const ::com::sun::star::util::URL& aURL,
        const ::com::sun::star::uno::Sequence< ::com::sun::star::beans::PropertyValue >& lArgs )
        throw (::com::sun::star::uno::RuntimeException);
    virtual void SAL_CALL addStatusListener( const ::com::sun::star::uno::Reference< ::com::sun::star::frame::XStatusListener >& xControl,
        const ::com::sun::star::util::URL& aURL ) throw (::com::sun::star::uno::RuntimeException);
    virtual void SAL_CALL removeStatusListener( const ::com::sun::star::uno::Reference< ::com::sun::star::frame::XStatusListener >& xControl,
        const ::com::sun::star::util::URL& aURL ) throw (::com::sun::star::uno::RuntimeException);
};
```

We can now implement the `Addon` class methods in `addons.cxx`

```cpp

// This is the service name an Add-On has to implement
#define SERVICE_NAME "com.sun.star.frame.ProtocolHandler"

// ProtocolHandler implementation "Addon" class methods

void SAL_CALL Addon::initialize( const Sequence< Any >& aArguments ) throw ( Exception, RuntimeException)
{
    Reference < XFrame > xFrame;
    if ( aArguments.getLength() )
    {
        aArguments[0] >>= xFrame;
        mxFrame = xFrame;
    }
}

Reference< XDispatch > SAL_CALL Addon::queryDispatch( const URL& aURL, const ::rtl::OUString& sTargetFrameName, sal_Int32 nSearchFlags )
                throw( RuntimeException )
{
    Reference < XDispatch > xRet;
    if ( aURL.Protocol.equalsAscii("inco.niocs.test.protocolhandler:") )
    {
	printf("DEBUG>>> Addon::queryDispatch() called. this = %p, command = %s\n", this,
	    OUStringToOString( aURL.Path, RTL_TEXTENCODING_ASCII_US ).getStr()); fflush(stdout);
        if ( aURL.Path.equalsAscii( "InsertDate" ) )
            xRet = new DateTimeWriterDispatchImpl( mxFrame );
        else if ( aURL.Path.equalsAscii( "InsertTime" ) )
            xRet = xRet = new DateTimeWriterDispatchImpl( mxFrame );
    }

    return xRet;
}

// This is a method which can query multiple URLs at once.
Sequence < Reference< XDispatch > > SAL_CALL Addon::queryDispatches( const Sequence < DispatchDescriptor >& seqDescripts )
            throw( RuntimeException )
{
    sal_Int32 nCount = seqDescripts.getLength();
    Sequence < Reference < XDispatch > > lDispatcher( nCount );

    for( sal_Int32 i=0; i<nCount; ++i )
        lDispatcher[i] = queryDispatch( seqDescripts[i].FeatureURL, seqDescripts[i].FrameName, seqDescripts[i].SearchFlags );

    return lDispatcher;
}


// Helper functions for the implementation of UNO component interfaces.
OUString Addon_getImplementationName()
throw (RuntimeException)
{
    return OUString ( IMPLEMENTATION_NAME );
}

Sequence< ::rtl::OUString > SAL_CALL Addon_getSupportedServiceNames()
throw (RuntimeException)
{
    Sequence < ::rtl::OUString > aRet(1);
    ::rtl::OUString* pArray = aRet.getArray();
    pArray[0] =  OUString ( SERVICE_NAME );
    return aRet;
}

Reference< XInterface > SAL_CALL Addon_createInstance( const Reference< XComponentContext > & rContext)
    throw( Exception )
{
    return (cppu::OWeakObject*) new Addon( rContext );
}

// Implementation of the recommended/mandatory interfaces of a UNO component.
// XServiceInfo
::rtl::OUString SAL_CALL Addon::getImplementationName()
    throw (RuntimeException)
{
    return Addon_getImplementationName();
}

sal_Bool SAL_CALL Addon::supportsService( const ::rtl::OUString& rServiceName )
    throw (RuntimeException)
{
    return cppu::supportsService(this, rServiceName);
}

Sequence< ::rtl::OUString > SAL_CALL Addon::getSupportedServiceNames(  )
    throw (RuntimeException)
{
    return Addon_getSupportedServiceNames();
}

```

Now we need to implement the `XDispatch` implementing class `DateTimeWriterDispatchImpl` in `addon.cxx` with helper functions to write date/time to cell A1

```cpp
// XDispatch implementer class "DateTimeWriterDispatchImpl" methods

void SAL_CALL DateTimeWriterDispatchImpl::dispatch( const URL& aURL, const Sequence < PropertyValue >& lArgs ) throw (RuntimeException)
{
    if ( aURL.Protocol.equalsAscii("inco.niocs.test.protocolhandler:") )
    {
	printf("DEBUG>>> DateTimeWriterDispatchImpl::dispatch() called. this = %p, command = %s\n", this,
	    OUStringToOString( aURL.Path, RTL_TEXTENCODING_ASCII_US ).getStr()); fflush(stdout);
        if ( aURL.Path.equalsAscii( "InsertDate" ) )
        {
            WriteCurrDate( mxFrame );
        }
        else if ( aURL.Path.equalsAscii( "InsertTime" ) )
        {
	    WriteCurrTime( mxFrame );
        }
    }
}

void SAL_CALL DateTimeWriterDispatchImpl::addStatusListener( const Reference< XStatusListener >& xControl, const URL& aURL ) throw (RuntimeException)
{
}

void SAL_CALL DateTimeWriterDispatchImpl::removeStatusListener( const Reference< XStatusListener >& xControl, const URL& aURL ) throw (RuntimeException)
{
}


// Helper functions to write current date/time to sheet

// Local function to write a string to cell A1

void WriteStringToCell( Reference< XFrame > &rxFrame, OUString aStr )
{
    if ( !rxFrame.is() )
	return;
    
    Reference< XController > xCtrl = rxFrame->getController();
    if ( !xCtrl.is() )
	return;

    Reference< XModel > xModel = xCtrl->getModel();
    if ( !xModel.is() )
	return;

    Reference< XSpreadsheetDocument > xSpreadsheetDocument( xModel, UNO_QUERY );
    if ( !xSpreadsheetDocument.is() )
	return;

    Reference< XSpreadsheets > xSpreadsheets = xSpreadsheetDocument->getSheets();
    if ( !xSpreadsheets->hasByName("Sheet1") )
	return;

    Any aSheet = xSpreadsheets->getByName("Sheet1");

    Reference< XSpreadsheet > xSpreadsheet( aSheet, UNO_QUERY );

    Reference< XCell > xCell = xSpreadsheet->getCellByPosition(0, 0);

    xCell->setFormula(aStr);
    
    printf("DEBUG>>> Wrote \"%s\" to Cell A1\n",
	OUStringToOString( aStr, RTL_TEXTENCODING_ASCII_US ).getStr()); fflush(stdout);
}

void GetCurrDateTime( oslDateTime* pDateTime )
{
    if ( !pDateTime )
	return;
    TimeValue aTimeval;
    osl_getSystemTime( &aTimeval );
    osl_getDateTimeFromTimeValue( &aTimeval, pDateTime );
}

// Local function to write Date to cell A1
void WriteCurrDate( Reference< XFrame > &rxFrame )
{
    char buf[12];
    oslDateTime aDateTime;
    GetCurrDateTime( &aDateTime );
    sprintf(buf, "%04d/%02d/%02d", aDateTime.Year, aDateTime.Month, aDateTime.Day);
    WriteStringToCell( rxFrame, OUString::createFromAscii(buf) );
}

// Local function to write Time to cell A1
void WriteCurrTime( Reference< XFrame > &rxFrame )
{
    char buf[10];
    oslDateTime aDateTime;
    GetCurrDateTime( &aDateTime );
    sprintf(buf, "%02d%02d%02d", aDateTime.Hours, aDateTime.Minutes, aDateTime.Seconds);
    WriteStringToCell( rxFrame, OUString::createFromAscii(buf) );
}


```

The final step is to write component operation function to register our component in `component.cxx`. Since we have only one component to register, we use a different approach than in [Part4](part4.md).

```cpp
/**
 * This function is called to get service factories for an implementation.
 *
 * @param pImplName       name of implementation
 * @param pServiceManager a service manager, need for component creation
 * @param pRegistryKey    the registry key for this component, need for persistent data
 * @return a component factory
 */
extern "C" SAL_DLLPUBLIC_EXPORT void * SAL_CALL component_getFactory(const sal_Char * pImplName, void * /*pServiceManager*/, void * pRegistryKey)
{
    void * pRet = 0;

    if (rtl_str_compare( pImplName, IMPLEMENTATION_NAME ) == 0)
    {
        Reference< XSingleComponentFactory > xFactory( createSingleComponentFactory(
            Addon_createInstance,
            OUString( IMPLEMENTATION_NAME ),
            Addon_getSupportedServiceNames() ) );

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

___

##<a name="buildsec"></a>Download, build and test the extension

**Please remember to remove any previous extensions from Part5 using `$LOROOT/instdir/program/unopkg remove`**

The whole project can be downloaded and built be doing :

```
$ git clone https://github.com/niocs/ProtocolHandlerExtension.git
$ cd ProtocolHandlerExtension
$ make
$ $LOROOT/instdir/program/unopkg add /home/$username/libreoffice5.3_sdk/LINUXexample.out/bin/ProtocolHandlerExtension.oxt
```

To remove the extension, do :
```
$ $LOROOT/instdir/program/unopkg remove ProtocolHandlerExtension.oxt
```

On opening Calc you should notice the top level menu called `Add-On example` with menu entries `Insert Date` and `Insert Time`. Add some text to any cell and try clicking on the new menu items and see if the text gets bold/italicized. The two new toolbar icons ![date icon](date.png) and ![time icon](time.png) also should be visible. On clicking `Insert Date`/`Insert Time`, the current date/time(UTC)  will be inserted to cell A1. You should also see the debug statements printed in the terminal as below when LO starts and you click the newly added UI elements.

```
## Just after Calc Loads.

DEBUG>>> Created Addon object : 0x2bfebd0
DEBUG>>> Addon::queryDispatch() called. this = 0x2bfebd0, command = InsertDate
DEBUG>>> Created DateTimeWriterDispatchImpl object : 0x2bfdeb0
DEBUG>>> Created Addon object : 0x2bff5d0
DEBUG>>> Addon::queryDispatch() called. this = 0x2bff5d0, command = InsertTime
DEBUG>>> Created DateTimeWriterDispatchImpl object : 0x2bff6e0

DEBUG>>> Created Addon object : 0x2ff5740
DEBUG>>> Addon::queryDispatch() called. this = 0x2ff5740, command = InsertDate
DEBUG>>> Created DateTimeWriterDispatchImpl object : 0x2ffbb50
DEBUG>>> Created Addon object : 0x2ff5860
DEBUG>>> Addon::queryDispatch() called. this = 0x2ff5860, command = InsertTime
DEBUG>>> Created DateTimeWriterDispatchImpl object : 0x2ff59a0

### After you click the UI elements : 

DEBUG>>> DateTimeWriterDispatchImpl::dispatch() called. this = 0x2ffbb50, command = InsertDate
DEBUG>>> Wrote "2016/10/26" to Cell A1
DEBUG>>> DateTimeWriterDispatchImpl::dispatch() called. this = 0x2ff59a0, command = InsertTime
DEBUG>>> Wrote "053312" to Cell A1
DEBUG>>> DateTimeWriterDispatchImpl::dispatch() called. this = 0x2bfdeb0, command = InsertDate
DEBUG>>> Wrote "2016/10/26" to Cell A1
DEBUG>>> DateTimeWriterDispatchImpl::dispatch() called. this = 0x2bff6e0, command = InsertTime
DEBUG>>> Wrote "053316" to Cell A1
```
___


## <a name="prebuilt"></a>Test the prebuilt extension

**Please remember to remove any previous extension from Part5 using `unopkg remove`.**

Download the extension as :
```
$ git clone https://github.com/niocs/ProtocolHandlerExtension.git
$ cd ProtocolHandlerExtension
```

In `ProtocolHandlerExtension` directory you will see a file called `ProtocolHandlerExtension.oxt`. This is the prebuilt extension. Install this extension using

```
$ unopkg add /path/to/ProtocolHandlerExtension.oxt
```

To remove this extension do :

```
$ unopkg remove ProtocolHandlerExtension.oxt
```

On opening calc you should see the top level menu called `Add-On example` with menu entries `Insert Date` and `Insert Time`. The two new toolbar icons ![date icon](date.png) and ![time icon](time.png) also should be visible. On clicking `Insert Date`/`Insert Time`, the current date/time(UTC) will be inserted to cell A1.