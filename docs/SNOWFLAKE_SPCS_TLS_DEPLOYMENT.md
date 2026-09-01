# Snowflake SPCS TLS Deployment Guide

This guide captures the recommended TLS deployment model for this Spring Boot
service when it calls Thales CRDP from Snowpark Container Services (SPCS).

Use this guide for:

- Snowflake secret setup
- CRDP backend endpoint validation
- SPCS service specification for this caller service
- optional CRDP backend service examples for `no-tls`, `tls-cert-opt`, and
  `tls-cert`
- service function wiring for Snowflake

## Recommended model

For this Java service, prefer:

- non-secret settings in the service spec `env:` block
- secret material in Snowflake secrets mounted as files
- `CRDP_CLIENT_PKCS12_B64_FILE` for the PKCS12 payload
- `CRDP_CLIENT_PKCS12_PASSWORD_FILE` for the PKCS12 password
- `CRDP_CA_CERT_PATH` for the CA PEM file

This repo now supports:

- plain HTTP
- HTTPS with server verification
- HTTPS with optional PKCS12 client certificate authentication

## How the app decides whether TLS is on

Primary flags:

```properties
CRDP_SSL_ENABLED=true|false
CRDP_SSL_VERIFY_SERVER=true|false
```

Behavior:

- `CRDP_SSL_ENABLED=false` means the client uses `http://`
- `CRDP_SSL_ENABLED=true` means the client uses `https://`
- `CRDP_SSL_VERIFY_SERVER=true` validates the server certificate chain and
  hostname
- `CRDP_SSL_VERIFY_SERVER=false` disables certificate and hostname verification
  and should only be used for non-production testing

Host resolution:

- if `CRDP_HOST` already includes `http://` or `https://`, that scheme wins
- otherwise the app uses `CRDP_SSL_ENABLED` to choose the scheme

## Validate the CRDP backend endpoint

The Snowflake internal DNS name is only the hostname. It does not itself prove
that the service is using HTTP or HTTPS.

Use these commands:

```sql
SELECT SYSTEM$GET_SERVICE_DNS_DOMAIN('SF_TUTS.PUBLIC');
DESC SERVICE thales_backend_service;
SHOW ENDPOINTS IN SERVICE thales_backend_service;
```

What to confirm:

- `dns_name` looks like `thales-backend-service.yourhashcode.svc.spcs.internal`
- `port` matches the CRDP listener
- `protocol` is `HTTPS` when CRDP is deployed in TLS mode

Typical mapping:

- `no-tls` -> port `8090`, protocol `HTTP`
- `tls-cert-opt` -> port `8091`, protocol `HTTPS`
- `tls-cert` -> port `8091`, protocol `HTTPS`

## Prepare the PKCS12 as base64

Snowflake generic string secrets are a good fit for the PKCS12 when stored as
base64 text.

PowerShell:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("E:\codex\work\snowflake\thales-snow-scs-crdp-service\certs\crdp-client.p12")) | Set-Content "E:\codex\work\snowflake\thales-snow-scs-crdp-service\certs\crdp-client.p12.b64"
```

You will also need:

- `crdp-ca.pem`
- the PKCS12 password

## Create Snowflake secrets

Recommended secrets:

- `CRDP_CA_PEM_SECRET`
- `CRDP_CLIENT_P12_B64_SECRET`
- `CRDP_CLIENT_P12_PASSWORD_SECRET`

Example:

```sql
CREATE OR REPLACE SECRET SF_TUTS.PUBLIC.CRDP_CA_PEM_SECRET
  TYPE = GENERIC_STRING
  SECRET_STRING = $$
-----BEGIN CERTIFICATE-----
...contents of crdp-ca.pem...
-----END CERTIFICATE-----
$$;

CREATE OR REPLACE SECRET SF_TUTS.PUBLIC.CRDP_CLIENT_P12_B64_SECRET
  TYPE = GENERIC_STRING
  SECRET_STRING = $$
...single-line base64 of crdp-client.p12...
$$;

CREATE OR REPLACE SECRET SF_TUTS.PUBLIC.CRDP_CLIENT_P12_PASSWORD_SECRET
  TYPE = GENERIC_STRING
  SECRET_STRING = 'your-p12-password-here';
