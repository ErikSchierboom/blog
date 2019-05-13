---
title: "Config files"
date: 2015-02-01
tags: 
  - C#
  - .NET
  - Configuration
---

When developing .NET applications, the importance of `Web.config` and `App.config` files is evident. They are used for many configuration purposes, such as specifying connection strings and web server settings. 

If you examine `Web.config` or `App.config` files, you'll find they are XML files. This is an example of a basic config file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <connectionStrings>
    <add name="Default" connectionString="Server=myServer;Database=myDB;" />
    <add name="Backup" connectionString="Server=myBackupServer;Database=myDB;" />
  </connectionStrings>
</configuration>
```

The root element of a config file is the `<configuration>` element, which can have zero or more children. These child elements are known as *configuration sections*. Each config section in turn can also have zero or more child elements, which are known as *configuration elements*.

In our example, there is only one config section: `<connectionStrings>`. Its two `<add>` child config elements define the connection strings, using the `name` and `connectionString` attributes. 

Retrieving the connection string in your application is easy:

```csharp
var def = ConfigurationManager.ConnectionStrings["Default"].ConnectionString;
var bak = ConfigurationManager.ConnectionStrings["Backup"].ConnectionString;
```

The `ConnectionStrings` property of the `ConfigurationManager` class is a wrapper around the `<connectionString>` element in your config file. If you use the connection string's name in its indexer, you get back a strongly-typed class describing the connection string.

Note that to use the `ConfigurationManager` class, your application needs to reference the `System.Configuration` assembly.

### App settings

Another commonly used config section is `<appSettings>`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <appSettings>
    <add key="RequireLogin" value="True" />
    <add key="Timeout" value="10" />
  </appSettings>
</configuration>
```

The `<appSettings>` config section is used for application-specific configuration. You can think of it as a dictionary that maps `string` keys to `string` values.

Retrieval is easy using the key as an indexer:

```csharp
var requireLogin = ConfigurationManager.AppSettings["RequireLogin"];     // Returns "True"
var timeout      = ConfigurationManager.AppSettings["Timeout"]; // Returns "10"
```

As values are returned as a `string`, any type conversions are up to you:

```csharp
var requireLogin = Convert.ToBoolean(ConfigurationManager.AppSettings["RequireLogin"]);
var timeout      = Convert.ToUInt32(ConfigurationManager.AppSettings["Timeout"]);
```

If you try to retrieve an app setting with a key that does not exist, you get back `null`.

## Config sections
The `<connectionStrings>` and `<appSettings>` config sections are built-in. If you want to use a non-built in config sections, you need to explicitly register them. This is done in the `<configSections>` element in your config file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="glimpse" type="Glimpse.Core.Configuration.Section, Glimpse.Core" />
  </configSections>
  <glimpse defaultRuntimePolicy="On" endpointBaseUri="~/Glimpse.axd">
    <logging level="Trace" />
  </glimpse>
</configuration>
```

Here we register a config section named `"glimpse"`, implemented by the `Glimpse.Core.Configuration.Section` type found in the `Glimpse.Core` assembly. The name of the config section element *must* match the registered name, hence the `<glimpse>` config section.

The `<glimpse>` config section itself is interesting. Besides having the `<logging>` child config element, it also has config options itself, expressed as attributes. This shows that there are two ways to define config options in a config section:

1. Through child elements and their attributes
2. Using attributes on the config section element itself

For simple configuration options, attributes on the config section element probably suffice. However, this has some disadvantages:

- You can't have more than one option with the same name
- You can't use nesting; there is only one level at which to define attributes

The following is an example of a config section with nested child config elements:

```xml
<glimpse defaultRuntimePolicy="On" endpointBaseUri="~/Glimpse.axd">
  <runtimePolicies>
    <ignoredTypes>
      <add type="Glimpse.AspNet.Policy.LocalPolicy, Glimpse.AspNet"/>
    </ignoredTypes>
  </runtimePolicies>
</glimpse>
```

### Creating custom configuration sections
For many applications, just using the `<appSettings>` config section suffices. However, there are several disadvantages to this approach:

- Values are always returned as a `string`
- Limited discoverability; it is not easy to find which settings are supported
- You cannot discern between required and optional options
- Related config options cannot be grouped
- There is no warning or error when using unsupported settings

Config sections have none of these disadvantages, so how to create our own config section? Suppose we want to create a custom config section to replace the following `<appSettings>` section:

```xml
<appSettings>
  <add key="RequireLogin" value="True" />
  <add key="Timeout" value="10" />
