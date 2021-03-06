# WhiteShark
WhiteShark is a binary serialization format with a much smaller byte size footprint than JSON or Java native serialization format, at the cost of a little overhead in processing time.

# Supported Languages
As a serialization format, WhiteShark can be used with any programming language.
This repository provides the Java library.

# Requirements
WhiteShark Java implementation requires Java 7 SE or later.
Optional `tests` package requires `org.json` package you can find at https://github.com/douglascrockford/JSON-java

# Usage
We do not provide a JAR file yet for the library, so you have to import the `com.xhaleera.whiteshark` package into your project.

## Serialization
Serialization is very simple with WhiteShark. Supported types are `null`, booleans, integers, floating-point numbers, strings, arrays and objects.

By design, serialization of object fields is done on an opt-in basis. In other words, you have to specify explicitly the serializable fields.
This is done through the `@WhiteSharkSerializable` annotation.

### Example
```java
class MySerializableClass {
	public string notSerializedString;

	@WhiteSharkSerializable
	public string serializedString;
}
```

### Serialization of Native Java `Map` and `Collection` Interfaces
`Map` and `Collection` instances are considered as common objects by WhiteShark. This means only their fields annotated with `@WhiteSharkSerializable` will be serialized by default.

In the following example, the `serializedMapInstance` field will be serialized, but not its content.

```java
class MySerializableClass {
	public string notSerializedString;

	@WhiteSharkSerializable
	public string serializedString;

	@WhiteSharkSerializable
	public HashMap<String,Object> serializedMapInstance;
}
```

To serialize the content of a `Map` (limited at this time to `Map<String,?>`) or a `Collection`, add `@WhiteSharkSerializableMap` or `@WhiteSharkSerializableCollection` annotation to this field.

```java
	...
	@WhiteSharkSerializable
	@WhiteSharkSerializableMap
	public HashMap<String,Object> serializedMapInstance;
}
```

You can also add these annotations at the type level, if your class extends `Map<String,?>` or `Collection`.

```java
@WhiteSharkSerializableCollection
class MyCollection extends Collection<Integer> {
	...
}

class MySerializableClass {
	...
	@WhiteSharkSerializable
	public MyCollection serializedCollection;
}
```

If your class or field implements both interfaces, it is possible to add the two annotations and `Map` and `Collection` serializations will apply.

Adding `@WhiteSharkSerializableMap` or `@WhiteSharkSerializableCollection` annotation to a not eligible field or type has no effect.

### Calling the Default Serializer
A WhiteShark stream of serailized data starts with a header. This header contains a custom 4-byte long alphanumeric identifier that indicates the potential stream usage. It allows you during deserialization to ensure the data you're receiving is the right one, and acting accordingly if not.

To serialize an object with default options, just call `WhiteSharkSerializer.serialize()` with the correct arguments.

```java
FileOutputStream fileStream = new FileOutputStream(new File(path));
string streamId = "STRG";
string toSerialize = "It's not very useful to serialize me, but... you know...";
WhiteSharkSerializer.serialize(streamId, fileStream, toSerialize);
fileStream.close();
```

### Calling the Serializer with Options
An alternative version to `WhiteSharkSerializer.serialize()` allows you to pass some options to the serializer.

At this time, only one option is supported:
* **`WhiteSharkConstants.OPTIONS_OBJECTS_AS_GENERICS`**: If set, serialized objects won't include class information and, thus, won't be mapped to their original class on deserialization (not set by default)

```java
FileOutputStream fileStream = new FileOutputStream(new File(path));
string streamID = "STRG";
string toSerialize = "It's not very useful to serialize me, but... you know...";
WhiteSharkSerializer.serialize(streamID, fileStream, toSerialize, WhiteSharkConstants.OPTIONS_OBJECTS_AS_GENERICS);
fileStream.close();
```

*Disabling class mapping, i.e. serializing objects as generics, can also be achieved on a class-by-class basis thanks to the `@WhiteSharkAsGenerics` class annotation.*

### External Class Mapping
Serialization is used to store objects permanently, in a database for example. Thus, serialization and deserialization is generally done using the same code base.

However, serialized data may be used over a network to achieve data formatting in an efficient way. Then, serialization and deserialization will probably occur on different code bases. As an example, a Java multiuser server communicating with a Unity C# client. Different languages imply different class name specifications (PHP packages are separated with `\`, but Java package separator is `.`).

WhiteShark provides a way to automate external class mapping using the `WhiteSharkExternalClassMapper` class. You can pass a properly configured instance to specific variants of `serialize()` and `deserialize()` methods and of the `WhiteSharkProgressiveDeserializer` constructor.

```java
	WhiteSharkExternalClassMapper mapper = new WhiteSharkExternalClassMapper();
	mapper.mapClass(MyJavaClass.class, "\Xhaleera\MyPHPClass");
	WhiteSharkSerializer.serialize("STID", inputStream, objectToSerialize, mapper);
```

## Immediate Deserialization
Immediate deserialization is the simplest and fastest method as it can deserialize your WhiteShark stream in just one call. However, it requires the WhiteShark stream to be fully available during deserialization.

To proceed, just call `WhiteSharkImmediateDeserializer.deserialize()` then pass you stream identifier and input stream.

```java
FileInputStream inStream = new FileInputStream(new File(path));
string streamId
Object o = WhiteSharkImmediateDeserializer.deserialize(streamId, inStream);
inStream.close();
```

