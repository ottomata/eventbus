# Eventbus - HTTP JSONSchema event validation and production to Kafka

Eventbus takes in JSON events via HTTP POST, validates and then produces them
using the configurable logic.

Throughout this code, an 'event' is meant to refer to a parsed JSON object with
a strict JSONSchema, and a 'schema' refers to an instance of a JSONSchema for
an event.

# Usage

For configuration of the default Eventbus implementation (see below), edit
config.yaml and run `npm start`.  The service will listen for HTTP POST
requests with Content-Type set to `application/json` at `/v1/events`.  The POST
body should be an array of JSON event objects.

# Architecture

Eventbus attempts to be a generic HTTP POST event intake, event schema validation
and event producing service.  The schema validation and event produce implementation
is left up to the user.  This service ships with a schema URL based
validator and Kafka produce implementation (using node-rdkafka), but you can
plug in your own by implementing a factory module that returns an instantiated `Eventbus`
and use it via `eventbus_factory_module` application config. See documentation below for more on
the default Kafka Eventbus.


## Eventbus implementation configuration

The Eventbus class is generic enough to be used with any type of validation and event production via
function injection.  The HTTP route that handles POSTing of events needs to have an instantiated
Eventbus instance.  To make this configurable (without editing the route code), the route in
routes/events.js will look for `app.conf.eventbus_factory_module` and require it.  This module is expected to export a function called `factory` that takes a `conf` object and a bunyan `logger`.
The function should return a Promise of an instantiated Eventbus.

Once the `eventbus` Promise resolves, the `/v1/events` route will be added and will use the instantiated Eventbus to validate and produce incoming events.

## Eventbus

The `Eventbus` class in lib/eventbus.js handles event validation and produce logic.
It is instantiated with `validate` and a `produce` functions that each take a single
event.  `validate` should either return a Promise of the validated event
(possibly augmented, e.g. field defaults populated) or throw an `EventInvalidError`.
`produce` is expected to return a Promise of the `produce` result, or throw an
`Error`.

Once instantiated with these injected functions, an `Eventbus` instance can be used
by calling the `process` function with an array of events.  `process` will
catch both validation and produce errors, and will return an object with errors
for each event grouped by the type of error.  E.g. if you passed in 3 events,
and 1 was valid, 1 was invalid, and another encountered an unknown error, `process`
will return an object like:

```$javascript
{
    'success': [{ ... }],
    'invalid': [{ ... }],
    'error': [{ ... }]
}
```
with each array containing an `EventStatus` instance representing the event's process result.

### Error events

The Eventbus constructor also takes an optional `mapToErrorEvent` function.
`mapToErrorEvent` takes an `event` and an `error` and returns a new error event representing
the error.  This new error event should be suitable for producing through the same
instantiated `Eventbus`.  If `mapToErrorEvent` is provided and event processing fails for any
reason, those errors will be converted to event errors via this function, and then produced.
There should be no difference between the kind of events that `mapToErrorEvent` returns and
the kind of events that your instantiated `Eventbus` can handle.
`eventbus.produce([errorEvents])` should work.

# Default Eventbus - Schema URI validation & producing with Kafka

If `eventbus_factory_module` is not specified, this service will use provided configruation
to instantiate and use an Eventbus that validates events with JSONSchemas discovered via
schema URLs, and produces valid events to Kafka.

## EventValidator & Event Schema URLs

While the `Eventbus` class that handles event validation and production can
be configured to do validation in any way, the default Eventbus uses the
`EventValidator` class to validate events with schemas obtained from
schema URIs in the events themselves. Every event should contain a URI
to the JSONSchema for the event.  `EventValidator` will extract and use those URIs to look up
(and cache) those schemas to use for event validation.  The `EventValidator` instance
used by the default Eventbus can request schcema URIs from the local filesystem with
`file://` or an `http://` based URIs. The field in the each event where the schema URI is
located is configurable with the `schema_uri_field` config.  When an event is received, the
schema URI will be extracted from the `schema_uri_field`. The JSONSchema at the
URI will be downloaded used to validate the event.  The extracted schema URI
can configurable be prefixed with `schema_base_uri` and suffixed with
`schema_file_extension`.  The default `schema_uri_field` is '$schema'.  If you
use the defaults, all of your events should have a `$schema` field set to
a resolvable schema URI.  IMPORTANT: The event's schema URI should match
EXACTLY what is set for the JSONSchema `$id`. The event validator implementation
will cache compiled schemas by the value of the schema's `$id`.  If this is not
set properly, schemas may not be cached, and each event will result in
schema URI lookup and validator compilation.

