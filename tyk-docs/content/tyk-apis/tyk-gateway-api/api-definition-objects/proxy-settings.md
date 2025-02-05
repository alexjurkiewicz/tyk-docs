---
date: 2017-03-13T15:08:55Z
Title: Proxy Settings in the API Definition
menu:
  main:
    parent: "API Definition Objects"
weight: 4
---

The `proxy` section outlines the API proxying functionality. You can define where Tyk should listen, and where Tyk should proxy traffic to.

* `proxy.preserve_host_header`: Set to `true` to preserve the host header. If `proxy.preserve_host_header` is set to `true` in an API definition then the host header in the outbound request is retained to be the inbound hostname of the proxy.

* `proxy.listen_path`: The path to listen on, e.g. `/api` or `/`. Any requests coming into the host, on the port that Tyk is configured to run on, that go to this path will have the rules defined in the API Definition applied. Versioning assumes that different versions of an API will live on the same URL structure. If you are using URL-based versioning (e.g. `/v1/function`, `/v2/function/`) then it is recommended to set up a separate non-versioned definition for each version as they are essentially separate APIs.
    
Proxied requests are literal, no re-writing takes place, for example, if a request is sent to the listen path of: `/listen-path/widgets/new` and the URL to proxy to is `http://your.api.com/api/` then the *actual* request that will land at your service will be: `http://your.api.com/api/listen-path/widgets/new`.
    
This behaviour can be circumvented so that the `listen_path` is stripped from the outgoing request. See the section on `strip_listen_path` below.

* `proxy.target_url`: This defines the target URL that the request should be proxied to if it passes all checks in Tyk.

* `proxy.strip_listen_path`: By setting this to `true`, Tyk will attempt to replace the `listen-path` in the outgoing request with an empty string. This means that in the above scenario where `/listen-path/widgets/new` and the URL to proxy to is `http://your.api.com/api/` becomes `http://your.api.com/api/listen-path/widgets/new`, actually changes the outgoing request to be: `http://your.api.com/api/widgets/new`.

* `proxy.enable_load_balancing`: Set this value to `true` to have a Tyk node distribute traffic across a list of servers. **Required: ** You must fill in the `target_list` section.

* `proxy.target_list`: A list of upstream targets (can be one or many hosts).

* `proxy.check_host_against_uptime_tests`: If uptime tests are enabled, Tyk will check the hostname of the outbound request against the downtime list generated by the host checker. If the host is found, then it is skipped.

* `proxy.service_discovery`: The service discovery section tells Tyk where to find information about the host to proxy to. In a clustered environment this is useful if servers are coming online and offline dynamically with new IP addresses. The service discovery module can pull out the required host data from any service discovery tool that exposes a RESTful endpoint that outputs a JSON object.

```json
{
  "enable_load_balancing": true,
  "service_discovery": {
    "use_discovery_service": true,
    "query_endpoint": "http://127.0.0.1:4001/v2/keys/services/multiobj",
    "use_nested_query": true,
    "parent_data_path": "node.value",
    "data_path": "array.hostname",
    "port_data_path": "array.port",
    "use_target_list": true,
    "cache_timeout": 10
  },
}
```
        

* `proxy.service_discovery.use_discovery_service`: Set this to `true` to enable the discovery module.

* `proxy.service_discovery.query_endpoint`: The endpoint to call.

* `proxy.service_discovery.data_path`: The namespace of the data path. For example, if your service responds with:

```json
{
  "action": "get",
  "node": {
    "key": "/services/single",
    "value": "http://httpbin.org:6000",
    "modifiedIndex": 6,
    "createdIndex": 6
  }
}
```

Then your name space would be `node.value`.

* `proxy.service_discovery.use_nested_query`: Sometimes the data you are retrieving is nested in another JSON object. For example, this is how Etcd responds with a JSON object as a value key:

```json
{
  "action": "get",
  "node": {
    "key": "/services/single",
    "value": "{\"hostname\": \"http://httpbin.org\", \"port\": \"80\"}",
    "modifiedIndex": 6,
    "createdIndex": 6
  }
}
```


In this case, the data actually lives within this string-encoded JSON object. So in this case, you set the `use_nested_query` to `true`, and use a combination of the `data_path` and `parent_data_path` (below)

* `proxy.service_discovery.parent_data_path`: This is the namespace of where to find the nested value In the above example, it would be `node.value`.
    
You would then change the `data_path` setting to be `hostname`.
    
Tyk will decode the JSON string and then apply the `data_path` namespace to that object in order to find the value.

* `proxy.service_discovery.port_data_path`: In the above nested example, we can see that there is a separate PORT value for the service in the nested JSON. In this case you can set the `port_data_path` value and Tyk will treat `data_path` as the hostname and zip them together (this assumes that the hostname element does not end in a slash or resource identifier such as `/widgets/`).
    
In the above example, the `port_data_path` would be `port`.

* `proxy.service_discovery.target_path`: The target path to append to the host:port combination provided by the service discovery engine.

* `proxy.service_discovery.use_target_list`: If you are using load_balancing, set this value to `true` and Tyk will treat the data path as a list and inject it into the target list of your API definition.

* `proxy.service_discovery.cache_timeout`: Tyk caches target data from a discovery service. In order to make this dynamic you can set a cache value when the data expires and new data is loaded.

* `proxy.disable_strip_slash`: This boolean option allows you to add a way to disable the stripping of the slash suffix from a URL.

#### Internal proxy setup

The transport section allows you to specify a custom proxy and set the minimum TLS versions and any SSL ciphers.

This is an example of `proxy.transport` definition followed by explanations for every field.
```json
{
  "transport": {
    "proxy_url": "http(s)://proxy.url:1234",
    "ssl_min_version": 771,
    "ssl_ciphers": [
      "TLS_RSA_WITH_AES_128_GCM_SHA256", 
      "TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA"
    ],
    "ssl_insecure_skip_verify": true,
    "ssl_force_common_name_check": false
  }
}
```
* `proxy.transport.proxy_url`: Use this setting to specify your custom forward proxy and port.

* `proxy.transport.ssl_min_version`: Use this setting to specify your minimum TLS version:

&nbsp;&nbsp;You need to use the following values for this setting:

| TLS Version   | Value to Use   |
|---------------|----------------|
|      1.0      |      769       |
|      1.1      |      770       |
|      1.2      |      771       |
|      1.3      |      772       |

* `proxy.transport.ssl_ciphers`: You can add `ssl_ciphers` which takes an array of strings as its value. Each string must be one of the allowed cipher suites as defined at https://golang.org/pkg/crypto/tls/#pkg-constants

* `proxy.transport.ssl_insecure_skip_verify`: Boolean flag to control at the API definition whether it is possible to use self-signed certs for some APIs, and actual certs for others. This also works for `TykMakeHttpRequest` & `TykMakeBatchRequest` in virtual endpoints.

* `proxy.transport.ssl_force_common_name_check`: Use this setting to force the validation of a hostname against the certificate Common Name.
