# spdlog

Very fast, header only, C++ logging library. [![Build Status](https://travis-ci.org/gabime/spdlog.svg?branch=master)](https://travis-ci.org/gabime/spdlog)&nbsp; [![Build status](https://ci.appveyor.com/api/projects/status/d2jnxclg20vd0o50?svg=true)](https://ci.appveyor.com/project/gabime/spdlog)


## Install
#### Just copy the headers:

* Copy the source [folder](https://github.com/gabime/spdlog/tree/v1.x/include/spdlog) to your build tree and use a C++11 compiler.

#### Or use your favorite package manager:

* Ubuntu: `apt-get install libspdlog-dev`
* Homebrew: `brew install spdlog`
* FreeBSD:  `cd /usr/ports/devel/spdlog/ && make install clean`
* Fedora: `yum install spdlog`
* Gentoo: `emerge dev-libs/spdlog`
* Arch Linux: `yaourt -S spdlog-git`
* vcpkg: `vcpkg install spdlog`
 

## Platforms
 * Linux, FreeBSD, Solaris, AIX
 * Windows (vc 2013+, cygwin)
 * Mac OSX (clang 3.5+)
 * Android

## Features
* Very fast - performance is the primary goal (see [benchmarks](#benchmarks) below).
* Headers only, just copy and use.
* Feature rich using the excellent [fmt](https://github.com/fmtlib/fmt) library.
* Fast asynchronous mode (optional)
* [Custom](https://github.com/gabime/spdlog/wiki/3.-Custom-formatting) formatting.
* Conditional Logging
* Multi/Single threaded loggers.
* Various log targets:
    * Rotating log files.
    * Daily log files.
    * Console logging (colors supported).
    * syslog.
    * Windows debugger (```OutputDebugString(..)```)
    * Easily extendable with custom log targets  (just implement a single function in the [sink](include/spdlog/sinks/sink.h) interface).
* Severity based filtering - threshold levels can be modified in runtime as well as in compile time.



## Benchmarks

Below are some [benchmarks](https://github.com/gabime/spdlog/blob/v1.x/bench/bench.cpp) done in Ubuntu 64 bit, Intel i7-4770 CPU @ 3.40GHz

#### Synchronous mode
```
*******************************************************************************
Single thread, 1,000,000 iterations
*******************************************************************************
basic_st...		    Elapsed: 0.226664	4,411,806/sec
rotating_st...	 	    Elapsed: 0.214339	4,665,499/sec
daily_st...		    Elapsed: 0.211292	4,732,797/sec
null_st...		    Elapsed: 0.102815	9,726,227/sec

*******************************************************************************
10 threads sharing same logger, 1,000,000 iterations
*******************************************************************************
basic_mt...		    Elapsed: 0.882268	1,133,441/sec
rotating_mt...              Elapsed: 0.875515	1,142,184/sec
daily_mt...		    Elapsed: 0.879573	1,136,915/sec
null_mt...		    Elapsed: 0.220114	4,543,105/sec
``` 
#### Asynchronous mode
```
*******************************************************************************
10 threads sharing same logger, 1,000,000 iterations 
*******************************************************************************
async...		Elapsed: 0.429088	2,330,524/sec
async...		Elapsed: 0.411501	2,430,126/sec
async...		Elapsed: 0.428979	2,331,116/sec

```

## Usage samples

```c++
#include "spdlog/spdlog.h"
#include "spdlog/sinks/stdout_color_sinks.h"
void stdout_example()
{
    // create color multi threaded logger
    auto console = spdlog::stdout_color_mt("console");
    console->info("Welcome to spdlog!");
    console->error("Some error message with arg: {}", 1);

    auto err_logger = spdlog::stderr_color_mt("stderr");
    err_logger->error("Some error message");

    // Formatting examples
    console->warn("Easy padding in numbers like {:08d}", 12);
    console->critical("Support for int: {0:d};  hex: {0:x};  oct: {0:o}; bin: {0:b}", 42);
    console->info("Support for floats {:03.2f}", 1.23456);
    console->info("Positional args are {1} {0}..", "too", "supported");
    console->info("{:<30}", "left aligned");

    spdlog::get("console")->info("loggers can be retrieved from a global registry using the spdlog::get(logger_name)");

    // Runtime log levels
    spdlog::set_level(spdlog::level::info); // Set global log level to info
    console->debug("This message should not be displayed!");
    console->set_level(spdlog::level::trace); // Set specific logger's log level
    console->debug("This message should be displayed..");

    // Customize msg format for all loggers
    spdlog::set_pattern("[%H:%M:%S %z] [%n] [%^---%L---%$] [thread %t] %v");
    console->info("This an info message with custom format");

    // Compile time log levels
    // define SPDLOG_DEBUG_ON or SPDLOG_TRACE_ON
    SPDLOG_TRACE(console, "Enabled only #ifdef SPDLOG_TRACE_ON..{} ,{}", 1, 3.23);
    SPDLOG_DEBUG(console, "Enabled only #ifdef SPDLOG_DEBUG_ON.. {} ,{}", 1, 3.23);
}
```
---
#### Basic file logger
```c++
#include "spdlog/sinks/basic_file_sink.h"
void basic_logfile_example()
{
    try 
    {
        auto my_logger = spdlog::basic_logger_mt("basic_logger", "logs/basic-log.txt");
    }
    catch (const spdlog::spdlog_ex &ex)
    {
        std::cout << "Log init failed: " << ex.what() << std::endl;
        return 1;
    }
}
```
---
#### Rotating files
```c++
#include "spdlog/sinks/rotating_file_sink.h"
void rotating_example()
{
    // Create a file rotating logger with 5mb size max and 3 rotated files
    auto rotating_logger = spdlog::rotating_logger_mt("some_logger_name", "logs/rotating.txt", 1048576 * 5, 3);
}
```

---
#### Daily files
```c++

#include "spdlog/sinks/daily_file_sink.h"
void daily_example()
{
    // Create a daily logger - a new file is created every day on 2:30am
    auto daily_logger = spdlog::daily_logger_mt("daily_logger", "logs/daily.txt", 2, 30);
}

```

---
#### Periodic flush
```c++
// periodically flush all *registered* loggers every 3 seconds:
// warning: only use if all your loggers are thread safe!
spdlog::flush_every(std::chrono::seconds(3));

```


---
#### Logger with multi sinks - each with different format and log level
```c++

// create logger with 2 targets with different log levels and formats.
// the console will show only warnings or errors, while the file will log all.
void multi_sink_example()
{
    auto console_sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt>();
    console_sink->set_level(spdlog::level::warn);
    console_sink->set_pattern("[multi_sink_example] [%^%l%$] %v");

    auto file_sink = std::make_shared<spdlog::sinks::basic_file_sink_mt>("logs/multisink.txt", true);
    file_sink->set_level(spdlog::level::trace);

    spdlog::logger logger("multi_sink", {console_sink, file_sink});
    logger.set_level(spdlog::level::debug);
    logger.warn("this should appear in both console and file");
    logger.info("this message should not appear in the console, only in the file");
}
```

---
#### Asynchronous logging
```c++
#include "spdlog/async.h"
#include "spdlog/sinks/basic_file_sink.h"
void async_example()
{
    // default thread pool settings can be modified *before* creating the async logger:
    // spdlog::init_thread_pool(8192, 1); // queue with 8k items and 1 backing thread.
    auto async_file = spdlog::basic_logger_mt<spdlog::async_factory>("async_file_logger", "logs/async_log.txt");
    // alternatively:
    // auto async_file = spdlog::create_async<spdlog::sinks::basic_file_sink_mt>("async_file_logger", "logs/async_log.txt");   
}

```

---
#### Asynchronous logger with multi sinks  
```c++
#include "spdlog/sinks/stdout_color_sinks.h"
#include "spdlog/sinks/rotating_file_sink.h"

void multi_sink_example2()
{
    spdlog::init_thread_pool(8192, 1);
    auto stdout_sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt >();
    auto rotating_sink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>("mylog.txt", 1024*1024*10, 3);
    std::vector<spdlog::sink_ptr> sinks {stdout_sink, rotating_sink};
    auto logger = std::make_shared<spdlog::async_logger>("loggername", sinks.begin(), sinks.end(), spdlog::thread_pool(), spdlog::async_overflow_policy::block);
    spdlog::register_logger(logger);
}
```
 
---
#### User defined types
```c++
// user defined types logging by implementing operator<<
#include "spdlog/fmt/ostr.h" // must be included
struct my_type
{
    int i;
    template<typename OStream>
    friend OStream &operator<<(OStream &os, const my_type &c)
    {
        return os << "[my_type i=" << c.i << "]";
    }
};

void user_defined_example()
{
    spdlog::get("console")->info("user defined type: {}", my_type{14});
}

```
---
#### Custom error handler
```c++
void err_handler_example()
{
    // can be set globally or per logger(logger->set_error_handler(..))
    spdlog::set_error_handler([](const std::string &msg) { spdlog::get("console")->error("*** LOGGER ERROR ***: {}", msg); });
    spdlog::get("console")->info("some invalid message to trigger an error {}{}{}{}", 3);
}

```
---
#### syslog 
```c++
#include "spdlog/sinks/syslog_sink.h"
void syslog_example()
{
    std::string ident = "spdlog-example";
    auto syslog_logger = spdlog::syslog_logger_mt("syslog", ident, LOG_PID);
    syslog_logger->warn("This is warning that will end up in syslog.");
}
```
---
#### Android example 
```c++
#incude "spdlog/sinks/android_sink.h"
void android_example()
{
    std::string tag = "spdlog-android";
    auto android_logger = spdlog::android_logger("android", tag);
    android_logger->critical("Use \"adb shell logcat\" to view this message.");
}
```

## Documentation
Documentation can be found in the [wiki](https://github.com/gabime/spdlog/wiki/1.-QuickStart) pages.
