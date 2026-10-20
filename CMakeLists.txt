cmake_minimum_required(VERSION 3.16)
project(SoapyBirria CXX)

find_package(SoapySDR REQUIRED)
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

# By default, build the BirriaSDR submodule as a static library linked into the
# plugin. Set BIRRIASDR_ROOT to a built checkout (build/libbirriasdr.so) instead;
# it must be at the commit the submodule pins unless BIRRIASDR_ANY_VERSION is ON.
set(BIRRIASDR_ROOT "" CACHE PATH "Built BirriaSDR source tree (empty: use the submodule)")
option(BIRRIASDR_ANY_VERSION "Accept a BIRRIASDR_ROOT checkout at any commit" OFF)
if(BIRRIASDR_ROOT)
  execute_process(COMMAND git -C "${CMAKE_CURRENT_SOURCE_DIR}" rev-parse :BirriaSDR
                  OUTPUT_VARIABLE BIRRIASDR_PINNED OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
  execute_process(COMMAND git -C "${BIRRIASDR_ROOT}" rev-parse HEAD
                  OUTPUT_VARIABLE BIRRIASDR_FOUND OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
  if(NOT BIRRIASDR_ANY_VERSION AND NOT BIRRIASDR_PINNED STREQUAL BIRRIASDR_FOUND)
    message(FATAL_ERROR "${BIRRIASDR_ROOT} is at '${BIRRIASDR_FOUND}', but SoapyBirria pins "
                        "BirriaSDR ${BIRRIASDR_PINNED}. Check out that commit, or pass "
                        "-DBIRRIASDR_ANY_VERSION=ON.")
  endif()
  find_library(BIRRIASDR_LIB birriasdr HINTS "${BIRRIASDR_ROOT}/build" REQUIRED)
  set(BIRRIASDR_TARGET ${BIRRIASDR_LIB})
else()
  if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/BirriaSDR/CMakeLists.txt")
    message(FATAL_ERROR "BirriaSDR submodule missing: run git submodule update --init")
  endif()
  set(BUILD_SHARED_LIBS OFF)
  set(BUILD_TESTING OFF)
  add_subdirectory(BirriaSDR EXCLUDE_FROM_ALL)
  set(BIRRIASDR_TARGET birriasdr)
endif()

SOAPY_SDR_MODULE_UTIL(
    TARGET birriaSupport
    SOURCES Registration.cpp Device.cpp
    LIBRARIES ${BIRRIASDR_TARGET}
)
if(BIRRIASDR_ROOT)
  target_include_directories(birriaSupport PRIVATE "${BIRRIASDR_ROOT}/include")
  set_target_properties(birriaSupport PROPERTIES
      BUILD_RPATH "${BIRRIASDR_ROOT}/build")
endif()
