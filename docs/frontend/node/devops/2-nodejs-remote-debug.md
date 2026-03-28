---
tags:
  - nodejs-debug
  - remote-debug
  - nodejs
  - devops
---

# How to Remote Debug a Node.js Process

Remote debugging Node.js applications is essential for diagnosing issues in staging, production, or any environment where direct access to the application's code editor is not available. This guide covers the most common methods.

## 1. Using Chrome DevTools (V8 Inspector) with SSH Tunneling

This is the most common and robust method for remote debugging, especially when your Node.js application is running on a remote server without a graphical interface.

### How it works
Node.js includes a built-in V8 Inspector. When enabled, it exposes a debugging endpoint (WebSocket URL) that Chrome DevTools can connect to. For remote servers, an SSH tunnel forwards the remote debugging port to your local machine, making it appear as if the Node.js process is running locally.

### Steps

1.  **Start your Node.js application on the remote server with the `--inspect` flag.**
    By default, `--inspect` listens on `127.0.0.1:9229`. For remote debugging, it's crucial that the debugger listens on an address accessible within the server's network, but typically you'll still use `127.0.0.1` and rely on SSH tunneling for security.

    ```bash
    # On the remote server
    node --inspect=127.0.0.1:9229 your-app.js
    # Or, if you need to bind to an external interface (less secure, generally avoid):
    # node --inspect=0.0.0.0:9229 your-app.js
    ```

    You will see output similar to:
    ```output
    Debugger listening on ws://127.0.0.1:9229/a1b2c3d4-e5f6-7890-1234-567890abcdef
    For help, see: https://nodejs.org/en/docs/inspector
    ```

2.  **Create an SSH tunnel from your local machine to the remote server.**
    This command forwards a port on your local machine (e.g., `9229`) to the debugging port on the remote server (`127.0.0.1:9229`).

    ```bash
    # On your local machine's terminal
    ssh -L 9229:127.0.0.1:9229 user@your_remote_host
    ```
    *   `-L 9229:127.0.0.1:9229`: Forwards local port 9229 to remote host's 127.0.0.1:9229.
    *   `user`: Your username on the remote server.
    *   `your_remote_host`: The IP address or hostname of your remote server.

3.  **Open Chrome DevTools on your local machine.**
    *   Open your Chrome browser and navigate to `chrome://inspect`.
    *   Under the "Devices" section, ensure "Discover network targets" is checked.
    *   You should see your Node.js process listed under "Remote Target" with the address `localhost:9229` (or whatever local port you forwarded to). If not, click "Configure..." and add `localhost:9229`.
    *   Click the "inspect" link next to your Node.js process.

    *(Screenshot description: Chrome DevTools `chrome://inspect` page showing a "Remote Target" section with a discovered Node.js process, typically showing `localhost:9229` and an "inspect" link.)*