## Streams
A 'stream' here refers to the destination of an event.  It is closely related
to Kafka's concept of a topic. Much of the time, a stream might correspond 1:1 with
a Kafka topic.  The default Kafka Eventbus extracts a stream name out of the event
using the configurable `stream_field`, and then prefixes it with `topic_prefix` to
build the destination Kafka topic.

This allows for more flexible constrol of the actual produce destination of events,
without requiring the events themselves to encode their final destination.
Wikimedia uses this feature to prefix topics with source datacenter names.
Event creators shouldn't need to know which datacenter they are operating in.

# HTTP API

See `spec.yaml`.

# Configuration

Configuration is passed to the service via the `config.yaml` file, which
has a `services` object.  This object contains a service named `eventbus`.
The `conf` object of this service will be passed to the `eventbus_factory_module`.
To use a custom Eventbus implementation, set `eventbus_factory_module` to your
javascript module that exports a function to instantiate an Eventbus with `conf`.
See the section above entitled 'Eventbus implementation configuration'.

## Default Kafka Eventbus configuration

The following values in `conf` will be used to instantiate a a default Eventbus
that extracts JSONSchemas from schema URIs, validates events using those
schemas, and then produces them to Kafka.

Note that all `*_field` configs
point to a field in an event, and uses dotted notation to access sub-objects.
E.g. "value" will be extracted from `{ meta: { schema_uri: "value" } }` if
`schema_uri_field` is set to 'meta.schema_uri'.

Property                    |         Default | Description              
----------------------------|-----------------|--------------------------
`port`                      |            6927 | port on which the service will listen
`interface`                 |       localhost | hostname on which to listen
`user_agent`                |        eventbus | The UserAgent seen when making remote http requests (e.g. for remote schemas)
`stream_field`              |         $schema | The name of the stream this event belongs to.  This will be used with `topic_prefix` to build the destination Kafka topic name.
`topic_prefix`              |       undefined | If given, destinaton Kafka topics will be this + the stream name extracted using `stream_field`.
`id_field`                  |       undefined | This is mainly used for logging.  If given, this will be extracted from each event to build an event id that will be added to log messages dealing with the event.
`key_field`                 |       undefined | If given, the value extracted from this field will be used as the Kafka message key.
`partition_field`           |       undefined | If given, the value extracted from this field will be used as the Kafka message partition.
`schema_uri_field`          | $schema         | The value extracted from this field will be used (with `schema_base_uri` and `schema_file_extension`) to download the event's JSONSchema for validation.
`schema_base_uri`           |       undefined | If given, this will be prefixed to every extracted schema URI.
`schema_file_extension`     |       undefined | If given, this will be appendede to every extracted schema URI unless the filename in the URI already has an extension.
`kafka.conf`                |                 | node-rdkafka (and librdkafka) configuration.  This will be passed directly to the node-rdkafka `kafka.Producer` constructor.  Make sure you set kafka.conf.metadata_broker_list.
`kafka.topic_conf`          |                 | node-rdkafka (and librdkafka) topic specific configuration.  This will be passed directly to the node-rdkafka `kafka.Producer` constructor.


# service-template-node

This service is based on Wikimedia's [service-tempate-node](https://github.com/wikimedia/service-template-node).  It is a fork of that 'template'
repository.  See also the [ServiceTemplateNode documentation](https://www.mediawiki.org/wiki/ServiceTemplateNode).


# TODO

- Tests for utils, validators, produanncer, eventbus, etc.
- topic (stream) -> schema mapping config reading?  from local config files and remote service too?
- monitoring/metrics (for kafka, etc.)
- name bikeshedding, probably won't use 'Eventbus' name.
- security review of AJV
- close() method for Eventbus.
- Use options{} for  constructor param instead of positional params.  Will make it easier
  to instantiate custom classes.

## Questions/Thoughts:
- We should leave off file extensions from versioned schemas in the schema repo, so they work without appending them to the schema uris

- Should we configure schemas for topics by final topic or by un prefixed stream name?  -- by stream!  topic should be abstracted.

- Should we have a query param to allow/disallow partial batch production on any error?
  i.e. should remaining events be produced if one fails?  This is the default behavior,
  but maybe users want to configure this.

- Should `?hasty=true` (fire and forget) mode use a non guarunteed Kafka producer?

