# This file is part of the FidelityFX SDK.
#
# Copyright � 2023 Advanced Micro Devices, Inc.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files(the �Software�), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and /or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions :
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED �AS IS�, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.

cmake_minimum_required(VERSION 3.17)

# Set configuration types based on what we are building for
if (NOT FFX_API_CUSTOM)
	message(STATUS "Building FidelityFX SDK with DX11 & DX12 & VK backends")
	set(CMAKE_CONFIGURATION_TYPES Debug Release RelWithDebInfo)
	set(CMAKE_DEBUG_POSTFIX d)
	set_property(GLOBAL PROPERTY USE_FOLDERS ON)
	
	# Get warnings for everything
if (CMAKE_COMPILER_IS_GNUCC)
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} -Wall")
endif()
if (MSVC)
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} /W3")
endif()
    option (FFX_API_DX11 "Build FidelityFX DX11 backend" ON)
	option (FFX_API_DX12 "Build FidelityFX DX12 backend" ON)
	option (FFX_API_VK "Build FidelityFX Vulkan backend" ON)
endif()

# Set platform
# MATCHES (not STREQUAL): a cross-compile toolchain file can append its own
# flags (e.g. /winsysroot:...) after /machine:x64, which would otherwise
# fail an exact-equality check without changing the actual target arch.
if(CMAKE_GENERATOR_PLATFORM STREQUAL "x64" OR CMAKE_EXE_LINKER_FLAGS MATCHES "/machine:x64")
    set(FFX_PLATFORM_NAME x64)
elseif(CMAKE_GENERATOR_PLATFORM STREQUAL "Win32" OR CMAKE_EXE_LINKER_FLAGS MATCHES "/machine:X86")
    set(FFX_PLATFORM_NAME x86)
else()
    message(FATAL_ERROR "Unsupported target platform - only supporting x64 and Win32 currently")
endif()

# Pre-compile shaders
set(FFX_AUTO_COMPILE_SHADERS ON CACHE BOOL "Compile shaders automatically as a prebuild step.")

if(CMAKE_GENERATOR STREQUAL "Ninja")
    set(USE_DEPFILE TRUE)
else()
    set(USE_DEPFILE FALSE)
endif()

# Set Visual Studio version
if(CMAKE_GENERATOR STREQUAL "Visual Studio 14 2015")
    set(FFX_VS_VERSION 2015)
elseif(CMAKE_GENERATOR STREQUAL "Visual Studio 15 2017")
	set(FFX_VS_VERSION 2017)
elseif(CMAKE_GENERATOR STREQUAL "Visual Studio 16 2019")
    set(FFX_VS_VERSION 2019)
elseif(CMAKE_GENERATOR STREQUAL "Visual Studio 17 2022")
	set(FFX_VS_VERSION 2022)
endif()

# Set configuration types based on what we are building for
if (NOT FFX_API_CUSTOM)

	# Embed PDBs in the debug versions of the libs
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /Z7")

	# Write both debug and release versions of the static libs to the /bin folder as they are uniquely named
	set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG   ${CMAKE_HOME_DIRECTORY}/bin/ffx_sdk/)
	set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${CMAKE_HOME_DIRECTORY}/bin/ffx_sdk/)
	set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELWITHDEBINFO ${CMAKE_HOME_DIRECTORY}/bin/ffx_sdk/)

else()

	# Ouput the static libs to the /bin folder as they are uniquely named
	set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${BIN_OUTPUT}/ffx_sdk)
	foreach( OUTPUTCONFIG ${CMAKE_CONFIGURATION_TYPES} )
		string( TOUPPER ${OUTPUTCONFIG} OUTPUTCONFIG )
		set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${BIN_OUTPUT}/ffx_sdk/ )
	endforeach( OUTPUTCONFIG CMAKE_CONFIGURATION_TYPES )

endif()

# Add requred compile definitions
add_compile_definitions(_UNICODE)
add_compile_definitions(UNICODE)
add_compile_definitions(_DISABLE_CONSTEXPR_MUTEX_CONSTRUCTOR) # Fixes crashes with older redists

if(FFX_VS_VERSION STREQUAL 2015 OR FFX_VS_VERSION STREQUAL 2017)
    message(NOTICE "Forcing the SDK path for VS 2015 and VS 2017")
    set(CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION "10.0.18362.0")
endif()

if (FFX_FSR3)
	option(FFX_FSR3UPSCALER "Build FFX FSR3 Upscaler API" ON)
	option(FFX_FI "Build FFX Frame Interpolation API" ON)
	option(FFX_OF "Build FFX Optical Flow API" ON)
endif()

# Setup common variables
set(FFX_SC_EXECUTABLE ${CMAKE_CURRENT_SOURCE_DIR}/tools/binary_store/FidelityFX_SC.exe)

