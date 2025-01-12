# Backend Communication Design Patterns Introduction

Backend communication patterns are fundamental ways systems talk to each other. Think of them like different methods of communication in real life - just like how we have phone calls (synchronous), text messages (asynchronous), or broadcast announcements (pub/sub).

# Request-Response Pattern

This is the most basic pattern in backend communication. Let me explain with a real-world analogy first:

Imagine you're at a restaurant:
1. You (the client) ask the waiter (server) for a menu
2. The waiter goes to get it and brings it back to you
3. You wait until you get the menu before doing anything else

In technical terms:
```javascript
// Server (Node.js example)
const express = require('express');
const app = express();

app.get('/menu', (req, res) => {
    // Process the request
    const menu = {
        items: [
            { name: 'Pizza', price: 10 },
            { name: 'Burger', price: 8 }
        ]
    };
    
    // Send response
    res.json(menu);
});

app.listen(3000);
```

```javascript
// Client
async function getMenu() {
    try {
        const response = await fetch('http://restaurant.com/menu');
        const menu = await response.json();
        console.log(menu);
    } catch (error) {
        console.error('Failed to get menu:', error);
    }
}
```

Key characteristics:
- Client initiates the request
- Server processes and responds
- Client waits for the response (in synchronous cases)
- One request leads to one response

# Synchronous vs Asynchronous Workloads

Let's understand this with a real-world example:

Synchronous (like a phone call):
- You call customer service
- You wait on the line while they process your request
- You can't do anything else while waiting
- You get immediate response

```javascript
// Synchronous example
function makePhoneCall() {
    // This blocks until complete
    const result = processCustomerRequest();
    console.log(result);
}
```

Asynchronous (like sending an email):
- You send an email to customer service
- You can do other things while waiting
- They respond when they're done
- You check your email later

```javascript
// Asynchronous example
async function sendEmailRequest() {
    // This doesn't block
    const promise = processCustomerRequest();
    console.log('Doing other things...');
    
    // Later when we need the result
    const result = await promise;
    console.log(result);
}
```

# Push Pattern

Push is when the server initiates communication with the client. Think of it like push notifications on your phone.

Example using WebSocket:
```javascript
// Server
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
    // When something important happens
    setInterval(() => {
        ws.send(JSON.stringify({
            type: 'notification',
            message: 'New message received!'
        }));
    }, 5000);
});
```

```javascript
// Client
const ws = new WebSocket('ws://localhost:8080');

ws.onmessage = (event) => {
    const notification = JSON.parse(event.data);
    console.log('Received:', notification);
};
```

# Polling

Polling is like repeatedly checking your mailbox to see if you have new mail. The client keeps asking the server "Any updates?" at regular intervals.

```javascript
// Client-side polling example
function startPolling() {
    setInterval(async () => {
        try {
            const response = await fetch('/api/updates');
            const updates = await response.json();
            
            if (updates.hasNew) {
                console.log('New updates:', updates.data);
            }
        } catch (error) {
            console.error('Polling failed:', error);
        }
    }, 5000); // Check every 5 seconds
}
```

# Long Polling

Long polling is an improved version of regular polling. Instead of checking repeatedly, the client makes a request and the server holds the connection open until there's new data or a timeout occurs.

```javascript
// Server (Node.js)
app.get('/updates', (req, res) => {
    const timeout = 30000; // 30 seconds
    
    const checkForUpdates = () => {
        if (hasNewData()) {
            res.json({ data: getNewData() });
        } else if (timeout <= 0) {
            res.status(204).end(); // No content
        } else {
            setTimeout(checkForUpdates, 1000);
            timeout -= 1000;
        }
    };
    
    checkForUpdates();
});
```

```javascript
// Client
async function longPoll() {
    try {
        const response = await fetch('/updates');
        const data = await response.json();
        
        // Process data
        console.log(data);
        
        // Immediately start next long poll
        longPoll();
    } catch (error) {
        // Wait before retrying on error
        setTimeout(longPoll, 5000);
    }
}
```

# Server-Sent Events (SSE)

SSE is a one-way channel from server to client, perfect for real-time updates like stock prices or news feeds.

```javascript
// Server (Node.js)
app.get('/stock-updates', (req, res) => {
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    
    const sendStockUpdate = () => {
        const stockPrice = getLatestStockPrice();
        res.write(`data: ${JSON.stringify(stockPrice)}\n\n`);
    };
    
    // Send update every second
    const intervalId = setInterval(sendStockUpdate, 1000);
    
    // Clean up on client disconnect
    req.on('close', () => clearInterval(intervalId));
});
```

```javascript
// Client
const eventSource = new EventSource('/stock-updates');

eventSource.onmessage = (event) => {
    const stockData = JSON.parse(event.data);
    updateStockDisplay(stockData);
};
```

# Publish-Subscribe (Pub/Sub)

Pub/Sub is like a newspaper subscription system. Publishers send messages to topics, and subscribers receive messages from topics they're interested in.

```javascript
// Using Redis as a pub/sub system
const Redis = require('ioredis');
const publisher = new Redis();
const subscriber = new Redis();

// Subscriber
subscriber.subscribe('news-updates', (err, count) => {
    if (err) console.error('Failed to subscribe:', err);
    console.log(`Subscribed to ${count} channels`);
});

subscriber.on('message', (channel, message) => {
    console.log(`Received ${message} from ${channel}`);
});

// Publisher
function publishNews(news) {
    publisher.publish('news-updates', JSON.stringify(news));
}

// Example usage
publishNews({
    title: 'Breaking News',
    content: 'Important update!'
});
```