```

## Caller service specification

This is the recommended SPCS service definition for the Spring Boot caller.

Assumptions:

- CRDP backend service is `thales_backend_service`
- backend DNS name is `thales-backend-service.yourhashcode.svc.spcs.internal`
- CRDP TLS endpoint listens on `8091`
- client certificate auth is required or available

```sql
CREATE SERVICE thales_udf_service
IN COMPUTE POOL my_compute_pool
FROM SPECIFICATION $$
spec:
  containers:
    - name: udf
      image: /sf_tuts/public/my_repo:latest
      env:
        PORT: "8083"
        INPUT_FORMAT: "spcs"

        CRDP_HOST: "thales-backend-service.yourhashcode.svc.spcs.internal"
        CRDP_PORT: "8091"
        CRDP_SSL_ENABLED: "true"
        CRDP_SSL_VERIFY_SERVER: "true"

        CRDP_CA_CERT_PATH: "/snowflake/secrets/crdp-ca/secret_string"
        CRDP_CLIENT_PKCS12_B64_FILE: "/snowflake/secrets/crdp-p12/secret_string"
        CRDP_CLIENT_PKCS12_PASSWORD_FILE: "/snowflake/secrets/crdp-p12-password/secret_string"

        BATCHSIZE: "1000"
        DEFAULTMETADATA: "1001000"
        DEFAULTMODE: "internal"
        DEFAULTCHARPOLICY: "char-internal"
        DEFAULTNBRCHARPOLICY: "nbr-char-internal"
        DEFAULTNBRNBRPOLICY: "nbr-nbr-internal"
        DEFAULTINTERNALCHARPOLICY: "char-internal"
        DEFAULTINTERNALNBRCHARPOLICY: "nbr-char-internal"
        DEFAULTINTERNALNBRNBRPOLICY: "nbr-nbr-internal"
        DEFAULTEXTERNALCHARPOLICY: "char-external"
        DEFAULTEXTERNALNBRCHARPOLICY: "nbr-char-external"
        DEFAULTEXTERNALNBRNBRPOLICY: "nbr-nbr-external"
        DEFAULTREVEALUSER: "admin"
      secrets:
        - snowflakeSecret: SF_TUTS.PUBLIC.CRDP_CA_PEM_SECRET
          directoryPath: "/snowflake/secrets/crdp-ca"
        - snowflakeSecret: SF_TUTS.PUBLIC.CRDP_CLIENT_P12_B64_SECRET
          directoryPath: "/snowflake/secrets/crdp-p12"
        - snowflakeSecret: SF_TUTS.PUBLIC.CRDP_CLIENT_P12_PASSWORD_SECRET
          directoryPath: "/snowflake/secrets/crdp-p12-password"
  endpoints:
    - name: udfendpoint
      port: 8083
      public: false
$$
MIN_INSTANCES = 1
MAX_INSTANCES = 1;
```

Notes:

- if CRDP is another internal SPCS service, you generally do not need
  `EXTERNAL_ACCESS_INTEGRATIONS`
- if CRDP is external to Snowflake, then you will need an external access
  integration and matching network rules

## Caller service specification for plain HTTP

If the CRDP backend is still `no-tls`, use this simpler shape:

```sql
CREATE SERVICE thales_udf_service
IN COMPUTE POOL my_compute_pool
FROM SPECIFICATION $$
spec:
  containers:
    - name: udf
      image: /sf_tuts/public/my_repo:latest
      env:
        PORT: "8083"
        INPUT_FORMAT: "spcs"
        CRDP_HOST: "thales-backend-service.yourhashcode.svc.spcs.internal"
        CRDP_PORT: "8090"
        CRDP_SSL_ENABLED: "false"
        CRDP_SSL_VERIFY_SERVER: "false"
        BATCHSIZE: "1000"
        DEFAULTMODE: "internal"
  endpoints:
    - name: udfendpoint
      port: 8083
      public: false
$$
MIN_INSTANCES = 1
MAX_INSTANCES = 1;
```

## Optional CRDP backend service examples

These examples show how the backend service could be represented at a high
level in SPCS. The exact container image and startup command depend on your
CRDP image packaging.

### Backend `no-tls`

```sql
CREATE SERVICE thales_backend_service
IN COMPUTE POOL my_compute_pool
FROM SPECIFICATION $$
spec:
  containers:
    - name: crdp
      image: /sf_tuts/public/thales_crdp_repo:latest
      env:
        KEY_MANAGER_HOST: "your-cm-host"
        REGISTRATION_TOKEN: "replace-me"
        SERVER_MODE: "no-tls"
  endpoints:
    - name: crdp-http
      port: 8090
      public: false