# A Linux/macOS host cross-compiling to the Windows PE/MSVC-ABI target (see
# open-shaders' Linux-ClangCL preset) can't run this bundled Windows PE tool
# directly; wrap it to run under Wine. Every CMakeShaders*.txt file across
# all backends invokes ${FFX_SC_EXECUTABLE} directly as COMMAND, so patching
# this single definition covers all of them.
if(CMAKE_HOST_UNIX)
    find_program(FFX_SC_WINE_EXECUTABLE NAMES wine)
    if(NOT FFX_SC_WINE_EXECUTABLE)
        message(FATAL_ERROR "FidelityFX-SDK: cross-compiling from a Linux/macOS host needs `wine` on PATH to run the bundled FidelityFX_SC.exe shader compiler.")
    endif()

    # Wine ships its own built-in d3dcompiler_47.dll (a partial
    # reimplementation, not Microsoft's real compiler), which
    # FidelityFX_SC.exe's `-compiler=fxc` mode resolves by default via
    # ordinary DLL search order. That builtin's HLSL parser has real bugs
    # (observed: rejects `globallycoherent`, some switch/case patterns, and
    # -- distinctly from a genuine upstream bug -- misparses valid syntax
    # that a real d3dcompiler_47.dll accepts fine), which surfaces as
    # seemingly-random "ffx_sc failed: No shader permutations generated"
    # errors that don't reproduce on native Windows. Explicitly pointing
    # `-d3ddll=` at a real d3dcompiler_47.dll (same source already trusted
    # by the DirectXTK/DirectXTex overlay ports' fxc2 workaround) routes
    # around Wine's builtin entirely; verified this alone fixes real
    # FSR1/FSR2/FSR3Upscaler shaders that failed under the builtin DLL.
    set(FFX_SC_D3DCOMPILER_DLL "${CMAKE_CURRENT_BINARY_DIR}/d3dcompiler_47.dll")
    if(NOT EXISTS "${FFX_SC_D3DCOMPILER_DLL}")
        file(
            DOWNLOAD "https://raw.githubusercontent.com/WasabiIceCream/fxc2/v1.0.0/d3dcompiler_47.dll"
            "${FFX_SC_D3DCOMPILER_DLL}"
            EXPECTED_HASH SHA512=ad8c7ce7f6f353da5e2cf816e1a69f1ec14011612e8041e4f9bb6ebed3e0fa4e4ebc069155a0c66e23811467012c201893b9b3b7a947d089ce2c749d5e8910c6
        )
    endif()
    find_program(FFX_SC_WINEPATH_EXECUTABLE NAMES winepath)
    if(NOT FFX_SC_WINEPATH_EXECUTABLE)
        message(FATAL_ERROR "FidelityFX-SDK: `winepath` (ships alongside `wine`) not found on PATH.")
    endif()
    execute_process(
        COMMAND "${FFX_SC_WINEPATH_EXECUTABLE}" -w "${FFX_SC_D3DCOMPILER_DLL}"
        OUTPUT_VARIABLE FFX_SC_D3DCOMPILER_DLL_WINPATH
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )

    # This wrapper is the COMMAND executable for every shader-generation
    # custom command in every backend (hundreds of them, each a real Wine
    # invocation). Unconditional file(WRITE ...) here would bump its mtime
    # on every single reconfigure, which Ninja's dependency tracking treats
    # as "this executable changed" -- invalidating and re-running every one
    # of those custom commands (multiple real minutes of Wine invocations)
    # even for a reconfigure that touched nothing shader-related. Guard it
    # the same way the d3dcompiler_47.dll download above already is.
    set(FFX_SC_WINE_WRAPPER "${CMAKE_CURRENT_BINARY_DIR}/FidelityFX_SC_wine.sh")
    if(NOT EXISTS "${FFX_SC_WINE_WRAPPER}")
        file(
            WRITE "${FFX_SC_WINE_WRAPPER}"
            "#!/bin/sh\nexec \"${FFX_SC_WINE_EXECUTABLE}\" \"${FFX_SC_EXECUTABLE}\" \"-d3ddll=${FFX_SC_D3DCOMPILER_DLL_WINPATH}\" \"$@\"\n"
        )
        file(
            CHMOD "${FFX_SC_WINE_WRAPPER}"
            PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
        )
    endif()
    set(FFX_SC_EXECUTABLE "${FFX_SC_WINE_WRAPPER}")
endif()
set(FFX_INCLUDE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/include)
set(FFX_SHARED_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src/shared)
set(FFX_HOST_PATH ${CMAKE_CURRENT_SOURCE_DIR}/include/FidelityFX/host)
set(FFX_GPU_PATH ${CMAKE_CURRENT_SOURCE_DIR}/include/FidelityFX/gpu)
set(FFX_COMPONENTS_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src/components)
set(FFX_HOST_BACKENDS_PATH ${FFX_HOST_PATH}/backends)
set(FFX_BACKENDS_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src/backends)
set(FFX_SRC_BACKENDS_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src/backends)

