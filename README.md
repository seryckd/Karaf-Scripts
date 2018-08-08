# Karaf Scripts

[Apache Karaf](https://karaf.apache.org/) has a powerful shell for scripting (Gogo). Here are some useful scripts that can be pasted into a console or added to a x.script file under ${karaf}/etc/scripts.

*Tips*

* Set the console colours
* Type Convert to Integer
* Type Convert to String
* Use String Split to Parse Command Results
* Get bundle id for a known bundle
* Loading a Java Class
* Loading a Java Class (not from BundleContext)
* Call a Java Static Method
* Call a Java Instance Method

*Sample Scripts*

* Jasypt - Show all PBE Algorithms
* Jasypt - Encrypt Password
* Base 64 Encoding and Decoding
* Show the value of a property from a pid

## Tips

### Set the console colours

The default console colours are hard to read on a Windows prompt. This variable can be used to change the colours.

    # Type  FullType      Description
    #
    # rs    reserved      Reserved characters like '{', '}', or '|'
    # st    string        A string enclosed with single (') or double (") quotes
    # nu    number        A numeric value
    # va    variable      A variable (starts with '$')
    # vn    variablename  The name of a variable when used without the starting '$'
    # fu    function      A group, command or alias
    # bf    badfunction   If does not match on function
    # co    constant      Keywords like 'null', 'false', 'true'
    # un    unknown       No other match
    # re    repair        The line being typed until a match is made

    # Color        Light  Dark
    # Name         Value  Value
    #
    # Black         30     90
    # Red           31     91
    # Green         32     92
    # Yellow        33     93
    # Blue          34     94
    # Magenta       35     95
    # Cyan          36     96
    # White         37     97

    # Set the colours so they are readable on Windows command prompt
    HIGHLIGHTER_COLORS = "rs=35:st=32:nu=32:co=32:va=36:vn=36:fu=37:bf=91:re=97"

### Type Convert to Integer

Most of the time the shell can work out what type a variable should be, but sometimes it needs a helping hand.

A variable can be type cast to an integer using the trick of adding zero (0). This has to be done using the expression operation. Note that in the expression the dollar sign ($) is not needed to refer to a variable.

    > v='275'
    275
    > $v length
    3

    > %(v+0)
    275
    > %(v+0) length
    Error executing command: Cannot coerce length() to any of []

### Type Convert to String

Most of the time the shell can work out what type a variable should be, but sometimes it needs a helping hand.

A variable can be forced to a string using the trick of concatenating with the empty string.

    >v=90
    90

    > $v getBytes
    Error executing command: Cannot coerce getbytes() to any of []
    > ($v'') getBytes
    [57, 48]

### Use String Split to Parse Command Results

String.split() is very handy for taking a string and splitting into an array.

As an example, let's look at finding the version of the bundle called camel-core.

    > list | grep camel-core
     74 | Active   |  50 | 2.19.1                             | camel-core

use split to break this apart by pipe (|). Split takes a regex value so we have to escapce the pipe character to use it as a match character.

    > (list | grep camel-core) split '\\|'
     74
     Active
      50
     2.19.1
     camel-core

Use square brackets to make this into an array and access the 4th entry

    > [((list | grep camel-core) split '\\|')] get 3
     2.19.1

The extra spaces can be tidied up with String.trim()

    > ([((list | grep camel-core) split '\\|')] get 3) trim
    2.19.1

### Get bundle id for a known bundle

The methods on the BundleContext take a bundle id. If you only know the bundle symbolic name then this will return the bundle id. It returns an empty string if the symbolic name cannot be found.

    [ ((list -s -t 0 | grep my-symbolic-name) split ' ') ] get 0

Example

    # Get the bundle id for the Apache Commons Pool bundle
    [ ((list -s -t 0 | grep org.apache.commons.pool) split ' ') ] get 0

    # and store it in a variable for later use
    bundleId = [ ((list -s -t 0 | grep org.apache.commons.pool) split ' ') ] get 0

### Loading a Java Class

The BundleContext object can be used to load a Java class. Normal OSGi class loading rules apply, you can only load a class that is visible by the bundle used to load it.

For a class accessible by the BundleContext (bundle 0) we can use

    # return a Class object loaded by BundleContext
    ($.context bundle) loadClass my-class

Example

    # Math class can be loaded by the BundleContext
    ($.context bundle) loadClass java.lang.Math

### Loading a Java Class (not from BundleContext)

To use a different bundle to load the class we have to specify the bundle id

    # return a Class object loaded by a specific bundle
    ($.context bundle my-bundle-id) loadClass my-class

Example

    # The 'Password' class is only available to the 'Jetty :: Utilities' bundle.
    # Assuming that the bundle id is 148 (it will be different in every Karaf container)
    ($.context bundle 148) loadClass org.eclipse.jetty.util.security.Password

For scripting, it would be very useful to automatically discover the bundle id based upon a known constant, such as the bundle symbolic name. We can do that with

    bundleId = [ ((list -s -t 0 | grep my-symbolic-name) split ' ') ] get 0
    ($.context bundle %(bundleId+0)) loadClass org.eclipse.jetty.util.security.Password

Notice we have to give a hint to the shell that $bundleId should be treated as an integer.

### Call a Java Static Method

If you have a Class variable then a static method can be called like this

    # my-args is a space separated list
    my-class my-method my-args

Examples

    # Call Math.random()
    > (($.context bundle) loadClass java.lang.Math) random
    0.2641391214192922

    # Call Math.min(5,4)
    > (($.context bundle) loadClass java.lang.Math) min 5 4
    4

    # Store as variable
    > math = ($.context bundle) loadClass java.lang.Math
    ... outputs Math.class.toString()
    > $math random
    0.3777290630677428
    > $math min 5 4
    4
    > $math round 32.1
    32

### Call a Java Instance Method

The new operator can be used to create a new instance of a Java class.

    # returns an instance of my-class and assigns to the variable $instance
    instance = new my-class my-args

The instance variable is then used to call the methods

    # my-args is a space separated list
    my-instance my-method my-args

Examples

    # Create a new string with the string 'hello'
    # Note that the quotes around hello are optional. I just like them
    > str = new (($.context bundle) loadClass java.lang.String) 'hello'
    hello

    # length()
    > $str length
    5

    # indexOf("el")
    # Again, quotes around the el string are optional.
    > $str indexOf 'el'
    1

    # Returning the length without using an intermediate variable
    > (new (($.context bundle) loadClass java.lang.String) 'hello') length
    5

## Sample Scripts

### Jasypt - Show all PBE Algorithms

Uses a utility method in Jasypt (org.jasypt.registry.AlgorithmRegistry.getAllPBEAlgorithms) to view all the available PBE algorithms in the JVM.

    # Show all the available Password Based Encryption (PBE) algorithms available
    # See https://docs.oracle.com/javase/8/docs/technotes/guides/security/SunProviders.html for details
    jasypt-pbe-algorithms = {
        ja_jasypt_id = [ ((list -s -t 0 | grep org.apache.servicemix.bundles.jasypt$) split ' ') ] get 0
	    (($.context bundle %(ja_jasypt_id+0)) loadClass org.jasypt.registry.AlgorithmRegistry) getAllPBEAlgorithms
    };

### Jasypt - Encrypt Password

Karaf can be configured to accept encrypted passwords in configuration files. But the passwords often have to be encrypted outside of Karaf. This script uses the class StandardPBEStringEncryptor to encrypt on the command line.

    # Usage: jasypt-encrypt my-secret my-password
    jasypt-encrypt = {
        ja_jasypt_id = [ ((list -s -t 0 | grep org.apache.servicemix.bundles.jasypt$) split ' ') ] get 0
        _ea_encryptor = new (($.context bundle %(ja_jasypt_id+0)) loadClass org.jasypt.encryption.pbe.StandardPBEStringEncryptor)
        $_ea_encryptor setPassword $1
        $_ea_encryptor setAlgorithm 'PBEWITHHMACSHA384ANDAES_128'
        _value = $_ea_encryptor encrypt $2
        echo Encrypted value: $_value
        _value = ''
        _ea_encryptor =	''
    };

The secret needs to be the same as the one used in the decryptor.

Alternatively the password and algorithm can be stored in the $karaf/etc/system.properties and pulled into the script using the system:property command.

	$_ea_encryptor setPassword ((system:property jasypt.encrypt.secret) trim)
	$_ea_encryptor setAlgorithm ((system:property jasypt.encrypt.algorithm) trim)

### Base 64 Encoding and Decoding

The Java Base64 utilities can be invoked directly

    # usage: base64-encode my-string
    base64-encode = {
    	((($.context bundle) loadClass java.util.Base64) getEncoder) encode ($1'' getBytes)
    };

    # usage: base64-encode-tostring my-string
    base64-encode-tostring = {
	    ((($.context bundle) loadClass java.util.Base64) getEncoder) encodeToString ($1'' getBytes)
    };

    # usage: base64-decode my-encoded-string
    base64-decode = {
	    ((($.context bundle) loadClass java.util.Base64) getDecoder) decode $1
    };

    # usage: base64-decode-tostring my-encoded-string
    base64-decode-tostring = {
    	new (($.context bundle) loadClass java.lang.String) (((($.context bundle) loadClass java.util.Base64) getDecoder) decode $1) UTF-8
    };

Examples

    > base64-encode 'hello'
    [97, 71, 86, 115, 98, 71, 56, 61]

    > a=base64-encode 'hello'
    [97, 71, 86, 115, 98, 71, 56, 61]
    > base64-decode $a
    [104, 101, 108, 108, 111]
    > base64-decode-tostring $a
    hello

    > base64-encode-tostring 'hello world'
    aGVsbG8gd29ybGQ=
    > base64-decode-tostring aGVsbG8gd29ybGQ=
    > hello world

### Show the value of a property from a pid

Shows the value of a property from a pid.

    # Return the value of property 'featuresBoot' from the pid org.apache.karaf.features
    boot-features-list = {
        config:property-get -p org.apache.karaf.features featuresBoot
    };

