# C++ Project Structure with Interfaces, Dependency Injection, and CMake

## Overview

We want a project structure where: - The **orchestrator** (top-level
main) coordinates everything. - The **processor** (business logic)
performs work, depending only on an interface for extraction. - The
**extractor** (infrastructure) interacts with the database. -
**Dependency Injection** is used so that the processor is testable
without a database.

The goal: - Processor should only depend on an **interface**, not a
concrete DB. - Orchestrator should wire things together but not drag in
DB infrastructure headers or libs. - Only the `db_extractor` module
links against DB infrastructure.

------------------------------------------------------------------------

## Interfaces

**IExtractor.h**

``` cpp
#pragma once
#include <string>
#include <vector>

class IExtractor {
public:
    virtual ~IExtractor() = default;
    virtual std::vector<std::string> fetchRecords(const std::string& query) = 0;
};
```

CMake:

``` cmake
add_library(extractor_interface INTERFACE)
target_include_directories(extractor_interface INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

------------------------------------------------------------------------

## Processor

**Processor.h**

``` cpp
#pragma once
#include "IExtractor.h"

class Processor {
public:
    Processor(IExtractor& extractor) : extractor_(extractor) {}

    void process(const std::string& query) {
        auto records = extractor_.fetchRecords(query);
        // do work...
    }

private:
    IExtractor& extractor_;
};
```

CMake:

``` cmake
add_library(processor STATIC Processor.cpp Processor.h)
target_link_libraries(processor PUBLIC extractor_interface)
```

------------------------------------------------------------------------

## Database Extractor

**DbExtractor.h**

``` cpp
#pragma once
#include "IExtractor.h"
#include <vector>
#include <string>

class DbExtractor : public IExtractor {
public:
    std::vector<std::string> fetchRecords(const std::string& query) override;
};
```

**DbExtractor.cpp**

``` cpp
#include "DbExtractor.h"
#include <some_db_driver.h>

std::vector<std::string> DbExtractor::fetchRecords(const std::string& query) {
    // talk to DB driver
    return { "record1", "record2" };
}
```

CMake:

``` cmake
add_library(db_extractor STATIC DbExtractor.cpp DbExtractor.h)
target_link_libraries(db_extractor PRIVATE extractor_interface db_driver_lib)
```

------------------------------------------------------------------------

## Orchestrator

**Orchestrator.cpp**

``` cpp
#include "Processor.h"
#include "DbExtractor.h"

int main() {
    DbExtractor extractor;
    Processor processor(extractor);
    processor.process("SELECT * FROM table");
}
```

CMake:

``` cmake
add_executable(orchestrator Orchestrator.cpp)
target_link_libraries(orchestrator PRIVATE processor db_extractor)
```

------------------------------------------------------------------------

## Unit Testing

**test_processor.cpp**

``` cpp
#include "Processor.h"
#include "IExtractor.h"
#include <gtest/gtest.h>

class MockExtractor : public IExtractor {
public:
    std::vector<std::string> fetchRecords(const std::string& query) override {
        return { "mock_record" };
    }
};

TEST(ProcessorTest, UsesExtractor) {
    MockExtractor mock;
    Processor processor(mock);
    processor.process("fake query");
    // assert results...
}
```

CMake:

``` cmake
add_executable(processor_tests test_processor.cpp)
target_link_libraries(processor_tests PRIVATE processor gtest)
```

------------------------------------------------------------------------

## Project Layout

    project-root/
    âââ CMakeLists.txt
    âââ include/
    â   âââ IExtractor.h
    âââ src/
    â   âââ processor/
    â   â   âââ CMakeLists.txt
    â   â   âââ Processor.cpp
    â   â   âââ Processor.h
    â   âââ db_extractor/
    â   â   âââ CMakeLists.txt
    â   â   âââ DbExtractor.cpp
    â   â   âââ DbExtractor.h
    â   âââ orchestrator/
    â       âââ CMakeLists.txt
    â       âââ Orchestrator.cpp
    âââ tests/
        âââ CMakeLists.txt
        âââ test_processor.cpp

------------------------------------------------------------------------

## Root `CMakeLists.txt`

``` cmake
cmake_minimum_required(VERSION 3.16)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(extractor_interface INTERFACE)
target_include_directories(extractor_interface INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)

add_subdirectory(src/processor)
add_subdirectory(src/db_extractor)
add_subdirectory(src/orchestrator)
add_subdirectory(tests)
```

------------------------------------------------------------------------

## Handling Orchestrator Dependency Exposure

### Problem

The orchestrator has to provide a concrete extractor, so it normally
includes `DbExtractor.h`. This leaks implementation details.

### Solution: Factory Wrapper

**extractor_factory.h**

``` cpp
#pragma once
#include "IExtractor.h"
#include <memory>

std::unique_ptr<IExtractor> makeDbExtractor();
```

**extractor_factory.cpp**

``` cpp
#include "DbExtractor.h"

std::unique_ptr<IExtractor> makeDbExtractor() {
    return std::make_unique<DbExtractor>();
}
```

**Orchestrator.cpp**

``` cpp
#include "Processor.h"
#include "extractor_factory.h"

int main() {
    auto extractor = makeDbExtractor();
    Processor processor(*extractor);
    processor.process("...");
}
```

This way the orchestrator only includes the factory and `IExtractor.h`,
not `DbExtractor.h`.

------------------------------------------------------------------------

## Library Types Recap

-   **INTERFACE**: for header-only libraries like `extractor_interface`.
-   **STATIC**: recommended for `processor` and `db_extractor`.
-   **SHARED**: only if you need runtime-loaded shared libraries.
-   **OBJECT**: advanced use for sharing compiled objects across
    targets.

Suggested setup: - `extractor_interface`: INTERFACE - `processor`:
STATIC - `db_extractor`: STATIC - `orchestrator`: EXECUTABLE - `tests`:
EXECUTABLE


