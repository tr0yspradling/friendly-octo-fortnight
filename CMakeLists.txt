cmake_minimum_required(VERSION 3.15)
project(starterkit C)

# On macOS raylib's amalgamated `rglfw.c` includes Objective-C sources.
# Enable Objective-C language support so the file can be compiled correctly
# when configuring on that platform.
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    enable_language(OBJC)
endif()

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug)
endif()

# Common flags
set(COMMON_CFLAGS -Wall -Wextra -Wshadow -Wno-unused-parameter -Wno-switch -pedantic-errors -fcommon)

# OS specific settings
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    add_compile_definitions(LINUX)
    set(OS_LIBS m pthread dl rt X11 GL)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    add_compile_definitions(WINDOWS)
    set(OS_LIBS opengl32 gdi32 ole32 comdlg32)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    add_compile_definitions(OSX)
    # Let CMake manage the target architecture to avoid passing stray tokens such
    # as a standalone "arm64" to the compiler.  Default to the host architecture
    # but allow callers to override CMAKE_OSX_ARCHITECTURES if they need a
    # universal build.
    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.9" CACHE STRING "" FORCE)
    if(NOT DEFINED CMAKE_OSX_ARCHITECTURES OR CMAKE_OSX_ARCHITECTURES STREQUAL "")
        set(CMAKE_OSX_ARCHITECTURES "${CMAKE_HOST_SYSTEM_PROCESSOR}" CACHE STRING "" FORCE)
    endif()
    list(APPEND OS_LIBS
        "-framework CoreVideo" "-framework IOKit" "-framework Cocoa"
        "-framework GLUT" "-framework OpenGL")
endif()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

file(GLOB PROJECT_SOURCES
    src/*.c
    src/systems/*.c
    src/managers/*.c
    src/scenes/*.c
)
add_executable(${PROJECT_NAME} ${PROJECT_SOURCES})

target_include_directories(${PROJECT_NAME} PRIVATE src src/managers src/scenes)

target_compile_options(${PROJECT_NAME} PRIVATE ${COMMON_CFLAGS})
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_definitions(${PROJECT_NAME} PRIVATE DEBUG)
else()
    target_compile_options(${PROJECT_NAME} PRIVATE -Ofast -ffast-math)
    target_compile_definitions(${PROJECT_NAME} PRIVATE RELEASE NDEBUG)
endif()

# Vendor libraries
file(GLOB RAYLIB_SOURCES vendor/raylib/*.c)
add_library(raylib STATIC ${RAYLIB_SOURCES})
target_compile_options(raylib PRIVATE -w -fcommon)
target_compile_definitions(raylib PUBLIC PLATFORM_DESKTOP)
target_include_directories(raylib PUBLIC vendor/raylib vendor/raylib/external/glfw/include)

# rglfw.c pulls in Objective-C sources on macOS; build it using the
# Objective-C compiler so the Cocoa files are handled correctly.
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set_source_files_properties(vendor/raylib/rglfw.c PROPERTIES LANGUAGE OBJC)
endif()

file(GLOB FLECS_SOURCES vendor/flecs/*.c)
add_library(flecs STATIC ${FLECS_SOURCES})
target_include_directories(flecs PUBLIC vendor/flecs)

file(GLOB CJSON_SOURCES vendor/cJSON/*.c)
add_library(cjson STATIC ${CJSON_SOURCES})
target_include_directories(cjson PUBLIC vendor/cJSON)

file(GLOB CHIPMUNK_SOURCES vendor/Chipmunk2D/*.c)
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    list(FILTER CHIPMUNK_SOURCES EXCLUDE REGEX "cpHastySpace.c")
endif()
add_library(chipmunk STATIC ${CHIPMUNK_SOURCES})
target_include_directories(chipmunk PUBLIC vendor/Chipmunk2D vendor/Chipmunk2D/chipmunk)

file(GLOB TINYFILEDIALOGS_SOURCES vendor/tinyfiledialogs/*.c)
add_library(tinyfiledialogs STATIC ${TINYFILEDIALOGS_SOURCES})
target_include_directories(tinyfiledialogs PUBLIC vendor/tinyfiledialogs)

file(GLOB NUKLEAR_SOURCES vendor/Nuklear/*.c)
add_library(nuklear STATIC ${NUKLEAR_SOURCES})
target_compile_definitions(nuklear PRIVATE STBRP_STATIC)
target_include_directories(nuklear PUBLIC vendor/Nuklear)

foreach(lib flecs cjson chipmunk tinyfiledialogs nuklear)
    target_compile_options(${lib} PRIVATE ${COMMON_CFLAGS})
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        target_compile_definitions(${lib} PRIVATE DEBUG)
    else()
        target_compile_options(${lib} PRIVATE -Ofast -ffast-math)
        target_compile_definitions(${lib} PRIVATE RELEASE NDEBUG)
    endif()
endforeach()

# Link everything
target_link_libraries(${PROJECT_NAME}
    raylib
    flecs
    cjson
    chipmunk
    tinyfiledialogs
    nuklear
    ${OS_LIBS}
)
