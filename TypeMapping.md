# Mapping types #

Most of the objects that are sent and received using the Hessian protocol contain a type, namely a class name attached to them. In strongly typed implementations, this type is used to correctly create instances of this objects, but in dynamic languages like PHP this is not mandatory.

However, there are cases when you need to map some remote type to a local type, for example an object that implements ActiveRecord that will get filled remotedly and then will be saved in a database.

To have HessianPHP create objects of the right class, you will need to create maps that will tell the library what class to create.

# The typeMap property #

In the configuration of both client and servers you can define mappings of concrete remote types to local types using names or wildcards.

The typeMap field in the configuration object is an associative array in which keys are the local type and values are remote types.

You can use the `*` wildcard in either the local or remote types for easier mapping.
For example ` 'array' => '*.IList*' ` will map every incoming object marked with the type IList to a PHP array

Example:

```
$options = new HessianOptions();
$options->typeMap['Item'] = 'com.test.model.Item'; // specific remote type mapped to local 'Item' class
$options->typeMap['Person'] = '*.Person'; // will match every class name that ends in 'Person' to the local type
```

Alternatively, you can use a shorter array version:

```
$options->typeMap = array(
 'DTO1' => 'com.test.DTO1',
 'Reference' => 'com.test.Reference',
);
```

**NOTES**

  * If you want to map remote types to local arrays, just write 'array' as the key.
  * Using the default ObjectFactory component that comes with this library, when a type cannot be resolved, you will get a stdClass object filled with properties and an extra field called type with the unresolved type name.

TODO: Test namespaces in PHP 5.3