# Promote common variables up so the common code can also access it if needed
if (FFX_API_CUSTOM)

	set(FFX_PLATFORM_NAME ${FFX_PLATFORM_NAME} PARENT_SCOPE)
	set(FFX_VS_VERSION ${FFX_VS_VERSION} PARENT_SCOPE)
	set(FFX_SC_EXECUTABLE ${FFX_SC_EXECUTABLE} PARENT_SCOPE)
	set(FFX_INCLUDE_PATH ${FFX_INCLUDE_PATH} PARENT_SCOPE)
	set(FFX_SHARED_PATH ${FFX_SHARED_PATH} PARENT_SCOPE)
	set(FFX_HOST_PATH ${FFX_HOST_PATH} PARENT_SCOPE)
	set(FFX_GPU_PATH ${FFX_GPU_PATH} PARENT_SCOPE)
	set(FFX_COMPONENTS_PATH ${FFX_COMPONENTS_PATH} PARENT_SCOPE)
	set(FFX_HOST_BACKENDS_PATH ${FFX_HOST_BACKENDS_PATH} PARENT_SCOPE)
	set(FFX_SRC_BACKENDS_PATH ${FFX_SRC_BACKENDS_PATH} PARENT_SCOPE)

endif()

# ------------------------------------------------------
# Define common source groupings used by all effects

# Set shared and global public sources
file(GLOB FFX_SHARED_SOURCES
    "${FFX_SHARED_PATH}/*.cpp"
    "${FFX_SHARED_PATH}/*.h")

set(FFX_PUBLIC_SOURCES
    "${FFX_HOST_PATH}/ffx_assert.h"
    "${FFX_HOST_PATH}/ffx_error.h"
	"${FFX_HOST_PATH}/ffx_fx.h"
	"${FFX_HOST_PATH}/ffx_interface.h"
    "${FFX_HOST_PATH}/ffx_types.h"
    "${FFX_HOST_PATH}/ffx_util.h")

# Init shader sources (for easy viewing)
set(FFX_PUBLIC_SHADER_SOURCES
    "${FFX_GPU_PATH}/ffx_common_types.h"
    "${FFX_GPU_PATH}/ffx_core.h"
    "${FFX_GPU_PATH}/ffx_core_cpu.h"
    "${FFX_GPU_PATH}/ffx_core_glsl.h"
    "${FFX_GPU_PATH}/ffx_core_gpu_common.h"
    "${FFX_GPU_PATH}/ffx_core_gpu_common_half.h"
    "${FFX_GPU_PATH}/ffx_core_hlsl.h"
    "${FFX_GPU_PATH}/ffx_core_portability.h")

project (FidelityFX-SDK)

# Components
add_subdirectory(${FFX_COMPONENTS_PATH}/opticalflow)
add_subdirectory(${FFX_COMPONENTS_PATH}/frameinterpolation)
add_subdirectory(${FFX_COMPONENTS_PATH}/fsr3)
add_subdirectory(${FFX_COMPONENTS_PATH}/fsr3upscaler)
add_subdirectory(${FFX_COMPONENTS_PATH}/fsr2)
add_subdirectory(${FFX_COMPONENTS_PATH}/fsr1)
add_subdirectory(${FFX_COMPONENTS_PATH}/spd)
add_subdirectory(${FFX_COMPONENTS_PATH}/cacao)
add_subdirectory(${FFX_COMPONENTS_PATH}/lpm)
add_subdirectory(${FFX_COMPONENTS_PATH}/blur)
add_subdirectory(${FFX_COMPONENTS_PATH}/vrs)
add_subdirectory(${FFX_COMPONENTS_PATH}/cas)
add_subdirectory(${FFX_COMPONENTS_PATH}/dof)
add_subdirectory(${FFX_COMPONENTS_PATH}/lens)
add_subdirectory(${FFX_COMPONENTS_PATH}/parallelsort)
add_subdirectory(${FFX_COMPONENTS_PATH}/denoiser)
add_subdirectory(${FFX_COMPONENTS_PATH}/sssr)
add_subdirectory(${FFX_COMPONENTS_PATH}/classifier)

# Add appropriate graphics backend if requested
if(FFX_API_DX11)
    add_subdirectory(${FFX_SRC_BACKENDS_PATH}/dx11)
endif()
if(FFX_API_DX12)
    add_subdirectory(${FFX_SRC_BACKENDS_PATH}/dx12)
endif()

if(FFX_API_VK)
    add_subdirectory(${FFX_SRC_BACKENDS_PATH}/vk)
endif()