# Multiplexing vs Demultiplexing (H2 Proxying vs Connection Pooling)

This concept is like a smart traffic system. Let me break it down:

## Multiplexing
Imagine you have a single highway (connection) that can handle multiple cars (requests) simultaneously in different lanes.

In technical terms:
```javascript
// HTTP/2 multiplexing example using Node.js
const http2 = require('http2');
const server = http2.createSecureServer({
    key: fs.readFileSync('server-key.pem'),
    cert: fs.readFileSync('server-cert.pem')
});

server.on('stream', (stream, headers) => {
    // Multiple requests can be handled over the same connection
    if (headers[':path'] === '/resource1') {
        stream.respond({
            'content-type': 'application/json',
            ':status': 200
        });
        stream.end(JSON.stringify({ data: 'Resource 1' }));
    } else if (headers[':path'] === '/resource2') {
        stream.respond({
            'content-type': 'application/json',
            ':status': 200
        });
        stream.end(JSON.stringify({ data: 'Resource 2' }));
    }
});
```

## Connection Pooling
Think of this like having a team of workers (connections) ready to handle tasks (requests):

```javascript
// Database connection pooling example
const { Pool } = require('pg');

const pool = new Pool({
    user: 'dbuser',
    host: 'database.server.com',
    database: 'mydb',
    password: 'secretpassword',
    port: 5432,
    max: 20, // Maximum number of connections in the pool
    idleTimeoutMillis: 30000 // How long a connection can be idle before being closed
});

async function queryDatabase() {
    const client = await pool.connect();
    try {
        const result = await client.query('SELECT * FROM users');
        return result.rows;
    } finally {
        client.release(); // Return the connection to the pool
    }
}
```

# Stateful vs Stateless

This is like the difference between a waiter who remembers your order history (stateful) and one who needs to ask every time (stateless).

## Stateful Example
```javascript
// Stateful session management
const express = require('express');
const session = require('express-session');
const app = express();

app.use(session({
    secret: 'my-secret-key',
    resave: false,
    saveUninitialized: false,
    cookie: { secure: true }
}));

app.post('/login', (req, res) => {
    // Store user state in session
    req.session.user = {
        id: 123,
        name: 'John',
        preferences: ['dark mode', 'notifications on']
    };
    res.send('Logged in');
});

app.get('/profile', (req, res) => {
    // Access stored state
    if (req.session.user) {
        res.json(req.session.user);
    } else {
        res.status(401).send('Not logged in');
    }
});
```

## Stateless Example
```javascript
// Stateless JWT authentication
const jwt = require('jsonwebtoken');

app.post('/login', (req, res) => {
    // Create token with all necessary information
    const token = jwt.sign({
        userId: 123,
        name: 'John',
        preferences: ['dark mode', 'notifications on']
    }, 'secret-key');
    
    res.json({ token });
});

app.get('/profile', (req, res) => {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) return res.status(401).send('No token provided');
    
    try {
        // Verify and decode token - no server-side state needed
        const userData = jwt.verify(token, 'secret-key');
        res.json(userData);
    } catch (err) {
        res.status(401).send('Invalid token');
    }
});
```

# Sidecar Pattern

The sidecar pattern is like having a motorcycle sidecar that provides additional functionality to the main vehicle. It's a way to add features to an application without changing its code.

Here's a practical example using a logging sidecar:

```javascript
// Main application
const express = require('express');
const app = express();

app.get('/api/data', (req, res) => {
    // Just focus on business logic
    res.json({ data: 'some important data' });
});

app.listen(3000);
```

```javascript
// Logging sidecar
const express = require('express');
const proxy = require('express-http-proxy');
const winston = require('winston');
const sidecar = express();

// Set up logging
const logger = winston.createLogger({
    level: 'info',
    format: winston.format.json(),
    transports: [
        new winston.transports.File({ filename: 'requests.log' })
    ]
});

// Proxy all requests, but log them first
sidecar.use((req, res, next) => {
    logger.info({
        timestamp: new Date(),
        method: req.method,
        path: req.path,
        headers: req.headers
    });
    next();
});

// Forward requests to main application
sidecar.use(proxy('http://localhost:3000'));

sidecar.listen(4000);
```

In this example:
1. The main application focuses solely on business logic
2. The sidecar handles cross-cutting concerns (logging)
3. Requests go through the sidecar first (port 4000)
4. The sidecar forwards requests to the main app (port 3000)

Common sidecar use cases:
- Logging and monitoring
- Security (authentication, encryption)
- Rate limiting
- Caching
- Service mesh communication

The beauty of the sidecar pattern is that you can:
- Add functionality without modifying the main application
- Use different technologies for the sidecar
- Update the sidecar independently
- Reuse the same sidecar across different applications

For example, you could add rate limiting to your application:

```javascript
// Rate limiting sidecar
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100 // limit each IP to 100 requests per windowMs
});

sidecar.use(limiter);
```

These patterns form the foundation of modern backend systems. They can be mixed and matched depending on your specific needs. For example:
- Use stateless services with JWT for scalability
- Add sidecars for cross-cutting concerns
- Implement connection pooling for database efficiency
- Use HTTP/2 multiplexing for better performance
- Choose between polling and WebSocket based on real-time requirements