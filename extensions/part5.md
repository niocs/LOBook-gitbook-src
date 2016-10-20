# Libreoffice extension development with C++ - Part 5 - Build a simple addon with a menu and toolbar buttons.

### Audience selection

1. Audience type **[AUD-C](README.md)** can jump directly to **[Test the prebuilt extension](#prebuilt)**
2. Audience type **[AUD-B](README.md)** can jump directly to **[Download, build and test the extension](#buildsec)**

___


An **addon** is a type of extension which adds UI elements to LO and allows user interaction.
LO uses *"Command URLs"* for communication between UI layer and the application objects.
> When a user chooses an item in the user interface, a command URL is dispatched to the application framework and processed in a chain of responsibility until an object accepts the command and executes it, thus consuming the command URL. This mechanism is known as the *dispatch framework*

The user interface of LO can be changed by an addon extension by providing an xml file with *.xcu* extension in the *oxt* file. We can add new menus and toolbar items and configure them to send command URLS. It is also possible to disable existing commands. We will see in part6 and part7 that we can make components that can handle/consume commandURLs from the UI and run actions for them.

The command URL is of the form
```
<Protocol>:<Path>
```
Example :
```
inco.niocs.test.extension:InsertDate
```
In this case the protocol part is `inco.niocs.test.extension` and the path is `InsertDate`. When creating new command URLs, we should use a protocol part that is likely to be unique. For example using a scheme like `com.<companyname>.<company-namespace>.<extension-namespace>` for protocol part would ensure uniqueness across all command URLs in LO.

In this part, we add a new menu called `Add-On example` with menu entry called `Make text Bold`. On user click it would send the LO built-in command URL `.uno:Bold`. Here the protocol part is `.uno` and the path is `Bold`. On issuing this command to the dispatch framework of LO, it would toggle the selected text's bold attribute.

To add this menu we do not need any C++ component as LO already has the functionality to handle the command `.uno:Bold`. We only need to create an xml file as follows :

**Addons.xcu**
```xml
<oor:component-data xmlns:oor="http://openoffice.org/2001/registry" xmlns:xs="http://www.w3.org/2001/XMLSchema" oor:name="Addons" oor:package="org.openoffice.Office">
    <node oor:name="AddonUI">
        <!--  We add our custom menus here -->
    </node>
</oor:component-data>

```
The above xml is the same for all addons, so no need to change any values in it.

Now we add xml content inside the above code to create a custom menu `Add-On example` inside the LO's menu bar.

```xml
<node oor:name="OfficeMenuBar">
    <node oor:name="inco.niocs.test.addon.example" oor:op="replace">
        <prop oor:name="Title" oor:type="xs:string">
            <value xml:lang="en-US">Add-On example</value>
        </prop>
        <prop oor:name="Target" oor:type="xs:string">
            <value>_self</value>
        </prop>
	<!--   INSERT SUBMENU's here  -->
    </node>
</node>
```

The top level node of the above xml segment has `name` attribute set as `OfficeMenuBar`. This indicates that the menu we are going to add is a top-level menu in LO menu bar.
The child node should have an unique value for `name` attribute for the newly inserted menu and should have `op` set as `replace`. The replace operation must be used to add a new node to a set or extensible node. Thus the real meaning of the operation is "add or replace".

Teh child elements of our menu node should have `prop` nodes with further `value` child nodes for the properties `Title` and `Target`.
We set property `Title` to the value `Add-On example` which is the display title of the new menu. The `Target` property can have one of `_top`, `_parent`, `_self`, `_blank` indicating which frame of LO should accept the command URL emitted from the menu. We choose `_self` so that the current frame get the command.

Now we need to add xml code for inserting the menu item titled `Make text Bold` that produces the command `.uno:Bold`.

```xml
<node oor:name="Submenu">
    <node oor:name="m1" oor:op="replace">
        <prop oor:name="URL" oor:type="xs:string">
            <value>.uno:Bold</value>
        </prop>
        <prop oor:name="Title" oor:type="xs:string">
            <value xml:lang="en-US">Make text Bold</value>
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
            <value>.uno:Italic</value>
        </prop>
        <prop oor:name="Title" oor:type="xs:string">
            <value xml:lang="en-US">Make text Italic</value>
        </prop>
        <prop oor:name="Target" oor:type="xs:string">
            <value>_self</value>
        </prop>
        <prop oor:name="Context" oor:type="xs:string">
            <value>com.sun.star.sheet.SpreadsheetDocument</value>
        </prop>
    </node>
</node>
```

We add as a child to our Top level menu a node named `Submenu` indicating that we are going to add a set of sub-menus.
The first menu entry we add is indicated by creating a child node under the `Submenu` node. This node should have a unique name say `m1` under the top level menu we added.
The `m1` node will then have the following property values :

1. URL : indicate the command URL the menu entry creates which is `.uno:Bold`.

2. Title : The display name of the menu entry.

3. Target : Set the value as `_self` as the action need to take place in the current frame.

4. Context : This indicate the type of document for which menu entry should be shown. We set it to `com.sun.star.sheet.SpreadsheetDocument` to indicate that the menu entry should only be shown for spreadsheet docs. The possible values for `Context` are : 
   1. Writer: com.sun.star.text.TextDocument
   2. Spreadsheet: com.sun.star.sheet.SpreadsheetDocument
   3. Presentation: com.sun.star.presentation.PresentationDocument
   4. Draw: com.sun.star.drawing.DrawingDocument
   5. Formula: com.sun.star.formula.FormulaProperties
   6. Chart: com.sun.star.chart2.ChartDocument
   7. Bibliography: com.sun.star.frame.Bibliography

Similarly we create another menu entry for `Make text italic` using a node named `m2` with command URL `.uno:Italic`.

Now we add two new toolbar entries with our own command URL's `inco.niocs.test.addon.example:InsertDate` and `inco.niocs.test.addon.example:InsertTime`. In this project we only define these toolbar entries and not the command URL's executor/consumer which we will see in upcoming part6.

```xml
<!-- INSIDE AddonUI node -->
<node oor:name="OfficeToolBar">
    <node oor:name="inco.niocs.test.addon.example" oor:op="replace">
    
        <!-- The InsertDate tool bar item -->
        <node oor:name="m1" oor:op="replace">
            <prop oor:name="URL" oor:type="xs:string">
                <value>inco.niocs.test.addon.example:InsertDate</value>
            </prop>
            <prop oor:name="Title" oor:type="xs:string">
                <value xml:lang="en-US">Insert Date</value>
            </prop>
            <prop oor:name="Target" oor:type="xs:string">
                <value>_self</value>
            </prop>
            <prop oor:name="Context" oor:type="xs:string">
                <value>com.sun.star.sheet.SpreadsheetDocument</value>
            </prop>
        </node>

        <!-- The InsertTime tool bar item -->
        <node oor:name="m1" oor:op="replace">
            <prop oor:name="URL" oor:type="xs:string">
                <value>inco.niocs.test.addon.example:InsertTime</value>
            </prop>
            <prop oor:name="Title" oor:type="xs:string">
                <value xml:lang="en-US">Insert Time</value>
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
<!-- INSIDE AddonUI node -->
```

For creating toolbar entries, we need to add a `OfficeToolBar` node inside the `AddonUI` node. Inside that we create a node indicating creation of a new toolbar named uniquely. Here we name it `inco.niocs.test.addon.example`. Under this node, we create two nodes `m1` and `m2` (unique names under **our** toolbar namespace). Each node has the following property values :

1. *URL* : The command URL to produce on user click on the toolbar item.

2. *Title* : The display title for the item.

3. *Target* : The frame to which the command URL needs to send.

4. *Context* : We set it to `com.sun.star.sheet.SpreadsheetDocument` to indicate that the menu entry should only be shown for spreadsheet docs.

Next we set images/icons for the toolbar/menu items we created.

```xml
<!-- INSIDE AddonUI node -->
<node oor:name="Images">

    <!-- Images for InsertDate -->
    <node oor:name="inco.niocs.test.addon.example.date" oor:op="replace">
        <prop oor:name="URL">
            <value>inco.niocs.test.addon.example:InsertDate</value>
        </prop>
        <node oor:name="UserDefinedImages">
            <prop oor:name="ImageSmallURL">
	        <value>%origin%/images/date.png</value>
            </prop>	
        </node>
    </node>
    
    <!-- Images for InsertTime -->
    <node oor:name="inco.niocs.test.addon.example.time" oor:op="replace">
        <prop oor:name="URL">
            <value>inco.niocs.test.addon.example:InsertTime</value>
        </prop>
        <node oor:name="UserDefinedImages">
            <prop oor:name="ImageSmallURL">
	        <value>%origin%/images/time.png</value>
            </prop>	
        </node>
    </node>
</node>
<!-- INSIDE AddonUI node -->
```

For adding icons for the command URL's we added, we need to add a node named `Images` inside `AddonUI` node. Inside it we add two nodes `inco.niocs.test.addon.example.date` and `inco.niocs.test.addon.example.time` (unique names) for the InsertDate and InsertTime commands respectively. The sole property value of these two nodes is `URL` and we set them to the corresponding command URLs. We now need to add a node named `UserDefinedImages` which contains the info on the set of images for that command URL. This node can have the following property values :

1. *ImageSmall* : The value should be hexbinary of the image of size 16x16 px.

2. *ImageBig* : The value should be hexbinary of the image of size 26x26 px.

3. *ImageSmallHC* : The value should be hexbinary of the image of size 16x16 px used for high contrast environments.

4. *ImageBigHC* : The value should be hexbinary of the image of size 26x26 px used for high contrast environments.

5. *ImageSmallURL* : The value should be a url to the external *small* image that resides in the oxt file. The special string `%origin%` can be used to resolve to the location where our extension is placed after installing it with `unopkg` or the extension manager. Example `%origin%/images/small/date.png`.

6. *ImageBigURL* : Same as 5 but for big size image.

7. *ImageSmallHCURL* : Same as 5 but for high contrast image.

8. *ImageBigHCURL* : Same as 6 but for high contrast image.


The `xcu` file we created needs to be added to the manifest.xml of the extension file. This is done via Makefile in the `SimpleAddonExtension` github project.


___

##<a name="buildsec"></a>Download, build and test the extension

The whole project can be downloaded and built using the following steps.

```
$ git clone https://github.com/niocs/SimpleAddonExtension.git
$ cd SimpleAddonExtension
$ make
```

This will create the extension file `SimpleAddonExtension.oxt` inside `/home/$username/libreoffice5.3_sdk/LINUXexample.out/bin`.
Use unopkg to install this extension to LO as :

```
$ $LOROOT/instdir/program/unopkg add /path/to/SimpleAddonExtension.oxt
```

To remove the extension use :
```
$ $LOROOT/instdir/program/unopkg remove SimpleAddonExtension.oxt
```

On opening calc you should see the top level menu called `Add-On example` with menu entries `Make text Bold` and `Make text Italic`. Add some text to any cell and try clicking on the new menu items and see if the text gets bold/italicized. The two new toolbar icons also should be visible. Clicking them would not produce any results though but can notice that calc will log errors like
```
warn:ucb.ucp.gio:7313:1:ucb/source/ucp/gio/gio_content.cxx:399: ignoring GError "The specified location is not supported" for <inco.niocs.test.addon.example:InsertDate>
warn:unotools.misc:7313:1:unotools/source/misc/mediadescriptor.cxx:689: caught Exception "Unable to retrieve value of property 'IsDocument'!" while opening <inco.niocs.test.addon.example:InsertDate>
warn:filter.config:7313:1:filter/source/config/cache/typedetection.cxx:455: caught Exception "Could not open stream for <inco.niocs.test.addon.example:InsertDate>" while querying type of <inco.niocs.test.addon.example:InsertDate>
warn:fwk.dispatch:7313:1:framework/source/dispatch/loaddispatcher.cxx:132: caught LoadEnvException 0 "type detection failed" while dispatching <inco.niocs.test.addon.example:InsertDate>
warn:ucb.ucp.gio:7313:1:ucb/source/ucp/gio/gio_content.cxx:399: ignoring GError "The specified location is not supported" for <inco.niocs.test.addon.example:InsertTime>
warn:unotools.misc:7313:1:unotools/source/misc/mediadescriptor.cxx:689: caught Exception "Unable to retrieve value of property 'IsDocument'!" while opening <inco.niocs.test.addon.example:InsertTime>
warn:filter.config:7313:1:filter/source/config/cache/typedetection.cxx:455: caught Exception "Could not open stream for <inco.niocs.test.addon.example:InsertTime>" while querying type of <inco.niocs.test.addon.example:InsertTime>
warn:fwk.dispatch:7313:1:framework/source/dispatch/loaddispatcher.cxx:132: caught LoadEnvException 0 "type detection failed" while dispatching <inco.niocs.test.addon.example:InsertTime>
```
You should also notice that the new menu/toolbars don't appear in Writer or Impress.
___


## <a name="prebuilt"></a>Test the prebuilt extension

Download the extension as :
```
$ git clone https://github.com/niocs/SimpleAddonExtension.git
$ cd SimpleAddonExtension
```

In `SimpleAddonExtension` directory you will see a file called `SimpleAddonExtension.oxt`. This is the prebuilt extension. Install this extension using

```
$ unopkg add /path/to/SimpleAddonExtension.oxt
```

To remove this extension do :

```
$ unopkg remove SimpleAddonExtension.oxt
```

On opening calc you should see the top level menu called `Add-On example` with menu entries `Make text Bold` and `Make text Italic`. Add some text to any cell and try clicking on the new menu items and see if the text gets bold/italicized. The two new toolbar icons ![date icon](date.png) and ![time icon](time.png) also should be visible. Clicking them would not produce any results though. You should also notice that the new menu does not appear in Writer or Impress or Draw.

