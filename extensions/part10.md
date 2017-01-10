# Libreoffice extension development with C++ - Part 10 - Creating a Calc addin(formula).

### Audience selection

1. Audience type **[AUD-C](README.md)** can jump directly to **[Test the prebuilt extension](#prebuilt)**
2. Audience type **[AUD-B](README.md)** can jump directly to **[Download, build and test the extension](#buildsec)**

___

LO Extensions can be used to add custom functions/formulas to Calc, such an extension is called an Addin. In this part we add couple of custom functions `getticker` and `getcompname`.
`getticker` functions takes an index `i` ranging from 0 to 506 (the upper limit may vary) and returns the ith ticker of `S&P500` index. `getcompname` takes an `S&P500` ticker as input and returns its company name.
For building such an extension we need to create a service implementation that implements `com.sun.star.sheet.AddIn` service and our own custom service that supports a custom interface that supports the functions `getticker` and `getcompname`. `com.sun.star.sheet.AddIn` supports the interface `com.sun.star.sheet.XAddIn` that has the following definition

```
published interface XAddIn: com::sun::star::lang::XLocalizable
{

   string getProgrammaticFuntionName( [in] string aDisplayName );
   string getDisplayFunctionName( [in] string aProgrammaticName );
   string getFunctionDescription( [in] string aProgrammaticName );
   string getDisplayArgumentName(
          [in] string aProgrammaticFunctionName,
          [in] long nArgument );

  string getArgumentDescription(
         [in] string aProgrammaticFunctionName,
         [in] long nArgument );

  string getProgrammaticCategoryName( [in] string aProgrammaticFunctionName );
  string getDisplayCategoryName( [in] string aProgrammaticFunctionName );

};
```
The methods of `XAddIn` interface are very self descriptive. The display name of a function is the name of the function as visible in Calc. Programmatic function name is the name as defined by our custom interface. `XAddIn` inherits from `com::sun::star::lang::XLocalizable` which has the following methods :

```
published interface XLocalizable: com::sun::star::uno::XInterface
{
   void setLocale( [in] Locale eLocale );
   Locale getLocale();
};
```

We now define our custom interface `com::sun::star::sheet::addin::XSP500Addin` and service `SP500Addin` as follows:

```
#ifndef __com_sun_star_sheet_addin_xsp500addin_idl__
#define __com_sun_star_sheet_addin_xsp500addin_idl__

#include <com/sun/star/uno/XInterface.idl>

module com { module sun { module star { module sheet { module addin {
                interface XSP500Addin : com::sun::star::uno::XInterface
                {
                    string getTicker( [in] long nIdx );
                    string getName( [in] string aTicker );
                };

                service SP500Addin : XSP500Addin;
                
}; }; }; }; };

```

Next we define our service implementation `SP500AddinImpl` in `SP500.hxx` as usual and implement the required methods of `XAddIn`, `XSP500Addin`, `XLocalizable`, `XServiceInfo` in `SP500.cxx`:

```cpp
// Include the data header file where the array of tickers and name maps are stored.
#include "data.hxx"

const OUString SP500AddinImpl::aFunctionNames[NUMFUNCTIONS] = {
    "getTicker",
    "getName"
};

const OUString SP500AddinImpl::aDisplayFunctionNames[NUMFUNCTIONS] = {
    "getticker",
    "getcompname"
};

const OUString SP500AddinImpl::aDescriptions[NUMFUNCTIONS] = {
    "Gets SP500 ticker of given index",
    "Gets company name of given ticker"
};

const OUString SP500AddinImpl::aFirstArgumentNames[NUMFUNCTIONS] = {
    "Index",
    "Ticker"
};

const OUString SP500AddinImpl::aFirstArgumentDescriptions[NUMFUNCTIONS] = {
    "Index of SP500 ticker to be returned",
    "Ticker whose company name is to be returned"
};

sal_Int32 SP500AddinImpl::getFunctionID( const OUString aProgrammaticFunctionName ) const
{
    for ( sal_Int32 nIdx = 0; nIdx < nNumFunctions; ++nIdx )
        if ( aProgrammaticFunctionName == aFunctionNames[nIdx] )
            return nIdx;
    return -1;
}

OUString SP500AddinImpl::getTicker( sal_Int32 nIdx )
{
    if ( nIdx < 0 || nIdx >= aTickers.size())
        return "Bad Index";
    return aTickers[nIdx];
}

OUString SP500AddinImpl::getName( const OUString& aTicker )
{
    auto it = aTickerToName.find(aTicker);
    if ( it == aTickerToName.end() )
        return "Bad Ticker";
    return it->second;
}


OUString SP500AddinImpl::getProgrammaticFuntionName( const OUString& aDisplayName )
{
    printf("DEBUG >>> getProgrammaticFuntionName(%s)\n", OUStringToOString( aDisplayName, RTL_TEXTENCODING_ASCII_US ).getStr()); fflush(stdout);
    for ( sal_Int32 nIdx = 0; nIdx < nNumFunctions; ++nIdx )
        if ( aDisplayName == aDisplayFunctionNames[nIdx] )
            return aFunctionNames[nIdx];
    return "";
}

OUString SP500AddinImpl::getDisplayFunctionName( const OUString& aProgrammaticName )
{
    printf("DEBUG >>> getDisplayFunctionName(%s)\n", OUStringToOString( aProgrammaticName, RTL_TEXTENCODING_ASCII_US ).getStr()); fflush(stdout);
    sal_Int32 nFIdx = getFunctionID( aProgrammaticName );
    return ( nFIdx == -1 ) ? "" : aDisplayFunctionNames[nFIdx];
}

OUString SP500AddinImpl::getFunctionDescription( const OUString& aProgrammaticName )
{
    sal_Int32 nFIdx = getFunctionID( aProgrammaticName );
    return ( nFIdx == -1 ) ? "" : aDescriptions[nFIdx];
}

OUString SP500AddinImpl::getDisplayArgumentName( const OUString& aProgrammaticFunctionName, sal_Int32 nArgument )
{
    sal_Int32 nFIdx = getFunctionID( aProgrammaticFunctionName );
    return ( nFIdx == -1 || nArgument != 0 ) ? "" : aFirstArgumentNames[nFIdx];
}

OUString SP500AddinImpl::getArgumentDescription( const OUString& aProgrammaticFunctionName, sal_Int32 nArgument )
{
    sal_Int32 nFIdx = getFunctionID( aProgrammaticFunctionName );
    return ( nFIdx == -1 || nArgument != 0 ) ? "" : aFirstArgumentDescriptions[nFIdx];
}

OUString SP500AddinImpl::getProgrammaticCategoryName( const OUString& aProgrammaticFunctionName )
{
    printf("DEBUG>>> inside getProgrammaticCategoryName()\n");fflush(stdout);
    return "Add-In";
}

OUString SP500AddinImpl::getDisplayCategoryName( const OUString& aProgrammaticFunctionName )
{
    printf("DEBUG>>> inside getDisplayCategoryName()\n");fflush(stdout);
    return "Add-In";
}


```

We use a golang script `getsp500.go` (provided in the source repository) to fetch and parse the `S&P500` wikipedia page `https://en.wikipedia.org/wiki/List_of_S%26P_500_companies` and generate the c++ header file `data.hxx` which we include in `SP500.cxx`
`data.hxx` contains the following two globals `aTickers` and `aTickerToName` which we use in implementing `XSP500Addin` functions:

```cpp
#include <map>
#include <vector>
#include <rtl/ustring.hxx>

using rtl::OUString;


std::vector<OUString> aTickers = {
    "A",
    "AAL",
    "AAP",
    "AAPL",
    "ABBV",
    "ABC",
    "ABT",
    "ACN",
    "ADBE",
    ...
    ...
    "YHOO",
    "YUM",
    "ZBH",
    "ZION",
    "ZTS"
};

std::map<OUString, OUString> aTickerToName = {
    {"A", "Agilent Technologies Inc"},
    {"AAL", "American Airlines Group"},
    {"AAP", "Advance Auto Parts"},
    {"AAPL", "Apple Inc."},
    {"ABBV", "AbbVie"},
    {"ABC", "AmerisourceBergen Corp"},
    {"ABT", "Abbott Laboratories"},
    {"ACN", "Accenture plc"},
    {"ADBE", "Adobe Systems Inc"},
    {"ADI", "Analog Devices, Inc."},
    {"ADM", "Archer-Daniels-Midland Co"},
    ...
    ...
    {"YHOO", "Yahoo Inc."},
    {"YUM", "Yum! Brands Inc"},
    {"ZBH", "Zimmer Biomet Holdings"},
    {"ZION", "Zions Bancorp"},
    {"ZTS", "Zoetis"}
};
```

##<a name="buildsec"></a>Download, build and test the extension

The whole project can be downloaded, built and installed by doing :

```
$ git clone https://github.com/niocs/SP500Addin.git
$ cd SP500Addin
$ make
$ $LOROOT/instdir/program/unopkg add /home/$username/libreoffice5.3_sdk/LINUXexample.out/bin/SP500Addin.oxt
```

To remove the extension, do :
```
$ $LOROOT/instdir/program/unopkg remove SP500Addin.oxt
```

To test `getTicker()` formula, go to any cell and type `=GETTICKER(210)` and hit enter, it should now show the string `GT` which is the 211th ticker in `S&P500` index sorted alphabetically. If you give an invalid index input, then it should produce the string `Bad Index` as output.
Similarly to test `getCompname()` go to any cell and type `=GETCOMPNAME("GT")` and it should produce the output `Goodyear Tire & Rubber` which is the company name of the ticker `GT`. If you provide any invalid `S&P500` ticker as input then it should produce `Bad Ticker` as output.


## <a name="prebuilt"></a>Test the prebuilt extension

Download the extension as :
```
$ git clone https://github.com/niocs/SP500Addin.git
$ cd SP500Addin
```

In `SP500Addin` directory you will see a file called `SP500Addin.oxt`. This is the prebuilt extension. Install this extension using

```
$ unopkg add /path/to/SP500Addin.oxt
```

To remove this extension do :

```
$ unopkg remove SP500Addin.oxt
```

To test `getTicker()` formula, go to any cell and type `=GETTICKER(210)` and hit enter, it should now show the string `GT` which is the 211th ticker in `S&P500` index sorted alphabetically. If you give an invalid index input, then it should produce the string `Bad Index` as output.
Similarly to test `getCompname()` go to any cell and type `=GETCOMPNAME("GT")` and it should produce the output `Goodyear Tire & Rubber` which is the company name of the ticker `GT`. If you provide any invalid `S&P500` ticker as input then it should produce `Bad Ticker` as output.
