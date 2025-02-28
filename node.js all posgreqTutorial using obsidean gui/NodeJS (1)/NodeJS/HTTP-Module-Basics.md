# HTTP Module Basics in Node.js

A comprehensive guide to Node.js's built-in HTTP module for creating web servers and making HTTP requests.

## Creating a Basic Server

### Simple HTTP Server
```javascript
const http = require('http');

// Create server
const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hello World\n');
});

// Listen on port 3000
server.listen(3000, () => {
    console.log('Server running at http://localhost:3000/');
});
```

### Server with Different Routes
```javascript
const server = http.createServer((req, res) => {
    // Get URL and method
    const { url, method } = req;
    
    // Handle different routes
    if (url === '/') {
        res.writeHead(200, { 'Content-Type': 'text/plain' });
        res.end('Home Page\n');
    }
    else if (url === '/api/users' && method === 'GET') {
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ users: ['John', 'Jane'] }));
    }
    else {
        res.writeHead(404, { 'Content-Type': 'text/plain' });
        res.end('Not Found\n');
    }
});
```

## Handling Requests

### Processing Request Headers
```javascript
const server = http.createServer((req, res) => {
    // Get all headers
    console.log(req.headers);
    
    // Get specific headers
    const userAgent = req.headers['user-agent'];
    const contentType = req.headers['content-type'];
    
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end(`User Agent: ${userAgent}\n`);
});
```

### Handling POST Data
```javascript
const server = http.createServer((req, res) => {
    if (req.method === 'POST') {
        let body = '';
        
        // Collect data chunks
        req.on('data', chunk => {
            body += chunk.toString();
        });
        
        // Process the complete request body
        req.on('end', () => {
            try {
                const data = JSON.parse(body);
                res.writeHead(200, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({ received: data }));
            } catch (error) {
                res.writeHead(400, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({ error: 'Invalid JSON' }));
            }
        });
    }
});
```

## Making HTTP Requests

### GET Request
```javascript
http.get('http://api.example.com/data', (res) => {
    let data = '';
    
    // Receive data in chunks
    res.on('data', (chunk) => {
        data += chunk;
    });
    
    // Process complete response
    res.on('end', () => {
        try {
            const parsedData = JSON.parse(data);
            console.log(parsedData);
        } catch (error) {
            console.error('Error parsing JSON:', error);
        }
    });
}).on('error', (error) => {
    console.error('Error:', error);
});
```

### POST Request
```javascript
const data = JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com'
});

const options = {
    hostname: 'api.example.com',
    port: 443,
    path: '/users',
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Content-Length': data.length
    }
};

const req = http.request(options, (res) => {
    let responseData = '';
    
    res.on('data', (chunk) => {
        responseData += chunk;
    });
    
    res.on('end', () => {
        console.log('Response:', responseData);
    });
});

req.on('error', (error) => {
    console.error('Error:', error);
});

// Write data to request body
req.write(data);
req.end();
```

## Error Handling

### Server Error Handling
```javascript
const server = http.createServer((req, res) => {
    try {
        // Server logic here
        throw new Error('Something went wrong');
    } catch (error) {
        console.error('Server Error:', error);
        res.writeHead(500, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Internal Server Error' }));
    }
});

// Handle server-level errors
server.on('error', (error) => {
    console.error('Server Error:', error);
});

// Handle client connection errors
server.on('clientError', (error, socket) => {
    socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});
```

### Request Error Handling
```javascript
const req = http.request(options, (res) => {
    // Handle response
});

req.on('error', (error) => {
    console.error('Request failed:', error);
});

req.on('timeout', () => {
    console.error('Request timed out');
    req.abort();
});

req.setTimeout(5000); // 5 second timeout
```

## Working with HTTPS

### HTTPS Server
```javascript
const https = require('https');
const fs = require('fs');

const options = {
    key: fs.readFileSync('private-key.pem'),
    cert: fs.readFileSync('certificate.pem')
};

const server = https.createServer(options, (req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Secure Hello World\n');
});

server.listen(443, () => {
    console.log('HTTPS server running on port 443');
});
```

## URL Parsing

### Using URL Module
```javascript
const { URL } = require('url');

const server = http.createServer((req, res) => {
    const url = new URL(req.url, `http://${req.headers.host}`);
    
    // Get pathname
    console.log('Path:', url.pathname);
    
    // Get query parameters
    console.log('Query params:', url.searchParams);
    
    // Get specific query parameter
    const name = url.searchParams.get('name');
    
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end(`Hello ${name || 'World'}\n`);
});
```

## Related Topics
- [[HTTPS-Security]] - Secure HTTP communication
- [[Express-Framework]] - Express.js web framework
- [[HTTP-Headers]] - Detailed HTTP headers guide
- [[REST-API]] - Building REST APIs

## Practice Projects
1. Build a basic HTTP server
2. Create a simple REST API
3. Implement a proxy server
4. Build a file server

## Resources
- [Node.js HTTP Documentation](https://nodejs.org/api/http.html)
- [HTTP/1.1 Specification](https://tools.ietf.org/html/rfc2616)
- [[Learning-Resources#HTTP|HTTP Learning Resources]]

## Tags
#nodejs #http #server #networking #api
