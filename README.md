# pod-to-pod authentication with oidc

This demo shows how to utilize kubernetes built-in service account tokens to authenticate between pods.

This mechanism can restrict what pod is allowed to communicate with what other pod.
It utilizes [ServiceAccount token volume projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection) to get JWT-s that are scoped to the appropriate audience. Using the same audience for multiple target services is highly discouraged as it can lead to replay-attacks.


## Setup demo environment

```bash
kind create cluster --config ./cluster.yaml
kubectl apply -f ./oidc-reviewer.yaml
kubectl apply -f ./echo-server-with-oidc-auth.yaml
kubectl apply -f ./oidc-test.yaml
kubectl apply -f ./echo-server.yaml
```


## Testing scenarios

### Valid request

Using the correct service account name and the correct audience results in the request passing through correctly:

```bash
kubectl exec -it -n default deploy/oidc-test-valid -- sh -c 'curl -v -I -H "Authorization: Bearer $(cat /var/run/secrets/echo-server/token)" echo-server.echo-server.svc.cluster.local'
```

Request succeeds with a 200 and passes through the proxy

```
*   Trying 10.96.93.28:80...
* Connected to echo-server.echo-server.svc.cluster.local (10.96.93.28) port 80 (#0)
> HEAD / HTTP/1.1
> Host: echo-server.echo-server.svc.cluster.local
> User-Agent: curl/8.1.2
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjhlOUt2Ny14bzJ5ZXRvMWViaEJiSW5aZzFfbTRWYld4VGlsMGF2T1p2b2cifQ.eyJhdWQiOlsiZWNoby1zZXJ2ZXIuZWNoby1zZXJ2ZXIuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjg3MzQ4NTQwLCJpYXQiOjE2ODczNDc5NDAsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJvaWRjLXRlc3QtdmFsaWQtNWM2NzhjYzhmOS1ucno3OSIsInVpZCI6ImQ1YjBmMmVlLWQwMzMtNDdlZS04Njc4LWE4OTg1YzQ5YjJjZSJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoib2lkYy10ZXN0IiwidWlkIjoiNTZlNWI3MGMtNGM1MS00MTFjLWE4ZjQtYWI2MTllMDgyNTFiIn19LCJuYmYiOjE2ODczNDc5NDAsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0Om9pZGMtdGVzdCJ9.fnIW_4isR7vLlQkSQehbVXa8shhrNMNfXsIFPadlEbgSIY_fhvCOfnLxm1mUB7JlEX1ws7Sl3X6-qwZmKzC4_gXom3QpGNeXH6OuhCSZG7_TYRjR2VUCznzlpk_s-iKZug5PHzn61UU20gA4bXXiN6d4hRF3G1rccgHYixGDRrvAynu5zhqQIJeDI-3hSTg12WH79Le2afBwCUGMWxTvvzwQmxe55FBtCinh7ktHBh-sz3Gr32Ci_OLxx2RAXZaJq-cJviNxy6ajtMpgN7T3B43QiC1LooMIFqDKtmB6eeYExfM90O0PliON-ln9NP6IGku7yWvHT-rzYcOim45j1g
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Content-Length: 2423
Content-Length: 2423
< Content-Type: application/json; charset=utf-8
Content-Type: application/json; charset=utf-8
< Date: Wed, 21 Jun 2023 11:48:25 GMT
Date: Wed, 21 Jun 2023 11:48:25 GMT
< Etag: W/"977-yIPOvtzvjKjMQqkip2VwA59uKqc"
Etag: W/"977-yIPOvtzvjKjMQqkip2VwA59uKqc"
< Gap-Auth: system:serviceaccount:default:oidc-test
Gap-Auth: system:serviceaccount:default:oidc-test

<
* Connection #0 to host echo-server.echo-server.svc.cluster.local left intact

```


oauth2-proxy logs show the following:

```
10.244.0.36:54020 - 63cb4b54-9fcb-4f36-91b4-658a869df287 - system:serviceaccount:default:oidc-test [2023/06/21 11:48:25] echo-server.echo-server.svc.cluster.local HEAD / "/" HTTP/1.1 "curl/8.1.2" 200 0 0.012
```

