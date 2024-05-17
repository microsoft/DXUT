# Copyright (c) Microsoft Corporation.
# Licensed under the MIT License.

cmake_minimum_required (VERSION 3.20)

set(DXUT_VERSION 11.31)

project (DXUT
  VERSION ${DXUT_VERSION}
  DESCRIPTION "DXUT for DirectX 11"
  HOMEPAGE_URL "http://go.microsoft.com/fwlink/?LinkId=320437"
  LANGUAGES CXX)

# https://devblogs.microsoft.com/cppblog/spectre-mitigations-in-msvc/
option(ENABLE_SPECTRE_MITIGATION "Build using /Qspectre for MSVC" OFF)

option(DISABLE_MSVC_ITERATOR_DEBUGGING "Disable iterator debugging in Debug configurations with the MSVC CRT" OFF)

option(ENABLE_CODE_ANALYSIS "Use Static Code Analysis on build" OFF)

option(DIRECTXTK_INTEGRATION "Support mixing with DirectX Tool Kit" OFF)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")

if(DEFINED VCPKG_TARGET_ARCHITECTURE)
    set(DIRECTX_ARCH ${VCPKG_TARGET_ARCHITECTURE})
elseif(CMAKE_GENERATOR_PLATFORM MATCHES "^[Ww][Ii][Nn]32$")
    set(DIRECTX_ARCH x86)
elseif(CMAKE_GENERATOR_PLATFORM MATCHES "^[Xx]64$")
    set(DIRECTX_ARCH x64)
elseif(CMAKE_GENERATOR_PLATFORM MATCHES "^[Aa][Rr][Mm]$")
    set(DIRECTX_ARCH arm)
elseif(CMAKE_GENERATOR_PLATFORM MATCHES "^[Aa][Rr][Mm]64$")
    set(DIRECTX_ARCH arm64)
elseif(CMAKE_VS_PLATFORM_NAME_DEFAULT MATCHES "^[Ww][Ii][Nn]32$")
    set(DIRECTX_ARCH x86)
elseif(CMAKE_VS_PLATFORM_NAME_DEFAULT MATCHES "^[Xx]64$")
    set(DIRECTX_ARCH x64)
elseif(CMAKE_VS_PLATFORM_NAME_DEFAULT MATCHES "^[Aa][Rr][Mm]$")
    set(DIRECTX_ARCH arm)
elseif(CMAKE_VS_PLATFORM_NAME_DEFAULT MATCHES "^[Aa][Rr][Mm]64$")
    set(DIRECTX_ARCH arm64)
endif()

include(GNUInstallDirs)

#--- Core Library
set(CORE_LIBRARY_HEADERS
    Core/dxerr.h
    Core/DXUT.h
    Core/DXUTDevice11.h
    Core/DXUTmisc.h
)

set(CORE_LIBRARY_SOURCES
    Core/dxerr.cpp
    Core/DXUT.cpp
    Core/DXUTDevice11.cpp
    Core/DXUTmisc.cpp
)

if(NOT DIRECTXTK_INTEGRATION)
    list(APPEND CORE_LIBRARY_HEADERS
      Core/DDSTextureLoader.h
      Core/ScreenGrab.h
      Core/WICTextureLoader.h
    )
    list(APPEND CORE_LIBRARY_SOURCES
      Core/DDSTextureLoader.cpp
      Core/ScreenGrab.cpp
      Core/WICTextureLoader.cpp
    )
endif()

add_library(${PROJECT_NAME} STATIC ${CORE_LIBRARY_SOURCES} ${CORE_LIBRARY_HEADERS})

#--- Optional Library
set(OPT_LIBRARY_HEADERS
    Optional/DXUTcamera.h
    Optional/DXUTgui.h
    Optional/DXUTLockFreePipe.h
    Optional/DXUTres.h
    Optional/DXUTsettingsdlg.h
    Optional/SDKmesh.h
    Optional/SDKmisc.h
)

set(OPT_LIBRARY_SOURCES
    Optional/DXUTcamera.cpp
    Optional/DXUTgui.cpp
    Optional/DXUTres.cpp
    Optional/DXUTsettingsdlg.cpp
    Optional/SDKmesh.cpp
    Optional/SDKmisc.cpp
)

if(NOT MINGW)
    set(OPT_LIBRARY_HEADERS
        ${OPT_LIBRARY_HEADERS}
        Optional/DXUTguiIME.h
        Optional/ImeUi.h
    )

    set(OPT_LIBRARY_SOURCES
        ${OPT_LIBRARY_SOURCES}
        Optional/DXUTguiIME.cpp
        Optional/ImeUi.cpp
    )
endif()

