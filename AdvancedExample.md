# Advanced example #

In this example, you will see how to correctly map remote types to local types and how to use the advanced object filtering options to transform an object into another automatically.

The described situation only applies if you want to have transparent interoperability and somelevel of type safety. Clients and servers in PHP can use normal arrays and even stdClass objects without much trouble, but this tends not to be the case with some Java or C# services, for example.

# The situation #

Let's assume that there is a remote Hessian service that handles CRUD operations for a list of people.This service is coded in Java and it uses a single 'Person' class as a data transfer object.

The signature of the Person class looks like this (Java):

```
package com.sample.hessian;

import java.io.Serializable;
import java.util.Calendar;

public class Person implements Serializable {
	
	private Integer id;
	private String firstName;
	private String lastName;
	private Calendar birthDate;

	// getters and setters
}
```

The service has operation for add, delete, update and get a person's record.We are particularly interested in adding persons to the remote service and getting the generated id, which is exactly what the method add() does. Takes a Person object and returns the same object with a newly generated id.

Looks like this: ` Person add(Person newPerson); `

We create a Person class in PHP compatible with this object, fill some data and then send it to the add() method.

```

require 'HessianClient.php';

class Person{
	var $id;
	var $firstName;
	var $lastName;
	var $birthDate;
}

$p = new Person();
$p->firtName = 'John';
$p->lastName = 'Sheppard';
$p->birthDate = new DateTime('1970-06-14 12:00:00'); // using the standard DateTime object

$url = 'http://localhost:8080/remoting/PersonService';
$proxy = new HessianClient($url);
$result = $proxy->add($p);
print_r($result);

```

When you run this code, depending in the remote server configuration, you will get either a very nasty Fault exception or an empty stream, which is not good at all. What happened?

Notice the birthDate property and its type, Calendar. This is a very common way to represent dates in Java, so the service should automatically map this into either timestamps or DateTime objects, right?

It turns out that this particular service does not do that. Assume there is a remote method called sample() that returns an already filled Person object. When you call this method, with the default configuration this is what you get (using print\_r() for clarity):

```

..
$sample = $proxy->sample();
print_r($sample);
...

stdClass Object
(
    [__type] => com.sample.hessian.Person
    [id] => 
    [firstName] => admin
    [lastName] => admin
    [birthDate] => stdClass Object
        (
            [__type] => com.caucho.hessian.io.CalendarHandle
            [type] => 
            [date] => DateTime Object(
                )
        )
)
```

There are a few things to notice from this output. First, none of the objects have a concrete class, just stdClass. Also, there is a property called `__type` that contains what seems to be a fully qualified Java class name.

Notice that the birthDate property seems to be an object of type com.caucho.io.CalendarHandle that contains a native DateTime. Apparently, the remote service doesn't know how to translate a Calendar object into a Hessian Datetime automatically, so it creates this wrapper.

For this example, we want to make the remote service understand our simple Person objects automatically and in order to make this work, we must do three things:

  1. Map the type 'com.sample.hessian.Person' to 'Person'
  1. Create some temporary object to map against the com.caucho.hessian.io.CalendarHandle wrapper
  1. Extract the DateTime date object from the Calendar and assign it to the birthDate field

First, the type mapping.

# Matching remote and local types #

In standard configuration, HessianPHP will try to match the remote type with local declared classes and instantiate them, but if it doesn't find a match, it will create simple stdClass objects so the data is not lost.

We already created a Person class that matches the remote class, so in order to use it, we must create a mapping using the HessianOptions object, like this:

```
$options = new HessianOptions();
$options->typeMap['Person'] = 'com.sample.hessian.Person';

$proxy = new HessianClient($url, $options);
```

Mapping is configured as an associative array in which the key is the name of the local class and the value is the name of the remote type. But sometimes, using the fully qualified name can be error prone or too specific for our needs. Fortunately, you can use wildcards for the mapping.

` $options->typeMap['Person'] = '*.Person'; `

In this line we say, match any type that ends with 'Person' to the local class. This is very useful, for example, when the remote service uses generic types, present in both Java and C# and you want to match these types to some simple PHP class.

**NOTE:** Mapping to concrete types works both ways, meaning that the mapped remote type will be sent along with the object in the data transmision. On the other hand, wildcard mapping works one way only, the type sent to the remote service will be the local PHP type.

Now we will create the temporary holder for the Calendar object and map it:

```
...

class CalendarHandler{
	public $type;
	public $date;
	function __construct($date=null){
		$this->date = $date;
	}
}

$options = new HessianOptions();
$options->typeMap['Person'] = '*.Person';
$options->typeMap['CalendarHandler'] = 'com.caucho.hessian.io.CalendarHandle';
$proxy = new HessianClient($url, $options);
...
```

If you call the sample method, you will notice that now the objects have correspondent types. But in order to send objects, you will need to use the CalendarHandler instead of a normal DateTime, like this:

```
$p = new Person();
$p->firstName = 'John';
$p->lastName = 'Sheppard';
$p->birthDate = new CalendarHandler(new DateTime('1970-06-14 12:00:00'));
... send the object
```