### Using an invalid service account

When you try to use a service account that isn't configured with `OAUTH2_PROXY_ALLOWED_GROUPS`, the request will fail:

```sh
kubectl exec -it -n default deploy/oidc-test-invalid-sa -- sh -c 'curl -v -I -H "Authorization: Bearer $(cat /var/run/secrets/echo-server/token)" echo-server.echo-server.svc.cluster.local'
```

Request fails with a 403 response:

```
*   Trying 10.96.93.28:80...
* Connected to echo-server.echo-server.svc.cluster.local (10.96.93.28) port 80 (#0)
> HEAD / HTTP/1.1
> Host: echo-server.echo-server.svc.cluster.local
> User-Agent: curl/8.1.2
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjhlOUt2Ny14bzJ5ZXRvMWViaEJiSW5aZzFfbTRWYld4VGlsMGF2T1p2b2cifQ.eyJhdWQiOlsiZWNoby1zZXJ2ZXIuZWNoby1zZXJ2ZXIuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjg3MzQ4Nzk5LCJpYXQiOjE2ODczNDgxOTksImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJvaWRjLXRlc3QtaW52YWxpZC1zYS02Y2Q3ODdjNS05enRxcCIsInVpZCI6IjQ4MDBhZDE5LTc0ZGItNDI1NS1iZjU1LTQ2ZTg0MWVkZmVkOSJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjFjNWQ0MjcxLTlhNzEtNGJiZC04ZjRiLTdjOGU5ZWQwMjdjMyJ9fSwibmJmIjoxNjg3MzQ4MTk5LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWZhdWx0In0.VSHue0teNtfsdcqqiwoTVuq4VFGvMWJbkVJwgC2UN84ohiCCuiUQW_3W8642DbP8KHungWZLkh-d6xW3B1ToIpz6TfvdQXM5n2xQcKRtsP7W8V97Qil8_Neuq4rUB6SLkcQmm6E3Us32xAUeB4f0jlg_0Etgr_9itYZpsURiuuPY2pFrjTGZ8hOLFwzdQh6H50HU5x0UuDmL35f83bC5_fzYpo5iArgG8hQcL9RgHnzmqX9kNj0xkQJ332zigtVZSuGN62O86HGtrzfHE35HLIhjK-tcNXoj6CByIjPIA8Bx4_e5P9GpbFR8Ok3eWjm7yt1cM4ZvizwbkoceufeYdQ
>
< HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
< Date: Wed, 21 Jun 2023 11:51:36 GMT
Date: Wed, 21 Jun 2023 11:51:36 GMT
< Content-Type: text/html; charset=utf-8
Content-Type: text/html; charset=utf-8

<
* Connection #0 to host echo-server.echo-server.svc.cluster.local left intact
```
oauth2-proxy logs show the following:


```
10.244.0.35:51366 - b132f7c5-8a65-42d1-86cc-f74fb907d2f3 - system:serviceaccount:default:default [2023/06/21 11:51:36] [AuthFailure] Invalid authorization via session: removing session Session{email:system:serviceaccount:default:default user:system:serviceaccount:default:default PreferredUsername: token:true id_token:true created:2023-06-21 11:51:36.231564223 +0000 UTC m=+317.830617414 expires:2023-06-21 11:59:59 +0000 UTC groups:[system:serviceaccount:default:default]}
10.244.0.35:51366 - b132f7c5-8a65-42d1-86cc-f74fb907d2f3 - system:serviceaccount:default:default [2023/06/21 11:51:36] echo-server.echo-server.svc.cluster.local HEAD - "/" HTTP/1.1 "curl/8.1.2" 403 2785 0.001
```

### Using an invalid audience

When you try to use an invalid audience in the request that isn't configured with 
`OAUTH2_PROXY_CLIENT_ID`, `OAUTH_PROXY_EXTRA_JWT_ISSUERS` or `OAUTH_PROXY_EXTRA_AUDIENCES`, the request fails.


```sh
kubectl exec -it -n default deploy/oidc-test-invalid-aud -- sh -c 'curl -v -I -H "Authorization: Bearer $(cat /var/run/secrets/echo-server/token)" echo-server.echo-server.svc.cluster.local'
```

