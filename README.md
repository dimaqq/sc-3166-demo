A reproducer for styled-components#3166

#### What is going on

`styled-components` support 2 build-time environment variables `SC_ATTR` and `SC_DISABLE_SPEEDY`, plus their `REACT_APP_` aliases.

This triggers a bug in create-react-app / webpack where environment variables that are referenced but not defined are not optimised away.

Thus the entire† environment variable block is included.

† `create-react-app` cleans the shell environment, only variables starting with `REACT_APP_` are allowed, and apparently some predefined values.

#### References

Variables were introduced in:

https://github.com/styled-components/styled-components/pull/2501 (`REACT_APP_` prefix)

https://github.com/styled-components/styled-components/issues/3166 (previous bug report for this issue)

https://github.com/facebook/create-react-app/issues/8961 (upstream bug report)

#### Without `styled-components`

All good, no `NODE_ENV` or `WDS_SOCKET_HOST` in the build project.

#### With `styled-components`

The built project (beautified) contains the following code:

```js
            var b = "undefined" !== typeof e && (Object({
                    NODE_ENV: "production",
                    PUBLIC_URL: "",
                    WDS_SOCKET_HOST: void 0,
                    WDS_SOCKET_PATH: void 0,
                    WDS_SOCKET_PORT: void 0
                }).REACT_APP_SC_ATTR || Object({
                    NODE_ENV: "production",
                    PUBLIC_URL: "",
                    WDS_SOCKET_HOST: void 0,
                    WDS_SOCKET_PATH: void 0,
                    WDS_SOCKET_PORT: void 0
                }).SC_ATTR) || "data-styled",
                w = "undefined" !== typeof window && "HTMLElement" in window,
                k = "boolean" === typeof SC_DISABLE_SPEEDY && SC_DISABLE_SPEEDY || "undefined" !== typeof e && (Object({
                    NODE_ENV: "production",
                    PUBLIC_URL: "",
                    WDS_SOCKET_HOST: void 0,
                    WDS_SOCKET_PATH: void 0,
                    WDS_SOCKET_PORT: void 0
                }).REACT_APP_SC_DISABLE_SPEEDY || Object({
                    NODE_ENV: "production",
                    PUBLIC_URL: "",
                    WDS_SOCKET_HOST: void 0,
                    WDS_SOCKET_PATH: void 0,
                    WDS_SOCKET_PORT: void 0
                }).SC_DISABLE_SPEEDY) || !1,
                x = function() {
                    return n.nc
                };
```

#### With `styled-components` with environment variable

Since the upstream bug is triggered when the referenced environment variable is not set, actually setting it (e.g. empty) should work around the bug.

For example, building the project using `env REACT_APP_SC_ATTR="zz" yarn build` reduces the injected environment blocks to:

```js
            var b = "undefined" !== typeof e ? "zz" : "data-styled",
                w = "undefined" !== typeof window && "HTMLElement" in window,
                k = "boolean" === typeof SC_DISABLE_SPEEDY && SC_DISABLE_SPEEDY || "undefined" !== typeof e && (Object({
                    NODE_ENV: "production",
                    PUBLIC_URL: "",
                    WDS_SOCKET_HOST: void 0,
                    WDS_SOCKET_PATH: void 0,
                    WDS_SOCKET_PORT: void 0,
                    REACT_APP_SC_ATTR: "zz"
                }).REACT_APP_SC_DISABLE_SPEEDY || Object({
                    NODE_ENV: "production",
                    PUBLIC_URL: "",
                    WDS_SOCKET_HOST: void 0,
                    WDS_SOCKET_PATH: void 0,
                    WDS_SOCKET_PORT: void 0,
                    REACT_APP_SC_ATTR: "zz"
                }).SC_DISABLE_SPEEDY) || !1,
                x = function() {
                    return n.nc
                };
```

Setting more environment variables `env REACT_APP_SC_ATTR="zz" REACT_APP_SC_DISABLE_SPEEDY="" yarn build` reduces the injection further:


```js
            var b = "undefined" !== typeof e ? "zz" : "data-styled",
                w = "undefined" !== typeof window && "HTMLElement" in window,
                k = "boolean" === typeof SC_DISABLE_SPEEDY && SC_DISABLE_SPEEDY || "undefined" !== typeof e && Object({
                    NODE_ENV: "production",
                    PUBLIC_URL: "",
                    WDS_SOCKET_HOST: void 0,
                    WDS_SOCKET_PATH: void 0,
                    WDS_SOCKET_PORT: void 0,
                    REACT_APP_SC_DISABLE_SPEEDY: "",
                    REACT_APP_SC_ATTR: "zz"
                }).SC_DISABLE_SPEEDY || !1,
                x = function() {
                    return n.nc
                };
```

Note that the unprefixed `SC_DISABLE_SPEEDY` is filtered out by `react-scripts` and is not passed to webpack.

#### Supply truthy values to environment variables

`env REACT_APP_SC_ATTR="1" REACT_APP_SC_DISABLE_SPEEDY="1" yarn build` eliminates the injected blocks, at the cost of "speedy" / CSSOM API.

#### Environment variable value `false` is not falsy

`env REACT_APP_SC_ATTR=dummy REACT_APP_SC_DISABLE_SPEEDY=false yarn build` doesn't work as `"false"` is interpreted as a string:

```js
            var b = "undefined" !== typeof e ? "dummy" : "data-styled",
                w = "undefined" !== typeof window && "HTMLElement" in window,
                k = "boolean" === typeof SC_DISABLE_SPEEDY && SC_DISABLE_SPEEDY || "undefined" !== typeof e && "false" || !1,
                x = function() {
                    return n.nc
                };
```

Here, `k` becomes `"false"`, a string, and it's used later as `useCSSOMInjection: !k`, that is CSSOM injection is disabled.
