---
tags:
  - nodejs
  - nodejs-deploy
  - deploy
---
### Deploying Node.js on a Physical Server

Deploying Node.js applications to a physical server involves setting up the environment, preparing the application, and implementing a robust deployment strategy. The goal is to ensure reliability, security, and scalability.

**1. Server Setup:**
*   **Operating System:** Choose a stable Linux distribution (e.g., Ubuntu, CentOS).
*   **Node.js Installation:** Install Node.js and npm/yarn. Using a version manager like `nvm` is recommended for managing Node.js versions.
*   **Process Manager:** Install a process manager like PM2 to ensure your application runs continuously (see details below).
*   **Reverse Proxy:** Set up Nginx or Apache to handle incoming requests, SSL termination, load balancing, and static file serving.
*   **Firewall Configuration:** Restrict access to essential ports (e.g., 80 for HTTP, 443 for HTTPS) and your application's port if needed for internal communication.
*   **SSH Security:** Secure SSH access with key-based authentication, disable root login, and consider changing the default SSH port.

**2. Application Preparation:**
*   **Build Step:** Transpile code (e.g., TypeScript to JavaScript) if necessary.
*   **Install Production Dependencies:** Run `npm install --production` to install only necessary packages.
*   **Configuration Management:** Use environment variables (via `.env` files and libraries like `dotenv`) for sensitive information (database credentials, API keys) and port settings.

**3. Deployment Process:**
*   **Code Transfer:** Use tools like `scp`, `rsync`, or Git to transfer your application code to the server. Cloning the repository and pulling updates is a common practice.
*   **Start Application:** Use PM2 to start, monitor, and manage your Node.js process.
*   **Configure Reverse Proxy:** Point your Nginx/Apache server to proxy requests to your Node.js application's port.

**4. Security Best Practices:**
*   **Keep Software Updated:** Regularly update Node.js, npm, OS, and all dependencies.
*   **Least Privilege:** Run your Node.js process under a dedicated, unprivileged user.
*   **Secure Sensitive Data:** Avoid hardcoding secrets; use environment variables.
*   **SSL/TLS:** Encrypt all traffic using HTTPS, typically managed by the reverse proxy.

**5. Monitoring and Logging:**
*   **Process Monitoring:** Leverage PM2 for basic monitoring and automated restarts.
*   **Centralized Logging:** For advanced needs, use services like the ELK stack or cloud-based logging solutions.
*   **Health Checks:** Implement application health check endpoints.

#### Why Use PM2 for Node.js Deployment?

PM2 (Process Manager 2) is essential for managing Node.js applications on physical servers because it:

*   **Ensures High Availability:** Automatically restarts crashed applications.
*   **Simplifies Process Management:** Allows easy starting, stopping, reloading, and monitoring of applications.
*   **Provides Log Management:** Handles `stdout`/`stderr` redirection, rotation, and real-time log viewing.
*   **Optimizes Performance:** Supports clustering to leverage multi-core CPUs for better throughput.
*   **Enables Zero-Downtime Reloads:** Allows code updates without interrupting service.
*   **Automates Startup:** Generates scripts to start applications automatically on server boot.

### Why Not Use `console.log()` for Production Logging?

While `console.log()` is fine for development, it's unsuitable for production Node.js logging due to:

*   **Performance Bottlenecks:** Can be slow, impacting application responsiveness.
*   **Lack of Structure:** Plain text logs are hard to parse by log management tools.
*   **Limited Features:** Lacks essential features like log levels (debug, info, error), rotation, and remote shipping.
*   **Security Risks:** Potential for accidental logging of sensitive information.
*   **Complex Management:** Output redirection and reliability are harder to manage than with dedicated libraries.

Dedicated logging libraries (like Pino, Winston, Bunyan) offer structured logging, customizable levels, efficient streaming, and better integration with production environments.

### Deploying Node.js with Docker: Why Not PM2 Anymore?

In Dockerized Node.js deployments, PM2 is typically not used because:

*   **Docker Manages Container Lifecycle:** Docker handles container restarts, fulfilling the role of a process manager for the main application process.
*   **Single Process Per Container:** Best practice dictates one primary process per container. Running PM2 inside a container to manage one Node.js process is redundant and adds overhead.
*   **Resource Efficiency:** PM2 consumes resources; in a container, it's more efficient to let Docker manage the process.
*   **Simplified Logging:** Docker has its own logging drivers and aggregation mechanisms that work best with direct application logs.

Instead, Node.js applications are typically run directly via `CMD` or `ENTRYPOINT` in the Dockerfile (e.g., `node server.js`).

#### In Docker, Why Still Use Pino or Other Logging Frameworks?

Even in Docker, structured logging frameworks are vital for:

*   **Structured Logging:** Essential for log aggregation systems (ELK, Splunk) to parse, query, and analyze logs effectively. Frameworks like Pino produce JSON logs by default.
*   **Log Levels:** Control log verbosity (debug, info, error) for different environments and troubleshooting needs.
*   **Performance:** Optimized libraries like Pino offer high throughput and low overhead, crucial in resource-constrained containers.
*   **Standardization & Readability:** Ensures consistent, predictable log formats for easier debugging across distributed systems.
*   **Remote Shipping:** Libraries can integrate directly with specific logging services, complementing Docker's collection capabilities.

These frameworks manage the application's *internal* logging, providing insights that Docker's container-level logging doesn't.
