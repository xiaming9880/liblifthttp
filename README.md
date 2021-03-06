liblifthttp - Safe Easy to use C++17 HTTP client library.
=========================================================

[![CircleCI](https://circleci.com/gh/jbaldwin/liblifthttp/tree/master.svg?style=svg)](https://circleci.com/gh/jbaldwin/liblifthttp/tree/master)
[![language][badge.language]][language]
[![license][badge.license]][license]

[badge.language]: https://img.shields.io/badge/language-C%2B%2B17-yellow.svg
[badge.license]: https://img.shields.io/badge/license-Apache--2.0-blue

[language]: https://en.wikipedia.org/wiki/C%2B%2B17
[license]: https://en.wikipedia.org/wiki/Apache_License

You're using curl? Do you even lift?

https://github.com/jbaldwin/liblifthttp

**liblifthttp** is a C++17 HTTP client library that provides an easy to use API for both synchronous _and_ asynchronous requests.  It is built upon the rock solid libcurl and libuv libraries.

**liblifthttp** is licensed under the Apache 2.0 license.

# Overview #
* Easy to use Synchronous and Asynchronous HTTP Request APIs.
* Safe C++17 client library API, modern memory move semantics.
* Background IO thread(s) for sending and receiving Async HTTP requests.
* Request pooling for re-using HTTP requests.

# Usage #

## Examples

See all of the examples under the examples/ directory.  Below are some simple examples
to get your started on using liblifthttp with both the synchronous and asynchronous APIs.

```C++
// libcurl requires some global functions to be called before being used.
// LibLiftHttp will call these appropriately if you place the following in
// the projects main.cpp file(s) where necessary.
static lift::GlobalScopeInitializer g_lifthttp_gsi{};
```

### Simple Synchronous
```C++
// Synchronous requests can be created on the stack.
lift::Request request{"http://www.example.com"};
// This is the blocking synchronous HTTP call.
auto response = request.Perform();
std::cout << "LiftStatus: " << lift::to_string(response.LiftStatus()) << "\n";
std::cout << "HTTP Status Code: " << lift::to_string(response.StatusCode()) << "\n";
for(const auto& header : response.Headers())
{
    std::cout << header.Name() << ": " << header.Value() << "\n";
}
std::cout << response.Data(); 
```

### Simple Asynchronous
```C++
// Creating the event loop starts it immediately, it spawns a background thread for executing requests.
lift::EventLoop loop{};

// Create the request just like we did in the sync version, but now provide a lambda for on completion.
// NOTE: that the Lambda is executed ON the Lift event loop background thread.  If you want to handle 
// on completion processing on this main thread you need to std::move() it back via a queue or inter-thread 
// communication.  This is imporant if any resources are shared between the threads.
// NOTE: The request is created on the heap so ownership can be passed easily via an std::unique_ptr
// to the lift::EventLoop!  lift::Request::make() is a handy function to easily do so.
auto request_ptr = lift::Request::make(
    "http://www.example.com",
    std::chrono::seconds{10}, // Give the request 10 seconds to complete or timeout.
    [](lift::RequestPtr req_ptr, lift::Response response) 
    {
        std::cout << "LiftStatus: " << lift::to_string(response.LiftStatus()) << "\n";
        std::cout << "HTTP Status Code: " << lift::to_string(response.StatusCode()) << "\n";
        for(const auto& header : response.Headers())
        {
            std::cout << header.Name() << ": " << header.Value() << "\n";
        }
        std::cout << response.Data(); 
    },
);

// Now inject the request into the event to be executed.  Moving into the event loop is required,
// this passes ownership of the request to the event loop background worker thread.
loop.StartRequest(std::move(request_ptr));

// Block on this main thread until the lift event loop background thread has completed the request, or timed out.
while(loop.ActiveRequestCount() > 0) {
    std::this_thread::sleep_for(std::chrono::milliseconds{10});
}

// When loop goes out of scope here it will automatically stop the background thread and cleanup all resources.
```

## Requirements
    C++17 compiler (g++/clang++)
    CMake
    make and/or ninja
    pthreads/std::thread
    libcurl devel
    libuv devel
    zlib devel

## Instructions

### Building
    # This will produce a shared and static library to link against your project.
    mkdir Release && cd Release
    cmake -DCMAKE_BUILD_TYPE=Release ..
    cmake --build .

### CMake Projects
To use within your cmake project you can clone the project or use git submodules and then add subdirectory in the parent project's `CMakeList.txt`,
assuming the lift code is in a `liblifthttp/` subdirectory of the parent project:
    
    add_subdirectory(liblifthttp)

To link to the `<project_name>` then use the following:
    
    add_executable(<project_name> main.cpp)
    target_link_libraries(<project_name> PRIVATE lifthttp)

Include lift in the project's code by simply including `#include <lift/lift.hpp>` as needed.

Note that by default liblifthttp will attempt to use system versions of `libcurl-dev`, `libuv-dev`, `libcrypto-dev`, `libssl-dev`, and `libcares-dev`.  If your project, like some of mine do, require a custom built version 
of `libcurl` or any of the other libraries that curl links to then you can specify the following `cmake` variables to override where liblifthttp
will link `libcurl` development libraries.  These can be dynamic or static libraries.  Note that a custom `libuv-dev` link is not currently supported.

    ${CURL_INCLUDE} # The curl.h header location, default is empty.
    ${LIBSSL}       # The ssl library to link against, default is empty.
    ${LIBCRYPTO}    # The crypto library to link against, default is empty.
    ${LIBCURL}      # The curl library to link against, default is '-lcurl'.
    ${LIBCARES}     # The c-ares (dns) library to link against, default is empty.

## Benchmarks
Using the example benchmark code and a local `nginx` instance serving its default welcome page.  All benchmarks use `keep-alive` connections.  The benchmark is compared against `wrk` as that is basically optimal performance since
`wrk` does zero parsing of the response whereas `lift` does.

Here is the CPU the benchmarks were run on:

    cat /proc/cpuinfo 
    ...
    Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz

Here is how the benchmark application is called (similiar to `wrk`):

    $ ./examples/lift_benchmark 
    ./examples/lift_benchmark <url> <duration_seconds> <connections> <threads>

Using `nginx` as the webserver with the default `ubuntu` configuration.

| Connections | Threads | wrk Req/Sec | lift Req/Sec |
|------------:|--------:|------------:|-------------:|
| 1           | 1       | 28,093      | 18,009       |
| 100         | 1       | 130,946     | 47,077       |
| 100         | 2       | 165,981     | 73,181       |
| 100         | 3       | 173,978     | 88,881       |
| 100         | 4       | 171,778     | 96,355       |

## Contributing and Testing

This project has a [CircleCI](https://circleci.com/) implementation to compile and run unit tests as well as simple integration tests against a local `nginx` instance.

Currently tested distros:
* ubuntu:rolling
* fedora:latest

Currently tested compilers:
* g++
* clang

Contributing should ideally be a single commit if possible.  Any new feature should include relevant tests and examples 
are welcome if understanding how the feature works is difficult or provides some additional value the tests otherwise cannot.

CMake is setup to understand how to run the tests.  Building and then running `ctest` will
execute the tests locally.  Note that the integration tests that make HTTP calls require a webserver
on http://localhost:80/ that will respond with a 200 on the root directory and 404 on any other url.
A future iteration might include an embedded server that responds with more sophisticated tests.

```bash
apt-get install nginx
systemctl start nginx
mkdir Release && cd Release
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .
ctest -V
```

## Support

File bug reports, feature requests and questions using [GitHub Issues](https://github.com/jbaldwin/liblifthttp/issues)

Copyright © 2017-2020, Josh Baldwin
