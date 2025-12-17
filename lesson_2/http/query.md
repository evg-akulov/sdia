#### Request
```http
GET /api/v1/payments/{id} HTTP/1.1
Host: host.com
Content-Type: application/json
Authorization: Bearer <token>
```
#### Response
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "paymentId": "UUID",
    "user_name": "string",
    "phone": "string",
    "wallet_number": "UUID",
    "amount": 999999999.99,
    "kind": "string",
    "description": "string",
    "status": "string",
    "created": "yyyyMMdd-HH:mm:ss:SSSSSS",
    "updated": "yyyyMMdd-HH:mm:ss:SSSSSS"
}
```
