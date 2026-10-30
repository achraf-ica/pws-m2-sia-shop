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



## Lab 2 

## Run it

    docker compose up --build -d

Inventory: http://127.0.0.1:8000/docs — Orders: http://127.0.0.1:8001/docs

## Lab 2 questions

1. In section 3.4 you sent the same PUT twice and, in section 4.3, the same order twice.
   What did each second request change? After a timeout, which one may a client resend
   without asking, and which one may it not?
2. Orders reserves stock with POST /products/{id}/reservations. Why not let Orders read
   the stock with GET, then write the new value with PATCH? Describe two orders arriving
   at the same moment for the last unit.
3. When Inventory is stopped, Orders answers 503. What would a client have received
   without the try/except, and why is that answer wrong?
4. In compose.yaml, Orders reaches Inventory at http://inventory:8000. Why not
   http://127.0.0.1:8000, which worked in section 4?


## Answers

1. Idempotency of PUT vs. POST : 

The second PUT changed nothing: it replaced product 45 with the same full representation, so the server ended in the same state as after the first call. The second order, however, created a new order (id 2), reserved another batch of units and lowered the stock again. POST on /orders is not idempotent: each call has a new effect.

After a timeout, a client may resend the PUT without asking: if the first one went through, the retry is harmless, and if it didn’t, the retry finishes the job. It may not blindly resend the order: if the first request actually succeeded and only the answer was lost, the customer would be charged and the stock reserved twice. It must first check (e.g. GET /orders) or the API must offer an idempotency key.


2. Why a reservation resource instead of GET then PATCH


Reading the stock and then writing a new value are two separate steps, and another request can slip in between them (a race condition, “lost update”). Take the last unit, stock = 1:
Order A: GET → stock is 1, enough.

Order B: GET → stock is 1, enough.

Order A: PATCH stock=0.

Order B: PATCH stock=0.

Both orders are accepted, yet only one unit existed: the shop sold something it doesn’t have (and the stock written is wrong too if quantities differ). With POST .../reservations, Inventory does the check and the subtraction in one SQL statement (UPDATE ... SET stock = stock - ? WHERE id = ? AND stock >= ?). The database serializes the two statements: the first succeeds (stock 0), the second matches no row and gets a 409 Conflict. The decision stays with the service that owns the data.

3. When Inventory is stopped, Orders answers 503. What would a client have received without the try/except, and why is that answer wrong?

Without the try/except, the httpx.RequestError raised when Inventory is stopped would escape place_order and the client would receive a generic 500 Internal Server Error, an outcome that is not in the contract published by responses= (only 201, 409, 422, 502 and 503). That answer is wrong because a 500 says “Orders has a bug” when Orders works fine and its dependency is down, which is exactly what 503 Service Unavailable (“Inventory cannot be reached”) tells the client, along with the hint that retrying later is reasonable.


4. In compose.yaml, Orders reaches Inventory at http://inventory:8000. Why not
   http://127.0.0.1:8000, which worked in section 4?

Inside the Orders container, 127.0.0.1 points back to that same container, where Inventory is not running, so the request would fail. In section 4 both services shared one host and therefore one 127.0.0.1, but under Compose each service has its own network and is reached by its service name (inventory), which Docker’s internal DNS resolves to the right container, on the port it listens on internally (8000).
