# Simple C++ HTTP Server

This is a small multithreaded C++ HTTP server for POSIX systems.
It has no dependencies except a compliant C++11 compiler. 

## Installation

The server is self-contained within one source file (.cpp) and one header (.h).
Copy these into your project and you are good to go. There are no dependencies beyond POSIX and C++11. 
This means any semi-recent Linux, BSD, or Mac should work. 
Windows is not supported at the moment.

This repository contains more files than those that makeup the server. 
For example, it contains some unit tests and a testing framework to run these tests. 
Further, two examples servers using the API are provided. 

## Example

A minimum server looks as follows:

```cpp
#include "http_server.h"
#include <iostream>
#include <sstream>

int main(){
    io::http::Config config;
    config.port = 8888;
    io::http::run(
        config, 
        [](int worker_id, int worker_count, const io::http::Request&req, io::http::Response&res){
            std::ostringstream out;
            out << "Worker "<<worker_id << " of " << worker_count <<" works says hello and starts working on "<<req.resource << "\n";            
            res.body = out.str();
        },
        []{std::cerr << "Server is running. Hit CTRL+C once for graceful shutdown. Hit CTRL+C a second time to kill the server.\n";}
    );
    std::cerr << "Server was shutdown gracefully\n";
}
```

Compile the program and run it. The server can now be accessed using `curl` as follows:

```bash
curl localhost:8888/hello
```

The response is a text similar to 

```
Worker 1 of 4 works says hello and starts working on /hello
```

How many workers are spawned depends on your machine. 
Which worker answers a request is random.

Hitting CTRL+C, i.e., sending SIGINT, will cause the server to terminate gracefully. 
This means that all accepted requests are fully answered but no new connections are accepted.
Once all requests have terminated, the `run` method terminates.
All resources are freed and all destructors are executed. 
Hitting CTRL+C a second time, calls the SIGINT handler installed before `run` was invoked. 
Usually, this terminates the program without running any destructors. 

If the request handler throws an exception, it is caught and a "Bad Request" response is sent to the client. 
The server does not terminate.

The server answers POST and GET requests. 
The server does not differentiate between them. 
A GET request is handled as a POSt request that has an empty body.
There is currently no support for HTTP headers beyond a MIME-type in the response.

Request and response objects are recycled between calls. 
This extends to the string buffers.
This is done to avoid unnecessary memory allocations.

## Example with Worker Data

In some setups, every worker needs to have access to resources that are too expensive to reaquire in every request. 
Examples are memory allocations or database connections. 
For this usecase, a second server variant exists. 
It uses `ServerData` and `WorkerData`. 
`ServerData` is essentially read-only global data that is shared among all workers. 
`WorkerData` exists per worker and each worker has read and write access.
`WorkerData` is very similar to thread local storage.

The following example removes duplicates from a given list using a large boolean array. 
The array is allocated only once and not per request.

```cpp
#include "http_server.h"
#include <iostream>
#include <sstream>

const int max_num = 1000000;

struct ServerData{};

struct WorkerData{
    std::vector<bool>seen;
    
    WorkerData(int worker_id, int worker_count, const ServerData&server_data):
        seen(max_num, false){
        (void)worker_id;
        (void)worker_count;
        (void)server_data;
    }
};

int main(){
    io::http::Config config;
    config.port = 8888;
    ServerData server_data;
    io::http::run_with_worker_data<WorkerData>(
        config,
        server_data,
        [](int worker_id, int worker_count, const ServerData&server_data, WorkerData&worker_data, const io::http::Request&req, io::http::Response&res){
            (void)worker_id;
            (void)worker_count;
            (void)server_data;
            std::istringstream in(req.body);
            int num;
            while(in >> num){
                if(num < 0 || num >= max_num)
                    throw std::runtime_error("number out of bounds");
                worker_data.seen[num] = true;
            }
            std::ostringstream out;
            for(int i=0; i<max_num; ++i){
                if(worker_data.seen[i]){
                    out << i << ' ';
                    worker_data.seen[i] = false;
                }
            }
            res.body = out.str();
            if(!res.body.empty())
                res.body.back() = '\n';
            else
                res.body = "\n";
        },
        []{std::cerr << "Server is running. Hit CTRL+C once for graceful shutdown. Hit CTRL+C a second time to kill the server.\n";}
    );
    std::cerr << "Server was shutdown gracefully\n";
}
```

The server can be called as follows:

```bash
curl -d "42 107 50 1 2 3 1 2 3 42 43" localhost:8888
```

The response looks as follows:

```
1 2 3 42 43 50 107
```

## Documentation

A request object has the following structure:

```cpp
struct Request{
    std::string resource;
    std::string body;
};
```

`resource` contains the URL with the GET-parameters. 
`body` contains the uploaded file, if there is one, and is an empty string otherwise. 
The server does not discern between GET and POST requests. 
They are both answered by the same handler.
GET requests have an empty body.

A response object is a struct with some convenience constructors. It looks as follows:

```cpp
struct Response{
    std::string body;
    std::string mime_type;
    int status;

    Response();
    Response(std::string body);
    Response(std::string body, int status);
    Response(std::string body, std::string mime_type);
    Response(std::string body, std::string mime_type, int status);
};
```

If omitted from the constructor parameters, `status` is 200 which means that everything went well. 
If `mime_type` is ommited, then the constructed `Response` has an empty `mime_type` string.
If this string is empty when the handler terminates, no MIME-type header is sent to the client.
The request object passed to a handler is in a default constructed state, i.e., `status` is 200 and `mime_type` is empty.

Finally, the server can be configured using a `Config` object. It has the following structure:

```cpp
struct Config{
    int port;
    int worker_count;
    bool install_int_signal_handler;
    int max_request_body_size;
    int max_request_resource_size;
    int timeout_in_seconds;
    bool print_exception_message_in_answer;

    Config();
};
```

The default constructor sets sensible default values for every parameter. If in doubt, keep the default value. The meaning of these parameters are:

* port: The port on which the server should listen.
* worker_count: How many threads should be spawned.
* timeout_in_seconds: How long should the server keep a connection alive, if the client no longer answers?
* max_request_body_size: How many bytes may an uploaded file have at most?
* max_request_resource_size: How many bytes may an URL including GET parameters have at most?
* install_int_signal_handler: If true, the handler for SIGINT (CTRL+C) is replaced to allow for a graceful shutdown. In the awkward case, that you want to run multiple servers within one program from different threads, this value must be set to false, as only one server can replace the singal handler. The previous signal handler is restored after the server finishes.
* print_exception_message_in_answer: If true, the content of `std::exception::what` is put into the answer to the client.

*Warning*: The reason why `max_request_body_size` and `max_request_resource_size` exist is to prevent attacks from a rogue client. 
Any client can force the server to allocate the maximum values before any handler is run. 
Be sure that your server can allocate `(max_request_body_size+max_request_resource_size) * worker_count` bytes of memory.
