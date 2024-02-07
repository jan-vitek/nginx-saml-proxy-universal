# nginx-saml-proxy

This is a fork of [UWIT-IAM/nginx-saml-proxy](https://github.com/UWIT-IAM/nginx-saml-proxy) 
which brings the ability to be configured to work with 3rd party SAML Identity providers.

This docker image can be used as a standalone proxy for an nginx `auth_request`
authentication. By defaultu you supply it a UW-registered SAML Entity ID and ACS postback
URL, the proxy will take care of the rest.

If you need different SAML IdP than UW, you can send your IdP info in the headers of
SAML request.

Example nginx.conf for UW IdP would look like the following...

```
location / {
    auth_request /saml/status/group/uw_it_all;
    auth_request_set $auth_user $upstream_http_x_saml_user;
    error_page 401 = @login_required;
    proxy_set_header Remote-User $auth_user;
    proxy_pass http://secure:5000/;
}

# user needs 2FA
location /2fa {
    auth_request /saml/status/2fa;
    error_page 401 = @2fa_required;
    alias /usr/share/nginx/html;
}

location /saml/ { 
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://saml:5000/;
}

location @login_required {
    return 302 https://$http_host/saml/login$request_uri;
}

location @2fa_required {
    return 302 https://$http_host/saml/2fa$request_uri;
}
```

Example nginx.conf for 3rd party IdP would look like this:

```
location / {
    auth_request /saml/status;
    auth_request_set $auth_user $upstream_http_x_saml_user;
    error_page 401 = @login_required;
    proxy_set_header Remote-User $auth_user;
}

location @login_required {
  return 302 https://$http_host/saml/login$request_uri;
}

location /saml/ {
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Prefix /saml/;
    proxy_set_header X-Saml-Entity-Id saml_service_id;
    proxy_set_header X-Saml-Acs /saml/login;
    proxy_set_header X-Saml-Idp https://example.com/saml;
    proxy_set_header X-Saml-Idp-Url https://example.com/saml/auth;
    proxy_set_header X-Saml-Idp-Id-Attr EmailAddress;
    proxy_set_header X-Saml-Idp-Cert MIIFPzCCAyegAw ... zkpHmcFcgY;
    proxy_pass http://saml:5000/;
}
```

See the [example nginx config](test/nginx/server.conf) for more examples.

## SECRET_KEY

This app wants an environment variable `SECRET_KEY`, which should be a secure,
randomly-generated string. Otherwise, we generate one on the fly, which only
works as long as the app is running, and won't work in a distributed environment.
SECRET_KEY is used to sign cookies, so setting a new key effectively
invalidates all existing sessions.


## Service Provider (SP) Entity ID and ACS URL

There are two ways to declare your SP entity-id and acs-url. With both of
these, the `X-Forwarded-` headers listed are crucial.

### By inference

With this the saml proxy will make assumptions that the request URL and proxy
path are registered as an SP.

If host https://example.com has the following proxy config...

```
location /saml/ {
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Prefix /saml/;
    proxy_pass http://saml:5000/;
  }
```

Then the SP entity-id to register is `https://example.com/saml` and the ACS
endpoint to register is `https://example.com/saml/login`.

### Explicitly

You can also declare these items explicitly by passing them in as headers...

```
location /saml/ {
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Prefix /saml/;
    proxy_set_header X-Saml-Entity-Id https://samldemo.iamdev.s.uw.edu/saml;
    proxy_set_header X-Saml-Acs /saml/login;
    proxy_pass http://saml:5000/;
}
```

You typically won't need to explicitly declare `X-Saml-Acs`. `X-Saml-Entity-Id`
may need to be declared if for any reason the entity-id doesn't match with how
we infer it, such as with an existing entity-id, or when multiple hosts
share a single Entity ID.
