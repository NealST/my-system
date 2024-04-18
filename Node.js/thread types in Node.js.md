## Preface

Although we frequently hear the voice that there is only one thread in Node.js, but to be strictly, it's not the real fact. This article mainly concludes some different types of threads in Node.js. 

The reason why threads exist is that threads 
is designed to process different tasks simutaniously. Tasks of different types are assigned to different threads.

## Thread Types

### Libuv threads

Libuv is the library that provides an event loop, which makes it possible to perform asynchronous operations on a server. This library has 3 main parts: thread pool, event loop, and callback queue.

The thread pool has 4 threads allocated by default, and the size can be increased up to 1028 threads. The thread pool is used to run three types of operations:

* File system operations
* DNS search functions
* User-defined code(when using libuv API directly)

There is another thread comes from the event loop itself. The library strictly allocates only a single thread to run the event loop.

So when talking about I/O operations in Node.js, the libuv library is the exact component that handles them.

### Javascript engine

In node.js, the default javascript engine is V8. But at the same time, you can switch it to JavaScriptCore, SpiderMonkey or any other.

These engine have the ability to freely create new threads whenever it needs and the number of threads is not limited to a specific number. These threads are used to process some common tasks:

* Garbage collection
* Compilation of Javascript
* System tasks like monitoring and others

### Addons and Node-API

The addons and Node-API are basically a bridge that allows you to write custom C++ and C code and plug it into your Node.js application.

it's similar to the V8 engine that we do not know the exact number of threads produced by the host. Each addon can spawn multiple threads and use multiple resources whenever it needs to.

### Worker threads

It's a special type of thread, which has a unique feature that it's able to create new threads directly from Javascript code. No other components or APIs of Node.js have such power.

For example,

```javascript
const { Worker } = require('node:worker_threads');

const worker = new Worker('./worker.js');
```

The worker can encapsulate CPU-bound operations in Javascript without blocking the main thread.


