# Prebid Cache

This application stores short-term data for use in Prebid Server and Prebid.js, primarily targeting video, native, and AMP formats.

## Installation

First install Go version 1.15 or newer.

Note that prebid-cache is using Go modules. We officially support the most recent two major versions of the Go runtime. However, if you'd like to use a version <1.13 and are inside `GOPATH` `GO111MODULE` needs to be set to `GO111MODULE=on`.

Download and prepare Prebid Cache:

```
cd YOUR_DIRECTORY
git clone https://github.com/prebid/prebid-cache src/github.com/prebid/prebid-cache
cd src/github.com/prebid/prebid-cache
```

Run the automated tests:

```
./validate.sh
```

Or just run the server locally:

```
go build .
./prebid-cache
```

## API

### POST /cache

Adds one or more values to the cache. Values can be given as either JSON or XML. A sample request is below.

```json
{
  "puts": [
    {
      "type": "xml",
      "ttlseconds": 60,
      "value": "<tag>Your XML content goes here.</tag>"
    },
    {
      "type": "json",
      "ttlseconds": 300,
      "value": [1, true, "JSON value of any type can go here."]
    }
  ]
}
```

If any of the `puts` are invalid, then it responds with a **400** and none of the values will be retrievable. Assuming that all of the values are well-formed, then the server will respond with IDs which can be used to fetch the values later.

**Note**: `ttlseconds` is optional, and will only be honored on a _best effort_ basis. Callers should never _assume_ that the data will stay in the cache for that long.

```json
{
  "responses": [
    {"uuid": "279971e4-70f0-4b18-bd65-5c6e7aa75d40"},
    {"uuid": "147c9934-894b-4c1f-9a32-e7bb9cd15376"}
  ]
}
```

#### Puts Parameters

| Name        | Scope    | Type     | Description |
| --- | --- | --- | --- |
| `type`      | required | string | Only `"xml"` or `"json"` are supported |
| `value`     | required | string | Prebid Cache will respond with an error if an empty string is provided |
| `ttlseconds` | optional | integer | Represents the time to live in seconds of your data. Default value is 3600 seconds |
| `key` | optional | string | When included, your value will be stored under this key instead of a system-generated random UUID. Requires `request_limits.allow_setting_keys` to be set to `true` |

If `ttlseconds` is included, its value must be non-negative and no larger than the `request_limits.max_ttl_seconds` configuration, which defaults to 3600 seconds. The `key` parameter will be ignored unless the boolean configuration flag `request_limits.allow_setting_keys` is set to `true`. The following is a sample `config.yaml` configuration file that sets the ttl to 100 seconds and allows Prebid Cache to set custom keys: 

```
request_limits:
  max_ttl_seconds: 100
  allow_setting_keys: true
```

in the example below, A Prebid Cache instance that supports `key` by setting `allow_setting_keys` to `true`, includes `key` fields in some of the `"puts"` array elements:

```json
{
  "puts": [
    {
      "type": "xml",
      "ttlseconds": 60,
      "value": "<tag>Your XML content goes here.</tag>",
      "key": "CustomKeyValueHere"
    },
    {
      "type": "json",
      "ttlseconds": 300,
      "value": [1, true, "JSON value of any type can go here."]
    },
    {
      "type": "json",
      "value": "{\"someJsonField\":\"some JSON string value.\"}",
      "key": "AnotherCustomKeyValue"
    }
  ]
}
```

This would result in a response like this:

```json
{
  "responses": [
    {"uuid": "CustomKeyValueHere"},
    {"uuid": "147c9934-894b-4c1f-9a32-e7bb9cd15376"},
    {"uuid": "AnotherCustomKeyValue"}
  ]
}
```

Prebid Cache will use the custom keys for those elements that include them and will create system-generated `uuid`s for the ones that don't. Note that if configuration flag `allow_setting_keys` is set to `false` or simply not set inside the `config.yaml` file, Prebid Cache would generate random `uuid`s for all elements and not use the custom keys `"CustomKeyValueHere"` nor `"AnotherCustomKeyValue"` at all.

#### Overwritting values

Prebid Cache does not allow overwritting any value, for either autogenerated or custom keys. If an entry already exists for a given key, it will not be overwitten, and an empty string will be returned as the `uuid` value of that entry. Suppose we wanted to overwrite the entries under `"CustomKeyValueHere"` and the system-generated key `"147c9934-894b-4c1f-9a32-e7bb9cd15376"`.