</appSettings>
```

The first step is to define a class inheriting from `ConfigurationSection`. Within that class, create a property for each config setting:

```csharp
public class CustomConfigSection : ConfigurationSection
{
    [ConfigurationProperty("requireLogin")]
    public bool RequireLogin
    {
        get { return (bool)base["requireLogin"]; }
        set { base["requireLogin"] = value; }
    }

    [ConfigurationProperty("timeout")]
    public int Timeout
    {
        get { return (int)base["timeout"]; }
        set { base["timeout"] = value; }
    }
}
```

For a property to be recognized as a config setting, you decorate it with the `ConfigurationProperty` attribute, passing the element's name as its parameter. The properties themselves are just wrappers over an indexer (inherited from `ConfigurationSection`); the actual retrieval and storing of the properties is done in the base class. Note that the indexer returns an `object`, so casting to the correct type is up to you.

The last step is to register our custom config section in our config file. You might recall that to register a custom config section, we needed to specify the config section's type and its assembly. If we assume our `CustomConfigSection` class is in the `ConfigDemo` namespace in the `ConfigDemo` assembly, we can register and use it as follows:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="custom" type="ConfigDemo.CustomConfigSection, ConfigDemo" />
  </configSections>
  <custom requireLogin="True" timeout="10" />
</configuration>
```

We are now ready to use our custom config section to retrieve the config options:

```csharp
var customConfig = (CustomConfigSection)ConfigurationManager.GetSection("custom");
var requireLogin = customConfig.RequireLogin; // Returns true
var timeout      = customConfig.Timeout;      // Returns 10
```

To get an instance of our custom config section, `GetSection` is called with the config section's registered name. We then cast the returned value to our custom config section class. Finally, we can use our custom config section's properties to retrieve the (strongly-typed) config setting values.

### Customizing configuration properties
So far, the only benefit of our custom config section was that our properties could be non-string values. However, this is not all there is to config properties.

#### Required properties
Custom config sections allow config properties to be marked as optional or required. You do this using the `IsRequired` property of the `ConfigurationProperty` attribute.

The example below makes the `RequireLogin` property required and the `Timeout` property optional. Note that config properties are optional by default.

```csharp
public class CustomConfigSection : ConfigurationSection
{
    [ConfigurationProperty("requireLogin", IsRequired = true)]
    public bool RequireLogin
    {
        get { return (bool)base["requireLogin"]; }
        set { base["requireLogin"] = value; }
    }

    [ConfigurationProperty("timeout", IsRequired = false)]
    public int Timeout
    {
        get { return (int)base["timeout"]; }
        set { base["timeout"] = value; }
    }
}
``` 

This is now a valid configuration:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="custom" type="ConfigDemo.CustomConfigSection, ConfigDemo" />
  </configSections>
  <custom requireLogin="True" />
</configuration>
```

Should you forget to configure a required property, a runtime exception is thrown.

#### Default values
You can also assign default values to properties:

```csharp
public class CustomConfigSection : ConfigurationSection
{
    [ConfigurationProperty("requireLogin", DefaultValue = true)]
    public bool RequireLogin
    {
        get { return (bool)base["requireLogin"]; }
        set { base["requireLogin"] = value; }
    }

    [ConfigurationProperty("timeout", DefaultValue = 20)]
    public int Timeout
    {
        get { return (int)base["timeout"]; }
        set { base["timeout"] = value; }
    }
}
``` 

Take the following configuration file as an example:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="custom" type="ConfigDemo.CustomConfigSection, ConfigDemo" />
  </configSections>
  <custom requireLogin="False" />
</configuration>
```

In this config file, the `requireLogin` setting is explicitly set, overriding the default value. However, the `timeout` setting is not specified and will get assigned its default value:

```csharp
var customConfig = (CustomConfigSection)ConfigurationManager.GetSection("custom");
var requireLogin = customConfig.RequireLogin; // Returns false (config value)
var timeout      = customConfig.Timeout;      // Returns 20 (default value)
```

#### Validators
Sometimes, you want to limit a config element to a specific range of values. For this, you can use configuration validators. Let's start with an example, where we want to limit the values of our timeout config property to the range [0..30]:

