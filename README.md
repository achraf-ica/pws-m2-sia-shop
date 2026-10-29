# Shop — Web Services M2 SIA

## Lab 1
## Içame Achraf

**Lab 1:** The Inventory Service, Behind Two Doors

## Run it

```bash
python -m uvicorn services.inventory.main:app --reload --port 8000
```

## Questions

1. In section 3, what status line did `DELETE /products/42` return, and what did the `Allow` header say? Why that answer rather than a 404?
2. In section 4.5, the SOAP call failed. What did the HTTP status line say? Name one piece of infrastructure that is misled by it, and what it does wrong as a result.
3. Which of the two doors could a colleague use tomorrow without asking you anything? List precisely what they would need in each case.

## Answers

### 1. HTTP 405 — Method Not Allowed

The request returned **405 Method Not Allowed** because the `DELETE` method is not supported for this resource. The `Allow` header specifies `GET`, indicating that this is the allowed HTTP method.

We got a 405 instead of a 404 because the resource's endpoint exists, but the requested HTTP method is not allowed. A 404 would indicate that the requested resource was not found.

**HTTP response:**

```http
HTTP/1.1 405 Method Not Allowed
date: Thu, 29 Oct 2026 18:46:34 GMT
server: uvicorn
allow: GET
content-length: 31
content-type: application/json
Connection: close
```

**Response body (JSON):**

```json
{
  "detail": "Method Not Allowed"
}
```

### 2. SOAP Fault with HTTP 200 OK

The HTTP status line says `200 OK`, but the response body contains a SOAP Fault indicating that product 99 does not exist.

The infrastructure components that could be misled by this status line include:

- **Proxy:** May treat the response as successful and forward it without recognizing the application-level error.
- **Load balancer:** May consider the request successful when evaluating service health or request outcomes.
- **Monitoring dashboard:** May count the request as a success, causing error rates to appear lower than they actually are.

The problem is that the HTTP status indicates success, while the SOAP body reports an application-level failure.

**HTTP response headers:**

```http
HTTP/1.1 200 OK
date: Thu, 29 Oct 2026 18:51:29 GMT
server: uvicorn
content-length: 306
content-type: application/soap+xml
```

**Response body (SOAP XML):**

```xml
<?xml version="1.0"?>
<env:Envelope xmlns:env="http://www.w3.org/2003/05/soap-envelope">
  <env:Body>
    <env:Fault>
      <env:Code>
        <env:Value>env:Sender</env:Value>
      </env:Code>
      <env:Reason>
        <env:Text xml:lang="en">No product 99</env:Text>
      </env:Reason>
    </env:Fault>
  </env:Body>
</env:Envelope>
```

### 3. Using the REST API vs. the SOAP API

The **REST API** would generally be easier for a colleague to use without asking for additional guidance, provided that its documentation and endpoint conventions are available.

For the REST API, they would need:

- **Base URL:** The address of the service.
- **Endpoint:** The resource path, e.g. `/products/99`.
- **HTTP method:** The supported method, such as `GET`, `POST`, `PUT`, or `DELETE`.
- **Parameters and request body:** The expected inputs and their formats, if applicable.
- **Authentication details:** An API key or token, if required.
- **Expected responses:** HTTP status codes and response formats, typically JSON.

For the SOAP API, they would need:

- **Base URL and operation endpoint:** The service address and the URL to which SOAP requests must be sent.
- **WSDL:** The service description defining available operations, messages, and data types, if provided.
- **SOAP version:** SOAP 1.1 or SOAP 1.2.
- **XML request structure:** The required SOAP envelope, headers, and body.
- **Namespaces:** The correct XML namespaces used by the service.
- **Required HTTP headers:** For example, `Content-Type` and, depending on the SOAP version and service, `SOAPAction`.
- **Authentication details:** Credentials or tokens, if required.
- **Expected responses and faults:** The response structure and how SOAP faults are represented.

In conclusion, REST is often simpler to consume because its resources and HTTP methods are relatively straightforward. SOAP requires more precise knowledge of the XML message structure, namespaces, operations, and fault handling.



## Lab 1
