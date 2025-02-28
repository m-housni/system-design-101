# HTTP 1.0 -> HTTP 1.1 -> HTTP 2.0 -> HTTP 3.0 (QUIC)

Let's build the same demo application using Node.js.

**1. Server-Side (Node.js)**

```javascript
const http = require('http');
const https = require('https');
const fs = require('fs');
const http2 = require('http2');
const quic = require('node-quic'); // Install with: npm install node-quic

function handleRequest(req, res) {
  const protocol = req.httpVersionMajor + '.' + req.httpVersionMinor;
  const { method, url, headers, socket } = req;
  let body = '';

  req.on('data', chunk => {
    body += chunk;
  });

  req.on('end', () => {
    const responseText = `
    Protocol: HTTP/${protocol}
    Method: ${method}
    Path: ${url}
    Headers: ${JSON.stringify(headers)}
    Body: ${body}
    `;

    res.writeHead(200, { 'Content-Type': 'text/plain', 'Server': 'MyTestServer' });
    res.end(responseText);
  });
}

// HTTP 1.0/1.1
const httpServer = http.createServer(handleRequest);
httpServer.listen(5000, () => {
  console.log('HTTP 1.0/1.1 server listening on port 5000');
});

// HTTP 2.0
const options = {
  key: fs.readFileSync(__dirname + '/key.pem'),
  cert: fs.readFileSync(__dirname + '/cert.pem')
};

const http2Server = http2.createSecureServer(options, handleRequest);

http2Server.listen(5001, () => {
    console.log("HTTP2 Server listening on port 5001");
});

// HTTP 3.0 (node-quic)
async function startQuicServer() {
  const quicServer = quic.createQuicSocket({
    endpoint: { port: 4433 },
    certificate: fs.readFileSync(__dirname + '/cert.pem'),
    privateKey: fs.readFileSync(__dirname + '/key.pem'),
  });

  quicServer.on('session', async (session) => {
    session.on('stream', async (stream) => {
      let body = '';
      stream.on('data', (data) => {
          body += data.toString();
      });
      stream.on('end', async () => {
        const responseText = `
        Protocol: HTTP/3.0
        Method: GET
        Path: /
        Headers: {}
        Body: ${body}
        `;
        stream.end(`HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n${responseText}`);
      });
    });
  });

  await quicServer.listen();
  console.log('HTTP 3.0 server listening on port 4433');
}

startQuicServer();
```

**2. Client-Side (curl or Browser)**

**HTTP 1.0:**

```bash
curl --http1.0 http://localhost:5000/test
```

**HTTP 1.1:**

```bash
curl --http1.1 http://localhost:5000/test
```

**HTTP 2.0:**

* Generate self-signed certificates (if you haven't already):
    ```bash
    openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -subj '/CN=localhost'
    ```
* Test with curl:
    ```bash
    curl --http2 --insecure https://localhost:5001/test
    ```
* Test with a browser: Open `https://localhost:5001/test`.

**HTTP 3.0:**

* Install `node-quic`:
    ```bash
    npm install node-quic
    ```
* Test with curl (make sure your `curl` supports HTTP/3):
    ```bash
    curl --http3 --insecure https://localhost:4433/test
    ```

**To Run the Node.js Server:**

1.  **Save:** Save the code as `server.js`.
2.  **Install:** Install `node-quic` with `npm install node-quic`.
3.  **Generate Certificates:** Generate the self-signed certificates as described above.
4.  **Run:** Execute `node server.js`.

**Key Points:**

* **HTTP 1.0/1.1:** Uses the `http` module.
* **HTTP 2.0:** Uses the `http2` module, which requires HTTPS.
* **HTTP 3.0:** Uses the `node-quic` module, providing QUIC support.
* **Self-Signed Certificates:** Needed for HTTP/2 and HTTP/3.
* **`curl`:** Versatile tool for testing.
* **Browser:** Good for visual confirmation of HTTP/2.

This Node.js version provides a functional equivalent to the Python Flask example, allowing you to test the different HTTP protocols.