```csharp
public class CustomConfigSection : ConfigurationSection
{
    [ConfigurationProperty("timeout")]
    [IntegerValidator(MinValue = 0, MaxValue = 30)]
    public int Timeout
    {
        get { return (int)base["timeout"]; }
        set { base["timeout"] = value; }
    }
}
``` 

If you would assign a value to this config property not in the specified range, you'll get a runtime exception.

There are [several built-in](https://msdn.microsoft.com/en-us/library/system.configuration.configurationvalidatorattribute(v=vs.110).aspx) config validators, including a regex validator and one that allows you to specify a callback. If the validator you want is not available, you can create it yourself by defining a class that inherits from `ConfigurationValidatorAttribute`.

#### Converting values
As we saw earlier, config elements are not limited to the `string` type; we can have `int`, `boolean` and even `TimeSpan` config elements. If you want precise control over how your config element is converted, you can create a custom config converter.

To illustrate how this works, we'll create a config converter that reads and stores integer values as hexadecimal numbers:

```csharp
public class HexConverter : ConfigurationConverterBase
{
    public override object ConvertTo(ITypeDescriptorContext ctx, CultureInfo ci, 
                                     object value, Type type)
    {
        return string.Format("{0:X}", value);
    }

    public override object ConvertFrom(ITypeDescriptorContext ctx, CultureInfo ci, 
                                       object data)
    {
        return int.Parse(data.ToString(), NumberStyles.HexNumber);
    }
}
```

The `ConvertFrom()` method is responsible for converting config file value to the config property type, which in this case is a conversion from a hex `string` to an `int`. It is no surprise that the `ConvertTo()` method works the other way around, converting the `int` config property to its hex `string` config file value.

To use this config converter, we decorate our config element with a `TypeConverter` attribute pointing to our config converter type:

```csharp
public class CustomConfigSection : ConfigurationSection
{
    [ConfigurationProperty("hexTimeout")]
    [TypeConverter(typeof(HexConverter))]
    public int HexTimeout
    {
        get { return (int)base["hexTimeout"]; }
        set { base["hexTimeout"] = value; }
    }
}
``` 

We can now use hex values in our config file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="custom" type="ConfigDemo.CustomConfigSection, ConfigDemo" />
  </configSections>
  <custom hexTimeout="9A" />
</configuration>
```

The `HexTimeout` property will automatically convert the hex `string` config value to an `int`:

```csharp
var customConfig = (CustomConfigSection)ConfigurationManager.GetSection("custom");
var timeout = customConfig.HexTimeout; // Returns 154
```

### Custom configuration elements
So far, our custom config section only had config attributes, and no (child) config elements. Let's add a config element to our config section that represents an URL.

As you might have guessed, we create a class that inherits from `ConfigurationElement`:

```csharp
public class UrlConfigElement : ConfigurationElement
{
    [ConfigurationProperty("value")]
    public string Value
    {
        get { return (string)base["value"]; }
        set { base["value"] = value; }
    }
}
```

Including this config element in our config section is easy:

```csharp
public class CustomConfigSection : ConfigurationSection
{
    [ConfigurationProperty("url")]
    public UrlConfigElement Url
    {
        get { return (UrlConfigElement)base["url"]; }
    }
}
``` 

We declare the config property as we are used to, only now the type is our custom config element. Note that we don't need a setter, as setting values will be done in the `UrlConfigElement` itself. 

We can now use this custom config element in our config file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="custom" type="ConfigDemo.CustomConfigSection, ConfigDemo" />
  </configSections>
  <custom>
    <url value="http://www.google.com" />
  </custom>
</configuration>
```

Finally, using the config element is simple:

```csharp
var customConfig = (CustomConfigSection)ConfigurationManager.GetSection("custom");
var url = customConfig.Url.Value; // Returns "http://www.google.com"
```

It is interesting to note that the base class for config sections, `ConfigurationSection`, actually derives from the `ConfigurationElement` class. In other words, every config section is also a config element, which is why you can have config properties directly within a config section.

### Multiple configuration elements
What if want users to define multiple instances of our custom config element? To support this, we need to create another class, one that represents zero or more instances of our config element.

Let's start with the code for this collection class:

```csharp
[ConfigurationCollection(typeof(UrlConfigElement))]
public class UrlConfigElementCollection : ConfigurationElementCollection
{
    protected override ConfigurationElement CreateNewElement()
    {
        return new UrlConfigElement();
    }

    protected override object GetElementKey(ConfigurationElement element)
    {
        if (element == null)
        { 
            throw new ArgumentNullException("element");
        }

        return ((UrlConfigElement)element).Value;
    }
}
```

Our custom config element class inherits from the `ConfigurationElementCollection` class, which itself derives from the `ConfigurationElement` class. Therefore, you could also define config properties on the collection class itself. The class is decorated with the `ConfigurationCollection` attribute, which indicates the config element type the class is a collection of.

The two methods are abstract in the base class, so we must implement them. The first, `CreateNewElement()` simply returns an instance of the config element. The `GetElementKey()` method returns an object that uniquely identifies an instance of the config element type. Here we use the `UrlConfigElement`'s actual URL, stored in its `Value` property.

We can then include our config element collection in our config section like any regular configuration property:

```csharp
public class CustomConfigSection : ConfigurationSection
{
    [ConfigurationProperty("urls")]
    public UrlConfigElementCollection Urls
    {
        get { return (UrlConfigElementCollection)base["urls"]; }
    }
}
```

We can now use the collection config property to configure multiple items:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="custom" type="ConfigDemo.CustomConfigSection, ConfigDemo" />
  </configSections>
  <custom>
    <urls>
      <add value="http://www.google.com" />
      <add value="http://www.microsoft.com" />
      <add value="http://www.apple.com" />
    </urls>
  </custom>
</configuration>
```

Retrieval of those config items is straightforward:


```csharp
foreach (UrlConfigElement urlElement in customConfig.Urls)
{
    var url = urlElement.Value;
}

```

As there can be only value per key and we used the URL value as the element's key, duplicate URL's will be filtered from the result. The following config file thus results in the same collection as the example above:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="custom" type="ConfigDemo.CustomConfigSection, ConfigDemo" />
  </configSections>
  <custom>
    <urls>
      <add value="http://www.google.com" />
      <add value="http://www.google.com" />
      <add value="http://www.microsoft.com" />
      <add value="http://www.microsoft.com" />
      <add value="http://www.google.com" />
      <add value="http://www.apple.com" />
    </urls>
  </custom>
</configuration>
```

#### Custom child element name
The default element name used to add an element to the collection is `<add>`, but this can be customized in the `ConfigurationCollection` attribute:

```csharp
[ConfigurationCollection(typeof(UrlConfigElement), AddItemName = "url")]
public class UrlConfigElementCollection : ConfigurationElementCollection
```

We now add config elements using the `<url>` element:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="custom" type="ConfigDemo.CustomConfigSection, ConfigDemo" />
  </configSections>
  <custom>
    <urls>
      <url value="http://www.google.com" />
      <url value="http://www.microsoft.com" />
      <url value="http://www.apple.com" />
    </urls>
  </custom>
</configuration>
```

## Config sources
Config sources allow config sections to be configured in external files. This is done by adding the `configSource` attribute to your config section, pointing to the external config file.

The example below states that the `<custom>` config section is configured in the `Custom.config` file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="custom" type="ConfigDemo.CustomConfigSection, ConfigDemo" />
  </configSections>
  <custom configSource="Custom.config" />
</configuration>
```

The `Custom.config` file looks like this:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<custom requireLogin="True" timeout="25" />
```

There are several use cases for this, for example when a project is open-source and you don't want to share your local config settings or when you don't want any server-specific configuration in your config file.

## Config files hierarchy
ASP.NET applications support a hierarchy of `Web.config` files, which is not supported by the `ConfigurationManager` class. If you want to use hierarchical `Web.config` files, you can use the `WebConfigurationManager` class instead. This class offers the same functionality as the `ConfigurationManager` class, but adds an overload to the `GetSection()` method that allows you to specify the path in which to look for the `Web.config` file.

If you don't need the hierarchy support, you can just use the `ConfigurationManager` class, which has the advantage that it also works in console applications.

## Transforming Web.config files
It is not uncommon for `Web.config` settings to change in different environments. A good example are connection strings, which test- and production environment values are likely different. To change a config setting, you could manually edit the files before deploying your application, but this is tedious and error-prone. Luckily, config file transforms allow you to automate this process.

Let's illustrate how this works through an example. Say our `Web.config` file has the following section:

```xml
<appSettings>
  <add key="Environment" value="TEST" />
</appSettings>
```

If we want to change the value of the `Environment` app setting to `PRODUCTION` for `Release` builds, expand the `Web.config` file in your solution explorer and edit the `Web.Release.config` file as follows:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration xmlns:xdt="http://schemas.microsoft.com/XML-Document-Transform">
  <appSettings>
    <add key="Environment" value="PRODUCTION" xdt:Transform="SetAttributes(value)" 
                                              xdt:Locator="Match(key)" />
  </appSettings>
</configuration>
```

As you can see, the config transform file itself is also an XML file. Its contents look like the original config file, but with some differences: 

- The `<configuration>` element now defines the `xdt` namespace. This imports the XML attributes we'll use to transform our config file. 

- The value of the `Environment` config element has been changed to `PRODUCTION`, which is the value we want to replace it with. 

- An `xdt:Locator` attribute has been added. This attribute is used to find the config element to transform in the source config file. In this case, if there is an `<add>` child element of the `<appSettings>` element which `key` attribute matches `Environment`, that element will be transformed. 

- The `xdt:Transform` attribute specifies what transformation to apply if a matching element is found. In our example, we simply state that the `value` attribute should be set to `PRODUCTION` in the transform config file.

We can verify that this works by doing a `Release` mode publish of our ASP.NET application. The `Web.config` file that is published will look like this:

```xml
<appSettings>
  <add key="Environment" value="PRODUCTION" />
</appSettings>
```

Note that config transforms are *only* applied when publishing an ASP.NET application; if you do a regular build, the transformations won't be applied.

Interestingly, the default ASP.NET application template includes the following `Web.Release.config` file:

```xml
<?xml version="1.0"?>
<configuration xmlns:xdt="http://schemas.microsoft.com/XML-Document-Transform">
  <system.web>
    <compilation xdt:Transform="RemoveAttributes(debug)" />
  </system.web>
</configuration>
```

So by default, a `Release` mode publish removes the `debug` attribute from the `compilation` element.

#### Transform options
In our previous examples, we used the `SetAttributes` and `RemoveAttributes` values for the `xdt:Transform` attribute. However, there are more options:

* Replace: replace the element
* Insert: insert the element
* InsertBefore: insert the element before a specified element
* InsertAfter: insert the element after a specified element
* Remove: remove the element
* RemoveAll: remove all elements
* RemoveAttributes: remote attributes from the element
* SetAttributes: set attributes of the elements

With these transform options, you can create fairly advanced transformations.

#### Build configurations
It is important to note that you can have a config transform file for each build configuration in your project. If you add a `Staging` build configuration, you can create a `Web.Staging.config` file by right-clicking on your project and selecting `Add Config Transform`. 

### Transforming App.config files
If you want to transform an `App.config` config file, you'll find this is not supported by default. For this, we can install the [Config Transform](https://visualstudiogallery.msdn.microsoft.com/579d3a78-3bdd-497c-bc21-aa6e6abbc859) Visual Studio extension.

After you have installed the plugin, transforming an `App.config` file is almost the same as a `Web.config` file, but with one additional step. For every `App.config` you want to transform, you need to right-click on it and select the `Add Config Transforms` option. This will add `App.<build config>.config` files for each build configuration, similar to how it works with `Web.config` files. Note that you only need to do this once, unless you added a new build configuration, in which case you just run it again.

Once the `App.<build config>.config` files have been generated, you can use them in exactly the same way as `Web.config` transforms. The only difference is that just compiling the project is enough for the transform to be applied, you don't need to publish it.

So how does this plugin work? Well, the `Add Config Transforms` option not only added the config transform files, it also modified your project's `.csproj` file. If you open your project file, you'll find that the following MSBuild task has been added:

```xml
<UsingTask TaskName="TransformXml" 
           AssemblyFile="$(MSBuildExtensionsPath32)\Microsoft\VisualStudio\v$(VisualStudioVersion)\Web\Microsoft.Web.Publishing.Tasks.dll" />
```

This MSBuild task points to the DLL that is used to transform `Web.config` files, hence its name: `Microsoft.Web.Publishing.Tasks.dll`. If you examine the `.csproj` file further, you'll find that this task is used to automatically apply the transformation after the project has been compiled.

## Conclusion
Config files are an important part of the .NET ecosystem. Just using app settings will suffice in many cases, but its disadvantages start to show when the configuration options become more complex. Creating your own config section(s) really helps in those scenarios. Implementing a custom config section or element is surprisingly easy and has many immediate advantages. When combining them with automatic config transforms, you can create some advanced config setups.