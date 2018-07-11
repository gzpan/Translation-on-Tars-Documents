# 5. tars protocol packet size

Currently, the tars protocol limits the size of data packets.

The communicator (client) has no limit on the size of the delivered packet, and there is a limit on the received packet. The default is 10000000 bytes (close to 10M).

The server has no restrictions on the delivered packets , and has a size limit on the received packets. The default is 100000000 bytes (close to 100M).

## 5.1. Modify the client receiving packet size
Modify the size of the packet by modifying the tars_set_protocol of ServantProxy.
```cpp
ProxyProtocol prot;
prot.responseFunc = ProxyProtocol::tarsResponseLen<100000000>;
prot.requestFunc  = ProxyProtocol::tarsRequest;
ccserverPrx -> tars_set_protocol(prot);
```
100000000 represents the size of the limit, in bytes.

ccserverPrx is globally unique, just set it once.

In order to write codes conveniently, it is recommended to set it once in the initialization of the business thread .

First call stringToProxy and then set it.
```cpp
prot.requestFunc = ProxyProtocol::tarsRequest //Must exist, the default is not this function.
```
If it is called in tup mode. Set len
```cpp
prot.responseFunc = ProxyProtocol:: tupResponseLen<100000000>;
```
## 5.2. Modify the server to receive the packet size

Modify the packet size by setting the form of ServantProtocol.
```cpp
addServantProtocol(ServerConfig::Application + "." + ServerConfig::ServerName + ".BObj",AppProtocol::parseLenLen<100000000>);
```
It is recommended to set it in the initialize of the server and set it after addServant.

# 6. Tars defined return code
```
//Define the return code given by the TARS service
const int TARSSERVERSUCCESS       = 0;    //Server-side processing succeeded
const int TARSSERVERDECODEERR     = -1;   //Server-side decoding exception
const int TARSSERVERENCODEERR     = -2;   //Server-side encoding exception
const int TARSSERVERNOFUNCERR     = -3;   //There is no such function on the server side
const int TARSSERVERNOSERVANTERR  = -4;   //The server does not have the Servant object
const int TARSSERVERRESETGRID     = -5;   // server grayscale state is inconsistent
const int TARSSERVERQUEUETIMEOUT  = -6;   //server queue exceeds limit
const int TARSASYNCCALLTIMEOUT    = -7;   // Asynchronous call timeout
const int TARSINVOKETIMEOUT       = -7;   //call timeout
const int TARSPROXYCONNECTERR     = -8;   //proxy link exception
const int TARSSERVEROVERLOAD      = -9;   //Server overload, exceeding queue length
const int TARSADAPTERNULL         = -10;  //The client routing is empty, the service does not exist or all services are down.
const int TARSINVOKEBYINVALIDESET = -11;  //The client calls the set rule illegally
const int TARSCLIENTDECODEERR     = -12;  //Client decoding exception
const int TARSSERVERUNKNOWNERR    = -99;  //The server is in an abnormal position
```

# 7. Business Configuration
The Tars service framework provides the ability to pull the configuration of a service from tarsconfig to a local directory.

The method of use is very simple. In the initialize of the Server, call addConfig to pull the configuration file.

Take HelloServer as an example:
```
HelloServer::initialize()
{
      //Increase the object
      addServant<HelloImp>(ServerConfig::Application+"."+ ServerConfig::ServerName + ".HelloObj");

      //pull the configuration file
      addConfig("HelloServer.conf");
}
```
Description:
> * HelloServer.conf configuration file can be configured on the web management platform;
> * After HelloServer.conf is pulled to the local, the absolute path of the configuration file can be indicated by ServerConfig::BasePath + "HelloServer.conf";
> * The configuration file management is on the web management platform, and the web management platform can actively push the configuration file to the server;
> * The configuration center supports ip level configuration, that is, a service is deployed on multiple services, only partially different (related to IP). In this case, the configuration center can support the merging of configuration files and support viewing on the web management platform as well as modification;

Note:
> * For services that are not released to the management platform, you need to specify the address of Config in the service configuration file, otherwise you cannot use remote configuration.

# 8. Log

The Tars Service Framework provides a macro definition method for logging the system's rolling logs/by-day logs. These macros are thread-safe and can be used at will.

## 8.1. TLOGXXX Description

TLOGXXX is used to record the rolling log for debugging services. XXX includes four levels of DEBUG/WARN/ERROR/NONE, the meanings are as follows.

