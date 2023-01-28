cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

project(HelloWorld CXX)

include(FetchContent)

# The configuration we support
set(CMAKE_CONFIGURATION_TYPES "Debug;Release;Distribution")

# Only compile the JoltPhysics library and none of its examples
set(TARGET_UNIT_TESTS OFF)
set(TARGET_HELLO_WORLD OFF)
set(TARGET_PERFORMANCE_TEST OFF)
set(TARGET_SAMPLES OFF)
set(TARGET_VIEWER OFF)

# When turning this option on, the library will be compiled using doubles for positions. This allows for much bigger worlds.
set(DOUBLE_PRECISION OFF)

# When turning this option on, the library will be compiled with debug symbols
set(GENERATE_DEBUG_SYMBOLS ON)

# When turning this option on, the library will be compiled in such a way to attempt to keep the simulation deterministic across platforms
set(CROSS_PLATFORM_DETERMINISTIC OFF)

# When turning this option on, the library will be compiled with interprocedural optimizations enabled, also known as link-time optimizations or link-time code generation
set(INTERPROCEDURAL_OPTIMIZATION ON)

# Select X86 processor features to use, by default the library compiles with AVX2, if everything is off it will be SSE2 compatible.
set(USE_SSE4_1 ON)
set(USE_SSE4_2 ON)
set(USE_AVX ON)
set(USE_AVX2 ON)
set(USE_AVX512 OFF)
set(USE_LZCNT ON)
set(USE_TZCNT ON)
set(USE_F16C ON)
set(USE_FMADD ON)

# Include Jolt
FetchContent_Declare(
        JoltPhysics
        GIT_REPOSITORY "https://github.com/jrouwe/JoltPhysics"
        GIT_TAG "e1e33ac424a3c37daf808d61c85542bfbfa0349a"
		SOURCE_SUBDIR "Build"
)
FetchContent_MakeAvailable(JoltPhysics)

