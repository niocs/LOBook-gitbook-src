# Libreoffice extension development with C++ - Part 7 - AsyncJob Component.

### Audience selection

1. Audience type **[AUD-C](README.md)** can jump directly to **[Test the prebuilt extension](#prebuilt)**
2. Audience type **[AUD-B](README.md)** can jump directly to **[Download, build and test the extension](#buildsec)**

___

In this part we are going to see a more flexible alternative to ProtocolHandler component we say in [Part6](part6.md).
A *job* in LO is a UNO component that can be executed by the job execution environment upon an event. The job is protected from office termination.
The job component must implement the service `com.sun.star.task.Job` if it needs to block the thread in which it is executed or implement `com.sun.star.task.AsyncJob` if needed to be executed in a separate thread. If a job component implements both of these services, then the synchronous service is preferred by the job execution environment. Note that synchronous jobs should not display any user interfaces from it as repaint errors and threading issues will occur. In this part we will focus on implementing `AsyncJob` service.

The `AsyncJob` service supports the interface `com::sun::star::task::XAsyncJob` which has only one method `executeAsync()` with the following signature :

```cpp
void executeAsync(
     [in] sequence< com::sun::star::beans::NamedValue > Arguments,
     [in] XJobListener Listener)
          raises( com::sun::star::lang::IllegalArgumentException );
```

The `Arguments` paramter is a sequence of `NamedValue` structs. This contains information on the environment where the job is running, ie it tells if the job was created by the job executor or the dispatch framework or global event broadcaster service and possibly provides a document model or frame to work with.

As we are already familiar with writing components, we focus on just writing the `executeAsync()` method and methods supporting it.

Lets declare our class `AsyncJobImpl` which implements `XAsyncJob` interface.

```cpp
#define IMPLEMENTATION_NAME "inco.niocs.test.AsyncJobImpl"

class AsyncJobImpl : public cppu::WeakImplHelper2
<
    com::sun::star::task::XAsyncJob,
    com::sun::star::lang::XServiceInfo
>
{
    
private:
    ::com::sun::star::uno::Reference< ::com::sun::star::uno::XComponentContext > mxContext;

public:

    AsyncJobImpl( const ::com::sun::star::uno::Reference< ::com::sun::star::uno::XComponentContext > &rxContext )
        : mxContext( rxContext )
    {
        printf("DEBUG>>> Created AsyncJobImpl object : %p\n", this); fflush(stdout);
    }

    // XAsyncJob methods
    virtual void SAL_CALL executeAsync( const ::com::sun::star::uno::Sequence< ::com::sun::star::beans::NamedValue >& rArgs,
                                        const ::com::sun::star::uno::Reference< ::com::sun::star::task::XJobListener >& rxListener )
        throw(::com::sun::star::lang::IllegalArgumentException,
              ::com::sun::star::uno::RuntimeException) override;

    // XServiceInfo methods
    virtual ::rtl::OUString SAL_CALL getImplementationName()
        throw (::com::sun::star::uno::RuntimeException);
    virtual sal_Bool SAL_CALL supportsService( const ::rtl::OUString& aServiceName )
        throw (::com::sun::star::uno::RuntimeException);
    virtual ::com::sun::star::uno::Sequence< ::rtl::OUString > SAL_CALL getSupportedServiceNames()
        throw (::com::sun::star::uno::RuntimeException);

    // A struct to store some job related info when executeAsync() is called
    struct AsyncJobImplInfo
    {
        ::rtl::OUString aEnvType;
        ::rtl::OUString aEventName;
        ::rtl::OUString aAlias;
        ::com::sun::star::uno::Reference< ::com::sun::star::frame::XFrame > xFrame;

        ::rtl::OUString aGenericConfigList, aJobConfigList, aEnvironmentList, aDynamicDataList;
    };

private:

    ::rtl::OUString validateGetInfo( const ::com::sun::star::uno::Sequence< ::com::sun::star::beans::NamedValue >& rArgs,
                                     const ::com::sun::star::uno::Reference< ::com::sun::star::task::XJobListener >& rxListener,
                                     AsyncJobImplInfo& rJobInfo );
    
    void formatOutArgs( const ::com::sun::star::uno::Sequence< ::com::sun::star::beans::NamedValue >& rList,
                        const ::rtl::OUString aListName,
                        ::rtl::OUString& rFormattedList );

    void writeInfoToSheet( const AsyncJobImplInfo& rJobInfo );
};

```

Note that we declared three private methods which help us in implementing `executeAsync()` interface method.
`validateGetInfo()` takes the arguments received from `executeAsync()` and validates it and extracts needed information and fills it into a struct `AsyncJobImplInfo` which has the following fields :
1. *aEnvType* : This stores the type of enviroment. For instance, if our job was invoked from a menu/toolbar then this will have the string "DISPATCH".
2. *aEventName* : This indicate the name of the event if the job was called in response to an event.
3. *aAlias* : A Job can have an alias and jobs can be called via its alias. If this is the case then this member will contain the alias name used.
4. *xFrame* : This stores the Frame object passed on to `executeAsync()` for us to manipulate the UI/Model.
5. *aGenericConfigList* : Contains stringified version of the sublist in `executeAsync()`'s with name `Config`.
6. *aJobConfigList* : Contains stringified version of the sublist in `executeAsync()`'s with name `JobConfig`.
7. *aEnvironmentList* : Contains stringified version of the sublist in `executeAsync()`'s with name `Environment`.
8. *aDynamicDataList* : Contains stringified version of the sublist in `executeAsync()`'s with name `DynamicData`.

`formatOutArgs()` takes a sequence of `NamedValue` and stringifies it. `writeInfoToSheet()` takes the info struct object `AsyncJobImplInfo` and writes all info to `Sheet1` of the current Frame.
`executeAsync()` methods also has to call the method `jobFinished()` of the `XJobListener` object it received just before exiting the function.

Following are the implementations of these methods :

```cpp
// XAsyncJob method implementations

void SAL_CALL AsyncJobImpl::executeAsync( const Sequence<NamedValue>& rArgs,
                                          const Reference<XJobListener>& rxListener )
    throw(IllegalArgumentException, RuntimeException)
{
    printf("DEBUG>>> Called executeAsync() : this = %p\n", this); fflush(stdout);
    
    AsyncJobImplInfo aJobInfo;
    OUString aErr = validateGetInfo( rArgs, rxListener, aJobInfo );
    if ( !aErr.isEmpty() )
    {
        sal_Int16 nArgPos = 0;        
        if ( aErr.startsWith( "Listener" ) )
            nArgPos = 1;

        throw IllegalArgumentException(
            aErr,
            // resolve to XInterface reference:
            static_cast< ::cppu::OWeakObject * >(this),
            nArgPos ); // argument pos
    }
    
    writeInfoToSheet( aJobInfo );

    bool bIsDispatch = aJobInfo.aEnvType.equalsAscii("DISPATCH");
    Sequence<NamedValue> aReturn( ( bIsDispatch ? 1 : 0 ) );

    if ( bIsDispatch )
    {
        aReturn[0].Name  = "SendDispatchResult";
        DispatchResultEvent aResultEvent;
        aResultEvent.Source = (cppu::OWeakObject*)this;
        aResultEvent.State = DispatchResultState::SUCCESS;
        aResultEvent.Result <<= true;
        aReturn[0].Value <<= aResultEvent;
    }

    rxListener->jobFinished( Reference<com::sun::star::task::XAsyncJob>(this), makeAny(aReturn));
    
}


OUString AsyncJobImpl::validateGetInfo( const Sequence<NamedValue>& rArgs,
                                        const Reference<XJobListener>& rxListener,
                                        AsyncJobImpl::AsyncJobImplInfo& rJobInfo )
{
    if ( !rxListener.is() )
        return "Listener : invalid listener";

    // Extract all sublists from rArgs.
    Sequence<NamedValue> aGenericConfig;
    Sequence<NamedValue> aJobConfig;
    Sequence<NamedValue> aEnvironment;
    Sequence<NamedValue> aDynamicData;

    sal_Int32 nNumNVs = rArgs.getLength();
    for ( sal_Int32 nIdx = 0; nIdx < nNumNVs; ++nIdx )
    {
        if ( rArgs[nIdx].Name.equalsAscii("Config") )
        {
            rArgs[nIdx].Value >>= aGenericConfig;
            formatOutArgs( aGenericConfig, "Config", rJobInfo.aGenericConfigList );
        }
        else if ( rArgs[nIdx].Name.equalsAscii("JobConfig") )
        {
            rArgs[nIdx].Value >>= aJobConfig;
            formatOutArgs( aJobConfig, "JobConfig", rJobInfo.aJobConfigList );
        }
        else if ( rArgs[nIdx].Name.equalsAscii("Environment") )
        {
            rArgs[nIdx].Value >>= aEnvironment;
            formatOutArgs( aEnvironment, "Environment", rJobInfo.aEnvironmentList );
        }
        else if ( rArgs[nIdx].Name.equalsAscii("DynamicData") )
        {
            rArgs[nIdx].Value >>= aDynamicData;
            formatOutArgs( aDynamicData, "DynamicData", rJobInfo.aDynamicDataList );
        }
    }

    // Analyze the environment info. This sub list is the only guaranteed one!
    if ( !aEnvironment.hasElements() )
        return "Args : no environment";

    sal_Int32 nNumEnvEntries = aEnvironment.getLength();
    for ( sal_Int32 nIdx = 0; nIdx < nNumEnvEntries; ++nIdx )
    {
        if ( aEnvironment[nIdx].Name.equalsAscii("EnvType") )
            aEnvironment[nIdx].Value >>= rJobInfo.aEnvType;
        
        else if ( aEnvironment[nIdx].Name.equalsAscii("EventName") )
            aEnvironment[nIdx].Value >>= rJobInfo.aEventName;

        else if ( aEnvironment[nIdx].Name.equalsAscii("Frame") )
            aEnvironment[nIdx].Value >>= rJobInfo.xFrame;
    }

    // Further the environment property "EnvType" is required as minimum.

    if ( rJobInfo.aEnvType.isEmpty() ||
         ( ( !rJobInfo.aEnvType.equalsAscii("EXECUTOR") ) &&
           ( !rJobInfo.aEnvType.equalsAscii("DISPATCH") )
             )        )
        return OUString("Args : \"" + rJobInfo.aEnvType + "\" isn't a valid value for EnvType");

    // Analyze the set of shared config data.
    if ( aGenericConfig.hasElements() )
    {
        sal_Int32 nNumGenCfgEntries = aGenericConfig.getLength();
        for ( sal_Int32 nIdx = 0; nIdx < nNumGenCfgEntries; ++nIdx )
            if ( aGenericConfig[nIdx].Name.equalsAscii("Alias") )
                aGenericConfig[nIdx].Value >>= rJobInfo.aAlias;
    }

    return "";
}


void AsyncJobImpl::formatOutArgs( const Sequence<NamedValue>& rList,
                                  const OUString aListName,
                                  OUString& rFormattedList )
{
    OUStringBuffer aBuf( 512 );
    aBuf.append("List \"" + aListName + "\" : ");
    if ( !rList.hasElements() )
        aBuf.append("0 items");
    else
    {
        sal_Int32 nNumEntries = rList.getLength();
        aBuf.append(nNumEntries).append(" item(s) : { ");
        for ( sal_Int32 nIdx = 0; nIdx < nNumEntries; ++nIdx )
        {
            OUString aVal;
            rList[nIdx].Value >>= aVal;
            aBuf.append("\"" + rList[nIdx].Name + "\" : \"" + aVal + "\", ");
        }
        aBuf.append(" }");
    }

    rFormattedList = aBuf.toString();
}


void WriteStringsToSheet( const Reference< XFrame > &rxFrame, const std::vector<OUString>& rStrings )
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

    size_t nNumStrings = rStrings.size();
    for ( size_t nIdx = 0; nIdx < nNumStrings; ++nIdx )
    {
        Reference< XCell > xCell = xSpreadsheet->getCellByPosition(0, nIdx);
        xCell->setFormula(rStrings[nIdx]);
        printf("DEBUG>>> Wrote \"%s\" to Cell A%d\n",
               OUStringToOString( rStrings[nIdx], RTL_TEXTENCODING_ASCII_US ).getStr(), nIdx); fflush(stdout);
    }
}

void AsyncJobImpl::writeInfoToSheet( const AsyncJobImpl::AsyncJobImplInfo& rJobInfo )
{
    if ( !rJobInfo.xFrame.is() )
    {
        printf("DEBUG>>> Frame passed is null, cannot write to sheet !\n");
        fflush(stdout);
    }
    std::vector<OUString> aStrList = {
        "EnvType = " + rJobInfo.aEnvType,
        "EventName = " + rJobInfo.aEventName,
        "Alias = " + rJobInfo.aAlias,
        rJobInfo.aGenericConfigList,
        rJobInfo.aJobConfigList,
        rJobInfo.aEnvironmentList,
        rJobInfo.aDynamicDataList
    };

    WriteStringsToSheet( rJobInfo.xFrame, aStrList );
}

```

A Job component can be called from the menu or toolbar buttons using any of the three following methods :

1. By *Event name* : A Job component can be configured to be called when a particular event is raised. A UI element can raise an event by using a URL of the form `vnd.sun.star.job:event=<EVENTNAME>`, where `EVENTNAME` can be any of the [standard events](https://wiki.openoffice.org/wiki/Documentation/DevGuide/WritingUNO/Jobs/List_of_Supported_Events) or a custom event registered by the Job component.
2. By *Alias* : A Job can be called from the UI element by its alias by using the URL of the form `vnd.sun.star.job:alias=<ALIAS_OF_THE_JOB>`.
3. By *Implementation name* : A Job can be called using its implementation name by using URL of form : `vnd.sun.star.job:service=<IMPLEMENTAION_NAME>` example :  `vnd.sun.star.job:service=inco.niocs.test.AsyncJobImpl`.

We specify our job's alias, attached events in an xml file called `Job.xcu`.

```xml
<oor:component-data oor:name="Jobs" oor:package="org.openoffice.Office" xmlns:oor="http://openoffice.org/2001/registry" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <node oor:name="Jobs">
        <!--            vvvvvvvvvvv    AsyncJobImpl is the alias -->
        <node oor:name="AsyncJobImpl" oor:op="replace">
            <prop oor:name="Service" oor:type="xs:string">
                <value>inco.niocs.test.AsyncJobImpl</value>
            </prop>
        </node>
    </node>
    <node oor:name="Events">
        <!--
        <node oor:name="onFirstVisibleTask" oor:op="modify">
            <node oor:name="JobList">
                <node oor:name="AsyncJobImpl" oor:op="replace"/>
            </node>
        </node>
        -->
        <node oor:name="onMyOwnJobEvent" oor:op="replace">
            <node oor:name="JobList">
                <node oor:name="AsyncJobImpl" oor:op="replace"/>
            </node>
        </node>
    </node>
</oor:component-data>
```

We now add menu entries to call our job in the three different ways discussed before.

```xml
<oor:component-data xmlns:oor="http://openoffice.org/2001/registry" xmlns:xs="http://www.w3.org/2001/XMLSchema" oor:name="Addons" oor:package="org.openoffice.Office">
    <node oor:name="AddonUI">
        <node oor:name="OfficeMenuBar">
            <node oor:name="inco.niocs.test.asyncjobimpl.menu" oor:op="replace">
                <prop oor:name="Title" oor:type="xs:string">
                    <value/>
                    <value xml:lang="en-US">Add-On example</value>
                </prop>
                <node oor:name="Submenu">
                    <node oor:name="m1" oor:op="replace">
                        <prop oor:name="URL" oor:type="xs:string">
                            <value>vnd.sun.star.job:alias=AsyncJobImpl</value>
                        </prop>
                        <prop oor:name="Title" oor:type="xs:string">
                            <value xml:lang="en-US">AsyncJob (ALIAS)</value>
                        </prop>
                        <prop oor:name="Target" oor:type="xs:string">
                            <value>_self</value>
                        </prop>
                        <prop oor:name="Context" oor:type="xs:string">
                            <value>com.sun.star.sheet.SpreadsheetDocument</value>
                        </prop>
                    </node>
                    <node oor:name="m2" oor:op="replace">
                        <prop oor:name="URL" oor:type="xs:string">
                            <value>vnd.sun.star.job:event=onMyOwnJobEvent</value>
                        </prop>
                        <prop oor:name="Title" oor:type="xs:string">
                            <value xml:lang="en-US">AsyncJob (EVENT)</value>
                        </prop>
                        <prop oor:name="Target" oor:type="xs:string">
                            <value>_self</value>
                        </prop>
                        <prop oor:name="Context" oor:type="xs:string">
                            <value>com.sun.star.sheet.SpreadsheetDocument</value>
                        </prop>
                    </node>
                    <node oor:name="m3" oor:op="replace">
                        <prop oor:name="URL" oor:type="xs:string">
                            <value>vnd.sun.star.job:service=inco.niocs.test.AsyncJobImpl</value>
                        </prop>
                        <prop oor:name="Title" oor:type="xs:string">
                            <value xml:lang="en-US">AsyncJob (SERVICE)</value>
                        </prop>
                        <prop oor:name="Target" oor:type="xs:string">
                            <value>_self</value>
                        </prop>
                        <prop oor:name="Context" oor:type="xs:string">
                            <value>com.sun.star.sheet.SpreadsheetDocument</value>
                        </prop>
                    </node>
                </node>
            </node>
        </node>
    </node>
</oor:component-data>
```

##<a name="buildsec"></a>Download, build and test the extension

**Please remember to remove any previous extensions from Part5/Part6 using `$LOROOT/instdir/program/unopkg remove`**

The whole project can be downloaded and built by doing :

```
$ git clone https://github.com/niocs/AsyncJobExtension.git
$ cd AsyncJobExtension
$ make
$ $LOROOT/instdir/program/unopkg add /home/$username/libreoffice5.3_sdk/LINUXexample.out/bin/AsyncJobExtension.oxt
```

To remove the extension, do :
```
$ $LOROOT/instdir/program/unopkg remove AsyncJobExtension.oxt
```

On running Calc you will notice there are three menu entries inside the menu `Add-On example` each for triggering our component via three methods :
1. Call via Event name.
2. Call via Implementation name(service).
3. Call via Alias.

On clicking each of them, Sheet1's column A will be filled with job environment information. Depending on the menu entry you click, some of the content will be blank.

You will also notice the debug info printed on the terminal on each click. It would look something like :
```
DEBUG>>> Created AsyncJobImpl object : 0x3e78f10
DEBUG>>> Called executeAsync() : this = 0x3e78f10
DEBUG>>> Wrote "EnvType = DISPATCH" to Cell A0
DEBUG>>> Wrote "EventName = " to Cell A1
DEBUG>>> Wrote "Alias = AsyncJobImpl" to Cell A2
DEBUG>>> Wrote "List "Config" : 3 item(s) : { "Alias" : "AsyncJobImpl", "Service" : "inco.niocs.test.AsyncJobImpl", "Context" : "",  }" to Cell A3
DEBUG>>> Wrote "" to Cell A4
DEBUG>>> Wrote "List "Environment" : 2 item(s) : { "EnvType" : "DISPATCH", "Frame" : "",  }" to Cell A5
DEBUG>>> Wrote "List "DynamicData" : 1 item(s) : { "Referer" : "private:user",  }" to Cell A6
DEBUG>>> Created AsyncJobImpl object : 0x4772690
DEBUG>>> Called executeAsync() : this = 0x4772690
DEBUG>>> Wrote "EnvType = DISPATCH" to Cell A0
DEBUG>>> Wrote "EventName = onMyOwnJobEvent" to Cell A1
DEBUG>>> Wrote "Alias = " to Cell A2
DEBUG>>> Wrote "" to Cell A3
DEBUG>>> Wrote "" to Cell A4
DEBUG>>> Wrote "List "Environment" : 3 item(s) : { "EnvType" : "DISPATCH", "Frame" : "", "EventName" : "onMyOwnJobEvent",  }" to Cell A5
DEBUG>>> Wrote "List "DynamicData" : 1 item(s) : { "Referer" : "private:user",  }" to Cell A6
DEBUG>>> Created AsyncJobImpl object : 0x4780a70
DEBUG>>> Called executeAsync() : this = 0x4780a70
DEBUG>>> Wrote "EnvType = DISPATCH" to Cell A0
DEBUG>>> Wrote "EventName = " to Cell A1
DEBUG>>> Wrote "Alias = " to Cell A2
DEBUG>>> Wrote "" to Cell A3
DEBUG>>> Wrote "" to Cell A4
DEBUG>>> Wrote "List "Environment" : 2 item(s) : { "EnvType" : "DISPATCH", "Frame" : "",  }" to Cell A5
DEBUG>>> Wrote "List "DynamicData" : 1 item(s) : { "Referer" : "private:user",  }" to Cell A6
```

*NOTE : From the above we can note that each time a event/trigger is made our class `AsyncJobImpl` is instantiated.*

## <a name="prebuilt"></a>Test the prebuilt extension

**Please remember to remove any previous extension from Part5/Part6 using `unopkg remove`.**

Download the extension as :
```
$ git clone https://github.com/niocs/AsyncJobExtension.git
$ cd AsyncJobExtension
```

In `AsyncJobExtension` directory you will see a file called `AsyncJobExtension.oxt`. This is the prebuilt extension. Install this extension using

```
$ unopkg add /path/to/AsyncJobExtension.oxt
```

To remove this extension do :

```
$ unopkg remove AsyncJobExtension.oxt
```

On running Calc you will notice there are three menu entries inside the menu `Add-On example` each for triggering our component via three methods :
1. Call via Event name.
2. Call via Implementation name(service).
3. Call via Alias.

On clicking each of them, Sheet1's column A will be filled with job environment information. Depending on the menu entry you click, some of the content will be blank.
