# Libreoffice extension development with C++ - Part 9 - Adding a context menu.

### Audience selection

1. Audience type **[AUD-C](README.md)** can jump directly to **[Test the prebuilt extension](#prebuilt)**
2. Audience type **[AUD-B](README.md)** can jump directly to **[Download, build and test the extension](#buildsec)**

___

Context menu is one that appears when we right-click on a sheet. In this part we are going to add an entry to context menu and assign action to it.

In order to register a new entry in LO context menu, we need access to `Frame` object of the current window. For this we are going to make a `AsyncJob` component that gets executed via a normal addon menu in the menubar. Since we already know how to implement `AsyncJob` component, we here focus on how we use this component to register a **new context menu entry**.

A context menu entry can be added by implementing `com::sun::star::ui::XContextMenuInterceptor` interface which has a single method :
```cpp
::com::sun::star::ui::ContextMenuInterceptorAction SAL_CALL notifyContextMenuExecute(
	const ::com::sun::star::ui::ContextMenuExecuteEvent& rEvent )
	throw ( ::com::sun::star::uno::RuntimeException ); 
```

and then we register this implementation using `XContextMenuInterception` object's `registerContextMenuInterceptor()` method inside `executeAsync()` of our `AsyncJob` component implementation `RegisterInterceptorJobImpl`. Let us now focus on implementing `com::sun::star::ui::XContextMenuInterceptor`

```cpp
// interceptor.hxx
// XContextMenuInterceptor implementer class
class ContextMenuInterceptorImpl : public cppu::WeakImplHelper1< ::com::sun::star::ui::XContextMenuInterceptor >
{

public:

    ContextMenuInterceptorImpl()
    {
	printf("DEBUG>>> Created ContextMenuInterceptorImpl object : %p\n", this); fflush(stdout);
    }

    virtual ::com::sun::star::ui::ContextMenuInterceptorAction SAL_CALL notifyContextMenuExecute(
	const ::com::sun::star::ui::ContextMenuExecuteEvent& rEvent )
	throw ( ::com::sun::star::uno::RuntimeException ); 
};


// interceptor.cxx

ContextMenuInterceptorAction SAL_CALL ContextMenuInterceptorImpl::notifyContextMenuExecute(
    const ContextMenuExecuteEvent& rEvent )
    throw ( RuntimeException )
{

    printf("DEBUG>>> Inside notifyContextMenuExecute : this = %p\n", this); fflush(stdout);
    try {
	Reference< XIndexContainer > xContextMenu = rEvent.ActionTriggerContainer;
	if ( !xContextMenu.is() )
	{
	    logError("DEBUG>>> notifyContextMenuExecute : bad rEvent.ActionTriggerContainer\n");
	    return ContextMenuInterceptorAction_IGNORED;
	}
	Reference< XMultiServiceFactory > xMenuElementFactory( xContextMenu, UNO_QUERY );
	if ( !xMenuElementFactory.is() )
	{
	    logError("DEBUG>>> notifyContextMenuExecute : bad xMenuElementFactory\n");
	    return ContextMenuInterceptorAction_IGNORED;
	}

	Reference< XPropertySet > xSeparator( xMenuElementFactory->createInstance( "com.sun.star.ui.ActionTriggerSeparator" ), UNO_QUERY );
	if ( !xSeparator.is() )
	{
	    logError("DEBUG>>> notifyContextMenuExecute : cannot create xSeparator\n");
	    return ContextMenuInterceptorAction_IGNORED;
	}

	xSeparator->setPropertyValue( "SeparatorType", makeAny( ActionTriggerSeparatorType::LINE ) );
	
	Reference< XPropertySet > xMenuEntry( xMenuElementFactory->createInstance( "com.sun.star.ui.ActionTrigger" ), UNO_QUERY );
	if ( !xMenuEntry.is() )
	{
	    logError("DEBUG>>> notifyContextMenuExecute : cannot create xMenuEntry\n");
	    return ContextMenuInterceptorAction_IGNORED;
	}

	xMenuEntry->setPropertyValue( "Text", makeAny(OUString("Increment cell(s)")));
	xMenuEntry->setPropertyValue( "CommandURL", makeAny( OUString("vnd.sun.star.job:event=onIncrementClick") ) );

	sal_Int32 nCount = xContextMenu->getCount();
	sal_Int32 nIdx = nCount;
	xContextMenu->insertByIndex( nIdx++, makeAny( xSeparator ) );
	xContextMenu->insertByIndex( nIdx++, makeAny( xMenuEntry ) );

	return ContextMenuInterceptorAction_CONTINUE_MODIFIED;
    }
    catch ( Exception& e )
    {
	fprintf(stderr, "DEBUG>>> notifyContextMenuExecute : caught UNO exception: %s\n",
		OUStringToOString( e.Message, RTL_TEXTENCODING_ASCII_US ).getStr());
	fflush(stderr);
    }

    return ContextMenuInterceptorAction_IGNORED;
}

```

The method `notifyContextMenuExecute()` gets a struct object typed `ContextMenuExecuteEvent`. This struct contains the following fields :

1. ActionTriggerContainer : This is a collection of recursive menu entries. It also acts as a factory for creating new menu entries.
2. ExecutePosition : This contains info on the coordinates where context menu will be created.
3. SourceWindow : Contains the window where the context menu has been requested.
4. Selection : Provides the current selection inside the source window.

There are two types of menu entries :

1. com.sun.star.ui.ActionTrigger : Normal menu entry with actions.
2. com.sun.star.ui.ActionTriggerSeparator : A menu entry which is just a separator.

In the above code we create each of these menu entries using `ContextMenuExecuteEvent::ActionTriggerContainer`'s `XMultiServiceFactory` method `createInstance()`.

`com.sun.star.ui.ActionTrigger` menu entry has the following properties :

1. Text : This is the text label of the menu entry
2. CommandURL : This is where we assign the command URL which will be executed if the menu entry is selected by the user. We need to implement a `ProtocolHandler` or `AsyncJob` or `Job` component to handle this URL. In this case we just reuse our `AsyncJob` component `RegisterInterceptorJobImpl`. The URL we use here is `vnd.sun.star.job:event=onIncrementClick`
3. Image : This property contains an image that is shown left of the menu label. The use is optional so that no image is used if this member is not initialized.
4. SubContainer : This property contains an optional sub menu.

`com.sun.star.ui.ActionTriggerSeparator` menu entry has one optional property : `SeparatorType` which can take any of the values :
1. LINE
2. SPACE
3. LINEBREAK

Next after creating these new menu entries we insert them to the `ActionTriggerContainer` member of `ContextMenuExecuteEvent` object we received using the method `insertByIndex()`.

Finally the method `notifyContextMenuExecute()` needs to return one of the following possibilities ( of type `com.sun.star.ui.ContextMenuInterceptorAction` ) :

1. IGNORED : This indicates that our interceptor has ignored the call and we don't want any modifications done to the context menu. The next registered `com.sun.star.ui.XContextMenuInterceptor` would then be notified.
2. CANCELLED : The context menu must not be executed (ie no menu is shown). No remaining interceptor will be called.
3. EXECUTE_MODIFIED : We modified the context menu and should be executed without notifying the next registered `com.sun.star.ui.XContextMenuInterceptor`.
4. CONTINUE_MODIFIED : We modified the context menu and next registered interceptor should be notified.

We want our `XContextMenuInterceptor` implementation `ContextMenuInterceptorImpl` to be instantiated only once per LO lifetime. We could do this by making it a singleton, but in this case we make use of C++'s `static` variable to make it a singleton manually. For this we write a private method `getInterceptor` in the AsyncJob implementation class. This method accepts the current `XFrame` object and a reference to an integer that indicates the number of times this method was called for the current `XFrame` object. This method has a static variable to hold on to uno reference to `XContextMenuInterceptor` we implemented and makes sure we instantiate it only once.

```cpp
Reference< XContextMenuInterceptor >& RegisterInterceptorJobImpl::getInterceptor( const Reference<XFrame>& xFrame, sal_Int32& rCalls )
{
    static Reference< XContextMenuInterceptor > xInterceptor;
    static std::map<std::uintptr_t, sal_Int32> aFrame2Calls;
    if ( !xInterceptor.is() )
	xInterceptor = Reference< XContextMenuInterceptor >( (cppu::OWeakObject*) new ContextMenuInterceptorImpl(), UNO_QUERY );

    std::uintptr_t nFramePtrVal = reinterpret_cast<std::uintptr_t>( xFrame.get() );
    auto aItr = aFrame2Calls.find( nFramePtrVal );
    if ( aItr != aFrame2Calls.end() )
	rCalls = ++(aItr->second);
    else
    {
	aFrame2Calls[ nFramePtrVal ] = 1;
	rCalls = 1;
    }
    return xInterceptor;
}

```

Next we handle the events `onEnableInterceptClick1` ( event produced by the normal menu entry to enable the context menu ) and `onIncrementClick` ( event produced by the context menu ) in `executeAsync()` of our AsyncJob.

```cpp
if (  aEventName.equalsAscii("onEnableInterceptClick1") )
{
    sal_Int32 nNumCalls = 0;
    Reference< XContextMenuInterceptor > xInterceptor = getInterceptor( xFrame, nNumCalls );
    if ( nNumCalls > 1 )
    {
        printf("DEBUG>>> Interceptor is already enabled\n");
	fflush(stdout);
    }
    else
    {
        Reference< XController > xController = xFrame->getController();
        if ( xController.is() )
        {
            Reference< XContextMenuInterception > xContextMenuInterception( xController, UNO_QUERY );
            if ( xContextMenuInterception.is() )
            {
                xContextMenuInterception->registerContextMenuInterceptor( xInterceptor );
                printf("DEBUG>>> Registered ContextMenuInterceptorImpl to current frame controller.\n"); fflush(stdout);
            }
        }
    }
} else if ( aEventName.equalsAscii("onIncrementClick") ) {
    printf("DEBUG>>> Got request from DISPATCH envType and event name onIncrementClick, going to call IncrementCellValue().\n"); fflush(stdout);
    IncrementMarkedCellValues( xFrame );
}

```

`IncrementMarkedCellValues()` is executed as an action to user selecting our context menu entry. It imcrements all numerical cells by 1 under current selection.


##<a name="buildsec"></a>Download, build and test the extension

The whole project can be downloaded, built and installed by doing :

```
$ git clone https://github.com/niocs/CtxtMenuInterceptor.git
$ cd CtxtMenuInterceptor
$ make
$ $LOROOT/instdir/program/unopkg add /home/$username/libreoffice5.3_sdk/LINUXexample.out/bin/CtxtMenuInterceptor.oxt
```

To remove the extension, do :
```
$ $LOROOT/instdir/program/unopkg remove CtxtMenuInterceptor.oxt
```

Click on the menu `Add-On example > Enable context menu`. Now you should see something like below in the terminal.

```
DEBUG>>> Created RegisterInterceptorJobImpl object : 0x7ff86e8382e8
DEBUG>>> Called executeAsync() : this = 0x7ff86e8382e8
DEBUG>>> xFrame = 0x7ff88134ec78
DEBUG>>> Created ContextMenuInterceptorImpl object : 0x7ff86e831c10
DEBUG>>> Registered ContextMenuInterceptorImpl to current frame controller.
```

If you click again on the same menu, you should see the following in terminal indicating that the object `ContextMenuInterceptorImpl` is instantiated only once.

```
DEBUG>>> Created RegisterInterceptorJobImpl object : 0x7ff86e8383d8
DEBUG>>> Called executeAsync() : this = 0x7ff86e8383d8
DEBUG>>> xFrame = 0x7ff88134ec78
DEBUG>>> Interceptor is already enabled
```

Now populate few cells with **numbers** and place cursor on them or select them and right click. You should see the menu entry `Increment cell(s)` and a menu-separator just above that. When you click it the cells you selected with numbers should increment by 1. The following would be displayed in terminal on doing that :

```
DEBUG>>> Inside notifyContextMenuExecute : this = 0x7ff86e831c10
DEBUG>>> Created RegisterInterceptorJobImpl object : 0x7ff880089570
DEBUG>>> Called executeAsync() : this = 0x7ff880089570
DEBUG>>> xFrame = 0x7ff88134ec78
DEBUG>>> Got request from DISPATCH envType and event name onIncrementClick, going to call IncrementCellValue().
```

If you clicked `Increment cell(s)` context menu with non-numeric data in current/selected cell(s), then those cells will not be modified. The terminal would say something like :

```
DEBUG>>> Got request from DISPATCH envType and event name onIncrementClick, going to call IncrementCellValue().
DEBUG>>> IncrementCellRange : cell type is not numeric, skipping.
```


## <a name="prebuilt"></a>Test the prebuilt extension

Download the extension as :
```
$ git clone https://github.com/niocs/CtxtMenuInterceptor.git
$ cd CtxtMenuInterceptor
```

In `CtxtMenuInterceptor` directory you will see a file called `CtxtMenuInterceptor.oxt`. This is the prebuilt extension. Install this extension using

```
$ unopkg add /path/to/CtxtMenuInterceptor.oxt
```

To remove this extension do :

```
$ unopkg remove CtxtMenuInterceptor.oxt
```

Open calc and click on the menu `Add-On example > Enable context menu`. This should enable our context menu entry from now on. Now populate few cells with **numbers** and place cursor on them or select them and right click. You should see the menu entry `Increment cell(s)` and a menu-separator just above that. When you click it, the cells you selected with numbers should increment by 1.