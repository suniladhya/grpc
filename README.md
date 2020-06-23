# grpc
**GRPC: On The WIRE:** *A High performance Universal Open source framework*

* Built on the HTTP/2. HTTP/2 as Transport layer. it defines how to write request and interprete the response from the other side.

* Messaages are encoded with Protobuf seriallization - Pluggable. We can also use JSON.

## How it Works:

* HTTP/2 allows to take a single physical stream.

![grpc in in nutshell](https://grpc.io/img/landing-2.svg)

## Protocol Buffers

* gRPC services send and receive data as Protocol Buffer (Protobuf) messages, similar to data contracts in Windows Communication Foundation (WCF).

* grpc can  use Protocol buffers as **IDL** & "Underlying message interchange format"

* On the server side, the server implements this interface and runs a gRPC server to handle client calls

* On the client side, the client has a stub, that provides the same methods as the server.

* In .Net, most seriallization technique using WCF works through *reflection technique to analyze the object structure at runtime*

* where as in the *Protobuf Libraries* requires ti write a structure in a `.proto` file. The file is than compiled by the protobuf compilers.

* The Protobuf compiler, protoc, is maintained by Google. (alternative implementations are available). Generated Code is optimized for fast seriallization and deseriallization.

```protobuf
syntax "proto3";

option csharp_namespace = "TraderSys";

message Stock {

    int32 id = 1;
    string symbol = 2;
    string display_name = 3;
    int32 market_id = 4;
}
```

### Field Numbers

* To identify fields in the binary encoded data. They can't change from version to version in of the Service. *Advantage of backward and forward compatibility*

* 1 - 15 (Encoded as single bytes) has better performance. 

* 16 - 2047 (2 bytes)

### Generated Code

* Actual generated code, far more complicated. Each class contain all necessary codes to *seriallize & deseriallize to the binary wire format*

```C#
public class Stock
{
    public int Id { get; set; }
    public string Symbol { get; set; }
    public string DisplayName { get; set; }
    public int MarketId { get; set; }
}
```

* **Note:** *snake_case are converted to Pascal Case*

### Nested Types

```protobuf
 message Outer {
    message Inner {
        string text = 1;
    }
    Inner inner = 1;
}
```

```C#
var inner = new Outer.Types.Inner { Text = "Hello" };
```

### Repeated fields for lists and arrays

* *for* ```IList<T>``` and ```IEnumerable<T>```

```protobuf
message Person {
    // Other fields elided
    repeated string aliases = 8;
}
```

### Protobuf reserved fields

* Backward-compatibility guarantees in Protocol Buffer (Protobuf) rely on field numbers always representing the same data item.

* If a field is removed from a message in a new version of the service, that field number should never be reused. You can enforce this by using the reserved keyword.

```protobuf
syntax "proto3";

message Stock {

    reserved 3, 4;
    //reserved 2, 9 to 11, 15;
    //reserved 16 to max;
    int32 id = 1;
    string symbol = 2;

}
```

### Protobuf Any and Oneof fields for variant types

* Protocol Buffer (Protobuf) provides two simpler options for dealing with values that might be of more than one type.

* The Any type can represent any known Protobuf message type.

* To use the Any type, you must import the google/protobuf/any.proto definition

```protobuf
syntax "proto3"

import "google/protobuf/any.proto"

message Stock {
    // Stock-specific data
}

message Currency {
    // Currency-specific data
}

message ChangeNotification {
    int32 id = 1;
    google.protobuf.Any instrument = 2;
}
```

```C#
/*the Any class provides methods for setting the field, extracting the message, and checking the type*/
public void FormatChangeNotification(ChangeNotification change)
{
    if (change.Instrument.Is(Stock.Descriptor))
    {
        FormatStock(change.Instrument.Unpack<Stock>());
    }
    else if (change.Instrument.Is(Currency.Descriptor))
    {
        FormatCurrency(change.Instrument.Unpack<Currency>());
    }
    else
    {
        throw new ArgumentException("Unknown instrument type");
    }
}
/*
Protobuf's internal reflection code uses the Descriptor static field on each generated type to resolve Any field types. There's also a TryUnpack<T> method, but that creates an uninitialized instance of T even when it fails. It's better to use the Is method as shown earlier.
*/
```

* The oneof keyword to specify that only one of a range of fields can be set in any message.

* ```repeated``` can't be used with oneof

```protobuf
message Stock {
    // Stock-specific data
}

message Currency {
    // Currency-specific data
}

message ChangeNotification {
  int32 id = 1;
  oneof instrument {
    Stock stock = 2;
    Currency currency = 3;
  }
}
```

* Setting any field that's part of a oneof set will automatically clear any other fields in the set

```C#
// code includes an enum that specifies which of the fields has been set.
//Fields that aren't set return null or the default value, rather than throwing an exception.
public void FormatChangeNotification(ChangeNotification change)
{
    switch (change.InstrumentCase)
    {
        case ChangeNotification.InstrumentOneofCase.None:
            return;
        case ChangeNotification.InstrumentOneofCase.Stock:
            FormatStock(change.Stock);
            break;
        case ChangeNotification.InstrumentOneofCase.Currency:
            FormatCurrency(change.Currency);
            break;
        default:
            throw new ArgumentException("Unknown instrument type");
    }
}
```

### Enumerations

* An enum was previously used to determine the type of a Oneof field.

* Protobuf allows to define own enumeration types, and Protobuf will compile them to C# enum types.

* Due to multiple language support, the naming conventions for enumerations are different from the C# conventions. But code generator converts to C# enums.

```protobuf
enum AccountStatus {
  ACCOUNT_STATUS_UNKNOWN = 0;
  ACCOUNT_STATUS_PENDING = 1;
  ACCOUNT_STATUS_ACTIVE = 2;
  ACCOUNT_STATUS_SUSPENDED = 3;
  ACCOUNT_STATUS_CLOSED = 4;
}
```

```C#
//c# equivalent enum of the protobuf version
public enum AccountStatus
{
    Unknown = 0,
    Pending = 1,
    Active = 2,
    Suspended = 3,
    Closed = 4
}
```

* Protobuf enumeration definitions must have a zero constant as their first field. As in C#, you can declare multiple fields with the same value. But you must explicitly enable this option by using the allow_alias option in the enum:

```protobuf
enum AccountStatus {
  option allow_alias = true;
  ACCOUNT_STATUS_UNKNOWN = 0;
  ACCOUNT_STATUS_PENDING = 1;
  ACCOUNT_STATUS_ACTIVE = 2;
  ACCOUNT_STATUS_SUSPENDED = 3;
  ACCOUNT_STATUS_CLOSED = 4;
  ACCOUNT_STATUS_CANCELLED = 4;
}
```

* enumerations can be declared at the top level in a .proto file, or nested within a message definition. Nested enumerations—like nested messages—will be declared within the .Types static class in the generated message class.

* There's no way to apply the [Flags] attribute to a Protobuf-generated enum, and Protobuf doesn't understand bitwise enum combinations. Look at the following example:

```protobuf
enum Region {
  REGION_NONE = 0;
  REGION_NORTH_AMERICA = 1;
  REGION_SOUTH_AMERICA = 2;
  REGION_EMEA = 4;
  REGION_APAC = 8;
}

message Product {
  Region available_in = 1;
}
```

* If you set product.AvailableIn to Region.NorthAmerica | Region.SouthAmerica, it's serialized as the integer value 3. When a client or server tries to deserialize the value, it won't find a match in the enum definition for 3. The result will be Region.None.

* The best way to work with multiple enum values in Protobuf is to use a repeated field of the enum type.

* **Note**: 

### Protobuf maps for dictionaries

* In .NET, to represent arbitrary collections of named values in messages
dictionary types are used. (`IDictionary<TKey,TValue>`)

```PROTOBUF
/* declare a map type in Protobuf */
message StockPrices {
    map<string, double> prices = 1;
}
```

```PROTOBUF
message Order {
    message Attributes {
        map<string, string> values = 1;
    }
    repeated Attributes attributes = 1;
}
```

* The MapField properties generated from map fields are read-only, and will never be null. To set a map property, use the Add (`IDictionary<TKey,TValue> values`) method on the empty MapField property to copy values from any .NET dictionary.

```C#
public Order CreateOrder(Dictionary<string, string> attributes)
{
    var order = new Order();
    order.Attributes.Add(attributes);
    return order;
}
```

[WCF](https://www.c-sharpcorner.com/UploadFile/db2972/wcf-serialization-part-1-day-6/#:~:text=WCF%20supports%20three%20basic%20serializers%3A&text=WCF%20deserializes%20WCF%20messages%20into%20.&text=Net%20objects%20into%20WCF%20messages,a%20custom%20serializer%20like%20XMLSerializer.)

[further readings](https://docs.microsoft.com/en-us/dotnet/architecture/grpc-for-wcf-developers/wcf-services-to-grpc-comparison)
