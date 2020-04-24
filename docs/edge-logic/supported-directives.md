## Supported Directives

This section lists all the directives you can use in the CDN360 Edge Logic. Although most of them are unmodified from the open-source version of NGINX, many have been modified to better suit the needs of a CDN proxy server. We also introduced some proprietary directives. 

Each non-proprietary directive includes a direct link to the official NGINX documentation. A detailed description is provided if the directive has been modified in any way from the original version, such as limitations on the parameters of some directives. 

In the following list, the "standard" directives are available to all customers, and they should cover the most common use cases. The "advanced" ones are usually more resource -consuming and they will be granted on a case-by-case basis. Please contact customer service if you need one or more of them.


### [add_header](http://nginx.org/en/docs/http/ngx_http_headers_module.html#add_header) (standard)

This directive modifies the response headers to the client. We made a few major changes to the open-source version:
1. The following new parameter has been introduced to control the behavior more precisely:
```nginx
policy=repeat|overwrite|preserve
```
```overwrite```: If the header being added exists in the upstream response, the local configuration overrides the header value. If you want to remove a header, set the value to an empty string.

```preserve```: If the header being added exists in the upstream response, the header value is not changed;

```repeat```: (default) Add the header to the client response, regardless of whether the header exists in the upstream response.

The policy parameter also supports variables; the value must be one of the three above.

**Limitation**: For the following "built-in" headers, the behavior is always fixed regardless of the configured policy:

| **"built-in" Header Name** | **Behavior** |
| ---- | ---- |
| ```Cache-Control``` | ```repeat``` |
| ```Link``` | ```repeat``` |
| ```Last-Modified``` | ```overwrite``` |
| ```ETag``` | ```overwrite``` |


