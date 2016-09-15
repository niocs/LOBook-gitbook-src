# Libreoffice extension development with C++ - Part 2 - Working with a Spreadsheet document

In this part we extend the program developed in Part 1 [code](https://github.com/niocs/UNOCreateUseObject)
to manipulate the spreadsheet further.

1. we use `::cppu::bootstrap()` to get the remote office context of a running
   office instance

   ```cpp
   Reference< XComponentContext > xContext( ::cppu::bootstrap() );
   ```

2. Obtain the service manager from the component context

   ```cpp
   Reference< XMultiComponentFactory > xServiceManager = xContext->getServiceManager();
   ```

3. Create Desktop service

   ```cpp
   Reference< XInterface > xDesktop = xServiceManager->createInstanceWithContext( OUString("com.sun.star.frame.Desktop"), xContext );
   ```

4. Get `XDesktop2` interface from `XInterface` reference we got in previous step

   ```cpp
   Reference< XDesktop2 > xDesktop2( xDesktop, UNO_QUERY );
   ```

5. Load a blank spreadsheet document using `XDesktop2` interface

   ```cpp
   Reference< XComponent > xComponent = xDesktop2->loadComponentFromURL( OUString( "private:factory/scalc" ), // URL to the ods file                                                                          
                                                                         OUString( "_blank" ), 0,
                                                                         Sequence < ::com::sun::star::beans::PropertyValue >() );
   ```									 
   Note that the last parameter can be used to describe the properties of the document, and can
   use the property names in [MediaDescriptor](http://api.libreoffice.org/docs/idl/ref/servicecom_1_1sun_1_1star_1_1document_1_1MediaDescriptor.html)
   See below for details on `Sequence` and `PropertyValue` types.

6. In fact the `xComponent` object we got supports `XSpreadsheetDocument` interface, but
   we need to explicitly convert by letting the compiler know by :

   ```cpp
   Reference< XSpreadsheetDocument > xSpreadsheetDocument(xComponent, UNO_QUERY);
   ```

7. Get the "collection of spreadsheets" interface `XSpreadsheets` :

   ```cpp
   Reference< XSpreadsheets > xSpreadsheets = xSpreadsheetDocument->getSheets();
   ```

8. Create a new sheet named MySheet

   ```cpp
   xSpreadsheets->insertNewByName( OUString("MySheet"), (short)0 );
   ```

9. See what is the type of each element in the collection `XSpreadsheets`

   ```cpp
   Type aElemType = xSpreadsheets->getElementType();
   fprintf( stdout, "\n>>> xSpreadsheets container has elements each of type = %s\n",                                                                                                                         
            OUStringToOString( aElemType.getTypeName(), RTL_TEXTENCODING_ASCII_US ).getStr() );
   ```

   This should print the type as "com.sun.star.sheet.XSpreadsheet"

10. Get the object corresponding to the sheet named "MySheet" and query its
    `XSpreadsheet` interface

    ```cpp
    Any aSheet = xSpreadsheets->getByName( OUString("MySheet") );
    Reference< XSpreadsheet > xSpreadsheet( aSheet, UNO_QUERY );
    ```

11. Get cells A1, A2, A3 each by using `getCellByPosition` method of `XCellRange` interface inherited `XSpreadsheet`

    Set cell A1 to 105 and A2 to 501 and insert a formula to A3 =SUM(A1:A2)

    ```cpp
    Reference< XCell > xCell = xSpreadsheet->getCellByPosition(0, 0);
    xCell->setValue(105);

    xCell = xSpreadsheet->getCellByPosition(0, 1);
    xCell->setValue(501);

    xCell = xSpreadsheet->getCellByPosition(0, 2);
    xCell->setFormula("=sum(A1:A2)");
    ```
    
12. Get `XPropertySet` interface of `xCell` object which is really a `Cell` service object

    ```cpp
    Reference< XPropertySet > xCellProps( xCell, UNO_QUERY );
    ```

13. Set the property "CellStyle" of A3 cell to the value "Result" and
    the property "VertJustify" to the enum `CellVertJustify2::TOP`

    ```cpp
    xCellProps->setPropertyValue( OUString("CellStyle"), makeAny( OUString("Result") ) );
    xCellProps->setPropertyValue( OUString("VertJustify"), makeAny( CellVertJustify2::TOP ) );
    ```

    Note that `makeAny()` function converts any type passed to it to `Any` type.

14. Get `XModel` interface from the spreadsheet component object we got in step 5)
    then get `XController` interface object using `XModel`'s `getCurrentController()` method
    and query `XSpreadsheetView` from `XController` object. Using the `XSpreadsheetView` set the
    "MySheet" tab to be the active one.

    ```cpp
    Reference< XModel > xSpreadsheetModel( xComponent, UNO_QUERY );
    Reference< XController > xSpreadsheetController = xSpreadsheetModel->getCurrentController();
    Reference< XSpreadsheetView > xSpreadsheetView( xSpreadsheetController, UNO_QUERY );
    xSpreadsheetView->setActiveSheet( xSpreadsheet );
    ```

15. Get `XCellRangesQuery` supporting object from the "MySheet" object.
    Get the cell ranges where there are formulas in the new sheet
    (we know that there is only one cell where there are formulas)
    Then get the collection of formula cells in the obtained range
    using `XEnumerationAccess` and `XEnumeration` interfaces.

    ```cpp
    Reference< XCellRangesQuery > xCellQuery( aSheet, UNO_QUERY );
    Reference< XSheetCellRanges > xFormulaCells = xCellQuery->queryContentCells( (short)CellFlags::FORMULA );
    Reference< XEnumerationAccess > xFormulasEnumAccess = xFormulaCells->getCells();
    Reference< XEnumeration > xFormulaEnum = xFormulasEnumAccess->createEnumeration();
    ```

16. Loop over the enumerated collection of formula cells by using `XEnumeration` interface's
    `hasMoreElements()` and `nextElement()` methods
    Check [http://api.libreoffice.org](http://api.libreoffice.org) for the signature of `hasMoreElements()` and `nextElement()` methods
    as an exercise.

    ```cpp
    while ( xFormulaEnum->hasMoreElements() )
    {
        Any aNextElement = xFormulaEnum->nextElement();
        xCell = Reference< XCell >( aNextElement, UNO_QUERY );
        Reference< XCellAddressable > xCellAddress( xCell, UNO_QUERY );
        fprintf( stdout, ">>> Formula cell in column = %d, row = %d, with formula = %s\n",
                 xCellAddress->getCellAddress().Column,
                 xCellAddress->getCellAddress().Row,
                 OUStringToOString( xCell->getFormula(), RTL_TEXTENCODING_ASCII_US ).getStr() );
        fflush( stdout );
    }
    ```

The complete C++ program can be compiled and run using :

```bash
$ git clone https://github.com/niocs/ManipulateSpreadsheet.git
$ make
$ make ManipulateSpreadsheet.run
```

## Types

The relation between UNO types like any, string, short etc to the corresponding C++ types can be seen at
[CommonTypes](https://wiki.openoffice.org/wiki/Documentation/DevGuide/FirstSteps/Common_Types)

**Struct type** : These are similar to C++ struct where there are only public member variables only.
Example
    
`::com::sun::star::beans::PropertyValue`

IDL source file : PropertyValue.idl
 
```
    module com {  module sun {  module star {  module beans {
    published struct PropertyValue
    {
        string Name;
	
        long Handle;
	
   	any Value;

   	com::sun::star::beans::PropertyState State;

   };

   }; }; }; };
   
```

Note : Structs should not be contructed using service manager.

Instantiating of a struct in C++ can be done as usual using either static allocation/dynamic allocation using new keyword

Example :
   
```cpp
   ::com::sun::star::beans::PropertyValue aProp;
   aProp.Name = OUString( "Readonly" );
   aProp.Value = makeAny( ( sal_Bool )true );
```

**Any type** : This type can hold on to one of the C++ types that are convertible to UNO types.
To set an `Any` type variable, use `<<=` operator
To get from an `Any` type variable use `>>=` operator

Example :

```cpp
    sal_Int32 cellColor;
    Any any;
    any = rCellProps->getPropertyValue(OUString::createFromAscii( "CharColor" ));
    // extract the value from any
    any >>= cellColor;
```

**Sequence type** : This corresponds to "sequence" UNO type. This represents a homogenous collection of values of one UNO type with
variable number of elements. For remote access this interfaces that take or return "sequence" type is important for performance.

Example a) Empty sequence of `PropertyValue` struct :

```cpp
   Sequence< ::com::sun::star::beans::PropertyValue > loadProps;
```

Example b) Non empty sequence of `PropertyValue` structs :

```cpp
   Sequence< ::com::sun::star::beans::PropertyValue > loadProps( 1 );
   // the structs are default constructed
   loadProps[0].Name = OUString::createFromAscii( "ReadOnly" );
   loadProps[0].Value <<= true;

   Reference< XComponent > rComponent = rComponentLoader->loadComponentFromURL(
                                           OUString::createFromAscii("private:factory/swriter"), 
                                           OUString::createFromAscii("_blank"), 
                                           0, 
                                           loadProps);
```


## Element Access


Element access is the way to access a object from a collection of objects.
The most important element access interfaces are :

* `com::sun::star::container::XNameContainer`
* `com::sun::star::container::XIndexContainer`
* `com::sun::star::container::XEnumeration`

All these three inherit from `XElementAccess` interface and it has two methods :

```
type getElementType()
boolean hasElements()
```

`getElementType()` returns the type of elements of the container as a `com::sun::star::uno::Type` object.
If container is heterogenous it returns `void` type.

`hasElements()` tells whether the set contains any element at all.

1. Name access

   The basic interface inherited by `com::sun::star::container::XNameContainer` is `com::sun::star::container::XNameAccess`
   which has the following three methods

   ```
   any getByName( [in] string name)
   sequence< string > getElementNames()
   boolean hasByName( [in] string name)
   ```

2. Index access

   The basic interface inherited by `com::sun::star::container::XIndexContainer` is `com::sun::star::container::XIndexAccess`
   It has two methods :

   ```
   any getByIndex( [in] long index)
   long getCount()
   ```

   Example : `com::sun::star::sheet::XSpreadsheets` supports both Index access and Named access

3. Enumeration access

   The interface `com::sun::star::container::XEnumerationAccess` creates enumerations that allow traveling across a set of objects.
   It has one method:

   ```
   com::sun::star::container::XEnumeration createEnumeration()
   ```

   The returned `XEnumeration` object supports two methods :

   ```
   boolean hasMoreElements()
   any nextElement()
   ```

   We used the above methods in getting all formula cells in the C++ program developed above.

   In the program we used `XCellRangesQuery` interface to get cell ranges object corresponding
   to cells having formulas. `XCellRangesQuery` interface defines the method :

   ```
   XSheetCellRanges queryContentCells(short cellFlags)
   ```
   The returned `XSheetCellRanges` object has the following methods :

   ```
   XEnumerationAccess getCells()
   String getRangeAddressesAsString()
   sequence< com.sun.star.table.CellRangeAddress > getRangeAddresses()
   ```

   We used `getCells()` to get hold of `XEnumerationAccess` object to iterate through the cells with formula.
   