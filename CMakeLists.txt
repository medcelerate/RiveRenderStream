# RiveRenderStream: standalone Disguise RenderStream workload that renders .riv
# content with bidirectional texture support (Windows / D3D11 only).
#
# Usage:
#   # 1. Build the Rive runtime static libs once (slow, not driven by CMake):
#   #    scripts\build_rive.bat release
#   # 2. Configure + build:
#   #    cmake -B build -G "Visual Studio 17 2022" -A x64
#   #    cmake --build build --config Release
#
# Output: build\Release\RiveRenderStream.exe

cmake_minimum_required(VERSION 3.20)
project(RiveRenderStream LANGUAGES C CXX)

if(NOT WIN32)
    message(FATAL_ERROR
        "RiveRenderStream is Windows-only (Disguise RenderStream + D3D11).")
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# -----------------------------------------------------------------------------
# Rive runtime - prebuilt static libs produced by scripts/build_rive.bat, built
# --with_rive_canvas --with_rive_scripting so GPU Canvas + Luau are available.
# We glob the whole set rather than name each lib, since Rive's premake config
# shifts the exact list (librive, rive_pls_renderer, decoders, harfbuzz,
# sheenbidi, yoga, libpng/jpeg/webp, zlib, miniaudio, luau_vm, ...).
# -----------------------------------------------------------------------------
set(RIVE_DIR "${CMAKE_SOURCE_DIR}/third_party/rive-runtime")
file(GLOB RIVE_LIBS "${RIVE_DIR}/renderer/out/release/*.lib")

if(NOT RIVE_LIBS AND NOT CMAKE_CONFIGURATION_TYPES)
    message(WARNING
        "No Rive static libs found under ${RIVE_DIR}/renderer/out/release.\n"
        "Run scripts\\build_rive.bat release first.")
endif()

add_executable(RiveRenderStream src/Main.cpp)

target_include_directories(RiveRenderStream PRIVATE
    "${CMAKE_SOURCE_DIR}/src"
    "${CMAKE_SOURCE_DIR}/include"
    "${RIVE_DIR}/include"
    "${RIVE_DIR}/renderer/include"
)

target_link_libraries(RiveRenderStream PRIVATE
    ${RIVE_LIBS}
    d3d11
    dxgi
    dxguid
    d3dcompiler
    Shlwapi
)

target_compile_definitions(RiveRenderStream PRIVATE
    WIN32_LEAN_AND_MEAN
    NOMINMAX
    _CRT_SECURE_NO_WARNINGS
    # The runtime libs are built --with_rive_canvas; render_context.hpp gates a
    # data member (and the makeRenderCanvas API we call) on RIVE_CANVAS, so this
    # TU must match the libs' layout. RIVE_ORE follows the canvas flag.
    RIVE_CANVAS
    RIVE_ORE
)

if(MSVC)
    # Match the Rive libs: RTTI off (they use an internal "lite rtti"), static CRT.
    target_compile_options(RiveRenderStream PRIVATE /GR-)
    set_property(TARGET RiveRenderStream PROPERTY
        MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()