Request fails with a 403:

```
*   Trying 10.96.93.28:80...
* Connected to echo-server.echo-server.svc.cluster.local (10.96.93.28) port 80 (#0)
> HEAD / HTTP/1.1
> Host: echo-server.echo-server.svc.cluster.local
> User-Agent: curl/8.1.2
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjhlOUt2Ny14bzJ5ZXRvMWViaEJiSW5aZzFfbTRWYld4VGlsMGF2T1p2b2cifQ.eyJhdWQiOlsiaW52YWxpZCJdLCJleHAiOjE2ODczNDkwNzksImlhdCI6MTY4NzM0ODQ3OSwiaXNzIjoiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJrdWJlcm5ldGVzLmlvIjp7Im5hbWVzcGFjZSI6ImRlZmF1bHQiLCJwb2QiOnsibmFtZSI6Im9pZGMtdGVzdC1pbnZhbGlkLWF1ZC03NjdmZmJkYjk3LTI4MmhjIiwidWlkIjoiMGU2MzE0MDktODZkOC00M2JkLTk4ZGItNTY5YWY4OTZjYWE5In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJvaWRjLXRlc3QiLCJ1aWQiOiI1NmU1YjcwYy00YzUxLTQxMWMtYThmNC1hYjYxOWUwODI1MWIifX0sIm5iZiI6MTY4NzM0ODQ3OSwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6b2lkYy10ZXN0In0.uKgFlEcVeMk0vBVekTi8z8YJCKFIcGjyDHwzCj9J9VxFZr-eoYg3eW17AvIenfL-nUN_LNhycQFugAzecNgTDwY4dij2sPTgcgu2c2SnG1QPv9bTkYcFiO2bF8aV-tjuXI7FtNhRviqI5nR3sxdcghNE0JywKtKH17SBLwiKzZHKuGBGgdK7enPco7UbYRb3S4Ak3SjDu1Ed6GByuY4gfW5jiZGELzHVmf9jhDWU-vzAtURSiur07OJUWc1fsC5ZKIUKEMdCmVqOm8Jrys4wnFz1HN5dzPTJkVmuqopMqvoXq6qJLaZFHvtHfmCD8MH4az9k_ChABsdOmWJoTCI8rQ
>
< HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
< Cache-Control: no-cache, no-store, must-revalidate, max-age=0
Cache-Control: no-cache, no-store, must-revalidate, max-age=0
< Expires: Thu, 01 Jan 1970 00:00:00 UTC
Expires: Thu, 01 Jan 1970 00:00:00 UTC
< X-Accel-Expires: 0
X-Accel-Expires: 0
< Date: Wed, 21 Jun 2023 11:55:53 GMT
Date: Wed, 21 Jun 2023 11:55:53 GMT
< Content-Type: text/html; charset=utf-8
Content-Type: text/html; charset=utf-8

<
* Connection #0 to host echo-server.echo-server.svc.cluster.local left intact
```

oauth2-proxy logs show the following:

```
[2023/06/21 11:55:53] [oauthproxy.go:959] No valid authentication in request. Initiating login.
[2023/06/21 11:55:53] [jwt_session.go:51] Error retrieving session from token in Authorization header: [unable to verify bearer token, audience from claim aud with value [invalid] does not match with any of allowed audiences map[echo-server.echo-server.svc.cluster.local:{}]]
10.244.0.37:60058 - 1da260dc-68b9-4400-98e0-b985d2829167 - - [2023/06/21 11:55:53] echo-server.echo-server.svc.cluster.local HEAD - "/" HTTP/1.1 "curl/8.1.2" 403 8498 0.000
```

## Relevant reading material
- https://medium.com/in-the-weeds/service-to-service-authentication-on-kubernetes-94dcb8216cdc
- https://gefyra.dev/usecases/oauth2-demo/
- https://github.com/gefyrahq/gefyra-demos/blob/main/oauth2-demo/oauth2-demo.yaml
- https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#openid-connect-provider
- https://www.ianunruh.com/posts/oauth2-proxy-with-k8s-service-accounts/

