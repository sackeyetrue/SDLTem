cmake_minimum_required(VERSION 3.16..4.1.1 FATAL_ERROR)

include(FetchContent)

#--------------------------------------------------------------------
# Project and version.
#--------------------------------------------------------------------

set (SDLTEM_VERSION_MAJOR 0)
set (SDLTEM_VERSION_MINOR 1)
set (SDLTEM_VERSION_PATCH 0)
set (SDLTEM_VERSION_STRING ${SDLTEM_VERSION_MAJOR}.${SDLTEM_VERSION_MINOR}.${SDLTEM_VERSION_PATCH})

project(SDLTem
        VERSION ${SDLTEM_VERSION_STRING} 
        DESCRIPTION "SDL Starter SDLTEM C/C++"
        HOMEPAGE_URL "https://github.com/djoezeke/SDLTem"
  LANGUAGES  C CXX
)

#--------------------------------------------------------------------
# Preparations depending on the specified build options  
#--------------------------------------------------------------------

if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
  set(SDLTEM_IS_TOP_LEVEL ON)
else()
  set(SDLTEM_IS_TOP_LEVEL OFF)
endif()

#--------------------------------------------------------------------
# Configure variables   
#--------------------------------------------------------------------

set(SDLTEM_EXE_NAME ${PROJECT_NAME} CACHE STRING "Base name of executable output name")

set(SDLTEM_EXE_TARGET_NAME  ${SDLTEM_EXE_NAME})

option(SDLTEM_ENABLE_DEBUG "Enable debug build" OFF)
option(SDLTEM_PRODUCTION_BUILD "Make this a production build!" OFF)
option(SDLTEM_BUILD_TESTS "Build the ${PROJECT_NAME} test programs" ${SDLTEM_IS_TOP_LEVEL})
option(SDLTEM_ENABLE_DOXYGEN "Build documentation with Doxygen." ${SDLTEM_IS_TOP_LEVEL})
option(SDLTEM_ENABLE_WARNING "Enable warning messages." ${SDLTEM_IS_TOP_LEVEL})

# Set required C/C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

#--------------------------------------------------------------------
# Apply Microsoft C runtime library option
# This is here because it also applies to tests and examples
#--------------------------------------------------------------------
if (MSVC AND NOT USE_MSVC_RUNTIME_LIBRARY_DLL)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Release>:Release>")
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

#--------------------------------------------------------------------
# Configurations
#--------------------------------------------------------------------

# Default to build type "Release" unless debug is enabled
if(NOT CMAKE_BUILD_TYPE)
	if (NOT SDLTEM_ENABLE_DEBUG)
		message(STATUS "No build type selected, default to Release")
		set(CMAKE_BUILD_TYPE "Release")
	else()
		message(STATUS "No build type selected but tests are being built, default to Debug")
		set(CMAKE_BUILD_TYPE "Debug")
	endif()
endif()

# Add compiler warnings
if(SDLTEM_ENABLE_WARNING)
  if(MSVC)
    set(COMPILER_WARNINGS /WX /W4 /permissive-)
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "(GNU|Clang)")
    set(COMPILER_WARNINGS -Werror -Wall -Wextra -Wpedantic -Wshadow -Wconversion -Wsign-conversion)
  endif()
endif()

# Output directories for a build tree
if(NOT DEFINED CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
endif()

#--------------------------------------------------------------------
# Add Subdirectories
#--------------------------------------------------------------------

FetchContent_Declare(
  SDL3
  GIT_REPOSITORY https://github.com/libsdl-org/SDL.git
  GIT_TAG release-3.4.8
  GIT_SHALLOW TRUE
)

set(SDL_SHARED ON CACHE BOOL "Build SDL shared library" FORCE)
set(SDL_STATIC OFF CACHE BOOL "Build SDL static library" FORCE)
set(SDL_TEST_LIBRARY OFF CACHE BOOL "Build SDL test library" FORCE)
set(SDL_TESTS OFF CACHE BOOL "Build SDL test programs" FORCE)
set(SDL_DISABLE_INSTALL ON CACHE BOOL "Disable SDL install target" FORCE)

FetchContent_MakeAvailable(SDL3)

# Build the documentation   
if(SDLTEM_ENABLE_DOXYGEN)
    add_subdirectory(docs)
endif()

# Build the test apps   
if(SDLTEM_BUILD_TESTS)
    include(CTest)
    enable_testing()
    add_subdirectory(tests)

    # Set the unit test app project as the Visual Studio startup project
    # if the target compiler is some version of Microsoft Visual C++ and
    # if this project is the main project.
    if(MSVC AND CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
        set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT "${PROJECT_NAME}Test")
    endif()
endif()

#--------------------------------------------------------------------
# Build targets
#--------------------------------------------------------------------

set(SDLTEM_SOURCE_FILES "src/main.cpp")

# Add executable
add_executable("${SDLTEM_EXE_TARGET_NAME}" ${SDLTEM_SOURCE_FILES})
target_include_directories("${SDLTEM_EXE_TARGET_NAME}" PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}" "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_link_libraries("${SDLTEM_EXE_TARGET_NAME}" PRIVATE SDL3::SDL3)

if(COMPILER_WARNINGS)
  target_compile_options("${SDLTEM_EXE_TARGET_NAME}" PRIVATE ${COMPILER_WARNINGS})
endif()

if(SDLTEM_PRODUCTION_BUILD)
	# setup the ASSETS_PATH macro to be in the root folder of your exe
	target_compile_definitions("${SDLTEM_EXE_TARGET_NAME}" PUBLIC ASSETS_PATH="./assets/") 
	target_compile_definitions("${SDLTEM_EXE_TARGET_NAME}" PUBLIC PRODUCTION_BUILD=1) 
else()
	# This is useful to get an ASSETS_PATH in your IDE during development
	target_compile_definitions("${SDLTEM_EXE_TARGET_NAME}" PUBLIC ASSETS_PATH="${CMAKE_CURRENT_SOURCE_DIR}/assets/")
	target_compile_definitions("${SDLTEM_EXE_TARGET_NAME}" PUBLIC PRODUCTION_BUILD=0) 
endif()
