## **README.md**

### **Snowflake CRDP Service: Thales CipherTrust Integration**

This project provides a **Spring Boot** application that implements protection of sensitive data in Snowflake and exposes the Thales protection and reveal operations as a web service. Leveraging **Snowpark Container Services** (SPCS), the service runs within Snowflake to protect and reveal sensitive data by integrating with the **Thales CipherTrust REST Data Protection** (CRDP) service.

This service supports bulk operations for three data types: `character`, `number-character`, and `number-number`, aligning with the protection policies of the Thales CRDP service.

### **Features**

* **Bulk Data Processing**: Efficiently handles large volumes of data for both protection and revelation.
* **Dynamic Policy Selection**: Automatically selects the appropriate protection policy based on the data type (`char`, `nbr-char`, `nbr-nbr`) and the request's mode (`internal` or `external`).
* **Configurable Environment**: Customizable via environment variables and application properties for flexible deployment.
* **TLS-ready CRDP Client**: Supports HTTP, HTTPS, optional server verification, and optional PKCS12 client-certificate authentication.
* **Robust Error Handling**: Gracefully handles missing or invalid data, returning a configurable "bad data" tag.
* **User Context Awareness**: For reveal operations, the service extracts the Snowflake user from the request header (`sf-context-current_user`) to enforce access control based on Thales policies.

***

### **Getting Started**

#### **Prerequisites**

* **Java 17+**
* **Maven**
* **Docker** (for containerized deployment)
* **Access to a Thales CipherTrust Manager** with CRDP enabled and appropriate policies configured.
* **Snowflake Account** with permissions to create external functions or Snowpark Container Services.


The service exposes several **POST** endpoints to handle data protection and revelation. Each endpoint corresponds to a specific data type.

#### **Protection Endpoints**

* `POST /protectbulkchar`
* `POST /protectbulknbrchar`
* `POST /protectbulknbrnbr`

#### **Revelation Endpoints**

* `POST /revealbulkchar`
* `POST /revealbulknbrchar`
* `POST /revealbulknbrnbr`

***

### **Snowflake Integration**

Please review the documentation folder for more content on how to deploy the service.

For the current Snowpark Container Services TLS deployment pattern, see
[SNOWFLAKE_SPCS_TLS_DEPLOYMENT.md](E:/codex/work/snowflake/thales-snow-scs-crdp-service/SNOWFLAKE_SPCS_TLS_DEPLOYMENT.md).

### **CRDP TLS Configuration**

The outbound client to CRDP now supports both plain HTTP and TLS.

Recommended environment variables:

```properties
CRDP_HOST=thales-backend-service.yourhashcode.svc.spcs.internal
CRDP_PORT=8091
CRDP_SSL_ENABLED=true
CRDP_SSL_VERIFY_SERVER=true
CRDP_CA_CERT_PATH=/snowflake/secrets/crdp-ca/secret_string
CRDP_CLIENT_PKCS12_B64_FILE=/snowflake/secrets/crdp-p12/secret_string
CRDP_CLIENT_PKCS12_PASSWORD_FILE=/snowflake/secrets/crdp-p12-password/secret_string
```

Compatibility notes:

* `CRDPIP` and `CRDPIPPORT` are still supported for older deployments.
* If `CRDP_HOST` or `CRDPIP` already includes `http://` or `https://`, that scheme wins.
* If the host does not include a scheme, the application uses `https://` when `CRDP_SSL_ENABLED=true` and `http://` otherwise.
* `CRDP_SSL_VERIFY_SERVER=false` disables certificate and hostname verification and should only be used for non-production testing.

For Snowpark Container Services, prefer Snowflake `containers.secrets` mounted as files instead of putting PEM, PKCS12, or passwords directly in the Docker image or service `env:` block.
