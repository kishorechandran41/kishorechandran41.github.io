Title: My Front End Environment Learnings 
Date: 2021-10-12 17:30
Modified: 2021-10-12 17:30
Category: Front End
Tags: Front End
Slug: My-Front-End-Learnings 
Authors: Kishore Chandran
Summary: My Front End Learnings, not much just a start.


### Environment
This is the basic requirement for a system to do with its operation
some explanations are as below,
#### OS StartUp
 * Turn on, the bootloader gets loaded
 * Magic number is sent to RAM
 * Program Counter gets initialized 
 * Here the BIOS is set up and ready to start
 * Hardware is initialized -> IO and Memory
 * Now the control transfers to GREB
 * GREB inits RAM's full capability by Initializing STACK and HEAP for OS
 * C Tool chains and headers are initialized
 * Garbage Collector processes started

If above is the environment required to run the OS similarly we need environments to 
run our code, once such environment is Google's V8.

### Google's V8
V8 engine is a runtime environment for our code to run in Google Chrome.

```
+------------------------------------------------------------------------------------------+
| Google Chrome                                                                            |
|                                                                                          |
| +----------------------------------------+          +------------------------------+     |
| | Google V8                              |          |            WebAPIs           |     |
| | +-------------+ +---------------+      |          |                              |     |
| | |    Heap     | |     Stack     |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | +-------------+ +---------------+      |          |                              |     |
| |                                        |          |                              |     |
| +----------------------------------------+          +------------------------------+     |
|                                                                                          |
|                                                                                          |
| +---------------------+     +---------------------------------------+                    |
| |     Event loop      |     |          Task/Callback queue          |                    |
| |                     |     |                                       |                    |
| +---------------------+     +---------------------------------------+                    |
|                             +---------------------------------------+                    |
|                             |          Microtask queue              |                    |
|                             |                                       |                    |
|                             +---------------------------------------+                    |
|                                                                                          |
|                                                                                          |
+------------------------------------------------------------------------------------------+
```

V8 can act as an Interpreter and Compiler -> Just in time compiler.
Something that is to be noted about the V8 is how it manages its stack and heap, 
an element that is created in the environment is not destroyed until its manually removed.

It is to be noted that V8 runs on single thread and we must not push any high processing tasks.
The multithreading and high processing tasks are handled by Google Chrome using Event Loop and Task/Callback queue.

### Node.js
Node.js is also an JavaScript runtime just like V8, which can be used to run .js files 
from the console or terminal.