```json
{
  "puts": [
    {
      "type": "xml",
      "ttlseconds": 60,
      "value": "<tag>Some other XML content under a key already in use.</tag>",
      "key": "CustomKeyValueHere"
    },
    {
      "type": "json",
      "ttlseconds": 300,
      "value": [2, false, "Trying to store other JSON under a key already in use."],
      "key": "147c9934-894b-4c1f-9a32-e7bb9cd15376"
    },
    {
      "type": "xml",
      "ttlseconds": 30,
      "value": "<tag>Different XML content will be stored under a system-generated key.</tag>",
    }
  ]
}
```

Then we get a response with empty `uuid` fields for those `"puts"` elements that tried to overwrite entries:

```json
{
  "responses": [
    {"uuid": ""},
    {"uuid": ""},
    {"uuid": "efc6ca1d-3409-4b8b-96e5-aec508a57639"}
  ]
}
```

Those items in the cache did not get overwritten as a subsequent `GET` call would assert:

```
$ curl http://someprebidcachehost.com/cache\?uuid\=CustomKeyValueHere

HTTP/1.1 200 OK
Content-Type: application/xml

<tag>Your XML content goes here.</tag>
```

This is to prevent bad actors from trying to overwrite legitimate caches with malicious content, or a poorly coded app overwriting its own cache with new values, generating uncertainty of what is actually stored under a particular key. Note that cases like these are the only time where a subset of caches would not get stored. Under any other scenario, we expect the entire request to fail and no elements will be stored.

Trying to overwrite the value under an existing key is also the only instance where an unsuccessful `Put` is not considered an error. As such, Prebid Cache will not respond with an error message or return an error code on these particular instances.

### GET /cache?uuid={id}

Retrieves a single value from the cache. If the id isn't recognized, then it will return an HTTP 404. The following are sample requests and responses based on the POST call examples above.

GET */cache?uuid=279971e4-70f0-4b18-bd65-5c6e7aa75d40*

```
HTTP/1.1 200 OK
Content-Type: application/xml

<tag>Your XML content goes here.</tag>
```


GET */cache?uuid=147c9934-894b-4c1f-9a32-e7bb9cd15376*

```
HTTP/1.1 200 OK
Content-Type: application/json

[1, true, "JSON value of any type can go here."]
```

### Limitations

This section does not describe permanent API contracts; it just describes limitations on the current implementation.

- This application does *not* validate XML. If users `POST` malformed XML, they'll `GET` a bad response too.
- The host company can set a max length on payload size limits in the application config. This limit will vary from vendor to vendor.

## Backend Configuration

In order to store its data a Prebid Cache instance can use either of the following storage services: Aerospike, Cassandra, Memcache, Redis, or local memory. Select the storage service your Prebid Cache server will use by setting the `backend.type` property in the `config.yaml` file:

```yaml
backend:
  type: "aerospike"
```

