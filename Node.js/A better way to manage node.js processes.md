## Preface

When talking about Node.js processes management, the most common tool we think about is PM2, but my opinion is that pm2 is not the best choice. This article mainly introduces why many developers choose pm2 and a better way to manage node.js processes.

## What is pm2

As the official readme says, pm2 is
> a production process manager for Node.js applications with a built-in load balancer. It allows you to keep applications alive forever,to reload them without downtime and to facilitate common system admin tasks.  

In short: it's a tool that launches your Node.js based application, watches it to detect when it has failed, and restart it upon failure.  

It also provides some measure of application performance monitor(APM),but this is not the main feature.

## The strenghts and weakness of pm2

In terms of the role of process management, there are many alternatives such as System V, OpenRc and so on. But why pm2 is popular? I think that the main reasons includes:

* The majority of people developing Node.js application for server systems are not experts on application deployment.
* Software developers like to use tools written in the same language the are writing their own software in.
* That part about "a built-in load balancer" in the pm2 description quoted above.
* Someone wrote an article that suggested using it, the article became popular, and eventually turned in to gospel passed down through the ages.

Consider the following basic web server applications:
```javascript
'use strict'

const server = require('fastify)({logger: true});

server.route({
  path: '/',
  method: 'get',
  handler (req, res) {
    res.send('hello world')
  }
});

server.listen({port: process.env.PORT})

```

pm2 makes it easy to start,and keep running,such an application:

```shell
$ export PORT=8080
$ pm2 start index.js
```
It also provides a simple switch to enable load balancing,what it calls "cluster mode",that will utilize multiple CPUs(or CPU cores) for the server:
```shell
$ export PORT=8080
$ pm2 start index.js -i 2 # where 2 is the number of CPUs to use
```

From there, it provides tooling for inspecting and managing logs and viewing metrics around the process. You can consult their documentation for more information on these features.

But at the same time, pm2 brings some disadvantages: 

