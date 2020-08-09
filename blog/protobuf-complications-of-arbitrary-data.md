# Protobuf: The complications of arbitrary data

[gRPC](https://grpc.io) is a great tool, with it you get to describe your services using the [Protobuf](https://developers.google.com/protocol-buffers) [IDL](https://en.wikipedia.org/wiki/IDL_(programming_language), and have client and server code automatically generated for [most popular languages](https://grpc.io/docs/languages).

## The almighty JSON

Despite myself personally wishing for a binary interchange format for the web, and despite the arguments you'll hear about Protobuf vs Cap'nProto vs FlatBuffers as a message format, JSON is in fact the most popular encoding option in 2020.

A vast majority of APIs use only a REST+JSON API, and you're lucky to get an IDL that describes them. Perhaps the 2nd most popular option for APIs today is GraphQL, which is also backed by JSON.

I want APIs of the future to be accessible - that means having both expressive type-safe binary encodings as well as JSON encodings.

With [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway), you can turn your gRPC API into a full REST API, and even get a Swagger / OpenAPI definition file for your services. It is able to do this in part because the latest version of Protobuf has a [canonical JSON mapping](https://developers.google.com/protocol-buffers/docs/proto3#json) - which means you can write your Protobuf messages in JSON format.

## JSON the feable

JSON has numerous limitations with its type system. It is not possible to encode:

- Float values NaN or Infinity.
- Integers seperately from floats.
- Integers greater than 2<sup>53</sup>, because JSON numbers are 64-bit floats.
- Durations or Timestamps (in any consistent, type-safe manner).
- Binary data, except as e.g. base64 strings.

## Mapping Protobuf to JSON has limitations

Protobuf works around the JSON limitations using a few tricks:

- NaN and Infinity are turned into JSON strings.
- 32-bit integers are encoded as JSON numbers, 64-bit ones are encoded as JSON strings.
- Durations are encoded as nanosecond-precision JSON strings, e.g. `"1.000340012s"` or `"1s"`.
- Timestamps are encoded as RFC 3339 JSON strings.
- Bytes are encoded as base64 strings.
- Arbitrary protobuf message types are encoded using JSON `{"@type": "URL", "value": ...}` objects.

This generally works quite OK _so long as you have a concrete message type._ For example, if you have:

```protobuf
message MyMessage {
    float my_value = 1;
}
```

And the JSON encoded message:

```json
{"my_value": "NaN"}`
```

It is quite clear that the string `"NaN"` should be interpreted as a float value.

## But what about arbitrary data?

Let's say you need to consume some arbitrarily typed metadata and act on that. You have two real options:

1. [`google.protobuf.Value`](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#value) which represents a JSON-compatible value:

> represents a dynamically typed value which can be either null, a number, a string, a boolean, a recursive struct value, or a list of values. A producer of value is expected to set one of that variants, absence of any variant indicates an error.
>
> The JSON representation for Value is JSON value.

2. [`google.protobuf.Any`](https://developers.google.com/protocol-buffers/docs/proto3#any) which represents any protobuf message:

>  lets you use messages as embedded types without having their .proto definition. An Any contains an arbitrary serialized message as bytes, along with a URL that acts as a globally unique identifier for and resolves to that message's type.

### What's the difference?

Using `Value` for your arbitrary data:

```protobuf
message MyMessage {
    google.protobuf.Value my_value = 1;
}
```

You may perhaps send the following JSON messages:

```json
{"my_value": 1.2345}
{"my_value": "NaN"}
```

Which will in turn be the following protobuf messages:

```
struct:{fields:{key:"my_value" value:{string_value:"NaN"}}}
struct:{fields:{key:"my_value" value:{number_value:1.2345}}}
```

Oops! How do we know if `"NaN"` is actually intended to be a number or string? Because we have no schema for this message - we can't know!

### The benefits of `Any`

In contrast if we were to change our message to use `Any`:

```protobuf
message MyMessage {
    google.protobuf.Any my_value = 1;
}
```

Then we can denote that it is indeed a float value:

```json
{"my_value": {"@type": "google.protobuf.FloatValue", "value": "NaN"}}
{"my_value": {"@type": "google.protobuf.FloatValue", "value": 1.2345}}
```

Which will in turn be the following protobuf messages:

```
struct:{fields:{key:"my_value" value:{any:{[google.protobuf.FloatValue]:{value:nan}}}
struct:{fields:{key:"my_value" value:{any:{[google.protobuf.FloatValue]:{value:1.2345}}}
```

### `Any` with JSON encoding is limited

But there are some limitations here, too! For example, what if we want a map instead of a float value? The only pre-defined type for that is [`google.protobuf.Struct`](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#struct) but because it maps `string` to `google.protobuf.Value` we end up with the same problem:

```json
{"my_value": {"@type": "google.protobuf.Struct", "value": {"my_float_value": "NaN"}}}
```

Becomes:

```
struct:{fields:{key:"my_value" value:{any:{[google.protobuf.Struct]:{fields:{key:"my_float_value" value:{string_value:"NaN"}}}}}}
```

And there is no way to describe a _new message type_ with "@type" JSON fields.

In order to represent our message above, we would first need to declare our own message in protobuf directly - defeating the whole purpose.

### `Any` with protobuf is NOT limited

Of course, if you are not sending messages in JSON format but instead using a gRPC client - you can easily declare your own message type and then `Any` is very powerful.

Using a gRPC client with `Any` means you can declare your own message types and the server gets the same exact data types and can [use reflection to grab that information](https://blog.golang.org/protobuf-apiv2).

Using a gRPC client with only a `google.protobuf.Value` API means you are limited to the same restrictions as JSON input (e.g. there is no type-safe way to represent NaN).

## When should you use `Any` or `Value`?

The following simple rule should help you determine when to use `Any` or `Value`:

- If you want to send arbitrary data from a protobuf/gRPC client, use `Any` because it is more powerful.
- If you want to send arbitrary data from a JSON client, use `Value` because it accurately encapsulates all that JSON is capable of.

In our APIs, we typically support both: we want to make our APIs accessible to both JSON and protobuf/gRPC clients.

## Closing thoughts

Protobuf's support for JSON is actually quite nice / powerful. After writing this article, I actually stumbled across 