If needed, [proxy_hide_header](#proxy_hide_header) can be used to remove the "Cache-Control" or "Link" headers from the origin.

2. The new parameter ```qtl_if(condition)``` has been introduced to allow adding the header based on some condition. If the condition is true, the ```add_header``` directive adds the header and the value to the downstream response based on the policy. The ```qtl_if``` parameter should always be at the end of the directive configuration. A condition may be any of the following,

*   A variable name; false if the value of a variable is an empty string.
*   Comparison of a variable with a string using the "=" and "!=" operators.
*   Matching a variable against a regular expression using the operators "\~" (for case-sensitive matching) and "\~\*" (for case-insensitive matching). Negative operators "!\~" and "!\~\*" are also available. If a regular expression includes the "}" or ";" characters, enclose the whole expression in single or double quotes.

3. Another change made to this directive is to merge the configurations across different levels (server/location/if). However, if the same header name appears in multiple levels, the configuration of only the deepest layer takes effect for that header.

 Example configurations:
```nginx
add_header X-Cache-Status $upstream_cache_status policy=preserve;
```
Example with variable:
```nginx
set $cache_status_method "preserve";  
if ($arg_debug = cache_status)
    set $cache_status_method "overwrite";
}
add_header X-Cache-Status $upstream_cache_status policy=$cache_status_method;
```

### [allow](http://nginx.org/en/docs/http/ngx_http_access_module.html#allow)
<span class="badge">standard</span><span class="badge">ETA: July 2020</span>

Allows access for the specified network or address. (Work in progress to make this only apply on edge.)


### [auth_request](http://nginx.org/en/docs/http/ngx_http_auth_request_module.html#auth_request) (advanced)

Enables authorization based on the result of a subrequest and sets the URI to which the subrequest will be sent. No change to the public version. 


### [auth_request_set](http://nginx.org/en/docs/http/ngx_http_auth_request_module.html#auth_request_set) (advanced)

Sets the request variable to the given value after the authorization request completes. No change to the public version. 


### [break](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#break) (standard)

Stops processing the current set of ngx_http_rewrite_module directives. No change to the public version. 


### [deny](http://nginx.org/en/docs/http/ngx_http_access_module.html#deny) (standard, ETA: July 2020)

Denies access for the specified network or address. (Work in progress to make this only apply on edge.)

### custom_log_field (CDN360 Proprietary, advanced, ETA: May 2020)

**Syntax**: ```custom_log_field {custom log field id} {value or variable};```<br/>
**Default**: ```-```<br/>
**Context**: ```http, server, location, if in location```

This directive allows the users to add up to 2 customized fields into the access log. They can be referred to by the keywords "custom1" and "custom2" when you [configure the format](https://docs.google.com/document/d/155m9F0oFIDXRLeFmLqbdb0gWiHAyTWB8rPLWdRVGXoI/edit#heading=h.owglsmu6p2rb) of the download log, or using our [advanced traffic analysis](https://obd.quantil.com) tool.


### [error_page](http://nginx.org/en/docs/http/ngx_http_core_module.html#error_page) (advanced)

Defines the URI that will be shown for the specified errors. No change to the public version. 


### [eval_func](https://docs.google.com/document/d/1T4NVOiiv_OlYA6nzDcoTm7MpQMBz5E1nr-W78_7GNiQ/edit#bookmark=id.ff3eprwz0chu) (CDN360 Proprietary, advanced)

**Syntax**:    eval_func $result {function name} {parameters};
**Default**:    -
**Context**:  http, server, location, if in location

This is a directive to perform some common encoding, decoding, hash, hash-mac, encryption, decryption and comparison algorithms. It is added to the [rewrite module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html).  Supported functions are:

| **Type** | **Name** | **Syntax** |
|----------|----------|------------| 
| hash | **SHA256**, **MD5** | ```eval_func $output SHA256 $input;``` |
| BASE64<br>codec | BASE64_ENCODE<br>**BASE64_DECODE** | ```eval_func $output BASE64_ENCODE $input;``` |
| URL<br>codec | URL_ENCODE<br>**URL_DECODE** | ```eval_func $output URL_ENCODE $input;``` |
| HEX<br>codec | HEX_ENCODE<br>**HEX_DECODE** | ```eval_func $output HEX_ENCODE $input;``` |
| AES<br>cipher | **ENCRYPT_AES_256_CBC**<br>**DECRYPT_AES_256_CBC** |```eval_func $output ENCRYPT_AES_256_CBC $key $iv $message;``` |
| HMAC<br>generation | **HMAC** | ```eval_func $output HMAC $key $message {digest algorithm};```<br>```{digest algorithm}``` can be ```MD5```, ```SHA1```, ```SHA256``` |
| integer<br>comparator | COMPARE_INT | ```eval_func $output COMPARE_INT $data1 $data2;```<br>```$output``` will be "1" when ```$data1 > $data2```. "0" and "-1" for the other cases. |

**NOTE:** The output value of the functions in **bold** is a binary string which may not be printable. You need to use the BASE64_ENCODE, URL_ENCODE or HEX_ENCODE to convert it to a printable format.

Examples:
```nginx
    eval_func $secret_key SHA256 "mySecret123!";
    eval_func $text HEX_ENCODE $secret_key;
    #the value of $text should be 
    "ad8fdcda140f607697ec80a8c38e86af19f4bb79ee7f7544abcfaadd827901af"
    eval_func $aseout ENCRYPT_AES_256_CBC $secret_key $iv $message;
    eval_func $hmacout HMAC $secret_key $message SHA256;
```

### [expires](http://nginx.org/en/docs/http/ngx_http_headers_module.html#expires) (standard)

Enables or disables adding or modifying the “Expires” and “Cache-Control” response header fields. No change to the public version. 


### [gzip_types](http://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip_types) (advanced)

Enables gzipping of responses for the specified MIME types in addition to “text/html”. No change to the public version. We always have gzip on. 


### [if](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#if) (standard)

Control the server behavior based on the specified condition. No change to the public version, but [use with caution](#ifcaution)! 


### [include](http://nginx.org/en/docs/ngx_core_module.html#include) (standard)

Includes another file, or files matching the specified mask, into configuration. No change to the public version. 


### [internal](http://nginx.org/en/docs/http/ngx_http_core_module.html#internal) (advanced)

 Specifies that a given location can only be used for internal requests. No change to the public version. 

### [limit_rate](http://nginx.org/en/docs/http/ngx_http_core_module.html#limit_rate) (standard)

Limits the rate of response transmission to a client. We limit the value to be an integer in [1-8] followed by ‘m’.

### [limit_rate_after](http://nginx.org/en/docs/http/ngx_http_core_module.html#limit_rate_after) (standard)

Sets the initial amount of traffic after which the further transmission of a response to a client will be rate limited. We limit the value to be an integer in [1-8] followed by ‘m’.

### [location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) (standard)

Sets configuration depending on a request URI. No change to the public version. 

### origin_connect_timeout (CDN360 Proprietary, advanced)

**Syntax**:    origin_connect_timeout time;
**Default**:    origin_connect_timeout 60s;
**Context**:  http, server, location

This is a wrapper of the [proxy_connect_timeout](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_connect_timeout) directive. It defines a timeout for establishing a connection with the origin server. The value is limited to an integer in [1,30] followed by ‘s’.

### origin_header_modify (CDN360 Proprietary, advanced)

**Syntax**:    origin_header_modify field value policy=value qtl_if(condition);
**Default**:    -
**Context**:  http, server, location, if in location

This directive can be used to add, delete, or overwrite the response header fields from the origin **before** any other processing. The directive supports NGINX variables.

Possible values of policy are ```repeat, overwrite,``` and ```preserve.``` The policy parameter supports a variable as a value. The default policy is ```repeat```.

*   The ```repeat``` policy always adds the header and the value into the upstream response.
*   The ```overwrite``` policy overwrites the value if the header already exists in the upstream response. Otherwise, it adds the header and the value into the upstream response.
*   The ```preserve``` policy adds the header and the value into the upstream response only if the header does not exist in the upstream response.

The parameter ```qtl_if``` is introduced to add the header based on the condition. A condition can be one of the following:

*   A variable name; false if the value of a variable is an empty string.
*   A comparison of a variable with a string using the "=" and "!=" operators.
*   The matching of a variable against a regular expression using the operators "\~" (for case-sensitive matching) and "\~\*" (for case-insensitive matching). Negative operators "!\~" and "!\~\*" are also available. If a regular expression includes the "}" or ";" characters, enclose the whole expression in single or double quotes.

Examples: 

Added a header ```X-Status``` based on origin's status code:
```nginx
origin_header_modify X-Status Good qtl_if($upstream_response_status ~ "^[23]");
origin_header_modify X-Status ClientErr qtl_if($upstream_response_status ~ "^4");
origin_header_modify X-Status ServerErr qtl_if($upstream_response_status ~ "^5");
```
Delete the ```Cache-Control``` header in the origin's response:
```nginx
origin_header_modify Cache-Control "" policy=overwrite;
```
The directive is merged across different levels (http/server/location/location if). If the same header name exists in different levels, the configuration for that header name in the innermost level takes effect.

Although CDN360 has hierarchical cache structure, the directive changes the header only in the origin response. 


### origin_limit_rate (CDN360 Proprietary, standard)

**Syntax**: ```origin_limit_rate rate;```<br>
**Default**: ```origin_limit_rate 0;```<br>
**Context**: ```http, server, location```

This a wrapper of the [proxy_limit_rate](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_limit_rate) directive. It limits the speed of reading the response from the origin server.

### origin_pass (CDN360 Proprietary, standard)

**Syntax**: ```origin_pass _origin_name[URI]_;```<br>
**Default**: ```none```<br>
**Context**: ```location, if in location```

This directive specifies the origin to fetch the content. It is a wrapper of the NGINX [proxy_pass](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) directive. It takes one parameter that is an origin name specified in the "origins" field of the property JSON. The origin name can be optionally followed by a URI. Variables can be used in the URI. Examples:
```nginx
origin_pass my_origin;    #URI is not specified, 
origin_pass my_origin/$uri;
origin_pass my_origin/abc/$uri;
```
If the URI is omitted, the variable ```$request_uri``` (with all the query strings) is appended automatically when accessing the origin.

### origin_read_timeout (CDN360 Proprietary, advanced)

**Syntax**:    origin_read_timeout time;
**Default**:    origin_read_timeout 60s;
**Context**:  http, server, location

This is a wrapper of the [proxy_read_timeout](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout) directive. It defines a timeout for reading a response from the origin server. The value is limited to an integer in [1,600] followed by ‘s’ OR an integer in [1,10] followed by ‘m’. 


### origin_send_timeout (CDN360 Proprietary, advanced)

**Syntax**:    origin_send_timeout time;
**Default**:    origin_send_timeout 60s;
**Context**:  http, server, location

This is a wrapper of the [proxy_send_timeout](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_send_timeout) directive. It sets a timeout for transmitting a request to the origin server. The value is limited to an integer in [1,600] followed by ‘s’ OR an integer in [1,10] followed by ‘m’.

### [origin_set_header](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) (CDN360 Proprietary, standard)

**Syntax**:  origin_set_header field value qtl_if(condition);
**Default**: origin_set_header host $host;
**Contexts**: http, server, location, if in location

This is a wrapper of the [proxy_set_header](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) directive to allow redefining (overwriting) or appending fields to the request header passed to the origin server. The following changes were made to the open-source version:

1. This directive merges the configurations across different levels (server/location/if). However, if the same header name appears in multiple levels, only the deepest layer’s configuration takes effect for that header. Because CDN360 has a hierarchical cache structure, we try to make sure the headers set by this directive appear only in the requests to the origin servers (not parent cache servers).
2. The new parameter  ```qtl_if(condition)``` can be used to set the header based on some conditions. If the condition is true, the directive takes effect. The ```qtl_if``` parameter should always be configured at the end of the directive configuration. A condition may be one of the following:

*   A variable name; false if the value of a variable is an empty string.
*   Comparison of a variable with a string using the "=" and "!=" operators.
*   Matching a variable against a regular expression using the operators "\~" (for case-sensitive matching) and "\~\*" (for case-insensitive matching). Negative operators "!\~" and "!\~\*" are also available. If a regular expression includes the "}" or ";" characters, enclose the whole expressions in single or double quotes.

Because of the hierarchical cache structure, the built-in variables $scheme and $remote_addr cannot be used. If you need to pass the scheme or IP used by the client to the origin servers, use the following variables:

*   $request_scheme : scheme used by the client
*   $client_real_ip:  client’s IP address
*   $client_country_code:  client’s ISO3166 country code

For example:
```nginx
origin_set_header X-Forwarded-For $client_real_ip;
```

### [proxy_buffering](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering) (standard)

Enables or disables buffering of responses from the proxied server. No change to the public version. 

### [proxy_cache_bypass](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_bypass) (standard)

Defines conditions under which the response will not be taken from a cache. No change to the public version. 

### [proxy_cache_methods](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_methods) (standard)

Specify the HTTPS methods whose response will be cached.

### [proxy_cache_min_age](https://docs.google.com/document/d/1T4NVOiiv_OlYA6nzDcoTm7MpQMBz5E1nr-W78_7GNiQ/edit#bookmark=id.geuwzglsxykl) (CDN360 Proprietary, standard)

**Syntax**:        proxy_cache_min_age time;
**Default**:        proxy_cache_min_age 0s;
**Context**:       http, server, location, if in location

Description:

This directive allows you to configure the minimum cache time. If the received max-age from the origin is less than the specified minimum age, the max-age value is set to the configured minimum age value. For example, if the max-age value in the received HTTP header is 100s and the configured minimum age value is 200s, the effective cache time will be 200s. 

NGINX calculates the cache time from the headers in the upstream response or from the NGINX directives in the following order:

X-Accel-Expires > Cache-Control (max-age) > Expires > proxy_cache_valid (NGINX directive)

 When NGINX calculates the cache time from max-age value in the Cache-Control header, it compares the value with the value configured in the  proxy_cache_min_age and updates the cache time accordingly. Otherwise, NGINX ignores the value in the proxy_cache_min_age directive.

 Note: The time variable in this directive can have a number with one of the following suffixes or combination of the following suffixes:

*   s = seconds (default, example: 10s)
*   m = minutes (example: 5m)
*   h = hours (example: 1h)
*   d = days (example: 1d)
*   w = weeks (example: 1w)
*   M = months (example: 2M)
*   y = years (example: 1y)

If there is no suffix in the time, the configured value is considered in seconds.

### [proxy_cache_min_uses](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_min_uses) (advanced)

Sets the number of requests after which the response will be cached. No change to the public version. 

### [proxy_cache_use_stale](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_use_stale) (advanced)

Determines in which cases a stale cached response can be used during communication with the proxied server. No change to the public version. 

### [proxy_cache_valid](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_valid) (standard)

**Contexts:** http, server, location, if in location

Sets caching time for different response codes. Changing the public version to enable it in if in location (ETA: June 2020).

### [proxy_cache_vary](https://docs.google.com/document/d/1T4NVOiiv_OlYA6nzDcoTm7MpQMBz5E1nr-W78_7GNiQ/edit#bookmark=id.mu0spq8pii23) (CDN360 Proprietary, advanced)

**Syntax**: proxy_cache_vary on | off;
**Default**: proxy_cache_vary off;
**Context**: http, server, location

If proxy_cache_vary is "on", the CDN360 cache servers honor the "Vary" response header from the origin and cache different variations separately. However, the varied contents must be purged using "directory purge". An error will be returned if "file purge" is used for varied contents.

If proxy_cache_vary is "off", the CDN360 cache servers do not cache any response with the "Vary" header.

Related reading: [The support (and non-support) of "Vary"](#support-of-vary).

### [proxy_cookie_domain](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cookie_domain) (advanced)

Sets a text that should be changed in the domain attribute of the “Set-Cookie” header fields of a proxied server response. No change to the public version. 

### [proxy_cookie_path](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cookie_path) (advanced)

Sets a text that should be changed in the path attribute of the “Set-Cookie” header fields of a proxied server response. No change to the public version. 

### [proxy_hide_header](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_hide_header) (standard)

Sets response header fields that will not be passed to the client. No change to the public version. 

### proxy_ignore_cache_control (CDN360 Proprietary, advanced)

**Syntax:** proxy_ignore_cache_control directives…;
**Default:** none
**Contexts:** http, server, location, if in location 

Disables processing of certain cache-control directives in the proxy server. The following directives can be ignored: 

*   no-cache
*   no-store
*   private
*   max-age
*   s-maxage
*   stale-while-revalidate
*   stale-if-error

Examples: ignore the no-cache and no-store directives:
```nginx
proxy_ignore_cache_control no-cache no-store;
```
Please note that this directive does not modify the "Cache-Control" header from the origin.

### [proxy_ignore_headers](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ignore_headers) (standard)

Disables processing of certain response header fields from the proxied server. No change to the public version. 

### [proxy_next_upstream](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream) (advanced)

Specifies in which cases a request should be passed to the next upstream server. No change to the public version. 

### [proxy_next_upstream_timeout](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream_timeout) (advanced)

Limits the time during which a request can be passed to the next upstream server. No change to the public version.

### [proxy_next_upstream_tries](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream_tries) (advanced)

Limits the number of possible tries for passing a request to the next upstream server. No change to the public version. 

### [proxy_no_cache](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_no_cache) (standard)

Defines conditions under which the response will not be saved to a cache. No change to the public version. 

### [proxy_pass_header](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass_header) (standard)

Permits passing [otherwise disabled](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_hide_header) header fields from a proxied server to a client. No change to the public version. 


### [proxy_redirect](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_redirect) (advanced)

Sets the text that should be changed in the “Location” and “Refresh” header fields of a proxied server response. No change to the public version. 

### [proxy_ssl_protocols](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_protocols) (advanced)

Enables the specified protocols for requests to a proxied HTTPS server. No change to the public version.

### [realtime_log_sample_rate](https://docs.google.com/document/d/1T4NVOiiv_OlYA6nzDcoTm7MpQMBz5E1nr-W78_7GNiQ/edit#heading=h.e8ntes266nrp) (CDN360 Proprietary, advanced, ETA June 2020)

**Syntax:** realtime_log_sample_rate {sample rate};
**Default:** none
**Contexts:** http, server, location, if in location

This CDN360-proprietary directive sets the sample rate of the [Real-Time Log](https://docs.google.com/document/d/1ju14e1arEPLsGFmxaYExjcdO7bwdYQ-m4h7tdbqpEZI/edit#heading=h.tbbol2vdsupw). The parameter value can be an integer in [0,65536] . 0 turns of the real time logging. Variable is supported. By default, the sample rate is set for the entire site by the ```realTimeLog``` field of the property JSON. This directive can be used to change the sample rate or turn off real-time logging selectively for some locations.

### [return](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return) (standard)

Stops processing and returns the specified code to a client. No change to the public version. 

### [rewrite](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite) (standard)

Rewrite the request URI when a regular expression pattern is matched. No change to the public version. 

### [set](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#set) (standard)

Sets a value for the specified variable. No change to the public version. 

### [satisfy](http://nginx.org/en/docs/http/ngx_http_core_module.html#satisfy) (standard)

Allows access if all (all) or at least one (any) of the ngx_http_access_module, ngx_http_auth_basic_module, ngx_http_auth_request_module, or ngx_http_auth_jwt_module modules allow access. No change to the public version. 


### sanitize_accept_encoding (CDN360 Proprietary, advanced)

**Syntax:** sanitize_accept_encoding enc1 enc2 … ;
**Default:** sanitize_accept_encoding gzip;
**Contexts:** http, server

This directive processes the incoming "Accept-Encoding" header to consolidate the value. The goal is to increase the cache efficiency and hit ratio by limiting the maximum number of variations due to the "Accept-Encoding" header to 5.

Users can specify up to four combinations of content encoding algorithms after this directive. Each combination is a comma-separated list of one or more content-encoding algorithms, such as "gzip,br" or "br". For each request from the client, the CDN360 proxy server tries to match the Accept-Encoding header with the specified combinations from left to right. If all the algorithms in a combination are found in the header, the header value is replaced with that combination. If no match is found, the header value is set to "identity".

For example: if the configuration is:
```nginx
sanitize_accept_encoding "gzip,br" "gzip" "deflate" "br";
```
The processing logic will be:
```nginx
if (A-E-header.contains("gzip") && A-E-header.contains("br"))
    A-E-header="gzip,br";
else if (A-E-header.contains("gzip")) A-E-header="gzip";
else if (A-E-header.contains("deflate")) A-E-header="deflate";
else if (A-E-header.contains("br")) A-E-header="br";
else A-E-header="identity";
```

### [secure_link](http://nginx.org/en/docs/http/ngx_http_secure_link_module.html#secure_link) (advanced)

Defines a string with variables from which the checksum value and lifetime of a link will be extracted. No change to the public version.

### [secure_link_md5](http://nginx.org/en/docs/http/ngx_http_secure_link_module.html#secure_link_md5) (advanced)

Defines an expression for which the MD5 hash value will be computed and compared with the value passed in a request. No change to the public version.

### [secure_link_secret](http://nginx.org/en/docs/http/ngx_http_secure_link_module.html#secure_link_secret) (standard)

Defines a secret word used to check authenticity of requested links. No change to the public version.

### [slice](http://nginx.org/en/docs/http/ngx_http_slice_module.html#slice) (advanced)

**Contexts:** http, server

Sets the size of the slice when fetching large files from the origin. We made a change that prevents this directive from being used in any "location" block. If slice is enabled, the entire website must use the same slice size. This behavior avoids potential problems when trying to chase origin’s redirection. The value is limited to 0 OR an integer in [4,512] followed by ‘m’.

### sorted_querystring_filter_parameter (CDN360 Proprietary, advanced)

**Syntax:** sorted_querystring_filter_parameter param1 param2 … ;
**Default:** none
**Contexts:** http, server, location, if in location

Removes some query parameters from the variable ```$sorted_querystring_args```.
This feature is implemented on top of this [open-source project](https://github.com/wandenberg/nginx-sorted-querystring-module).

### [sub_filter](http://nginx.org/en/docs/http/ngx_http_sub_module.html#sub_filter) (advanced)

Sets a string to replace in the response and a replacement string. No change to the public version.

### [sub_filter_last_modified](http://nginx.org/en/docs/http/ngx_http_sub_module.html#sub_filter_last_modified) (advanced)

Allows preserving the “Last-Modified” header field from the original response during replacement to facilitate response caching. No change to the public version.

### [sub_filter_once](http://nginx.org/en/docs/http/ngx_http_sub_module.html#sub_filter_once) (advanced)

Indicates whether to look for each string to replace once or repeatedly. No change to the public version.

### [sub_filter_types](http://nginx.org/en/docs/http/ngx_http_sub_module.html#sub_filter_types) (standard)

Enables string replacement in responses with the specified MIME types in addition to “text/html”. No change to the public version.


### [valid_referers](http://nginx.org/en/docs/http/ngx_http_referer_module.html#valid_referers) (standard)

Specifies the “Referer” request header field values that will cause the embedded $invalid_referer variable to be set to an empty string. No change to the public version.