---
tags:
  - nodejs-dump
  - heapdump
  - memory-profiling
  - nodejs
  - devops
---

# How to Heapdump a Node.js Process

Heap dumping is crucial for diagnosing memory leaks and optimizing memory usage in Node.js applications. This guide summarizes various methods to generate heap dumps.

## 1. Using Chrome DevTools (V8 Inspector) - Recommended for Development/Debugging

This is the most common and user-friendly method for local development and debugging.

### How it works
Node.js comes with a built-in V8 Inspector that can be accessed via Chrome DevTools. This allows you to connect to a running Node.js process and take heap snapshots.

### Steps

1.  **Start your Node.js application with the `--inspect` flag:**
    ```bash
    node --inspect your-app.js
    # Or for a specific port:
    node --inspect=0.0.0.0:9229 your-app.js
    ```
    You will see output similar to:
    ```output
    Debugger listening on ws://127.0.0.1:9229/a1b2c3d4-e5f6-7890-1234-567890abcdef
    For help, see: https://nodejs.org/en/docs/inspector
    ```

2.  **Open Chrome Browser:**
    *   If running locally, open `chrome://inspect` in your Chrome browser.
    *   You should see your Node.js process listed under "Remote Target". Click "inspect" to open DevTools.
    *   If running on a remote server, you might need to use SSH tunneling:
        ```bash
        ssh -L 9229:127.0.0.1:9229 user@your_remote_host
        ```
        Then, in Chrome, click "Configure..." in `chrome://inspect`, add `localhost:9229`, and then connect.

3.  **Take a Heap Snapshot:**
    *   In Chrome DevTools, go to the "Memory" tab.
    *   Select "Heap snapshot" and click "Take snapshot". The snapshot will be generated and displayed in DevTools for analysis.

## 2. Using `heapdump` npm package (Older Projects / Programmatic Dumps)

The `heapdump` package allows programmatic generation of heap snapshots. While functional, it's often less preferred than the V8 Inspector for active debugging due to its maintenance status and the power of DevTools, especially in newer Node.js versions.

### Installation
```bash
npm install heapdump
```

### Usage Example (JavaScript)

```javascript
// app.js
const http = require('http');
const heapdump = require('heapdump');
const path = require('path');
const fs = require('fs');

// Ensure a directory for dumps exists
const dumpDir = '/tmp/nodejs_heapdumps';
if (!fs.existsSync(dumpDir)) {
    fs.mkdirSync(dumpDir);
}

let leakyArray = [];

http.createServer((req, res) => {
    // Simulate a memory allocation that might lead to a leak
    leakyArray.push(new Array(1024 * 10).join('x'));
    res.write('Hello from Node.js! Memory allocated.
');

    if (req.url === '/heapdump') {
        console.log('Taking heapdump...');
        const filename = path.join(dumpDir, `heapdump-${Date.now()}.heapsnapshot`);
        heapdump.writeSnapshot(filename, function(err){
            if (err) {
                console.error('Heapdump error:', err);
                res.end('Heapdump failed: ' + err.message + '
');
            } else {
                console.log(`Heapdump written to ${filename}`);
                res.end(`Heapdump saved to ${filename}
`);
            }
        });
    } else {
        res.end('Visit /heapdump to generate a heap snapshot.
');
    }
}).listen(8080);

console.log('Server running at http://localhost:8080/');

// To trigger a heapdump, visit http://localhost:8080/heapdump
// You can then analyze the .heapsnapshot file in Chrome DevTools (Memory tab -> Load profile)
```

### Example Output (Console)
```output
Server running at http://localhost:8080/
# After visiting http://localhost:8080/heapdump
Taking heapdump...
Heapdump written to /tmp/nodejs_heapdumps/heapdump-1678886400000.heapsnapshot
```

## 3. Using `llnode` (Post-mortem Debugging / Native Inspection)

`llnode` is a Node.js plugin for LLDB, allowing for post-mortem debugging and inspection of running Node.js processes, including heap inspection, from the command line. It's more advanced and typically used for complex native crashes or deeper analysis.

### Installation
Refer to the official `llnode` GitHub repository for installation instructions, as it often involves compiling against your specific Node.js version and LLDB.

### Usage Example
1.  **Get process ID (PID) of your Node.js app:**
    ```bash
    ps aux | grep node
    # Example Output:
    # user      12345  0.5  1.0 123456 45678 ?        Sl   Mar27   0:15 node your-app.js
    ```
2.  **Attach LLDB with `llnode`:**
    ```bash
    lldb -p <PID>
    # Example:
    # lldb -p 12345
    ```
3.  **Take heap snapshot within LLDB:**
    ```lldb
    (lldb) v8 heap snapshot
    ```

### Example Output (LLDB Console)
```output
(lldb) v8 heap snapshot
Writing heap snapshot to /tmp/heap-12345-0.heapsnapshot
```

## 4. Programmatic V8 Heapdump (Node.js 11.12.0+ via `v8` module) - Recommended for Production Programmatic Dumps

For Node.js versions 11.12.0 and above, you can directly use the built-in `v8` module to take heap snapshots without needing third-party packages like `heapdump`. This is generally preferred for programmatic snapshots in modern Node.js applications, especially in production environments where installing additional native modules (`heapdump`) might be problematic.

### Usage Example (JavaScript)

```javascript
// app.js
const http = require('http');
const v8 = require('v8');
const fs = require('fs');
const path = require('path');

const dumpDir = '/tmp/nodejs_v8_heapdumps';
if (!fs.existsSync(dumpDir)) {
    fs.mkdirSync(dumpDir);
}

let data = [];

http.createServer((req, res) => {
    // Simulate memory usage
    data.push(new Array(1024 * 50).join('x'));
    res.write('Hello from Node.js! Memory allocated.
');

    if (req.url === '/take-snapshot') {
        const filename = path.join(dumpDir, `snapshot-${Date.now()}.heapsnapshot`);
        console.log(`Starting heap snapshot to ${filename}...`);

        const snapshotStream = v8.getHeapSnapshot();
        const fileStream = fs.createWriteStream(filename);

        snapshotStream.pipe(fileStream)
            .on('finish', () => {
                console.log(`Heap snapshot written to ${filename}`);
                res.end(`Heap snapshot saved to ${filename}
`);
            })
            .on('error', (err) => {
                console.error('Error writing heap snapshot:', err);
                res.end(`Failed to save snapshot: ${err.message}
`);
            });
    } else {
        res.end('Visit /take-snapshot to generate a heap snapshot.
');
    }
}).listen(8081);

console.log('Server running at http://localhost:8081/');
// To trigger, visit http://localhost:8081/take-snapshot
```

### Example Output (Console)
```output
Server running at http://localhost:8081/
# After visiting http://localhost:8081/take-snapshot
Starting heap snapshot to /tmp/nodejs_v8_heapdumps/snapshot-1678886400000.heapsnapshot...
Heap snapshot written to /tmp/nodejs_v8_heapdumps/snapshot-1678886400000.heapsnapshot
```

### Open debug model through signal

Below command will open the node debug mode, then you can open chrome://inspect wot debug the process.
```shell
kill -user1  PID
```