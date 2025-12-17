#### Request
```http
POST /api/v1/payment HTTP/1.1
Host: host.com
Content-Type: application/json
Authorization: Bearer <token>

{
  "paymentId": "UUID",
  "wallet": "UUID",
  "amount": 999999999.99,
  "kind": "string",
  "description": "string",
  "timestamp": "yyyyMMdd-HH:mm:ss:SSSSSS"
}
```
#### Response
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
"paymentId": "UUID",
"timestamp": "yyyyMMdd-HH:mm:ss:SSSSSS"
}
```
paymentId - является Idempotency Key