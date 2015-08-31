## Note about multibyte character encodings (mbstring options) ##

PHP includes some options that change the way PHP deals with strings and string related functions, and some of these options can really mess up binary data if one is not careful. Concretely, mbstring.func\_overload and mbstring.internal\_encoding.

The former option automatically replaces some of the str_... functions with the equivalent mb\_str_... functions that work with different character sets available in the mbstring extension. However, if you are not careful, this will also affect the way you would normally work with binary data including files and streams, since functions like strlen() or substr() will work directly on characters instead of bytes, which is the normal behaviour in PHP.

The mbstring.internal\_encoding option, changes the way PHP represents strings internally, including variables that can be used with binary services, including HessianPHP. Usually, people would set this option to UTF-8 to automatically handle international text. Although the library tries to detect this and work accordingly, there may be cases when a
string sent or received with HessianPHP can not be decoded as expected. This can result in an exception with the message "Read past the end of stream" or simply a "Code not recognized" fault.

If you intend to use mbstring.func\_overload and internal UTF-8 encoding, here are some tips

  * Save **all** your PHP source files in the UTF-8 encoding, even if there are no international characters in them. Better be safe than sorry.
  * Make sure that your html pages are correctly encoded, including content type in the head section:
> 

&lt;meta http-equiv="Content-Type" content="text/html; charset=UTF-8" &gt;


  * If you want to work with text that comes from a remote service and local text (same source file or external file), make sure all of this files are also saved in UTF-8 encoding. Same goes for information that comes from a database.

As of PHP 5, converting between character encodings tends to be a somewhat frustating experience so perhaps this could help if you plan to change the mbstring options in your PHP installation.