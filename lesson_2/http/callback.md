#### Request
```http
POST /api/v1/callback HTTP/1.1
Host: host.com
Content-Type: application/json
Authorization: Bearer <token>

{
  "paymentId": "UUID",
  "status": "string",
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