add_library(${PROJECT_NAME}Opt STATIC ${OPT_LIBRARY_SOURCES} ${OPT_CORE_LIBRARY_HEADERS})

if(NOT MINGW)
    target_precompile_headers(${PROJECT_NAME} PRIVATE Core/DXUT.h)
    target_precompile_headers(${PROJECT_NAME}Opt PRIVATE Core/DXUT.h)
endif()

source_group(DXUT REGULAR_EXPRESSION Core/*.*)
source_group(DXUTOpt REGULAR_EXPRESSION Optional/*.*)

target_include_directories(${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Core>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}>)

target_include_directories(${PROJECT_NAME}Opt PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Optional>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}>
  PRIVATE Core/)

if(DIRECTXTK_INTEGRATION)
  target_compile_definitions(${PROJECT_NAME} PRIVATE USE_DIRECTXTK)
  target_compile_definitions(${PROJECT_NAME}Opt PRIVATE USE_DIRECTXTK)

  find_package(directxtk REQUIRED)
  target_link_libraries(${PROJECT_NAME} PRIVATE Microsoft::DirectXTK)
  target_link_libraries(${PROJECT_NAME}Opt PRIVATE Microsoft::DirectXTK)
endif()

target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_11)
target_compile_features(${PROJECT_NAME}Opt PUBLIC cxx_std_11)

if(MINGW)
    find_package(directxmath CONFIG REQUIRED)
else()
    find_package(directxmath CONFIG QUIET)
endif()

if(directxmath_FOUND)
    message(STATUS "Using DirectXMath package")
    target_link_libraries(${PROJECT_NAME} PUBLIC Microsoft::DirectXMath)
    target_link_libraries(${PROJECT_NAME}Opt PUBLIC Microsoft::DirectXMath)
endif()

#--- Package
include(CMakePackageConfigHelpers)

string(TOLOWER ${PROJECT_NAME} PACKAGE_NAME)

write_basic_package_version_file(
  ${PACKAGE_NAME}-config-version.cmake
  VERSION ${DXUT_VERSION}
  COMPATIBILITY AnyNewerVersion)

install(TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}-targets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

install(TARGETS ${PROJECT_NAME}Opt
  EXPORT ${PROJECT_NAME}Opt-targets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/build/${PROJECT_NAME}-config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}-config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/${PACKAGE_NAME})

install(EXPORT ${PROJECT_NAME}-targets
  FILE ${PROJECT_NAME}-targets.cmake
  NAMESPACE Microsoft::
  DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/${PACKAGE_NAME})

install(EXPORT ${PROJECT_NAME}Opt-targets
  FILE ${PROJECT_NAME}Opt-targets.cmake
  NAMESPACE Microsoft::
  DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/${PACKAGE_NAME})

install(FILES ${CORE_LIBRARY_HEADERS} ${OPT_LIBRARY_HEADERS}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME})

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}-config.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}-config-version.cmake
  DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/${PACKAGE_NAME})

if(MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE /W4 /GR- /fp:fast "$<$<NOT:$<CONFIG:DEBUG>>:/guard:cf>")
    target_compile_options(${PROJECT_NAME}Opt PRIVATE /W4 /GR- /fp:fast "$<$<NOT:$<CONFIG:DEBUG>>:/guard:cf>")

    if(ENABLE_SPECTRE_MITIGATION
       AND (MSVC_VERSION GREATER_EQUAL 1913)
       AND (NOT (CMAKE_CXX_COMPILER_ID MATCHES "Clang")))
      message(STATUS "Building Spectre-mitigated libraries.")
      target_compile_options(${PROJECT_NAME} PRIVATE "/Qspectre")
      target_compile_options(${PROJECT_NAME}Opt PRIVATE "/Qspectre")
    endif()

    if((MSVC_VERSION GREATER_EQUAL 1924)
       AND ((NOT (CMAKE_CXX_COMPILER_ID MATCHES "Clang")) OR (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 16.0)))
      target_compile_options(${PROJECT_NAME} PRIVATE /ZH:SHA_256)
      target_compile_options(${PROJECT_NAME}Opt PRIVATE /ZH:SHA_256)
    endif()

    if((MSVC_VERSION GREATER_EQUAL 1928)
       AND (CMAKE_SIZEOF_VOID_P EQUAL 8)
       AND ((NOT (CMAKE_CXX_COMPILER_ID MATCHES "Clang")) OR (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 13.0)))
      target_compile_options(${PROJECT_NAME} PRIVATE "$<$<NOT:$<CONFIG:DEBUG>>:/guard:ehcont>")
      target_compile_options(${PROJECT_NAME}Opt PRIVATE "$<$<NOT:$<CONFIG:DEBUG>>:/guard:ehcont>")
    endif()
else()
    target_compile_definitions(${PROJECT_NAME} PRIVATE $<IF:$<CONFIG:DEBUG>,_DEBUG,NDEBUG>)
    target_compile_definitions(${PROJECT_NAME}Opt PRIVATE $<IF:$<CONFIG:DEBUG>,_DEBUG,NDEBUG>)
endif()

if(NOT ${DIRECTX_ARCH} MATCHES "^arm")
    if(CMAKE_SIZEOF_VOID_P EQUAL 4)
        set(ARCH_SSE2 $<$<CXX_COMPILER_ID:MSVC>:/arch:SSE2> $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-msse2>)
    else()
        set(ARCH_SSE2 $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-msse2>)
    endif()

    target_compile_options(${PROJECT_NAME} PRIVATE ${ARCH_SSE2})
    target_compile_options(${PROJECT_NAME}Opt PRIVATE ${ARCH_SSE2})
endif()

if( CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
    set(WarningsLib "-Wno-deprecated-declarations" "-Wno-unused-const-variable" "-Wno-switch" "-Wno-ignored-attributes")
    target_compile_options(${PROJECT_NAME} PRIVATE ${WarningsLib})
    target_compile_options(${PROJECT_NAME}Opt PRIVATE ${WarningsLib})
elseif( CMAKE_CXX_COMPILER_ID MATCHES "MSVC" )
    target_compile_options(${PROJECT_NAME} PRIVATE /sdl /permissive- /JMC- /Zc:__cplusplus /Zc:inline)
    target_compile_options(${PROJECT_NAME}Opt PRIVATE /sdl /permissive- /JMC- /Zc:__cplusplus /Zc:inline)

    if(ENABLE_CODE_ANALYSIS)
      target_compile_options(${PROJECT_NAME} PRIVATE /analyze /WX)
      target_compile_options(${PROJECT_NAME}Opt PRIVATE /analyze)
    endif()

    if(CMAKE_INTERPROCEDURAL_OPTIMIZATION)
      message(STATUS "Building using Whole Program Optimization")
      target_compile_options(${PROJECT_NAME} PRIVATE /Gy /Gw)
      target_compile_options(${PROJECT_NAME}Opt PRIVATE /Gy /Gw)
    endif()

    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.26)
        target_compile_options(${PROJECT_NAME} PRIVATE /Zc:preprocessor /wd5104 /wd5105)
        target_compile_options(${PROJECT_NAME}Opt PRIVATE /Zc:preprocessor /wd5104 /wd5105)
    endif()

    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.28)
        target_compile_options(${PROJECT_NAME} PRIVATE /Zc:lambda)
        target_compile_options(${PROJECT_NAME}Opt PRIVATE /Zc:lambda)
    endif()

    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.34)
        target_compile_options(${PROJECT_NAME} PRIVATE /wd5262)
        target_compile_options(${PROJECT_NAME}Opt PRIVATE /wd5262)
    endif()

    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.35)
        target_compile_options(${PROJECT_NAME} PRIVATE /Zc:checkGwOdr $<$<VERSION_GREATER_EQUAL:${CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION},10.0.22000>:/Zc:templateScope>)
        target_compile_options(${PROJECT_NAME}Opt PRIVATE /Zc:checkGwOdr $<$<VERSION_GREATER_EQUAL:${CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION},10.0.22000>:/Zc:templateScope>)
    endif()
endif()

if(WIN32)
    if(${DIRECTX_ARCH} MATCHES "^arm64")
        message(STATUS "Building for Windows 10/Windows 11.")
        set(WINVER 0x0A00)
    elseif(${DIRECTX_ARCH} MATCHES "^arm")
        message(STATUS "Building for Windows 8.")
        set(WINVER 0x0602)
    else()
        message(STATUS "Building for Windows 7.")
        set(WINVER 0x0601)
        target_compile_definitions(${PROJECT_NAME} PRIVATE _WIN7_PLATFORM_UPDATE)
        target_compile_definitions(${PROJECT_NAME}Opt PRIVATE _WIN7_PLATFORM_UPDATE)
    endif()

    target_compile_definitions(${PROJECT_NAME} PRIVATE _UNICODE UNICODE _WIN32_WINNT=${WINVER})
    target_compile_definitions(${PROJECT_NAME}Opt PRIVATE _UNICODE UNICODE _WIN32_WINNT=${WINVER})

    if(DISABLE_MSVC_ITERATOR_DEBUGGING)
      target_compile_definitions(${PROJECT_NAME} PRIVATE _ITERATOR_DEBUG_LEVEL=0)
      target_compile_definitions(${PROJECT_NAME}Opt PRIVATE _ITERATOR_DEBUG_LEVEL=0)
    endif()
endif()
