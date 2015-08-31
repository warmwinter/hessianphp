## Interceptors ##

With Interceptors, you can perform custom operations around the execution of remote communication, in both clients and services.

An interceptor is an object that implements the **IHessianInterceptor** interface and it has 2 main methods: **beforeRequest** and **afterRequest** . These methods accept a single HessianCallingContext as a parameter. This object contains different information pertaining the current action such as the method being called, url, original configuration options and the writer and parser objects that handle the protocol at a low level.

Check the source code for a complete description of the properties and methods in these two classes.

## Configuration ##

Using interceptors is fairly simple.
  1. Create a class that derives from IHessianInterceptor
  1. Create an instance of the class
  1. Add it to the interceptors array in the configuration options passed to HessianPHP clients or servers.

## Example ##

This is a simple but useful interceptor that saves the binary streams involved in the communication and also the internal log of the parser activity.

```
class Interceptor implements IHessianInterceptor{
	
	function beforeRequest(HessianCallingContext $ctx){
		// the call property in the context is a HessianCall object
		// representing the RPC operation. We will log only if we are a client
		if($ctx->isClient)
			echo "Calling method: ".$ctx->call->method;
		else {
			$ipclient = $_SERVER['REMOTE_ADDR'];
		}
		// this tells the transport layer to save the original bytes
		$ctx->options->saveRaw = true; 
	}
	
	function afterRequest(HessianCallingContext $ctx){
		file_put_contents('outgoing.bin', $ctx->payload);
		file_put_contents('parserLog.txt', implode("\r\n", $ctx->parser->log));
		file_put_contents('writerLog.txt', implode("\r\n", $ctx->writer->log));
		file_put_contents('incoming.bin', $ctx->transport->rawData);
	}
}

// configuration
$interceptor = new Interceptor();

$options = new HessianOptions();
$options->interceptors = array($interceptor); // add interceptor 

$proxy = new HessianClient($url, $options);

// or

$server = new HessianService($testService, $options);
```

If you have diferent interceptors, they will execute in the order they were added to the array.