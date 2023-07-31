# Program Binding for CloudEvents - Version 1.0.3-wip

## Abstract

The Program Binding for CloudEvents defines how events are mapped to environment
variables and standard input of an OS program.

## Table of Contents

1. [Introduction](#1-introduction)

- 1.1. [Conformance](#11-conformance)
- 1.2. [Content Modes](#12-content-modes)
- 1.3. [Event Formats](#13-event-formats)

2. [Use of CloudEvents Attributes](#2-use-of-cloudevents-attributes)

- 2.1. [datacontenttype Attribute](#21-datacontenttype-attribute)
- 2.2. [data](#22-data)

3. [Process Message Mapping](#3-process-message-mapping)

- 3.1. [Binary Content Mode](#31-binary-content-mode)
- 3.2. [Structured Content Mode](#32-structured-content-mode)
- 3.3. [Batched Content Mode](#33-batched-content-mode)

4. [References](#4-references)

## 1. Introduction

[CloudEvents][ce] is a standardized and protocol-agnostic definition of the
structure and metadata description of events. This specification defines how the
elements defined in the CloudEvents specification are to be used by an OS programs
environment variables and standard input.

### 1.1. Conformance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119][rfc2119].

### 1.2. Content Modes

The CloudEvents specification defines three content modes for transferring
events: _structured_, _binary_ and _batch_. The Program binding supports
all three content modes. Every compliant implementation SHOULD
support both structured and binary modes.

In the _binary_ content mode, the value of the event `data` is piped on standard
input as-is, with the `datacontenttype` attribute
value declaring its media type in the environment variable `CE-CONTENT-TYPE`; all other
event attributes are mapped into environment variables.

In the _structured_ content mode, event metadata attributes and event data are
piped on standard input using an [event format](#13-event-formats) that supports
[structured-mode messages][ce-message].

In the _batched_ content mode, event metadata attributes and event data of
multiple events are batch piped on standard input using an 
[event format](#13-event-formats) that supports batching
[structured-mode messages][ce-message].

### 1.3. Event Formats

Event formats, used with the _structured_ content mode, define how an event is
expressed in a particular data format. All implementations of this specification
that support the _structured_ content mode MUST support the non-batching [JSON
event format][json-format], but MAY support any additional, including
proprietary, formats.

Event formats MAY additionally define how a batch of events is expressed. Those
can be used with the _batched_ content mode.

## 2. Use of CloudEvents Attributes

This specification does not further define any of the core [CloudEvents][ce]
event attributes.

This mapping is intentionally robust against changes, including the addition and
removal of event attributes, and also accommodates vendor extensions to the
event metadata.

### 2.1. datacontenttype Attribute

The `datacontenttype` attribute is assumed to contain a [RFC2046][rfc2046]
compliant media-type expression.

### 2.2. data

`data` is assumed to contain opaque application data that is encoded as declared
by the `datacontenttype` attribute.

An application is free to hold the information in any in-memory representation
of its choosing, but as the value is transposed into standard input as defined in this
specification, the assumption is that the `data` value is made available as a
sequence of bytes.

For instance, if the declared `datacontenttype` is
`application/json;charset=utf-8`, the expectation is that the `data` value is
made available as [UTF-8][rfc3629] encoded JSON text to standard input.

## 3. Process Message Mapping

The content mode is chosen by the sender of the event, which is either the
requesting or the responding party. Gestures that might allow solicitation of
events using a particular mode might be defined by an application, but are not
defined here. The _batched_ mode MUST NOT be used unless solicited, and the
gesture SHOULD allow the receiver to choose the maximum size of a batch.

The receiver of the event can distinguish between the three modes by inspecting
the `CE-CONTENT-TYPE` environment variable. If the value is prefixed with the CloudEvents
media type `application/cloudevents`, indicating the use of a known
[event format](#13-event-formats), the receiver uses _structured_ mode. If the
value is prefixed with `application/cloudevents-batch`, the receiver uses the
_batched_ mode. Otherwise it defaults to _binary_ mode.

If a receiver detects the CloudEvents media type, but with an event format that
it cannot handle, for instance `application/cloudevents+avro`, it MAY still
treat the event as binary and forward it to another party as-is.

When the `CE-CONTENT-TYPE` environment variable is not prefixed with the CloudEvents media
type, knowing when the message ought to be parsed as a CloudEvent can be a
challenge. While this specification can not mandate that senders do not include
any of the CloudEvents environment variables when the message is not a CloudEvent, it
would be reasonable for a receiver to assume that if the message has all of the
mandatory CloudEvents attributes as environment variables then it's probably a
CloudEvent. However, as with all CloudEvent messages, if it does not adhere to
all of the normative language of this specification then it is not a valid
CloudEvent.

### 3.1. Binary Content Mode

The _binary_ content mode accommodates any shape of event data, and allows for
efficient transfer and without transcoding effort.

#### 3.1.1. CE-CONTENT-TYPE environment variable

For the _binary_ mode, the `CE-CONTENT-TYPE` environment variable corresponds to
(MUST be populated from or written to) the CloudEvents `datacontenttype`
attribute. Note that a `CE-DATACONTENTTYPE` environment variable MUST NOT also be
present in the message.

#### 3.1.2. Event Data Encoding

The [`data`](#22-data) byte-sequence is used as standard input.

#### 3.1.3. Metadata Attributes

All other [CloudEvents][ce] attributes, including extensions, MUST be
individually mapped to and from distinct environment variable.

CloudEvents extensions that define their own attributes MAY define a secondary
mapping to environment variables for those attributes. Note that these attributes 
MUST be prefix with a `CE-` prefix as noted in
[Environment Variable Names](#3131-environment-variable-names).

##### 3.1.3.1. Environment Variable Names

Except where noted, all CloudEvents context attributes, including extensions,
MUST be mapped to environment variables with the same upper case name as the 
attribute name but prefixed with `CE-`.

Examples:

    * `time` maps to `CE-TIME`
    * `id` maps to `CE-ID`
    * `specversion` maps to `CE-SPECVERSION`

Note: environment variables are case-sensitive, and should all be upper-case.

##### 3.1.3.2. Environment Variable Values

The value for each environment variable is constructed from the respective attribute
type's [canonical string representation][ce-types] and require no special encoding.

#### 3.1.4. Examples

This example shows the _binary_ mode mapping of an event to a programs environment
variables and standard in:

| Name            | Value                           |
|-----------------|---------------------------------|
| CE-SPECVERSION  | 1.0                             |
| CE-TYPE         | com.example.someevent           |
| CE-TIME         | 2018-04-05T03:56:24Z            |
| CE-ID           | 1234-1234-1234                  |
| CE-SOURCE       | /mycontext/subcontext           |
| CE-CONTENT-TYPE | application/json; charset=utf-8 |
 .... further attributes ...

Standard In:
```text
{
    ... application data ...
}
```

### 3.2. Structured Content Mode

The _structured_ content mode keeps event metadata and data together in the
standard input payload, allowing simple forwarding of the same event across 
multiple routing hops, and across multiple protocols.

#### 3.2.1. Event Content-Type

An environment variable with the `CE-CONTENT-TYPE` name MUST be set to the media 
type of an [event format](#13-event-formats).

Example for the [JSON format][json-format]:

| Name            | Value                                       |
|-----------------|---------------------------------------------|
| CE-CONTENT-TYPE | application/cloudevents+json; charset=utf-8 |

#### 3.2.2. Event Data Encoding

The chosen [event format](#13-event-formats) defines how all attributes, and
`data`, are represented.

The event metadata and data is then rendered in accordance with the event format
specification and the resulting data becomes the standard input payload.

#### 3.2.3. Metadata Attributes

Implementations MAY include the same environment variables as defined for the
[binary mode](#313-metadata-attributes).

All CloudEvents metadata attributes MUST be mapped into the payload, even if
they are also mapped into environment variables.

#### 3.2.4. Examples

This example shows a JSON event format encoded event mapped to a programs 
environment variables and standard input:

| Name            | Value                                       |
|-----------------|---------------------------------------------|
| CE-CONTENT-TYPE | application/cloudevents+json; charset=utf-8 |

```text
{
    "specversion" : "1.0",
    "type" : "com.example.someevent",

    ... further attributes omitted ...

    "data" : {
        ... application data ...
    }
}
```

### 3.3. Batched Content Mode

In the _batched_ content mode several events are batched into standard input. The 
chosen [event format](#13-event-formats) MUST define how a batch is represented, 
including a suitable media type.

#### 3.3.1. Event Content-Type

The `CE-CONTENT-TYPE` environment variable MUST be set to the media type of
the batch mode for the [event format](#13-event-formats).

Example for the [JSON Batch format][json-batch-format]:

| Name            | Value                                             |
|-----------------|---------------------------------------------------|
| CE-CONTENT-TYPE | application/cloudevents-batch+json; charset=utf-8 |

#### 3.3.2. Event Data Encoding

The chosen [event format](#13-event-formats) defines how a batch of events and
all event attributes, and `data`, are represented.

The batch of events is then rendered in accordance with the event format
specification and the resulting data becomes the standard input payload.

#### 3.3.3. Examples

This example shows two batched CloudEvents mapped to a programs environment
variables and standard in:

| Name            | Value                                             |
|-----------------|---------------------------------------------------|
| CE-CONTENT-TYPE | application/cloudevents-batch+json; charset=utf-8 |

```text
[
    {
        "specversion" : "1.0",
        "type" : "com.example.someevent",

        ... further attributes omitted ...

        "data" : {
            ... application data ...
        }
    },
    {
        "specversion" : "1.0",
        "type" : "com.example.someotherevent",

        ... further attributes omitted ...

        "data" : {
            ... application data ...
        }
    }
]
```

## 4. References

- [RFC2046][rfc2046] Multipurpose Internet Mail Extensions (MIME) Part Two:
  Media Types
- [RFC2119][rfc2119] Key words for use in RFCs to Indicate Requirement Levels
- [RFC3629][rfc3629] UTF-8, a transformation format of ISO 10646
- [RFC4627][rfc4627] The application/json Media Type for JavaScript Object
  Notation (JSON)
- [RFC4648][rfc4648] The Base16, Base32, and Base64 Data Encodings
- [RFC6839][rfc6839] Additional Media Type Structured Syntax Suffixes
- [RFC7159][rfc7159] The JavaScript Object Notation (JSON) Data Interchange
  Format

[ce]: ../spec.md
[ce-message]: ../spec.md#message
[ce-types]: ../spec.md#type-system
[json-format]: ../formats/json-format.md 
[json-batch-format]: ../formats/json-format.md#4-json-batch-format
[json-value]: https://tools.ietf.org/html/rfc7159#section-3
[json-array]: https://tools.ietf.org/html/rfc7159#section-5
[rfc2046]: https://tools.ietf.org/html/rfc2046
[rfc2119]: https://tools.ietf.org/html/rfc2119
[rfc3629]: https://tools.ietf.org/html/rfc3629
[rfc4627]: https://tools.ietf.org/html/rfc4627
[rfc4648]: https://tools.ietf.org/html/rfc4648
[rfc6839]: https://tools.ietf.org/html/rfc6839#section-3.1
[rfc7159]: https://tools.ietf.org/html/rfc7159
