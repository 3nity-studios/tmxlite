cmake_minimum_required(VERSION 3.30.2)
project(tmxlite VERSION 1.4.4 LANGUAGES CXX)
SET(PROJECT_NAME tmxlite)

if(NOT DEFINED BUILD_SHARED_LIBS)
  SET(BUILD_SHARED_LIBS FALSE CACHE BOOL "Should tmxlite be built as a static or shared lib?")
endif()

if(NOT DEFINED CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "")
  SET(CMAKE_BUILD_TYPE "Release" CACHE STRING "Should tmxlite be built in release or debug mode?")
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo" "MinSizeRel")
endif()


SET(PROJECT_STATIC_RUNTIME FALSE CACHE BOOL "Use statically linked standard/runtime libraries?")
#option(BUILD_DOC "Build documentation" OFF)

SET(USE_RTTI TRUE CACHE BOOL "Use run time type information?")

SET(USE_EXTLIBS FALSE CACHE BOOL "Use external zlib, zstd and pugixml libraries instead of the included source?")
SET(USE_ZSTD FALSE CACHE BOOL "Enable zstd compression? (Already set to true if USE_EXTLIBS is true)")

if(USE_RTTI)
  if(CMAKE_COMPILER_IS_GNUCXX OR APPLE)
    if(PROJECT_STATIC_RUNTIME)
      SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -static")
    else()
      SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
    endif()
  endif()
else()
  if(CMAKE_COMPILER_IS_GNUCXX OR APPLE)
    if(PROJECT_STATIC_RUNTIME)
      SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -fno-rtti -static")
    else()
      SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -fno-rtti")
    endif()
  endif()
endif()

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(BUILD_SHARED_LIBS)
  SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -D_DEBUG_ -DTMXLITE_STATIC")
  SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3 -DNDEBUG -DTMXLITE_STATIC")
  SET(CMAKE_DEBUG_POSTFIX -s-d)
  SET(CMAKE_RELEASE_POSTFIX -s)
else()
  SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -D_DEBUG_")
  SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3 -DNDEBUG")
  SET(CMAKE_DEBUG_POSTFIX -d)
endif()

#disable msvc warning
if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif()

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)


#includes the list of source files in the src directory
SET(PROJECT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
include(${PROJECT_DIR}/CMakeLists.txt)

#if we want external zip and xml libs find them and tell the compiler
if(USE_EXTLIBS)
    add_definitions(-DUSE_EXTLIBS)
    add_definitions(-DUSE_ZSTD)

    if(TMXLITE_FETCH_EXTLIBS)
      FetchContent_Declare(
        pugixml
        GIT_REPOSITORY   https://github.com/zeux/pugixml.git
        GIT_TAG          db78afc2b7d8f043b4bc6b185635d949ea2ed2a8 # v1.14
        GIT_SHALLOW      ON
        GIT_PROGRESS     TRUE
        EXCLUDE_FROM_ALL
        SYSTEM
      )
      FetchContent_MakeAvailable(pugixml)

      FetchContent_Declare(
        ZLIB
        GIT_REPOSITORY   https://github.com/madler/zlib.git
        GIT_TAG          51b7f2abdade71cd9bb0e7a373ef2610ec6f9daf # 1.3.1
        GIT_SHALLOW      ON
        GIT_PROGRESS     TRUE
        EXCLUDE_FROM_ALL
        SYSTEM
      )
      FetchContent_MakeAvailable(ZLIB)
      
      FetchContent_Declare(
        zstd
        GIT_REPOSITORY   https://github.com/facebook/zstd.git
        GIT_TAG          794ea1b0afca0f020f4e57b6732332231fb23c70 # v1.5.6
        GIT_SHALLOW      ON
        GIT_PROGRESS     TRUE
        EXCLUDE_FROM_ALL
        SYSTEM
      )
      FetchContent_MakeAvailable(zstd)
    else()
      find_package(ZLIB REQUIRED)
      find_package(pugixml REQUIRED)
      find_package(zstd REQUIRED)
    endif()
else()
    #add miniz and pugixml from source
    SET(PROJECT_SRC ${PROJECT_SRC} ${LIB_SRC})
    if(USE_ZSTD)
      add_definitions(-DUSE_ZSTD)
      find_package(zstd REQUIRED)
    endif()
    
endif()

if(NOT BUILD_SHARED_LIBS)
    add_library(${PROJECT_NAME} STATIC ${PROJECT_SRC})
else()
    add_library(${PROJECT_NAME} SHARED ${PROJECT_SRC})
endif()   
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

if(USE_EXTLIBS)
    target_link_libraries(${PROJECT_NAME} ZLIB::ZLIB pugixml::static zstd::libzstd)
else()
    if(USE_ZSTD)
      target_link_libraries(${PROJECT_NAME} zstd::libzstd)
    endif()
endif()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/tmxlite.pc.in ${CMAKE_CURRENT_BINARY_DIR}/tmxlite.pc
               @ONLY)

target_include_directories(
  ${PROJECT_NAME}
  PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/thirdparty>
  $<INSTALL_INTERFACE:include>
)

include(GNUInstallDirs)
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/tmxlite DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
if(BUILD_SHARED_LIBS)
  install(TARGETS ${PROJECT_NAME} EXPORT tmxlite-targets DESTINATION ${CMAKE_INSTALL_LIBDIR})
else()
  install(TARGETS ${PROJECT_NAME} EXPORT tmxlite-targets
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

install(
  FILES       ${CMAKE_CURRENT_BINARY_DIR}/tmxlite.pc
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
  COMPONENT   libraries)

# https://cmake.org/cmake/help/latest/manual/cmake-packages.7.html#creating-packages
if(BUILD_SHARED_LIBS)
    set(config_name "Shared")
else()
    set(config_name "Static")
endif()
set(targets_config_filename "tmxlite${config_name}Targets.cmake")

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/tmxliteConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion    )

# A Package Config file that works from the build directory
export(EXPORT tmxlite-targets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/tmxliteTargets.cmake"
  NAMESPACE tmxlite::)

# A Package Config file that works from the installation directory
set(ConfigPackageLocation ${CMAKE_INSTALL_LIBDIR}/cmake/tmxlite)

install(EXPORT tmxlite-targets
  FILE ${targets_config_filename}
  NAMESPACE tmxlite::
  DESTINATION ${ConfigPackageLocation})

configure_package_config_file(
  cmake/tmxliteConfig.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/tmxliteConfig.cmake"
  INSTALL_DESTINATION ${ConfigPackageLocation})

configure_package_config_file(
  cmake/tmxliteConfigDependencies.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/tmxliteConfigDependencies.cmake"
  INSTALL_DESTINATION ${ConfigPackageLocation})

install(
  FILES
  "${CMAKE_CURRENT_BINARY_DIR}/tmxliteConfig.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/tmxliteConfigDependencies.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/tmxliteConfigVersion.cmake"
  DESTINATION ${ConfigPackageLocation}
  COMPONENT devel)
