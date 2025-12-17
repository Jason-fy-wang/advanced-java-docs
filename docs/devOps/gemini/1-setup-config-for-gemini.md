---
tags:
  - config
  - gemini
  - gemini-config
---
After we config network as below, we can access google success,  but the gemini happen `fetch fail`  still.  
[[3-v2RayN-TUN-mode]] 

How to resolve the gemini `fetch error?`
The reason for fetch error is Node didn't access network through TUN, so we have to config proxy for node :
```powershell
# proxy for Node
$env:ALL_PROXY="socks5://127.0.0.1:7889"

```

