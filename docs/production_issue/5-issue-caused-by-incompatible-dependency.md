---
tags:
  - production-issue
  - dependency-management
  - incompatible-dependency
  - mu-server
  - netty
---
# When a healthy-looking REST service accepts connections but never responds

## Incident summary

An internal REST service became unavailable after deployment. The process started normally and its port was listening, but requests connected and then waited indefinitely for a response. There were no startup exceptions or obvious application errors.

The cause was an incompatible dependency combination: **Netty had been upgraded independently to address a security vulnerability, while `mu-server` remained on its previous version.** The resulting runtime combination was not compatible.

This incident is a useful reminder that a successful build and a successful process startup do not prove that a framework's transitive runtime dependencies are compatible.

## Impact and symptoms

From a caller's perspective, the service looked alive but could not serve traffic:

- The service port was in the `LISTEN` state.
- Startup completed without an error or exception.
- An HTTP request could establish a connection.
- The request received no response and eventually hung or timed out.

This symptom profile is deceptive. It naturally leads to checks of load balancers, routing, firewall rules, thread pools, and application handlers. Those checks are reasonable, but an incompatible networking stack can fail before a request reaches application code.

## Investigation

We used elimination rather than assuming that the listening port meant the service was healthy.

### 1. Check recent application changes

We first reviewed the recent code changes and commit history to identify a behavioural change in the service itself. That review did not reveal an application-code change that explained the failure.

### 2. Treat startup success as a weak signal

The service had no startup error, but startup exercises only part of a web server's lifecycle. In particular, it may not exercise request decoding, channel-pipeline execution, response writing, or connection lifecycle handling under real traffic.

The key observation was therefore not merely that the port was open, but that the server accepted requests without making progress toward a response.

### 3. Inspect the resolved dependency graph

The breakthrough came from reviewing the dependency update made to remediate a Netty vulnerability. Netty had been pinned or upgraded directly, but the version of `mu-server` had not changed with it.

`mu-server` is built on Netty and relies on Netty's runtime behaviour and APIs. Overriding Netty outside the version range expected by the server framework can produce a partially functional process: it can bind a socket successfully, while request processing fails or stalls later in the lifecycle.

For Maven projects, inspect the *resolved* dependency set rather than relying on declarations in `pom.xml` alone:

```shell
mvn dependency:tree -Dincludes=io.netty
mvn dependency:tree -Dincludes=io.muserver
```

For Gradle projects:

```shell
./gradlew dependencyInsight --dependency netty --configuration runtimeClasspath
./gradlew dependencies --configuration runtimeClasspath
```

Look for direct overrides, dependency-management rules, exclusions, shaded artifacts, and multiple Netty module versions. All Netty modules used at runtime should resolve consistently, and the chosen version must be supported by the framework that embeds it.

## Root cause

The root cause was a **version-skewed server stack**:

```text
mu-server version A
        |
        | expects a compatible Netty release line
        v
Netty upgraded independently to version B
        |
        v
Server starts and binds its port, but request handling is incompatible
```

The dependency upgrade fixed the intended vulnerability concern but introduced an integration problem. Dependency compatibility is a property of the complete runtime graph, not of a single library in isolation.

## Resolution

Resolve the issue by selecting a supported pair of `mu-server` and Netty versions:

1. Check the `mu-server` release documentation and its published dependency metadata for its supported Netty version or release line.
2. Upgrade `mu-server` to a version that supports the required secure Netty version, or use the framework's managed Netty version if it already contains the required security fix.
3. Remove unnecessary direct Netty overrides so that one component owns version alignment.
4. Rebuild and deploy the complete dependency set, then verify a real HTTP request and response—not only process startup or port binding.

Avoid resolving this kind of problem by changing only one Netty module. Netty is released as a set of closely related modules; mixing module versions can create another form of version skew.

## Prevention checklist

When upgrading a transitive infrastructure dependency for a vulnerability fix:

- Identify the framework that owns the dependency and confirm its compatibility guidance before forcing an override.
- Record the before-and-after resolved dependency trees in the change or pull request.
- Use dependency convergence checks to prevent multiple versions of the same library family from reaching runtime.
- Add a smoke test that starts the service and performs an HTTP request through the normal server stack.
- Add a dependency-vulnerability scan, but treat its upgrade recommendation as input to compatibility validation rather than an automatic version bump.

## Takeaway

A listening port proves only that a process bound a socket. It does not prove that the HTTP stack can read, route, and respond to requests. When a newly deployed service starts cleanly but every request hangs, inspect recent dependency overrides early—especially where an embedded server framework and its network library have been upgraded independently.



