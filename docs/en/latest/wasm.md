---
title: WASM
---

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

APISIX supports WASM plugins written with [Proxy WASM SDK](https://github.com/proxy-wasm/spec#sdks).

This plugin requires APISIX to run on [APISIX-OpenResty](../how-to-build.md#step-6-build-openresty-for-apache-apisix), and is under construction.
Currently, only a few APIs are implemented. Please follow [wasm-nginx-module](https://github.com/api7/wasm-nginx-module) to know the progress.

## Programming model

The plugin supports the follwing concepts from Proxy WASM:

```
                    Wasm Virtual Machine
┌────────────────────────────────────────────────────────────────┐
│      Your Plugin                                               │
│          │                                                     │
│          │ 1: 1                                                │
│          │         1: N                                        │
│      VMContext  ──────────  PluginContext                      │
│                                           ╲ 1: N               │
│                                            ╲                   │
│                                             ╲  HttpContext     │
│                                               (Http stream)    │
└────────────────────────────────────────────────────────────────┘
```

* All plugins run in the same WASM VM, like the Lua plugin in the Lua VM
* Each plugin has its own VMContext (the root ctx)
* Each configured route/global rules has its own PluginContext (the plugin ctx).
For example, if we have a service configuring with WASM plugin, and two routes inherit from it,
there will be two plugin ctxs.
* Each HTTP request which hits the configuration will have its own HttpContext (the HTTP ctx).
For example, if we configure both global rules and route, the HTTP request will
have two HTTP ctxs, one for the plugin ctx from global rules and the other for the
plugin ctx from route.

## How to use

First of all, we need to define the plugin in `config.yaml`:

```yaml
wasm:
  plugins:
    - name: wasm_log # the name of the plugin
      priority: 7999 # priority
      file: t/wasm/log/main.go.wasm # the path of `.wasm` file
```

That's all. Now you can use the wasm plugin as a regular plugin.

For example, enable this plugin on the specified route:

```shell
curl -i http://127.0.0.1:9080/apisix/admin/routes/1  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/index.html",
    "plugins": {
         "wasm_log": {
             "conf": "blahblah"
         }
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:1980": 1
        }
    }
}'
```

Attributes below can be configured in the plugin:

| Name           | Type                 | Requirement | Default        | Valid                                                                      | Description                                                                                                                                         |
| --------------------------------------| ------------| -------------- | -------- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
|  conf         | string | required |   |  != ""        | the plugin ctx configuration which can be fetched via Proxy WASM SDK |