> * INFO: Information level, the internal log of the framework is printed at the INFO level, unless it is an error
> * DEBUG: Debug level, lowest level;
> * WARN: warning level;
> * ERROR: error level;

How to use:
```cpp
TLOGINFO("test" << endl);
TLOGDEBUG("test" << endl);
TLOGWARN("test" << endl);
TLOGERROR("test" << endl);
```
Description:

> * The current level of the service LOG can be set in the web management;
> * The logs of the Tars framework are printed by info. After setting it to info, you can see the frame log of Tars;
> * TLOGXXX is scrolled by size, you can modify the scroll size and number in the template configuration file of the service, usually do not use it;
> * TLOGXXX indicates that the circular log is not sent to the remote tarslog service;
> * The file name of TLOGXXX is related to the service name, usually app.server.log;
> * TLOGXXX is a global single piece, can be used anywhere, but if the framework is used before the LOG initialization, it will directly cout out;
> * #include "servant/TarsLogger.h" is required in the place of use

TLOGXXX is a macro, which is defined as follows:
```cpp
#define LOG (TarsRollLogger::getInstance()->logger())

#define LOGMSG(level,msg...) do{if(LOG->IsNeedLog(level)) LOG->log(level)<<msg;}while(0)

#define TLOGINFO(msg...) LOGMSG(TarsRollLogger::INFO_LOG,msg)
#define TLOGDEBUG(msg...) LOGMSG(TarsRollLogger::DEBUG_LOG,msg)
#define TLOGWARN(msg...) LOGMSG(TarsRollLogger::WARN_LOG,msg)
#define TLOGERROR(msg...) LOGMSG(TarsRollLogger::ERROR_LOG,msg)
```

The type returned by TarsRollLogger::getInstance()->logger() is TC_RollLogger*, so you can set the LOG through it, for example:
set to Info level:
```cpp
TarsRollLogger::getInstance()->logger()->setLogLevel(TC_RollLogger::INFO_LOG);
```
The LOG log is asynchronous by default and can also be set to sync:
```cpp
TarsRollLogger::getInstance()->sync(true);
```
You can also use LOG in any ostream, for example:
```cpp
ostream &print(ostream &os);
```
Can be used as follows:
```cpp
print(LOG->debug());
```

## 8.2. DLOG/FDLOG

Used to record daily logs, so-called by-day logs, mainly used to record important business information;
> * DLOG is the default button log, FDLOG can specify the file name of the log file;
> * DLOG/FDLOG logs will be automatically uploaded to tarslog, which can be set to not be uploaded to tarslog;
> * DLOG/FDLOG can modify the scrolling time, such as by minute, hour, etc.
> * DLOG/FDLOG log is asynchronous by default. It can be set to sync if necessary, but it must be asynchronous for remote upload to tarslog and cannot be set to sync.
> * DLOG/FDLOG can be set to not be recorded locally, only uploaded to the remote;
> * #include "servant/TarsLogger.h" is required in the place of use

## 8.3. Code example
```cpp
CommunicatorPtr c = new Communicator();

string logObj = "tars.tarslog.LogObj@tcp -h 127.0.0.1 -p 20500";

//Initialize the local scroll log
TarsRollLogger::getInstance()->setLogInfo("Test", "TestServer", "./");

//Initialize the time log
TarsTimeLogger::getInstance()->setLogInfo(c, logObj , "Test", "TestServer", "./");

//If the Tars service, do not need the above part of the code, the framework has been automatically initialized, do not initialize the business itself

//The default daily log does not need to be uploaded to the server
TarsTimeLogger::getInstance()->enableRemote("", false);

//The default daily log is scrolled by minute
TarsTimeLogger::getInstance()->initFormat("", "%Y%m%d%H%M");

//abc2 file does not need to be uploaded to the server
TarsTimeLogger::getInstance()->enableRemote("abc2", false);

//abc2 file scrolls by hour
TarsTimeLogger::getInstance()->initFormat("abc2", "%Y%m%d%H");

//abc3 file does not remember local
TarsTimeLogger::getInstance()->enableLocal("abc3", false);

int i = 100000;
while(i--)
{
      //Same as the previous one
      TLOGDEBUG(i << endl);

      //error level
      TLOGERROR(i << endl);

      DLOG << i << endl;

      FDLOG("abc1") << i << endl;
      FDLOG("abc2") << i << endl;
      FDLOG("abc3") << i << endl;

      if(i % 1000 == 0)
      {
          cout << i << endl;
      }
      usleep(10);
}
```

