---
title: Configure Ingress Ssl Passthrough
menu:
  product_voyager_10.0.0:
    identifier: ssl-passthrough-configuration
    name: Ssl Passthrough
    parent: config-ingress
    weight: 10
product_name: voyager
menu_name: product_voyager_10.0.0
section_menu_id: guides
---
> New to Voyager? Please start [here](/docs/concepts/overview.md).

# SSL Passthrough

The annotation `ingress.appscode.com/ssl-passthrough` allows to configure TLS termination in the backend and not in haproxy. When set to `true`, passes TLS connections directly to backend.

If `ssl-passthrough` is used, HAProxy will use `tcp`. For more details see  [here](https://www.haproxy.com/documentation/haproxy/deployment-guides/tls-infrastructure/). When `ssl-pasthrough` is enabled, Voyager automatically converts your HTTP ingress rules to TCP rules.

Please note that following features are not supported when using `ssl-pasthrough`:

- Multiple paths for HTTP rules.
- `headerRules` and `rewriteRules` for backends.
- Specifying TLS for TCP rules. So even if you define `spec.tls` for your HTTP hosts, it will be ignored.

Voyager will not modify your existing TCP rules. Instead it will cause a validation error if TLS defined for existing TCP rules on same port. In that case, you have to either ensure TCP hosts do not match with `spec.tls` or, just set `noTLS=true` for those TCP rules.

## Ingress Example

```yaml
apiVersion: voyager.appscode.com/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: default
  annotations:
    ingress.appscode.com/ssl-passthrough: "true"
spec:
  rules:
  - host: voyager.appscode.test
    http:
      port: 8443
      paths:
      - path: /foo
        backend:
          serviceName: test-server
          servicePort: 443
```

Generated haproxy.cfg:

```ini
# HAProxy configuration generated by https://github.com/appscode/voyager
# DO NOT EDIT!
global
	daemon
	stats socket /var/run/haproxy.sock level admin expose-fd listeners
	server-state-file global
	server-state-base /var/state/haproxy/
	# log using a syslog socket
	log /dev/log local0 info
	tune.ssl.default-dh-param 2048
	ssl-default-bind-ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK
	lua-load /etc/auth-request.lua
	hard-stop-after 30s
defaults
	log global
	# https://cbonte.github.io/haproxy-dconv/1.7/configuration.html#4.2-option%20abortonclose
	# https://github.com/appscode/voyager/pull/403
	option dontlognull
	option http-server-close
	# Timeout values
	timeout client 50s
	timeout client-fin 50s
	timeout connect 5s
	timeout server 50s
	timeout tunnel 50s
	# Configure error files
	# default traffic mode is http
	# mode is overwritten in case of tcp services
	mode http
frontend tcp-0_0_0_0-8443
	bind *:8443
	mode tcp
	default_backend test-server.default:443
backend test-server.default:443
	mode tcp
	server pod-test-server-777ccbbc49-g7q6t 172.17.0.4:6443
```

Now check the response:

```console
$ minikube service --url voyager-test-ingress
http://192.168.99.100:31692

$ curl -k https://192.168.99.100:31692
{"type":"http","host":"192.168.99.100:31692","serverPort":":6443","path":"/","method":"GET","headers":{"Accept":["*/*"],"User-Agent":["curl/7.47.0"]}}
```
