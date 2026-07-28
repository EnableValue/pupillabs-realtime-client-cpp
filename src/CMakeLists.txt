cmake_minimum_required(VERSION 3.20)

project(plrtspservice 
    VERSION 1.0.0 
    LANGUAGES CXX C
)

# ---------------------------------------------------------------------------
# Global Settings
# ---------------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Ensure symbols are hidden by default across platforms unless explicitly exported
set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_VISIBILITY_INLINES_HIDDEN YES)

# Standardize output locations for built libraries
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# ---------------------------------------------------------------------------
# Find Dependencies via vcpkg
# ---------------------------------------------------------------------------
find_package(unofficial-live555 CONFIG REQUIRED)

# ---------------------------------------------------------------------------
# Target Definition
# ---------------------------------------------------------------------------
# SHARED creates a .dll on Windows and a .dylib on macOS suitable for P/Invoke
add_library(${PROJECT_NAME} SHARED
    dllmain.cpp
    LoggingUsageEnvironment.cpp
    LoggingUsageEnvironment.hh
    RTSPService.hh
)

# Set include directories for the target
target_include_directories(${PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
)

# ---------------------------------------------------------------------------
# Compile Definitions (extracted from Visual Studio project)
# ---------------------------------------------------------------------------
target_compile_definitions(${PROJECT_NAME} PRIVATE
    # Export macro definition from MSBuild project
    PLRTSPSERVICE_EXPORTS
    # live555 defines matching your MSBuild project
    LIVEMEDIA_API=__declspec(dllimport)
    NO_OPENSSL=1
)

# Platform-specific definitions
if(WIN32)
    target_compile_definitions(${PROJECT_NAME} PRIVATE
        _WINDOWS
        _USRDLL
        WIN32_LEAN_AND_MEAN
    )
endif()

# ---------------------------------------------------------------------------
# Linking Dependencies
# ---------------------------------------------------------------------------
target_link_libraries(${PROJECT_NAME} PRIVATE
    unofficial::groupsock
    unofficial::liveMedia
    unofficial::UsageEnvironment
    unofficial::BasicUsageEnvironment
)

# Windows Sockets required by live555 on Windows
if(WIN32)
    target_link_libraries(${PROJECT_NAME} PRIVATE ws2_32)
endif()

# ---------------------------------------------------------------------------
# macOS Universal Binary / Architecture Support (Intel & ARM64)
# ---------------------------------------------------------------------------
if(APPLE)
    # Set rpath so Unity / .NET P/Invoke can load dependent dylibs relative to @loader_path
    set_target_properties(${PROJECT_NAME} PROPERTIES
        MACOSX_RPATH TRUE
        BUILD_WITH_INSTALL_RPATH TRUE
        INSTALL_RPATH "@loader_path"
    )
    
    # Note: If building a Universal Binary (x86_64;arm64) on macOS, specify:
    # -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" during your CMake configuration step.
endif()