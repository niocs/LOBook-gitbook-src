# Libreoffice extension development with C++ - Part 3 - How UNO objects connect and communicate with each other.

### Audience selection

1. Audience type **[AUD-C](README.md)** can skip this part and start with [Part4](part4.md)
2. Audience type **[AUD-B](README.md)** can jump directly to [code download and build/run section](#buildsec)

___

> UNO objects in different environments connect via the **interprocess bridge**. You can execute calls on UNO object instances, that are located in a different process. This is done by converting the method name and the arguments into a byte stream representation, and sending this package to the remote process, for example, through a socket connection.

By default LO will not listen on a resource for security reasons. We can make LO listen for UNO connections from external processess by
starting soffice with extra param :
```
$ $LOROOT/instdir/program/soffice --accept="socket,host=0,port=2002;urp;" --calc
```

After running, it can be verified using netstat -na, and it would show something like :

```
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:2002            0.0.0.0:*               LISTEN
```

## Import a UNO object

To import a reference to a UNO object from an exporting server(example : the LO instance), we use `com::sun::star::bridge::UnoUrlResolver` *service*
which supports the *interface* `com::sun::star::bridge::XUnoUrlResolver`. The interface definition is as follows :

```
interface XUnoUrlResolver: com::sun::star::uno::XInterface
{
      /** resolves an object on the UNO URL */
      com::sun::star::uno::XInterface resolve( [in] string sUnoUrl )  
          raises (com::sun::star::connection::NoConnectException,  
                  com::sun::star::connection::ConnectionSetupException,  
                  com::sun::star::lang::IllegalArgumentException); 
};
```

The string passed to `resolve()` method is called UNO URL with the following format :
```
 uno:connection-type,params;protocol-name,params;ObjectName
| I |           II         |        III         |   IV     |
```
Example :

```
uno:socket,host=localhost,port=2002;urp;StarOffice.ServiceManager
```
The parts I to IV are :

I. This identifies the URL as UNO URL and distinguishes it from others, such as http: or ftp: URLs.

II. connection-type indicates the transport layer mechanism to be used to transfer the byte stream for example TCP/IP sockets or named pipes. Optionally after this string there can be key=value pairs separated by comma indicating the specifics of the transport layer mechanism to be used. Example for sockets, it needs host and port.

III. This indicates the application layer protocol to be used. Example : urp (UNO Remote Protocol). This string can be followed optionally be key=value pairs separated by comma to customize the protocol to specific needs.

IV. This specifies the distinct name of the UNO object the server has exported. Note that it is not possible to access an arbitraty UNO object like in CORBA.

## How to import an object using UnoUrlResolver service.

1. Get local component context *without attempting to invoke LO*.
   ```cpp
   Reference< XComponentContext > xLocalComponentContext = cppu::defaultBootstrap_InitialComponentContext();
   ```
   Note that this is different from calling `cppu::bootstrap()` as in previous examples where it will start LO instance if not found running.
   Where as here no connections to LO is made yet.

2. Get initial service manager from local component context.
   ```cpp
   Reference< XMultiComponentFactory > xServiceManager = xLocalComponentContext->getServiceManager();
   ```

3. Create UnoUrlResolver service locally and query for XUnoUrlResolver interface.
   ```cpp
   Reference< XInterface > xInstance =
        xServiceManager->createInstanceWithContext(
            "com.sun.star.bridge.UnoUrlResolver", xLocalComponentContext );
   Reference< XUnoUrlResolver > xResolver( xInstance, UNO_QUERY );
   ```

4. Import the remote UNO object StarOffice.ServiceManager from port 2002 where soffice is listening.
   ```cpp
   xInstance = xResolver->resolve( OUString(
       "uno:socket,host=localhost,port=2002;urp;StarOffice.ServiceManager" ) );
   ```
   Note that the returned object is typed as `Reference< XInterface >` so we need to query for the required interface to use it.

___

### <a name="buildsec"></a>Download, build and run the program

First run LO as :

```
$ $LOROOT/instdir/program/soffice --accept="socket,host=0,port=2002;urp;" --calc
```

Then the C++ code for this section can be downloaded and run as :

```bash
$ git clone https://github.com/niocs/ImportingUNOObject.git
$ cd ImportingUNOObject
$ make
$ make ImportUNOObject.run
```

If it worked, you will see the string **"Connected successfully to the office"** in the output of `make ImportUNOObject.run`.

If you run the above **before** starting soffice with *listen instructions for port 2002*, the program will show the exception caught and will print something like :
```
Error: Connector : couldn't connect to socket
```
along with some warnings and `make` errors which can be ignored. The above error means the program was not able to connect to soffice to import the UNO object.

___

Usage of UnoUrlResolver has some disadvantages :
1. We won't be notified when brige terminates.
2. We cannot close the underlying interprocess connection.
3. We cannot give the remote process any of our local object as an initial object.

## Interprocess Bridge
This is the entity in any remote process responsible for remote connections. Bridge is *threadsafe* and allows multiple threads to execute remote calls.
The main thread(dispatcher) inside bridge will not block as it passes requests to worker threads. There are however two types of remote calls :
1. **Synchronous call** : The request is sent through the connection and lets the requesting thread *wait* for the reply. Any calls that have a
   *return value* or *out parameter* or throw exception other than `RuntimeException` must be synchronous.

2. *Asynchronous/oneway call* : This sends the request throught the connection and returns immediately without waiting for reply.

In the IDL reference of all methods it will be indicated if the corresponding request is synchronous or asynchronous using the [oneway] modifier.

**Note that there exists cases where oneway calls cause deadlocks in LO so we should not introduce new oneway methods when writing new components**

> Although the remote bridge supports asynchronous calls, this feature is disabled by default. Every call is executed synchronously. The oneway flag of UNO interface methods is ignored. However, the bridge can be started in a mode that enables the oneway feature and thus executes calls flagged with the [oneway] modifier as asynchronous calls. To do this, the protocol part of the connection string on both sides of the remote bridge must be extended by ',Negotiate=0,ForceSynchronous=0'

Example : use the below to start soffice with listening
```bash
$ soffice --accept="socket,host=0,port=2002;urp,Negotiate=0,ForceSynchronous=0;" --calc
```
and use the following UNO URL for connecting to soffice.
```
"uno:socket,host=localhost,port=2002;urp,Negotiate=0,ForceSynchronous=0;StarOffice.ServiceManager"
```

There is an alternative to using UnoUrlResolver using lower level interfaces described in the following OOO wiki pages :
1. [Opening a Connection](https://wiki.openoffice.org/wiki/Documentation/DevGuide/ProUNO/Opening_a_Connection)
2. [Creating a Bridge](https://wiki.openoffice.org/wiki/Documentation/DevGuide/ProUNO/Creating_the_Bridge)
3. [Closing a connection](https://wiki.openoffice.org/wiki/Documentation/DevGuide/ProUNO/Closing_a_Connection)
4. [Advanced Java example showing how the above steps are used](https://wiki.openoffice.org/wiki/Documentation/DevGuide/ProUNO/Example:_A_Connection_Aware_Client)

The complete code for the above can be found at $LOROOT/instdir/sdk/examples/DevelopersGuide/ProfUNO/InterprocessConn

Since we focus on in-process calc extensions, discussion on the above is outside the scope of this series of posts.


