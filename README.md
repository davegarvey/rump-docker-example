Demonstrates using Rump to migrate data across two separate Redis instances.

There are two Gateway containers `gateway-1` and `gateway-2` which each target their own Redis container `redis-1` and `redis-2`.

The `rump` container runs the Rump application every 5 seconds, which pulls data from `redis-1` and writes it to `redis-2`. It uses the pattern `apikey-*` to only move keys which start with `apikey-`, which means it will not move other keys such as analytics data.

The `docker-compose` file uses environment variables on the Gateway containers to override the Redis host setting so that each Gateway targets a corresponding Redis.

## 1. Create the deployment

Use `docker-compose up` to create the demo deployment.

The docker output will show the Rump container repeating this:

```
rump_1       | Using scan pattern: apikey-*
rump_1       |
rump_1       | signal: exit
rump_1       | done
```

The 'blank' output on line 2 shows that no data has been transferred.

## 2. Verify initial API access state

Check that we do not have access to the default test API on either Gateway.

Running both of these commands

```
curl http://127.0.0.1:8081/tyk-api-test/
```

```
curl http://127.0.0.1:8082/tyk-api-test/
```

Should result in:

```
{
    "error": "Authorization field missing"
}
```

This means that the API is active on both Gateways but we do not have access yet as the API configuration is expecting us to authenticate.

## 3. Create API key on Gateway 1

Create an API key on Gateway 1 which allows access to the API:

```
curl http://127.0.0.1:8081/tyk/keys \
-H 'x-tyk-authorization: 352d20ee67be67f6340b4c0605b044b7' \
-d '{"allowance":1000, "rate":1000, "per":60, "expires":-1, "quota_max":-1, "quota_renews":1406121006, "quota_remaining":0, "quota_renewal_rate":60, "access_rights":{ "1":{"api_name":"Tyk Test API", "api_id":"1"}}}'
```

This will return something similar to:

```
{
  "key":"df9ecf5f49b04fa9815f6481708310a2",
  "status":"ok",
  "action":"added",
  "key_hash":"95b32314"
}
```

You will need to note the `key` value in order to test the API in step 5.

## 4. Check to see if Rump is migrating the new data

In the Docker log output for the Rump container you should now see that key data is being read and written:

```
rump_1       | Using scan pattern: apikey-*
rump_1       | rw
rump_1       | signal: exit
rump_1       | done
```

The `rw` on line 2 indicates that a single record has been read (`r`) and written (`w`).

## 5. Verify API access state again

Now that we have an API key, we can use it to test API access. 

Because of Rump migrating the key data from `redis-1` to `redis-2`, we should be able to access the API via both Gateways. Update the `authorization` header in the `curl` commands to use the key generated in step 3.

Gateway 1:

```
curl http://127.0.0.1:8081/tyk-api-test/get \
-H 'authorization: df9ecf5f49b04fa9815f6481708310a2'
```

Gateway 2:

```
curl http://127.0.0.1:8082/tyk-api-test/get \
-H 'authorization: df9ecf5f49b04fa9815f6481708310a2'
```

Both of these commands should result in a successful response, similar to:

```
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Accept-Encoding": "gzip",
    "Authorization": "df9ecf5f49b04fa9815f6481708310a2",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.54.0"
  },
  "origin": "172.21.0.1, 101.100.177.76, 172.21.0.1",
  "url": "https://httpbin.org/get"
}
```