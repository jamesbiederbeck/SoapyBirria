cmake_minimum_required(VERSION 3.16)
project(SoapyBirria CXX)

find_package(SoapySDR REQUIRED)
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

# Point at a BirriaSDR checkout that has been built (build/libbirriasdr.so).
set(BIRRIASDR_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../BirriaSDR" CACHE PATH "BirriaSDR source tree")
find_library(BIRRIASDR_LIB birriasdr HINTS "${BIRRIASDR_ROOT}/build" REQUIRED)

SOAPY_SDR_MODULE_UTIL(
    TARGET birriaSupport
    SOURCES Registration.cpp Device.cpp
    LIBRARIES ${BIRRIASDR_LIB}
)
target_include_directories(birriaSupport PRIVATE "${BIRRIASDR_ROOT}/include")
set_target_properties(birriaSupport PROPERTIES
    BUILD_RPATH "${BIRRIASDR_ROOT}/build")