### Aerospike
Prebid Cache makes use of an Aerospike Go client that requires Aerospike server version 4.9+ and will not work properly with older versions. Full documentation of the Aerospike Go client can be found [here](https://github.com/aerospike/aerospike-client-go/tree/v6).
| Configuration field | Type | Description |
| --- | --- | --- |
| host | string | aerospike server URI |
| port | 4-digit integer | aerospike server port |
| namespace | string | aerospike service namespace where keys get initialized |

### Cassandra
Prebid Cache makes use of a Cassandra client that supports latest 3 major releases of Cassandra (2.1.x, 2.2.x, and 3.x.x). Full documentation of the Cassandra Go client can be found [here](https://github.com/gocql/gocql).
| Configuration field | Type | Description |
| --- | --- | --- |
| hosts | string | Cassandra server URI |
| keyspace | string | Keyspace defined in Cassandra server |

### Memcache:
| Configuration field | Type | Description |
| --- | --- | --- |
| config_host | string | Configuration endpoint for auto discovery. Replaced at docker build. |
| poll_interval_seconds | string | Node change polling interval when auto discovery is used |
| hosts | string array | List of nodes when not using auto discovery | 

### Redis:
Prebid Cache makes use of a Redis Go client compatible with Redis 6. Full documentation of the Redis Go client Prebid Cache uses can be found [here](https://github.com/go-redis/redis).
| Configuration field | Type | Description |
| --- | --- | --- |
| host | string | Redis server URI |
| port | integer | Redis server port |
| password | string | Redis password |
| db | integer | Database to be selected after connecting to the server |
| expiration | integer | Availability in the Redis system in Minutes |
| tls | field | Subfields: <br> `enabled`: whether or not pass the InsecureSkipVerify value to the Redis client's TLS config <br> `insecure_skip_verify`: In Redis, InsecureSkipVerify controls whether a client verifies the server's certificate chain and host name. If InsecureSkipVerify is true, crypto/t |

Sample configuration file `config/configtest/sample_full_config.yaml` shown below:
```yaml
port: 9000
admin_port: 2525
index_response: "Any index response"
log:
  level: "info"
rate_limiter:
  enabled: false
  num_requests: 150
request_limits:
  max_size_bytes: 10240
  max_num_values: 10
  max_ttl_seconds: 5000
  allow_setting_keys: true
backend:
  type: "memory"
  aerospike:
    default_ttl_seconds: 3600
    host: "aerospike.prebid.com"
    hosts: ["aerospike2.prebid.com", "aerospike3.prebid.com"]
    port: 3000
    namespace: "whatever"
    user: "foo"
    password: "bar"
  cassandra:
    hosts: "127.0.0.1"
    keyspace: "prebid"
  memcache:
    hosts: ["10.0.0.1:11211","127.0.0.1"]
  redis:
    host: "127.0.0.1"
    port: 6379
    password: "redis-password"
    db: 1
    expiration: 1
    tls:
      enabled: false
      insecure_skip_verify: false
compression:
  type: "snappy"
metrics:
  type: "none"
  influx:
    host: "metrics-host"
    database: "metrics-database"
    username: "metrics-username"
    password: "metrics-password"
    enabled: true
  prometheus:
    port: 8080
    namespace: "prebid"
    subsystem: "cache"
    timeout_ms: 100
    enabled: true
routes:
  allow_public_write: true
```

## Development

### Prerequisites

[Golang](https://golang.org/doc/install) 1.9.1 or greater and [Dep](https://github.com/golang/dep#installation) must be installed on your system.

### Automated tests

`./validate.sh` runs the unit tests and reformats your code with [gofmt](https://golang.org/cmd/gofmt/).
`./validate.sh --nofmt` runs the unit tests, but will _not_ reformat your code.

### Manual testing

Run `prebid-cache` locally with:

```bash
go build .
./prebid-cache
```

The service will respond to requests on `localhost:2424`, and the admin data will be available on `localhost:2525`

### Configuration

Configuration is handled by [Viper](https://github.com/spf13/viper#putting-values-into-viper). The easiest way to set config during development is by editing the [config.yaml](./config.yaml) file. You can also set the config through environment variables. For instance:

```bash
export PBC_COMPRESSION_TYPE="none"
```
##### Rate limiter configuration

Prebid Cache's rate limiting feature, that has the downside of considerable memory consumption, is enabled by default for a maximum of 100 requests per second. From the [config.yaml](./config.yaml) file, use the `rate_limiter.enabled` and `rate_limiter.num_requests` options to either disable the rate limiter or modify its request capacity. For instance adding the following in the `config.yaml` file:

```yaml
rate_limiter:
  enabled: false
```

disables the rate limiter. We could also disable it by setting the following environment variable:

```bash
export PBC_RATE_LIMITER_ENABLED="false"
```

In contrast, we could keep the rate limiter running and set its maximum number of requests to a value other than 100. For instance, to set them to 150, we could modify the `num_requests` field inside [config.yaml](./config.yaml):

```yaml
rate_limiter:
  num_requests: 150
```

Or via the following environment variable:
```bash
export PBC_RATE_LIMITER_NUM_REQUESTS=150
```

### Docker

Prebid Cache works in Docker out of the box. It comes with a Dockerfile that creates a container, downloads all dependencies, and instantly installs a working image for us to run Prebid Cache right away.
Using the `docker build` command we specify an image name and the location of the folder where we cloned or downloaded Prebid Cache to create an image ready to run. If we cloned Prebid Cache in `~/go/src/github.com/prebid/prebid-cache`, then we could use the command that follows to create the image `prebid-cache`.
```bash
docker build -t prebid-cache ~/go/src/github.com/prebid/prebid-cache
```
We can run Prebid Cache using the newly created image:
```bash
docker run -p 8000:8000 -t prebid-cache
```

### Profiling

[pprof stats](http://artem.krylysov.com/blog/2017/03/13/profiling-and-optimizing-go-web-applications/) can be accessed from a running app on `localhost:2525`
