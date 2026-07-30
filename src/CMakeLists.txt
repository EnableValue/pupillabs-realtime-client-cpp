cmake_minimum_required(VERSION 3.20)

project(plrtspservice 
    VERSION 1.0.0 
    LANGUAGES CXX C
)

# Standard GNU install paths (defines CMAKE_INSTALL_BINDIR, CMAKE_INSTALL_LIBDIR, CMAKE_INSTALL_INCLUDEDIR)
include(GNUInstallDirs)

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
# Find Dependencies (Windows Monolith vs Android/Linux Static Archives)
# ---------------------------------------------------------------------------

if(WIN32)
    # --- Windows Monolithic Package Setup ---
    list(APPEND CMAKE_PREFIX_PATH 
        "C:/dev/libs/live555/debug"
        "C:/dev/libs/live555/release"
    )
    
    find_package(live555 CONFIG REQUIRED)
    
    target_link_libraries(${PROJECT_NAME} PRIVATE
        live555::live555
        ws2_32
    )

else()
    # --- Android / Linux Direct Search Setup ---
    set(LIVE555_ROOT "$ENV{HOME}/libs/live555" CACHE PATH "Path to live555 installation root")

    # Locate the 4 separate static libraries
    find_library(LIVE_MEDIA_LIB liveMedia HINTS ${LIVE555_ROOT}/lib REQUIRED)
    find_library(GROUP_SOCK_LIB groupsock HINTS ${LIVE555_ROOT}/lib REQUIRED)
    find_library(USAGE_ENV_LIB UsageEnvironment HINTS ${LIVE555_ROOT}/lib REQUIRED)
    find_library(BASIC_USAGE_ENV_LIB BasicUsageEnvironment HINTS ${LIVE555_ROOT}/lib REQUIRED)

    # Include header directories
    target_include_directories(${PROJECT_NAME} PRIVATE
        ${LIVE555_ROOT}/include/liveMedia
        ${LIVE555_ROOT}/include/groupsock
        ${LIVE555_ROOT}/include/UsageEnvironment
        ${LIVE555_ROOT}/include/BasicUsageEnvironment
    )

    # Link static archives and Android system libraries
    target_link_libraries(${PROJECT_NAME} PRIVATE
        ${LIVE_MEDIA_LIB}
        ${GROUP_SOCK_LIB}
        ${BASIC_USAGE_ENV_LIB}
        ${USAGE_ENV_LIB}
    )

    if(ANDROID)
        target_link_libraries(${PROJECT_NAME} PRIVATE log android)
    endif()
endif()

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
# Symbol Visibility for P/Invoke / JNI
# ---------------------------------------------------------------------------
if(WIN32)
    # On Windows, symbols marked with __declspec(dllexport) via PLRTSPSERVICE_EXPORTS are exported
else()
    # On Android/macOS/Linux, make all symbols default-visible for dynamic loading/PInvoke
    set_target_properties(${PROJECT_NAME} PROPERTIES
        CXX_VISIBILITY_PRESET default
        VISIBILITY_INLINES_HIDDEN OFF
    )
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

# ---------------------------------------------------------------------------
# Installation Rules
# ---------------------------------------------------------------------------
install(TARGETS ${PROJECT_NAME}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}  # Installs .dll -> bin/
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}  # Installs .so on Linux/Android -> lib/
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}  # Installs .lib -> lib/
)

# Optionally copy header files required by users of plrtspservice
install(FILES RTSPService.hh LoggingUsageEnvironment.hh
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}      # Installs .hh -> include/
)