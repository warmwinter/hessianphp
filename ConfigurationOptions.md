# Introduction #

Most configuration options can be set using the standard HessianOptions class or an associative array, more on this later. Some configuration options affect general behaviour and others are aimed only to clients or servers.

# Details #

In order to configure hessian clients or servers you can create a HessianOptions object
and set its properties like this:

```
$options = new HessianOptions();
$options->version = 1;
$options->detectVersion = false;
$options->typeMap = array('ParamObjectJava' => 'ParamObject');
```

Then pass it to the constructor of either server or client:

```
$client = new HessianClient('http://localhost/remoteService', $options);
```

**Using arrays**

If you preffer to use arrays for configuration, you can pass an associative array where
keys contain the names of the properties, like this:

```
$service = new HessianService($wrappedObject, 
  array(
   'displayInfo' => true, 
   'ignoreOutput'=> true
  )
);
```

This example tells HessianPHP to use BASIC Authentication for a client proxy, using the conventions defined for the CURL library in PHP (the default transport):

```
$options = array(
  'transportOptions' => array( CURLOPT_USERPWD => 'user:password' )
);
$proxy = new HessianClient($url, $options);
```

## General configuration properties ##

Options that affect both client and servers.

| **Option** | **Description** |
|:-----------|:----------------|
| **version** | Protocol version to use, posible values are 1 and 2. The default value is 2. |
| **detectVersion** | Boollean, Defines if the library should try to detect protocl version based on incoming data. Default is false. |
| **typeMap** | Associative array defining mappings between local and remote types (classes) |
| **interceptors** | Array of interceptor objects that implement the IHessianInterceptor interface that will be executed in every request.|
| **timeZone** | String. String defining a timezone for datetime handling, which is required in newer versions of PHP. Default detected local time zone |
| **saveRaw** | Boolean. Tells the transport library to keep a copy of received bytes in raw form. Usefull for debugging or logging. |
| **headers** | Array of headers to be included in each request. Not implemented yet. |

## Client options ##

| **transport** | String/object. Defines the http communication library that clients use to send requests to remote services. Available options are 'CURL' and 'http'. See description below. |
|:--------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **transportOptions** | Mixed. Transport specific options to configure the request.                                                                                                                 |

**Transport options**

  * **'CURL' (Default)** uses the CURL extension to send POST requests which can be faster and better suited for secure communication.
  * **'http'** uses standard PHP http stream wrappers (stream\_context\_create and fopen) to send the request.

Optionally, you can pass an object that implements IHessianTransport using another
library or method. Transport specific options to configure the request. In the current provided implementations, the options are:

  * CURL: array of options to be passed to the curl\_setopt\_array() function
  * http: array to be merged with the options used in the stream\_context\_create() function.

Please consult the PHP Manual for more information about specific options for these libraries.

Options for the CURL transport can be found here:
http://www.php.net/manual/en/function.curl-setopt.php

The options for the 'http' transport can be found here:
http://www.php.net/manual/en/context.http.php

## Server options ##

| **serviceName** | String. Published name of service, the default is the class name of the serviced object |
|:----------------|:----------------------------------------------------------------------------------------|
| **displayInfo**| Boolean. Default false. When set true, the service will display an information page with the name of the service and available methods upon a browsing to the service PHP page (GET request). When false, it will issue an error telling the Hessian requires POST.|
| **ignoreOutput** | Boolean. Default false. When set to true, the service will ignore all text produced inside method calls, for example echo() and print() statements. NOTE: If any of the methods called in the service writes anything to the screen, it will result in a corrupted stream. |

## Advanced options ##

The following options exist for customizing internal behaviour of the library and it should
be used carefully.

| **objectFactory** | Object. Replaces the default object factory with an object that implements the IHessianObjectFactory interface. Can be used to get objects through IoC for example. |
|:------------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **parseFilters**  | Associative array in which keys are the names of basic Hessian types or classes/interfaces prefixed with @ and the keys are valid PHP callbacks to functions that process or transform the value just read from the service. Used for transforming or filtering mostly objects into a different format or even data type. Comes with a default filter 'date' to transform UNIX timestamps into native DateTime objects. |
| **writeFilters**  | Associative array in which keys are the names of basic PHP types or classes and the values are valid PHP callbacks to functions that will process or transform a value before is written into the output stream. Used to handle serialization of specific types, for instance, the IteratorWriter that serializes PHP STD iterators as lists or maps. |