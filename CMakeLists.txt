##
## CMake build script for LLVM.
##

cmake_minimum_required(VERSION 3.9)

project(LLVM CXX C)

if(TARGET llvm-project)
	return()
endif()

find_package(Threads REQUIRED)
if(UNIX)
	find_package(ZLIB REQUIRED)
endif()

# Set the default build type to 'Release'
if (NOT CMAKE_BUILD_TYPE)
	set(default_build_type "Release")
	message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
	set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE STRING "Choose the type of build." FORCE)
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
	set(IS_DEBUG_BUILD "YES")
else()
	set(IS_DEBUG_BUILD "NO")
endif()

# Use the ExternalProject module instead of just CMake because we want to build and
# install only some tools from LLVM, not all the libraries and tools.
include(ExternalProject)

# LLVM requires python3, so we first check whether python3 is installed as
# "python3", and if not, we suppose that `python` runs python3.
find_program(PYTHON_EXECUTABLE "python3")
if(NOT PYTHON_EXECUTABLE)
	find_program(PYTHON_EXECUTABLE "python")
endif()

ExternalProject_Add(llvm-project
	SOURCE_DIR "${PROJECT_SOURCE_DIR}/src"
	CMAKE_ARGS
		# Force a release build (we don't need to debug LLVM).
		# This has no effect on Windows with MSVC, but is useful on Linux.
		-DCMAKE_BUILD_TYPE=Release

		# Force Python3 (see the comment above when setting
		# PYTHON_EXECUTABLE).
		-DPYTHON_EXECUTABLE=${PYTHON_EXECUTABLE}

		# Targets to be build.
		-DLLVM_TARGETS_TO_BUILD=X86

		# Disable the generation of targets for tests (we don't need them).
		-DLLVM_INCLUDE_TESTS=NO

		# Our tools depending on LLVM require RTTI, so build LLVM with it.
		-DLLVM_REQUIRES_RTTI=YES

		# When building in the debug mode (=> assertions are enabled), we
		# have to build LLVM with assertions. This prevents link errors
		# when building the middle-end and back-end (see
		# https://github.com/oclint/oclint/issues/129).
		-DLLVM_ENABLE_ASSERTIONS=${IS_DEBUG_BUILD}

		# Disable the emission of warnings, which are useless since we do
		# not modify the LLVM sources (of course, except for a few
		# exceptions).
		-DLLVM_ENABLE_WARNINGS=NO

		# Enable the use of terminfo because in the back-end, we use the
		# coloring system from LLVM so we do not have to deal with issues
		# due to multi-platform support of coloring by ourselves.
		# Note to developers: On Linux, programs that link LLVM libraries
		#                     need to also link the tinfo library.
		-DLLVM_ENABLE_TERMINFO=YES

	# Disable the installation right after build (we want to install the
	# needed libraries and tools manually).
	INSTALL_COMMAND ""
)

# Forces a configure step for the given external project.
# The configure step for external projects is needed to (1) detect source-file
# changes and (2) fix infinite recursion of 'make' after a terminated build.
macro(force_configure_step target)
	# This solution is based on
	# http://comments.gmane.org/gmane.comp.programming.tools.cmake.user/43024
	ExternalProject_Add_Step(${target} force-configure
		COMMAND ${CMAKE_COMMAND} -E echo "Force configure of ${target}"
		DEPENDEES update
		DEPENDERS configure
		ALWAYS 1
	)
endmacro()

# We need to force the configuration step for LLVM. Otherwise, if we changed
# LLVM sources, they would not be rebuilt. This also fixes infinite recursion
# when running make after an interrupted LLVM build.
force_configure_step(llvm-project)

# Add libraries.
ExternalProject_Get_Property(llvm-project binary_dir)

add_library(llvm INTERFACE)
add_dependencies(llvm llvm-project)

set(LLVM_LIB_LIST
	LLVMInterpreter
	LLVMMCJIT
	LLVMRuntimeDyld
	LLVMOrcJIT
	LLVMExecutionEngine
	LLVMRuntimeDyld
	LLVMX86CodeGen
	LLVMX86AsmParser
	LLVMX86Disassembler
	LLVMBitWriter
	LLVMIRReader
	LLVMInstrumentation
	LLVMObject
	LLVMSupport
	LLVMipo
	LLVMAsmPrinter
	LLVMSelectionDAG
	LLVMX86Desc
	LLVMAsmParser
	LLVMBitReader
	LLVMVectorize
	LLVMMCParser
	LLVMCodeGen
	LLVMX86AsmPrinter
	LLVMX86Info
	LLVMObjCARCOpts
	LLVMScalarOpts
	LLVMX86Utils
	LLVMTransformUtils
	LLVMAnalysis
	LLVMTarget
	LLVMCore
	LLVMMC
	LLVMObject
	LLVMMCDisassembler
	LLVMProfileData
	LLVMDebugInfoCodeView
	LLVMPasses
	LLVMLinker
	LLVMInstCombine
	LLVMSupport
)

if(MSVC)
	set(DEBUG_DIR "Debug/")
	set(RELEASE_DIR "Release/")
endif()

foreach(LLVM_LIB ${LLVM_LIB_LIST})
	target_link_libraries(llvm INTERFACE debug     ${binary_dir}/${DEBUG_DIR}lib/${CMAKE_FIND_LIBRARY_PREFIXES}${LLVM_LIB}${CMAKE_STATIC_LIBRARY_SUFFIX})
	target_link_libraries(llvm INTERFACE optimized ${binary_dir}/${RELEASE_DIR}lib/${CMAKE_FIND_LIBRARY_PREFIXES}${LLVM_LIB}${CMAKE_STATIC_LIBRARY_SUFFIX})
endforeach(LLVM_LIB)

if(CMAKE_THREAD_LIBS_INIT)
	target_link_libraries(llvm INTERFACE debug     ${CMAKE_THREAD_LIBS_INIT})
	target_link_libraries(llvm INTERFACE optimized ${CMAKE_THREAD_LIBS_INIT})
endif()

if(UNIX)
	target_link_libraries(llvm INTERFACE debug     ${ZLIB_LIBRARIES} dl)
	target_link_libraries(llvm INTERFACE optimized ${ZLIB_LIBRARIES} dl)

	if (NOT APPLE)
		target_link_libraries(llvm INTERFACE debug     rt tinfo)
		target_link_libraries(llvm INTERFACE optimized rt tinfo)
	else()
		target_link_libraries(llvm INTERFACE debug     curses)
		target_link_libraries(llvm INTERFACE optimized curses)
	endif()
endif()

# Set include directories.
ExternalProject_Get_Property(llvm-project source_dir)
target_include_directories(llvm SYSTEM INTERFACE ${source_dir}/include)
target_include_directories(llvm SYSTEM INTERFACE ${binary_dir}/include)