if (("${CMAKE_SYSTEM_NAME}" STREQUAL "Windows" OR "${CMAKE_SYSTEM_NAME}" STREQUAL "WindowsStore") AND NOT MINGW)
	# 64 bit architecture
	set(CMAKE_VS_PLATFORM_TOOLSET_HOST_ARCHITECTURE "x64")

	# Set runtime library
	if (USE_STATIC_MSVC_RUNTIME_LIBRARY)
		set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
	else()
		set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
	endif()

	# Set general compiler flags
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++17 /Zc:__cplusplus /Gm- /Wall /WX /MP /nologo /diagnostics:classic /FC /fp:except- /Zc:inline")
	
	# Optionally generate debug symbols
	if (GENERATE_DEBUG_SYMBOLS)
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /Zi")
	endif()

	# Remove any existing compiler flag that enables RTTI
	string(REPLACE "/GR" "" CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})

	# Set compiler flag for disabling RTTI
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /GR-")

	if ("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "ARM")
		# On ARM the exception handling flag is missing which causes warnings
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc")
	endif()

	# Set compiler flags for various configurations
	set(CMAKE_CXX_FLAGS_DEBUG "/GS /Od /Ob0 /RTC1")
	set(CMAKE_CXX_FLAGS_RELEASE "/GS- /Gy /O2 /Oi /Ot")
	set(CMAKE_CXX_FLAGS_DISTRIBUTION "/GS- /Gy /O2 /Oi /Ot")

	# Set linker flags
	if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
		if (CROSS_PLATFORM_DETERMINISTIC)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /fp:precise")
		else()
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /fp:fast") # Clang doesn't use fast math because it cannot be turned off inside a single compilation unit
		endif()
		if ("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "x86" OR "${CMAKE_VS_PLATFORM_NAME}" STREQUAL "x64")
			if (USE_AVX512)
				set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /arch:AVX512")
			elseif (USE_AVX2)
				set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /arch:AVX2")
			elseif (USE_AVX)
				set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /arch:AVX")
			endif()	
			if (USE_SSE4_1)
				add_compile_definitions(JPH_USE_SSE4_1)
			endif()
			if (USE_SSE4_2)
				add_compile_definitions(JPH_USE_SSE4_2)
			endif()
			if (USE_LZCNT)
				add_compile_definitions(JPH_USE_LZCNT)
			endif()
			if (USE_TZCNT)
				add_compile_definitions(JPH_USE_TZCNT)
			endif()
			if (USE_F16C)
				add_compile_definitions(JPH_USE_F16C)
			endif()
			if (USE_FMADD AND NOT CROSS_PLATFORM_DETERMINISTIC)
				add_compile_definitions(JPH_USE_FMADD)
			endif()
		endif()
		set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /DJPH_FLOATING_POINT_EXCEPTIONS_ENABLED") # Clang turns Float2 into a vector sometimes causing floating point exceptions
		set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /DJPH_FLOATING_POINT_EXCEPTIONS_ENABLED")
	elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /showFilenames")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Qunused-arguments") # Clang emits warnings about unused arguments such as /MP and /GL
		if (USE_AVX512)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx512f -mavx512vl -mavx512dq -mavx2 -mbmi -mpopcnt -mlzcnt -mf16c")
		elseif (USE_AVX2)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx2 -mbmi -mpopcnt -mlzcnt -mf16c")
		elseif (USE_AVX)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx -mpopcnt")
		elseif (USE_SSE4_2)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse4.2 -mpopcnt")
		elseif (USE_SSE4_1)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse4.1")
		else()
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse2")
		endif()
		if (USE_LZCNT)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mlzcnt")
		endif()
		if (USE_TZCNT)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mbmi")
		endif()
		if (USE_F16C)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mf16c")
		endif()
		if (USE_FMADD AND NOT CROSS_PLATFORM_DETERMINISTIC)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfma")
		endif()
	endif()
elseif ("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux" OR "${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin" OR "${CMAKE_SYSTEM_NAME}" STREQUAL "iOS" OR MINGW OR EMSCRIPTEN)
	# Set general compiler flags
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -I. -Wall -Werror")

	# Optionally generate debug symbols
	if (GENERATE_DEBUG_SYMBOLS)
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")
	endif()

	if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
		# Somehow -Wcomment doesn't want to be turned off from code and we need this because Doxygen MathJax uses it
		# Also disable -Wstringop-overflow or it will generate false positives that can't be disabled from code when link-time optimizations are enabled
		# Also turn off automatic fused multiply add contractions, there doesn't seem to be a way to do this selectively through the macro JPH_PRECISE_MATH_OFF
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-comment -Wno-stringop-overflow -ffp-contract=off")
	else()
		# Do not use -ffast-math since it cannot be turned off in a single compilation unit under clang, see Core.h
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ffp-model=precise")

		# On clang 14 and later we can turn off float contraction through a pragma, older versions and deterministic versions need it off always, see Core.h
		if (CMAKE_CXX_COMPILER_VERSION LESS 14 OR CROSS_PLATFORM_DETERMINISTIC)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ffp-contract=off")
		endif()
	endif()

	# Platform specific compiler flags
	if (CMAKE_OSX_ARCHITECTURES MATCHES "arm64" OR "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "aarch64")
		# ARM64
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
	elseif ("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86_64" OR "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "AMD64")
		# X64
		if (USE_AVX512)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx512f -mavx512vl -mavx512dq -mavx2 -mbmi -mpopcnt -mlzcnt -mf16c")
		elseif (USE_AVX2)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx2 -mbmi -mpopcnt -mlzcnt -mf16c")
		elseif (USE_AVX)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx -mpopcnt")
		elseif (USE_SSE4_2)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse4.2 -mpopcnt")
		elseif (USE_SSE4_1)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse4.1")
		else()
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse2")
		endif()
		if (USE_LZCNT)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mlzcnt")
		endif()
		if (USE_TZCNT)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mbmi")
		endif()
		if (USE_F16C)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mf16c")
		endif()
		if (USE_FMADD AND NOT CROSS_PLATFORM_DETERMINISTIC)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfma")
		endif()
	endif()

	# Set compiler flags for various configurations
	set(CMAKE_CXX_FLAGS_DEBUG "")
	set(CMAKE_CXX_FLAGS_RELEASE "-O3")
	set(CMAKE_CXX_FLAGS_DISTRIBUTION "-O3")

	# Set linker flags
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -pthread")
endif()

set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE OFF)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_DISTRIBUTION OFF)

# On ARM, whole program optimization triggers an internal compiler error during code gen, so we don't turn it on
if (INTERPROCEDURAL_OPTIMIZATION AND NOT ("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "ARM64") AND NOT ("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "ARM"))
	include(CheckIPOSupported)
	check_ipo_supported(RESULT IS_IPO_SUPPORTED OUTPUT IPO_CHECK_OUTPUT)

	if (IS_IPO_SUPPORTED)
		set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE ON)
		set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_DISTRIBUTION ON)
	else()
		message(WARNING "Interprocedural optimizations are not supported: ${IPO_CHECK_OUTPUT}")
	endif()
endif()

# Set linker flags
set(CMAKE_EXE_LINKER_FLAGS_DISTRIBUTION "${CMAKE_EXE_LINKER_FLAGS_RELEASE}")

# Compile the HelloWorld application
add_executable(${PROJECT_NAME} ../Source/HelloWorld.cpp)
target_include_directories(HelloWorld PUBLIC ${JoltPhysics_SOURCE_DIR}/..)
target_link_libraries(${PROJECT_NAME} Jolt)

# Make this project the startup project
set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT "HelloWorld")
