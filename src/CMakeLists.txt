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
# LIVE555_ROOT Auto-Resolution
# ---------------------------------------------------------------------------
# If LIVE555_ROOT is not provided via -DLIVE555_ROOT=..., resolve path relative to source directory:
# source directory: <workspace>/plrtspservice/source
# install directory: <workspace>/installs/<platform_triple>
if(NOT LIVE555_ROOT)
    get_filename_component(WORKSPACE_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../.." ABSOLUTE)
    
    if(WIN32)
        set(LIVE555_ROOT "${WORKSPACE_ROOT}/installs/windows/x64")
    elseif(ANDROID)
        if(CMAKE_ANDROID_ARCH_ABI)
            set(TARGET_ABI ${CMAKE_ANDROID_ARCH_ABI})
        else()
            set(TARGET_ABI "arm64-v8a")
        endif()
        set(LIVE555_ROOT "${WORKSPACE_ROOT}/installs/android/${TARGET_ABI}")
    elseif(APPLE)
        set(LIVE555_ROOT "${WORKSPACE_ROOT}/installs/macos/arm64")
    else()
        set(LIVE555_ROOT "${WORKSPACE_ROOT}/installs/linux/x64")
    endif()
endif()

message(STATUS "[plrtspservice] Target Platform LIVE555_ROOT: ${LIVE555_ROOT}")

# Validate required Live555 include paths
if(NOT EXISTS "${LIVE555_ROOT}/include/liveMedia/liveMedia.hh")
    message(FATAL_ERROR "Could not locate Live555 headers inside: ${LIVE555_ROOT}/include. Please verify LIVE555_ROOT.")
endif()

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
    ${LIVE555_ROOT}/include
    ${LIVE555_ROOT}/include/liveMedia
    ${LIVE555_ROOT}/include/groupsock
    ${LIVE555_ROOT}/include/UsageEnvironment
    ${LIVE555_ROOT}/include/BasicUsageEnvironment
)

# ---------------------------------------------------------------------------
# Locate Shared Monolithic Live555 Libraries
# ---------------------------------------------------------------------------
if(WIN32)
    # 1. Locate MSVC Import Library (.lib)
    find_library(LIVE555_IMPORT_LIB
        NAMES live555 liblive555
        HINTS "${LIVE555_ROOT}/lib" "${LIVE555_ROOT}/bin"
        NO_DEFAULT_PATH
        REQUIRED
    )
    
    # 2. Locate Shared Dynamic Link Library (.dll)
    find_file(LIVE555_DLL
        NAMES live555.dll
        HINTS "${LIVE555_ROOT}/bin" "${LIVE555_ROOT}/lib"
        NO_DEFAULT_PATH
        REQUIRED
    )

    message(STATUS "[plrtspservice] Found Live555 Import Lib: ${LIVE555_IMPORT_LIB}")
    message(STATUS "[plrtspservice] Found Live555 Shared DLL: ${LIVE555_DLL}")

    # Create an Imported Target to wrap the Windows DLL/LIB pair cleanly
    add_library(Live555::Monolith SHARED IMPORTED GLOBAL)
    set_target_properties(Live555::Monolith PROPERTIES
        IMPORTED_IMPLIB "${LIVE555_IMPORT_LIB}"
        IMPORTED_LOCATION "${LIVE555_DLL}"
    )

    target_link_libraries(${PROJECT_NAME} PRIVATE
        Live555::Monolith
        ws2_32
    )

    # 3. Post-build step: Copy live555.dll to target execution directory
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${LIVE555_DLL}"
            "$<TARGET_FILE_DIR:${PROJECT_NAME}>"
        COMMENT "Copying live555.dll to output folder for runtime loading..."
    )

else()
    # macOS, Linux, Android (.so / .dylib)
    find_library(LIVE555_SHARED_LIB
        NAMES live555 liblive555
        HINTS "${LIVE555_ROOT}/lib"
        NO_CMAKE_FIND_ROOT_PATH
        NO_DEFAULT_PATH
        REQUIRED
    )

    message(STATUS "[plrtspservice] Found Live555 Shared Library: ${LIVE555_SHARED_LIB}")

    target_link_libraries(${PROJECT_NAME} PRIVATE
        ${LIVE555_SHARED_LIB}
    )

    if(ANDROID)
        target_link_libraries(${PROJECT_NAME} PRIVATE log android)
    endif()
endif()

# ---------------------------------------------------------------------------
# Compile Definitions & Flags
# ---------------------------------------------------------------------------
target_compile_definitions(${PROJECT_NAME} PRIVATE
    PLRTSPSERVICE_EXPORTS
    NO_OPENSSL=1
)


# Include header directories
target_include_directories(${PROJECT_NAME} PRIVATE
	${LIVE555_ROOT}/include/liveMedia
	${LIVE555_ROOT}/include/groupsock
	${LIVE555_ROOT}/include/UsageEnvironment
	${LIVE555_ROOT}/include/BasicUsageEnvironment
	${LIVE555_ROOT}/include
)

if(WIN32)
    target_compile_definitions(${PROJECT_NAME} PRIVATE
        _WINDOWS
        _USRDLL
        WIN32_LEAN_AND_MEAN
        LIVEMEDIA_API=__declspec\(dllimport\)
    )
endif()

# ---------------------------------------------------------------------------
# Symbol Visibility & RPATH Settings
# ---------------------------------------------------------------------------
if(WIN32)
    # Symbols exported via PLRTSPSERVICE_EXPORTS annotation macro
else()
    set_target_properties(${PROJECT_NAME} PROPERTIES
        CXX_VISIBILITY_PRESET default
        VISIBILITY_INLINES_HIDDEN OFF
    )
endif()

if(APPLE)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        MACOSX_RPATH TRUE
        BUILD_WITH_INSTALL_RPATH TRUE
        INSTALL_RPATH "@loader_path"
    )
elseif(UNIX AND NOT ANDROID)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        BUILD_WITH_INSTALL_RPATH TRUE
        INSTALL_RPATH "$ORIGIN"
    )
endif()

# ---------------------------------------------------------------------------
# Installation Rules
# ---------------------------------------------------------------------------
# Installation path resolves automatically to <workspace>/installs/<platform_triple>
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    get_filename_component(WORKSPACE_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../.." ABSOLUTE)
    if(WIN32)
        set(CMAKE_INSTALL_PREFIX "${WORKSPACE_ROOT}/installs/windows/x64" CACHE PATH "Default Install Prefix" FORCE)
    elseif(ANDROID)
        set(CMAKE_INSTALL_PREFIX "${WORKSPACE_ROOT}/installs/android/${TARGET_ABI}" CACHE PATH "Default Install Prefix" FORCE)
    elseif(APPLE)
        set(CMAKE_INSTALL_PREFIX "${WORKSPACE_ROOT}/installs/macos/arm64" CACHE PATH "Default Install Prefix" FORCE)
    else()
        set(CMAKE_INSTALL_PREFIX "${WORKSPACE_ROOT}/installs/linux/x64" CACHE PATH "Default Install Prefix" FORCE)
    endif()
endif()

install(TARGETS ${PROJECT_NAME}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(FILES RTSPService.hh LoggingUsageEnvironment.hh
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)