$$
MIN_INSTANCES = 1
MAX_INSTANCES = 1;
```

### Backend `tls-cert-opt`

For a production Snowflake deployment, prefer secrets mounted as files or a
startup wrapper script rather than embedding raw PEM blocks directly in `env:`.

```sql
CREATE SERVICE thales_backend_service
IN COMPUTE POOL my_compute_pool
FROM SPECIFICATION $$
spec:
  containers:
    - name: crdp
      image: /sf_tuts/public/thales_crdp_repo:latest
      env:
        KEY_MANAGER_HOST: "your-cm-host"
        REGISTRATION_TOKEN: "replace-me"
        SERVER_MODE: "tls-cert-opt"
        CERT_VALUE: "..."
        KEY_VALUE: "..."
  endpoints:
    - name: crdp-https
      port: 8091
      protocol: HTTPS
      public: false
$$
MIN_INSTANCES = 1
MAX_INSTANCES = 1;
```

### Backend `tls-cert`

```sql
CREATE SERVICE thales_backend_service
IN COMPUTE POOL my_compute_pool
FROM SPECIFICATION $$
spec:
  containers:
    - name: crdp
      image: /sf_tuts/public/thales_crdp_repo:latest
      env:
        KEY_MANAGER_HOST: "your-cm-host"
        REGISTRATION_TOKEN: "replace-me"
        SERVER_MODE: "tls-cert"
        CERT_VALUE: "..."
        KEY_VALUE: "..."
        TRUSTED_CA: "..."
  endpoints:
    - name: crdp-https
      port: 8091
      protocol: HTTPS
      public: false
$$
MIN_INSTANCES = 1
MAX_INSTANCES = 1;
```

If your CRDP image can only consume PEMs through environment variables, one
reasonable pattern is:

- store the PEM values in Snowflake generic string secrets
- inject them as environment variables or mounted files
- use an entrypoint wrapper to export or materialize the exact variables/files
  the CRDP image expects before starting the CRDP process

## Service functions for the Spring Boot caller

These examples assume:

- service name is `thales_udf_service`
- endpoint name is `udfendpoint`
- the service listens on `8083`
- the Spring Boot routes are the current REST endpoints in this repo

### Protect functions

```sql
CREATE OR REPLACE FUNCTION PROTECT_BULK_CHAR(V VARCHAR)
RETURNS VARCHAR
SERVICE = thales_udf_service
ENDPOINT = udfendpoint
AS '/protectbulkchar';

CREATE OR REPLACE FUNCTION PROTECT_BULK_NBRCHAR(V VARCHAR)
RETURNS VARCHAR
SERVICE = thales_udf_service
ENDPOINT = udfendpoint
AS '/protectbulknbrchar';

CREATE OR REPLACE FUNCTION PROTECT_BULK_NBRNBR(V VARCHAR)
RETURNS VARCHAR
SERVICE = thales_udf_service
ENDPOINT = udfendpoint
AS '/protectbulknbrnbr';
```

### Reveal functions

```sql
CREATE OR REPLACE FUNCTION REVEAL_BULK_CHAR(V VARCHAR)
RETURNS VARCHAR
SERVICE = thales_udf_service
ENDPOINT = udfendpoint
AS '/revealbulkchar';

CREATE OR REPLACE FUNCTION REVEAL_BULK_NBRCHAR(V VARCHAR)
RETURNS VARCHAR
SERVICE = thales_udf_service
ENDPOINT = udfendpoint
AS '/revealbulknbrchar';

CREATE OR REPLACE FUNCTION REVEAL_BULK_NBRNBR(V VARCHAR)
RETURNS VARCHAR
SERVICE = thales_udf_service
ENDPOINT = udfendpoint
AS '/revealbulknbrnbr';
```

Adjust names and signatures to match your existing Snowflake SQL layer.

## Validation checklist

After deployment:

```sql
SHOW SERVICES;
SHOW ENDPOINTS IN SERVICE thales_backend_service;
SHOW ENDPOINTS IN SERVICE thales_udf_service;
DESC SERVICE thales_backend_service;
DESC SERVICE thales_udf_service;
```

What to look for:

- backend endpoint protocol is `HTTPS` when TLS is enabled
- caller service is running
- caller env values point to the right backend host and port
- caller secret file paths are mounted as expected

For first smoke testing only, you can temporarily use:

```properties
CRDP_SSL_VERIFY_SERVER=false
```

That is useful when:

- the backend certificate SAN does not yet match the Snowflake internal DNS name
- you are still validating trust material

For production, keep:

```properties
CRDP_SSL_VERIFY_SERVER=true
```

## Rotation and restart note

Snowflake can update mounted secret files, but this Java service builds its
shared TLS client at startup. In practice, rotate the secret and then restart or
roll the service so the JVM reloads the TLS material.

