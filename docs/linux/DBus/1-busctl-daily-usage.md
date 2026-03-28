---
tags:
  - dbus
  - busctl
  - python
  - linux
---

# D-Bus & busctl Daily Usage Guide

D-Bus is a message bus system that allows applications to talk to one another. `busctl` is the modern introspection and management tool for D-Bus (part of systemd).

## 1. Finding Objects and Services

To interact with D-Bus, you first need to find the **Service**, the **Object Path**, and the **Interface**.

### List all available services
- **System Bus**: `busctl list`
  ```shell
  busctl list
  ```
  ```output
  NAME                             PID PROCESS         USER           CONNECTION
  :1.0                             474 systemd         root           :
  org.freedesktop.hostname1        474 systemd         root           :
  org.freedesktop.locale1          474 systemd         root           :
  org.freedesktop.login1           474 systemd         root           :
  org.freedesktop.systemd1         474 systemd         root           :
  org.freedesktop.timedate1        474 systemd         root           :
  ...
  ```

- **Session Bus (User)**: `busctl --user list`
  ```shell
  busctl --user list
  ```
  ```output
  NAME                             PID PROCESS         USER           CONNECTION
  :1.0                             987 systemd         user           :
  org.gnome.Terminal               123 gnome-terminal-server user :
  org.gnome.evolution.dataserver   456 evolution-calendar-factory user :
  ...
  ```

### Find Objects under a service
Once you have a service name (e.g., `org.freedesktop.NetworkManager`), list its objects:
- `busctl tree <Service>`
  ```shell
  # Example: List objects under NetworkManager
  busctl tree org.freedesktop.NetworkManager
  ```
  ```output
  / (root)
  └─org
    └─freedesktop
      └─NetworkManager
        ├─Settings
        │ └─Connection
        │   └─...
        ├─ActiveConnection
        │ └─...
        ├─Device
        │ ├─Wireless
        │ └─Wired
        └─...
  ```

### Introspect an Object (Find Methods, Properties, and Signals)
To see what a specific object can do:
- `busctl introspect <Service> <Path>`
  ```shell
  # Example: Introspect the NetworkManager root object
  busctl introspect org.freedesktop.NetworkManager /org/freedesktop/NetworkManager
  ```
  ```output
  NAME                                TYPE       SIGNATURE RESULT        FLAGS
  org.freedesktop.DBus.Introspectable
    .Introspect                         method     -         s             -
  org.freedesktop.DBus.Properties
    .Get                                method     ss        v             -
    .GetAll                             method     s         a{sv}         -
    .Set                                method     ssv       -             -
  org.freedesktop.NetworkManager
    .ActiveConnections                  property   ao        read
    .AllDevices                         property   ao        read
    .Version                            property   s         read
    .GetDevices                         method     -         ao            -
    .State                              property   u         read
    .CheckConnectivity                  method     -         u             -
    .StateChanged                       signal     u         -
  ...
  ```

---

## 2. Common busctl Commands

### Call a Method
- `busctl call <Service> <Path> <Interface> <Method> <Signature> [Arguments...]`
  ```shell
  # Example: Get the Systemd version (no arguments, no signature needed for method)
  busctl call org.freedesktop.systemd1 /org/freedesktop/systemd1 org.freedesktop.systemd1.Manager Version
  ```
  ```output
  s "v245.4-4ubuntu3.20"
  ```
  ```shell
  # Example: Get a specific session (requires signature 's' for string argument)
  busctl call org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager GetSession s "c1"
  ```
  ```output
  o "/org/freedesktop/login1/session/c1"
  ```

### Get a Property
- `busctl get-property <Service> <Path> <Interface> <Property>`
  ```shell
  # Example: Get the Version property of Systemd Manager
  busctl get-property org.freedesktop.systemd1 /org/freedesktop/systemd1 org.freedesktop.systemd1.Manager Version
  ```
  ```output
  s "v245.4-4ubuntu3.20"
  ```
  ```shell
  # Example: Get the State property of NetworkManager
  busctl get-property org.freedesktop.NetworkManager /org/freedesktop/NetworkManager org.freedesktop.NetworkManager State
  ```
  ```output
  u 70
  ```