* First, it wraps the application in what it calls a [processContainer](https://github.com/Unitech/pm2/blob/311c53298448fc4575fc689c4943692a664373ad/lib/ProcessContainer.js). This container starts the application in a subprocess with an IPC channel,monkey patches process.stdout and process.stdin, registers handler for typical process signals, and register handlers for uncaughtException and unhandledRejection. I recommend reading through the linked container source code to understand how pm2 is handling your process. Pay close attention to the stdout and stdin pathces. Some [issues in Pino](https://github.com/pinojs/pino/issues?q=is%3Aissue+sort%3Aupdated-desc+pm2+is%3Aclosed) regarding monkey patched stdout, with several of them being due to the patching done by pm2.

* Second, pm2 enables "load balancing" through the use of Node.js's cluster module. This module implements a round-robin load balancing algorithm, which pm2 seems to rely upon,as its primary balancing algorithm. The result is all network connections really going to a single process, the pm2 process, and then getting balanced to one of the forked processes. While the overhead introduced here may be negligible if pm2 is only being used to manage a singular application, I highly doubt it remains so when multiple applications are being managed by pm2.

* Finally, using pm2 to manage your process means you have wrapped your Node.js application up in another Node.js application thereby inheriting any performance penalties of the parent application in addition to your own application's performance characteristics.

## What instead

Assuming the application is being deployed to a full system,e.g. "bare metal" or some sort of virtual machine or VPS, we should utilize the OS's native process manager. A typical deployment host in this sort of setup is [Debian](https://www.debian.org/), which, at this time, uses systemd as its process manager.

As the description above, pm2 adds handlers for process signals and process errors to implement the feature of keeping your application alive, we can also achieve this goal in the same manner.

```javascript
'use strict'

const server = require('fastify')({ logger: true })

function handleSignal(sig) {
  server.log.info(`handling signal ${sig}`)
  server.close()
}
['SIGINT', 'SIGTERM'].forEach(sig => process.on(sig, handleSignal))

process.on('uncaughtException', error => {
  server.log.warn('got uncaughtException', error)
  server.close()
})
process.on('unhandledRejection', error => {
  server.log.warn('got unhandledRejection', error)
  server.close()
})

server.route({
  path: '/',
  method: 'get',
  handler (req, res) {
    res.send('hello world')
  }
})

server.listen({ port: process.env.PORT })

```

Subsequently, we can configure the process manager to manage the application:

1. Add a new unprivilged user for our application:
```shell
adduser --system myapp
```
2. Create a deployment location for the app:
```shell
mkdir -p /opt/apps/myapp && chown myapp /opt/apps/myapp
```
3. Deploy the app:
```shell
cd /opt/apps/myapp && tar xcf /tmp/myapp.tar.gz
```
4. Add a service description file as
/etc/systemd/system/myapp.service:

```shell
[Unit]
Description=My Cool Web Server
Requires=network.target

[Service]
Type=simple
Restart=always
RestartSec=1
User=myapp
Group=nogroup
WorkingDirectory=/opt/apps/myapp
Environment="PORT=8000"
ExecStart=/usr/bin/node /opt/apps/myapp/index.js

[Install]
WantedBy=multi-user.target
```
5. Activate the service:
```shell
systemctl daemon-reload
systemctl enable myapp
systemctl start myapp.service
```

6. Verify the service is working:
```shell
curl http://127.0.0.1:8000/
```

Through using the system tools, we gain a few benefits:

1. Anyone familiar with the standard OS tools will be familiar with how to manage the service.
2. We gain limited process privileges through the use of system accounts.
3. Our service will start with the system according to the service configuration
4. Logs are managed through the standard system log management:
```shell
journalctl -u myapp
```

### Clustering

There is one caveat to the above example: it's a singular process utilizing one CPU core according to the standard Node.js core usage. If we want to dedicate more resources to our application,we need to do a little more work.

First, we need to boot multiple instance of our application. With systemd, we can rewrite our service using a target instead:
1. Create /etc/systemd/system/myapp.target like:
```shell
[Unit]
Description=My Cool Web Server
Requires=myapp@1.service myapp@2.service

[Install]
WantedBy=multi-user.target
```

2. Create /etc/systemd/system/myapp@.service like:
```shell
[Unit]
Description=My Cool Web Server %I
Requires=network.target
PartOf=myapp.target

[Install]
WantedBy=myapp.service

[Service]
Type=simple
Restart=always
RestartSec=1
User=myapp
Group=nogroup
WorkingDirectory=/opt/apps/myapp
Environment="PORT=800%I"
ExecStart=/usr/bin/node /opt/apps/myapp/index.js
```

3. Install the service
```shell
systemctl daemon-reload
systemctl enable myapp.target
```

4. Start the service as many times as CPU resources desired:
```shell
systemctl start myapp.target
```

5. Verify they are runing
```
curl http://127.0.0.1:8001
curl http://127.0.0.1:8002
```

Second, we need a way to load balance traffic to those instances. To do so, we should use a reverse proxy. My preference is to use [HAProxy](https://www.haproxy.org/). In short, install HAProxy, provide a configuration like the following one, and enable the service:

```shell
frontend myapp-proxy
  bind 0.0.0.0:80
  use_backend myapp-backend

backend myapp-backend
  server myapp1 127.0.0.1:8001
  server myapp2 127.0.0.2:8002
```

We get the same sort of round-robin load balancing as the pm2 load balancing, but in a seperate process. This allow us to restart individual instance at will without downtime(e.g. systemctl restart myapp@1), among other niceties like TLS termination.

### Containerized Deployment

The deploy manner described above is an increasingly rare method of deploying application. Nowadays most deployments are done through some form of containerization. The most basic of which is Docker. In such a case, the container host acts as the process manager and the "container" is the process. This means that the container should be written such that the embedded application is booted directly.

```shell
FROM debian:stable-slim

# copy script and node_modules into the container

PORT 8000
CMD ["node", "/myapp/index.js"]
```

Note that we should use the same modified script as we used in the bare metal deployment. The container host is still going to send traditional process management signals and the application should recognize them.

## Conclusion

pm2 is a process manager that simplifies process management for developers that may not have much knowledge of how systems manage processes.But it comes with some inherent costs and caveats: namely it does not run applications in an unaltered environment. We should take care to deploy our applications with as little runtime environment changes as is necessary in order to reduce the number of things we need to investigate when something goes wrong. By utilizing the standard tooling we are able to run our applications with as little interference as possible. And we also gain the benefit of anyone being able to manage our applications through standard interfaces without having to learn new setups and tools.

