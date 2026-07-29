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
add_library(${PROJECT_NAME} SHARED
    dllmain.cpp
    LoggingUsageEnvironment.cpp
    LoggingUsageEnvironment.hh
    RTSPService.hh
)

target_include_directories(${PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
)

# ---------------------------------------------------------------------------
# Compile Definitions
# ---------------------------------------------------------------------------
target_compile_definitions(${PROJECT_NAME} PRIVATE
    # Macro for exporting symbols from your own library
    PLRTSPSERVICE_EXPORTS
    
    # live555 defines
    NO_OPENSSL=1
)

# Standard platform definitions
if(WIN32)
    target_compile_definitions(${PROJECT_NAME} PRIVATE
        _WINDOWS
        _USRDLL
        WIN32_LEAN_AND_MEAN
    )
endif()

# ---------------------------------------------------------------------------
# Symbol Visibility for P/Invoke
# ---------------------------------------------------------------------------
if(WIN32)
    # On Windows, symbols marked with __declspec(dllexport) via PLRTSPSERVICE_EXPORTS are exported
else()
    # On macOS/Linux, make all symbols default-visible for dynamic loading/PInvoke
    set_target_properties(${PROJECT_NAME} PROPERTIES
        CXX_VISIBILITY_PRESET default
        VISIBILITY_INLINES_HIDDEN OFF
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

if(WIN32)
    target_link_libraries(${PROJECT_NAME} PRIVATE ws2_32)
endif()

# ---------------------------------------------------------------------------
# macOS Universal Binary / P/Invoke Configuration
# ---------------------------------------------------------------------------
if(APPLE)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        MACOSX_RPATH TRUE
        BUILD_WITH_INSTALL_RPATH TRUE
        INSTALL_RPATH "@loader_path"
    )
endif()