### Set a Property
- `busctl set-property <Service> <Path> <Interface> <Property> <Signature> <Value>`
  ```shell
  # Example: Set a property (hypothetical, as many are read-only and require root)
  busctl set-property org.example.Service /org/example/Path org.example.Interface MyProperty b true
  ```
  ```output
  # No output on success
  ```

### Monitor D-Bus Traffic
- **Monitor all traffic**: `busctl monitor`
  ```shell
  # Example: Monitor all D-Bus messages on the system bus
  sudo busctl monitor
  ```
  ```output
  # Output will show all D-Bus messages in real-time. Press Ctrl+C to exit.
  TYPE "signal" SENDER "org.freedesktop.systemd1" DEST "(unspecified)"
  PATH "/org/freedesktop/systemd1" INTERFACE "org.freedesktop.systemd1.Manager" MEMBER "UnitNew"
  "s" "network.target"
  "o" "/org/freedesktop/systemd1/unit/network_2e_target"
  ...
  ```

- **Monitor specific service**: `busctl monitor <Service>`
  ```shell
  # Example: Monitor traffic for NetworkManager
  sudo busctl monitor org.freedesktop.NetworkManager
  ```
  ```output
  # Output will show D-Bus messages related to NetworkManager. Press Ctrl+C to exit.
  TYPE "signal" SENDER "org.freedesktop.NetworkManager" DEST "(unspecified)"
  PATH "/org/freedesktop/NetworkManager" INTERFACE "org.freedesktop.NetworkManager" MEMBER "StateChanged"
  "u" 70
  ...
  ```

---

## 3. Python D-Bus Examples

The most common library for D-Bus in Python is `dbus-python` or the more modern `pydbus`.

### Example using `pydbus` (Recommended for simplicity)

```python
from pydbus import SystemBus

# 1. Connect to the System Bus
bus = SystemBus()

# 2. Get a proxy object for a service
# Pattern: bus.get(".service.name", "/object/path")
systemd = bus.get(".systemd1", "/org/freedesktop/systemd1")

# 3. Call a method
# Note: pydbus maps D-Bus methods to Python methods
try:
    units = systemd.ListUnits()
    print("
--- ListUnits Output (truncated) ---")
    for unit in units[:5]: # Print first 5 units for brevity
        print(f"Unit: {unit[0]}, Description: {unit[1]}")
except Exception as e:
    print(f"Error calling ListUnits: {e}")

# 4. Get a property
try:
    version = systemd.Version
    print(f"
Systemd Version: {version}")
except Exception as e:
    print(f"Error getting Version property: {e}")
```

```output
--- ListUnits Output (truncated) ---
Unit: -.slice, Description: Root Slice
Unit: proc-sys-fs-binfmt_misc.automount, Description: Arbitrary Executable File Formats File System Automount Point
Unit: dev-hugepages.mount, Description: Huge Pages File System
Unit: dev-mqueue.mount, Description: POSIX Message Queue File System
Unit: systemd-journald.service, Description: Journal Service

Systemd Version: v245.4-4ubuntu3.20
```

### Example using `dbus-python` (Legacy/Low-level)

```python
import dbus

# Connect to system bus
bus = dbus.SystemBus()

# Get the object
try:
    obj = bus.get_object('org.freedesktop.login1', '/org/freedesktop/login1')

    # Select the interface
    interface = dbus.Interface(obj, 'org.freedesktop.login1.Manager')

    # Call a method
    sessions = interface.ListSessions()
    print("
--- ListSessions Output ---")
    for session_info in sessions:
        # session_info is a tuple: (session_id, user_id, user_name, seat_id, object_path)
        print(f"Session ID: {session_info[0]}, User: {session_info[2]}")

except dbus.exceptions.DBusException as e:
    print(f"Error interacting with D-Bus: {e}")
except Exception as e:
    print(f"An unexpected error occurred: {e}")
```

```output
--- ListSessions Output ---
Session ID: c1, User: user
```

---

## 4. Understanding Signatures

When using `busctl call` or `set-property`, you must provide a signature string:

| Signature | Type | Python Example |
| :--- | :--- | :--- |
| `s` | String | `"hello"` |
| `b` | Boolean | `True/False` |
| `y` | Byte | `0x01` |
| `n` | Int16 | `123` |
| `i` | Int32 | `123456` |
| `x` | Int64 | `123456789` |
| `as` | Array of Strings | `["a", "b"]` |
| `a{sv}` | Dict (String to Variant) | `{"key": "value"}` |