This works but it's pretty ugly and counter intuitive. In the next step we will create a filter that does this automatically for us.

# Filtering objects and types #

HessianPHP can apply filters both when reading from and writing to a remote service. These filters are applied only to specific primitive types or declared PHP classes or interfaces.

For example, one of the primitive Hessian types is simply called 'date' and it returns a UNIX timestamp, which in PHP is a very large number. In order to turn this into a DateTime object, the library uses a default parsing filter that looks like this:

` $options->parseFilters['date'] = array('HessianDatetimeAdapter','toObject'); `

Within the options objects there are two collections of filters called parseFilters and writeFilters.

These collections are arrays in which the key is the name of a primitive type or a class or interface name prefixed with the character @ to make them distinctive from primitive types, por example '@Person'. The value is a callback, a 'callable' variable, in the context of being able to be used with the call\_user\_func\_array() function. This means it can be the name of a global function, an array with a class name and a public static function, an array of an object and the name of an instance method or a closure (only for PHP 5.3).

The standard 'date' filter calls the toObject() static method in the HessianDatetimeAdapter class passing the value that results from parsing a date in the Hessian protocol, this is, a UNIX timestamp, creates a DateTime object and returns that instead.

**NOTE:** Every callback receives two arguments, the value and either a parser or a writer. The second parameter is used for advanced operations like generating new binary streams. Take a look at the code of the Hessian2IteratorWriter class for an example, the value returned from the function is a HessianStreamResult object that contains a new data stream that gets added to the main stream.

Now, for filtering CalendarHandler objects, we need to create two filters: One to convert a DateTime into a CalendarHandler for writing and another to replace the CalendarObject in the Person class with the DateTime inside it.

Actually, there are several ways to do this, but for the sake of the example we will only use static methods in the same CalendarHandler class.

```

class CalendarHandler{
	public $type;
	public $date;
	function __construct($date=null){
		$this->date = $date;
	}
	// extract the date property and return this instead of the calendar object
	public static function calendarToDateTime($calendar){
		if(isset($calendar->date) && ($calendar->date instanceof DateTime)){
			return $calendar->date;
		}
		return null;
	}
	// Wrap the birthDate property in a CalendarHandler before writing
	public static function writePerson(Person $data, $writer){
		if(isset($data->birthDate) && ($data->birthDate instanceof DateTime)){
			$cal = new CalendarHandler($data->birthDate);
			$data->birthDate = $cal;
		}
		return $data;
	}
}

$options = new HessianOptions();
// 1. this will apply when reading and parsing incoming data
$options->parseFilters['@CalendarHandler'] = array('CalendarHandler','calendarToDateTime');
// 2. this will apply when sending information to the service
$options->writeFilters['@Person'] = array('CalendarHandler','writePerson');

```

The line after comment 1 means: When you read data, each time you create an object of type CalendarHandler, call the method calendarToDateTime() with the original object as a parameter and retrieve whatever this method returns to use instead.

The last line (2) means: When you write data, if you find a Person object, call the method writePerson in the CalendarHandler class, retrieve the results and use them instead.

Now, we can finally call the remote add() method safely. Here's the entire script:

```

require 'HessianClient.php';

class CalendarHandler{
	public $type;
	public $date;
	function __construct($date=null){
		$this->date = $date;
	}
	public static function calendarToDateTime($data){
		if(isset($data->date) && ($data->date instanceof DateTime)){
			return $data->date;
		}
		return null;
	}
	
	public static function writePerson(Person $data, $writer){
		if(isset($data->birthDate) && ($data->birthDate instanceof DateTime)){
			$cal = new CalendarHandler($data->birthDate);
			$data->birthDate = $cal;
		}
		return $data;
	}
}

class Person{
	var $id;
	var $firstName;
	var $lastName;
	var $birthDate;
}

$url = 'http://localhost:8080/remoting/PersonService';

$options = new HessianOptions();
$options->typeMap['Person'] = '*.Person';
$options->typeMap['CalendarHandler'] = 'com.caucho.hessian.io.CalendarHandle';
$options->parseFilters['@CalendarHandler'] = array('CalendarHandler','calendarToDateTime');
$options->writeFilters['@Person'] = array('CalendarHandler','writePerson');

$proxy = new HessianClient($url, $options);
$p = new Person();
$p->firstName = 'John';
$p->lastName = 'Sheppard';
$p->birthDate = new DateTime('1970-06-14 12:00:00');

$john = $proxy->add($p);
echo $john->id.' '.$john->lastName.' '.$john->birthDate->format(DATE_ATOM);

```

The output for this will be something like this:

` 17 Sheppard 1970-06-14T12:00:00-04:00 `

# Conclusion #

Sometimes interacting with services written in different platforms in a natural way can be challenging and even if libraries like HessianPHP provide some help, the best thing is to understand the platform and make the service and data transfer objects as simple as posible, specially when dealing with dates and times.

# Resources #

The sample Java service is based on an article which can be found at http://www.devx.com/Java/Article/42551.