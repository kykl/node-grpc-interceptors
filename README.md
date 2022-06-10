# grpc-interceptors
This library provides a way to instrument Node.js gRPC clients and servers with interceptors/middleware. Server-side interceptors are [not yet implemented](https://github.com/grpc/grpc-node/issues/419) in grpc for node.  This library is a temporary fix for this.  Node module grpc-js is used.

# Fork

[Original repository here](https://github.com/echo-health/node-grpc-interceptors).  Only two small changes in file **server-proxy.js** were done in this fork.

This fork was to allow better password authentication through interceptors. In original library, it was not possible to raise an error if authorization was denied.

In this fork, the interceptor has now access to the callback (of the RPC call). That way, it's now possible to cancel the RPC by calling the callback with an error. This is necessary when doing simple user/password authentication through interceptors.

For much better security, use also [TLS mutual authentication](https://github.com/grpc/grpc/issues/6757#issuecomment-261703455).

## Install

```
npm install git+https://github.com/mfecteau34/node-grpc-interceptors.git
or better:
npm install grpcjs-interceptors
```

### Usage

This usage example takes for granted that two RPC metadata parameters are sent with the RPC call.  One parameter is "user", and the other "password".

If using the grpcurl utility for testing, add those parameters in the command line : **-rpc-header 'user: rpcuser' -rpc-header 'password: abc123'**

Lodash is also used to get the parameters easily.

```js
const interceptors = require('grpc-interceptors');
const _ = require('lodash');

const server = interceptors.serverProxy(new grpc.Server());
server.addService(proto.MyPackage.MyService.service, { Method1, Method2 });

// Original version did not have "callback" as a parameter.
const myMiddlewareFunc = async function (ctx, next, callback) {

    // do stuff before call
    console.log('Making gRPC call...');
    
    // Doing authentication
    let user = _.get(ctx,'call.metadata._internal_repr.user[0]','undefined')
    let password = _.get(ctx,'call.metadata._internal_repr.password[0]','undefined')
    if (user == "rpcuser" && password == "abc123") { 
        // Executing the RPC call only if the authentication was successful
        await next();
        // do stuff after call
        console.log(ctx.status.code);
    } else {
        // The whole point for this fork was to have access to callback function ...
        callback(new Error("Unauthorized"));
    }

}

server.use(myMiddlewareFunc);
```

