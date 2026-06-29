# Web Fundamentals

# How The Web Works?

Client? Server? Request and Response ???!!!

Client and Server are not the only components in the web!!

Let's talk about other parts of the "internet toolbox" 😄

---

# DNS

The **Domain Name System (DNS)** is like an address book for websites.

When you type a web address in your browser, the browser looks at the DNS to find the website's IP address (the actual address where the server lives) before it can retrieve the website.

The browser needs to know which server the website lives on so it can send HTTP messages to the right place.

Example:

```text
google.com
      ↓
DNS
      ↓
142.250.xxx.xxx
```

---

# TCP/IP

Maybe you studied them in college, so let me give a brief about them 😅

**TCP** = Transmission Control Protocol

**IP** = Internet Protocol

They are communication protocols that define how data should travel across the internet.

### TCP

Responsible for:

* Making sure data reaches the destination correctly.
* Making sure packets arrive in the correct order.
* Detecting lost packets and re-sending them.

### IP

Responsible for:

* Addressing devices.
* Routing packets between networks.

Together:

```text
TCP/IP
```

allow computers to communicate reliably over the internet.

---

# Files

Every website is made of files.

The files could be:

* HTML files
* CSS files
* JavaScript files
* Images
* Videos
* Fonts
* Backend generated content

---

# So, what don't you see?

When you type a web address:

1. The browser goes to a DNS server to find the real address of the server.
2. The browser sends an HTTP request message to the server.
3. The server processes the request.
4. If everything is OK, the server returns:

```text
200 OK
```

5. The server starts sending the website files to the browser.
6. Data travels in small chunks called **Packets**.
7. The browser receives the files and renders the page.

```text
Browser  -> Request  -> Server
Browser <- Response <- Server
```

---

# HTTP

**HTTP = HyperText Transfer Protocol**

A protocol used for communication between clients and servers.

### Characteristics

* Stateless
* Request / Response based
* Application Layer Protocol

Default Port:

```text
80
```

---

# HTTPS

**HTTP Secure**

HTTP + SSL/TLS Encryption

### Benefits

* Encryption
* Authentication
* Data Integrity

Default Port:

```text
443
```

---

# Request & Response

## Request

Sent from the client to the server.

Contains:

* URL
* Method
* Headers
* Body (optional)

Example:

```http
GET /api/products HTTP/1.1
Host: example.com
```

---

## Response

Sent from the server to the client.

Contains:

* Status Code
* Headers
* Body

Example:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

---

# HTTP Methods

## GET

Retrieve data.

```http
GET /api/products
```

## POST

Create a new resource.

```http
POST /api/products
```

## PUT

Update an entire resource.

```http
PUT /api/products/1
```

## PATCH

Partial update.

```http
PATCH /api/products/1
```

## DELETE

Delete a resource.

```http
DELETE /api/products/1
```

---

# HTTP Response Status Codes

## 1XX

Informational

## 2XX

Successful

Examples:

```text
200 OK
201 Created
204 No Content
```

## 3XX

Redirection

Examples:

```text
301 Moved Permanently
302 Found
```

## 4XX

Client Errors

Examples:

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
```

## 5XX

Server Errors

Examples:

```text
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
```

---

# Kestrel & Reverse Proxy

## Kestrel

A built-in ASP.NET Core web server.

Responsibilities:

* Receive requests
* Process requests
* Return responses

---

## Reverse Proxy

Sits between Kestrel and the outside network.

Receives HTTP requests from the internet and forwards them to Kestrel.

```text
Client
   ↓
Reverse Proxy
   ↓
Kestrel
```

### Common Types

### IIS

Microsoft's web server.

### YARP

Yet Another Reverse Proxy.

Built by Microsoft using .NET.

### Nginx

Popular on Linux servers.

### Apache

Another popular Linux web server.

---

# Request Pipeline

A request doesn't directly reach your controller.

It goes through multiple stages first:

```text
Request
    ↓
Reverse Proxy
    ↓
Kestrel
    ↓
Middlewares
    ↓
Routing
    ↓
Endpoint (Controller Action)
    ↓
Response
```

This is the high-level journey of almost every request inside an ASP.NET Core application.