4.  **Start Debugging in DevTools.**
    Once you click "inspect", a new DevTools window will open. You can now set breakpoints, step through code, inspect variables, and profile memory (as discussed in heapdump documentation) as if the application were running locally.

    *(Screenshot description: Chrome DevTools window with the "Sources" tab open, showing your Node.js application's code, with a breakpoint set on a line, and the call stack/scope panels visible.)*

## 2. Using VS Code for Remote Debugging

VS Code provides excellent integrated debugging capabilities, including direct remote debugging via SSH or by attaching to an inspector port.

### Method A: Remote SSH Extension (Recommended for Development)

This method uses VS Code's Remote - SSH extension to open a remote folder and debug directly, offering a seamless local-like development experience.

1.  **Install the Remote - SSH Extension in VS Code.**
    Search for "Remote - SSH" by Microsoft in the VS Code Extensions view and install it.

2.  **Connect to your Remote Host.**
    *   Click the Remote Explorer icon in the VS Code Activity Bar (usually on the left sidebar, looks like a monitor with a network icon).
    *   In the SSH Targets section, click the `+` icon to add a new SSH Host (e.g., `ssh user@your_remote_host`).
    *   Once configured, click the connect icon (folder with arrow) next to your host.

    *(Screenshot description: VS Code Remote Explorer sidebar showing SSH Targets, with an option to add a new host and a connect button next to an existing host.)*

3.  **Open your Project Folder on the Remote Host.**
    After connecting, VS Code will open a new window connected to the remote. Go to File > Open Folder and select your Node.js project directory on the remote machine.

4.  **Configure and Start Debugging.**
    *   Open your Node.js application file (e.g., `app.js`).
    *   Go to the Run and Debug view (Ctrl+Shift+D).
    *   Click "create a launch.json file" if you don't have one. Select "Node.js".
    *   A typical `launch.json` for a simple Node.js application might look like this:
        ```json
        {
            "version": "0.2.0",
            "configurations": [
                {
                    "type": "node",
                    "request": "launch",
                    "name": "Launch Program",
                    "skipFiles": [
                        "<node_internals>/**"
                    ],
                    "program": "${workspaceFolder}/your-app.js",
                    "runtimeArgs": [
                        "--inspect=0.0.0.0:9229"
                    ],
                    "restart": true,
                    "console": "integratedTerminal"
                }
            ]
        }
        ```
        **Note:** When using Remote-SSH, VS Code handles the tunneling automatically. You usually don't need `--inspect` or manual SSH tunneling. The `runtimeArgs` for `--inspect` might still be useful if you want to explicitly start the debugger, but often simply `program` is enough.

    *   Set breakpoints in your code.
    *   Start the debugger by clicking the green play button.

    *(Screenshot description: VS Code editor with `launch.json` open, showing a Node.js launch configuration, and the Run and Debug sidebar with breakpoints and watch expressions visible.)*

### Method B: Attach to Remote Process (Requires `--inspect` and SSH Tunnel)

This method is useful when you have a Node.js process already running with the inspector enabled and want to attach VS Code to it, similar to the Chrome DevTools approach.

1.  **Start your Node.js application on the remote server with `--inspect`** (same as step 1 in Chrome DevTools section).

    ```bash
    # On the remote server
    node --inspect=127.0.0.1:9229 your-app.js
    ```

2.  **Create an SSH tunnel from your local machine to the remote server** (same as step 2 in Chrome DevTools section).

    ```bash
    # On your local machine's terminal
    ssh -L 9229:127.0.0.1:9229 user@your_remote_host
    ```

3.  **Configure `launch.json` in VS Code to attach to the remote process.**
    On your *local* machine, open your VS Code project (or an empty folder).
    Create or modify your `.vscode/launch.json` to include an "Attach" configuration:

    ```json
    {
        "version": "0.2.0",
        "configurations": [
            {
                "type": "node",
                "request": "attach",
                "name": "Attach to Remote Node",
                "port": 9229, // The local port you forwarded to
                "address": "localhost",
                "restart": true,
                "localRoot": "${workspaceFolder}",
                "remoteRoot": "/path/to/your/remote/project/folder" // Absolute path on the remote server
            }
        ]
    }
    ```
    *   `port`: This is your *local* port that the SSH tunnel forwards from.
    *   `address`: `localhost` because the tunnel makes it appear local.
    *   `localRoot`: The root of your project on your local machine.
    *   `remoteRoot`: The absolute path to the project on the remote server. **This is crucial for mapping local files to remote files for breakpoints.**

4.  **Start Debugging.**
    *   In VS Code, go to the Run and Debug view.
    *   Select the "Attach to Remote Node" configuration from the dropdown.
    *   Click the green play button. VS Code will now connect to the remote Node.js process via your SSH tunnel.

    *(Screenshot description: VS Code Run and Debug sidebar, showing the debugger successfully attached to a remote process, with active breakpoints and variable inspection.)*