## Progressive Deserialization
Progressive deserialization is the method of choice if you need to deserialize your WhiteShark stream *on the flow*. For example, it applies to network communications, if your serialized data is chunked or if you can not or do not want to buffer your whole stream before deserialization occurs.

The following example exhibits how to deserialize a file in several steps:

```java
FileInputStream inStream = new FileInputStream(new File(path));
WhiteSharkProgressiveDeserializer.DeserializationResult result = null;
WhiteSharkProgressiveDeserializer deserializer = new WhiteSharkProgressiveDeserializer(streamId);
byte[] b = new byte[1024];
int c;
while (inStream.available() > b.length) {
	c = inStream.read(b);
	result = deserializer.update(b, 0, c);
	if (result.complete)
		break;
}
if (result != null && !result.complete) {
	c = inStream.read(b);
	result = deserializer.finalize(b, 0, Math.max(0, c));
}
inStream.close();
```

# Comparison with Other Serialization Formats
As a Java library, it is interesting to compare it against the Java built-in serialization API. It is also interesting to compare against the well-known and widely used JSON format.

## Differences with Java Built-in Serialization API
Both APIs provides serialization and deserialization of generic, primitive and custom types, including type mapping. However, WhiteShark is only able to serialize / deserialize publicly available properties when the Java built-in serialization API is capable of serializing whole objects.

Furthermore, WhiteShark has an opt-in philosophy and serializes only properties and types you have explicitly elected for serialization, when Java built-in serialization API works on an opt-out basis thanks to transient fields.

## Differences with JSON
WhiteShark and JSON share the same philosophy but has very different approaches.

WhiteShark serializes types and classes based on their native definition. However, JSON language syntax does not natively include type and class mapping, leaving the responsability to instantiante approriate types and classes during deserialization to the developer.

In that, WhiteShark is faster to implement.

## Output Size

### Data Set
As an example, we serialize a list of fictitious employees, thanks to a `Team` class extending `Vector<Employee>`.
The `Team` class also includes an array of 12 integers, containing the number of days for each month of a year.

| First Name | Last Name | Age | Male  | Height |
|------------|-----------|-----|-------|--------|
| Charlotte  | HUMBERT   | 30  | False | 1.8 m  |
| Eric       | BALLET    | 38  | True  | 1.65 m |
| Charles    | SAUVEUR   | 35  | True  | 1.8 m  |
| Carli      | BRUNA     | 26  | False | 1.6 m  |
| William    | MARTIN    | 31  | True  | 1.75 m |
| Marine     | DAVID     | 35  | False | 1.55 m |

Each employee has some additional data. For the sake of simplification, all employees have the exact same data.

First, a list of work day categories, implemented as a `Map`.

| Category | Count |
|----------|-------|
| missing  | 0     |
| ill      | 2     |
| vacation | 10    |
| years    | 3     |

Then, a list of meta data, also implemented as a `Map`.

| Meta          | Value      |
|---------------|------------|
| Date of birth | 1901-01-01 |
| Entry date    | 1902-01-01 |

And finally, a list of skills, implemented as a `Collection`.

| Skill           |
|-----------------|
| Management      |
| Human Resources |

Additionnally, we use external class mapping to map `com.xhaleera.whiteshark.tests.Employee` with `Xhaleera::WhiteShark::Test::Employee`.

### Results 
You can find in the following list the amount of data required to store the serialized stream in each format.

| Format        | Size        | Diff to WhiteShark |
|---------------|-------------|--------------------|
| WhiteShark    | 1,242 bytes | -                  |
| Java built-in | 1,762 bytes | +41.87%            |
| JSON          | 1,505 bytes | +21.18%            |

## Performance

### Test Protocol
Using the same data set, we proceed in sequence to 10,000 runs of:
* WhiteShark serialization,
* Java built-in serialization,
* JSON data construction and serialization,
* WhiteShark immediate deserialization,
* WhiteShark progressive deserialization,
* Java built-in deserialization,
* JSON deserialization

*Serializations and deserializations are done to and from memory buffers only.*

Each process is run 10 times, and we keep the minimal, maximal and average timing as a performance indicator.

### Results
This test protocol has been run on October 26th, 2015 on a MacBook Pro mi-2009 (2,53 GHz Intel Core 2 Duo, 8 Go 1067 MHz DDR3) running OS X 10.11.1 and Java 8 SE Update 51.

Values represent a single serialization or deserialization run and are expressed in milliseconds.

| Process                                | Minimal | Maximal | Average |
|----------------------------------------|--------:|--------:|--------:|
| WhiteShark serialization               | 0.1922  | 0.2219  | 0.21038 |
| Java built-in serizalization             | 0.0874  | 0.1604  | 0.12597 |
| JSON serialization                     | 0.0945  | 0.147   | 0.11596 |
| WhiteShark immediate deserialization   | 0.1416  | 0.2859  | 0.25071 |
| WhiteShark progressive deserialization | 0.5124  | 0.7714  | 0.58699 |
| Java built-in deserialization            | 0.1724  | 0.2083  | 0.19734 |
| JSON deserialization                   | 0.1042  | 0.1456  | 0.12283 |

At this time, WhiteShark Java implementation is about twice slower than the other options while serializing or deserializing thanks to the immediate method. Progressive deserialization is twice slower than the immediate method and three to five times slower than the other options.

Optimizations are planned to reduce that gap.

See [ARCHIVED_RESULTS.md](ARCHIVED_RESULTS.md) to compare evolution over time.

# Format Specifications
See [FORMAT_SPECS.md](FORMAT_SPECS.md)

# License
See [LICENSE.md](LICENSE.md)
