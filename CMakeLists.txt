project(Calibu)
cmake_minimum_required(VERSION 2.8)

if(POLICY CMP0042)
  cmake_policy(SET CMP0042 NEW) # MACOSX_RPATH
endif()


set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/)
include( install_package )

set(CALIBU_VERSION_MAJOR 0)
set(CALIBU_VERSION_MINOR 2)
set(CALIBU_VERSION ${CALIBU_VERSION_MAJOR}.${CALIBU_VERSION_MINOR})

option(BUILD_APPLICATIONS "Build Applications" ON)
option(BUILD_SHARED_LIBS "Build Shared Library" ON)

option(BUILD_MATLAB "Build MATLAB wrappers." OFF)
if(BUILD_MATLAB)
  find_package( MATLAB QUIET )
  if( MATLAB_FOUND )
    message( STATUS "Building matlab wrappers" )
  else()
    set(BUILD_MATLAB OFF)
  endif()
endif()

set(CMAKE_CXX_FLAGS "-std=c++14 -Wall ${CMAKE_CXX_FLAGS}")
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" AND NOT ANDROID)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
endif()

# Overide with cmake -DCMAKE_BUILD_TYPE=Debug {dir}
if( NOT CMAKE_BUILD_TYPE )
   set( CMAKE_BUILD_TYPE Release )
endif()

string(TOLOWER ${PROJECT_NAME} LIBRARY_NAME)

#######################################################
## Sources
set( INC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include/calibu )
set(HEADERS
  ${INC_DIR}/Calibu.h
  ${INC_DIR}/Platform.h
  ${INC_DIR}/exception.h
  ${INC_DIR}/calib/AutoDiffArrayCostFunction.h
  ${INC_DIR}/calib/Calibrator.h
  ${INC_DIR}/calib/CostFunctionAndParams.h
  ${INC_DIR}/calib/ReprojectionCostFunctor.h
  ${INC_DIR}/calib/LocalParamSe3.h
  ${INC_DIR}/cam/camera_crtp.h
  ${INC_DIR}/cam/camera_models_crtp.h
  ${INC_DIR}/cam/camera_models_poly.h
  ${INC_DIR}/cam/camera_models_kb4.h
  ${INC_DIR}/cam/camera_models_rational.h
  ${INC_DIR}/cam/camera_xml.h
  ${INC_DIR}/cam/stereo_rectify.h
  ${INC_DIR}/cam/camera_rig.h
  ${INC_DIR}/cam/rectify_crtp.h
  ${INC_DIR}/cam/camera_crtp_impl.h
  ${INC_DIR}/conics/Conic.h
  ${INC_DIR}/conics/ConicFinder.h
  ${INC_DIR}/conics/FindConics.h
  ${INC_DIR}/gl/Drawing.h
  ${INC_DIR}/image/AdaptiveThreshold.h
  ${INC_DIR}/image/Gradient.h
  ${INC_DIR}/image/ImageProcessing.h
  ${INC_DIR}/image/IntegralImage.h
  ${INC_DIR}/image/Label.h
  ${INC_DIR}/pcalib/base64.h
  ${INC_DIR}/pcalib/pcalib.h
  ${INC_DIR}/pcalib/pcalib_xml.h
  ${INC_DIR}/pcalib/response.h
  ${INC_DIR}/pcalib/response_impl.h
  ${INC_DIR}/pcalib/response_linear.h
  ${INC_DIR}/pcalib/response_poly.h
  ${INC_DIR}/pcalib/vignetting_dense.h
  ${INC_DIR}/pcalib/vignetting.h
  ${INC_DIR}/pcalib/vignetting_impl.h
  ${INC_DIR}/pcalib/vignetting_poly.h
  ${INC_DIR}/pcalib/vignetting_uniform.h
  ${INC_DIR}/pose/Ransac.h
  ${INC_DIR}/target/Hungarian.h
  ${INC_DIR}/target/LineGroup.h
  ${INC_DIR}/target/RandomGrid.h
  ${INC_DIR}/target/Target.h
  ${INC_DIR}/target/TargetGridDot.h
  ${INC_DIR}/target/GridDefinitions.h
  ${INC_DIR}/utils/Rectangle.h
  ${INC_DIR}/utils/Range.h
  ${INC_DIR}/utils/Utils.h
  ${INC_DIR}/utils/PlaneBasis.h
  ${INC_DIR}/utils/StreamOperatorsEigen.h
  ${INC_DIR}/utils/StreamOperatorsSophus.h
  )

set(SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
SET(SOURCES
  ${SRC_DIR}/cam/CameraXml.cpp
  ${SRC_DIR}/cam/rectify_crtp.cpp
  ${SRC_DIR}/cam/StereoRectify.cpp
  ${SRC_DIR}/conics/Conic.cpp
  ${SRC_DIR}/conics/ConicFinder.cpp
  ${SRC_DIR}/conics/FindConics.cpp
  ${SRC_DIR}/image/ImageProcessing.cpp
  ${SRC_DIR}/image/Label.cpp
  ${SRC_DIR}/pcalib/base64.cpp
  ${SRC_DIR}/pcalib/pcalib_xml.cpp
  ${SRC_DIR}/target/Hungarian.cpp
  ${SRC_DIR}/target/RandomGrid.cpp
  ${SRC_DIR}/target/TargetGridDot.cpp
  ${SRC_DIR}/utils/Utils.cpp
  )

#######################################################
## Find required dependencies
find_package( Eigen3 REQUIRED )
find_package( Sophus REQUIRED )
find_package( glog REQUIRED CONFIG)
set(HAVE_GLOG 1)
list( APPEND LINK_LIBS glog::glog )

find_package(tinyxml2 REQUIRED CONFIG)
list(APPEND LINK_LIBS tinyxml2 )


## Apply project include directories
list( APPEND CALIBU_INC
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR}/include
    ${CMAKE_BINARY_DIR}
    )

list( APPEND USER_INC   ${EIGEN3_INCLUDE_DIR} )
list( APPEND USER_INC   ${Sophus_INCLUDE_DIRS} )

#######################################################
## Find optional dependencies

# OpenCV is required for PNP methods and calibration application
find_package( OpenCV REQUIRED calib3d imgproc core )
if( OpenCV_FOUND )
    set( HAVE_OPENCV 1 )
    list( APPEND LINK_LIBS  ${OpenCV_LIBS})
    list( APPEND USER_INC ${OpenCV_INCLUDE_DIRS} )
    list( APPEND HEADERS ${INC_DIR}/pose/Pnp.h ${INC_DIR}/pose/Tracker.h )
    list( APPEND SOURCES ${SRC_DIR}/pose/Pnp.cpp ${SRC_DIR}/pose/Tracker.cpp )
endif()

#######################################################
## Optionally create unit tests

option(BUILD_TESTS "Build unit tests" OFF)

if(BUILD_TESTS)
  add_subdirectory(tests)
endif()

#######################################################
## Create configure file for inclusion in library

# Platform configuration vars
## Compiler configuration
include(SetPlatformVars)
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/src/config.h.in"
  "${CMAKE_CURRENT_BINARY_DIR}/include/calibu/config.h"
)
#list( APPEND HEADERS "${CMAKE_CURRENT_BINARY_DIR}/include/${LIBRARY_NAME}/config.h" )

# Include source/build directories first to preempt any installed Calibu
include_directories( ${CALIBU_INC} ${USER_INC} ${CMAKE_CURRENT_BINARY_DIR}/include )

if( BUILD_MATLAB )
  add_subdirectory( matlab )
endif()

# build calibu library
add_library( calibu ${SOURCES} )
target_link_libraries( calibu ${LINK_LIBS} )

#######################################################
## Setup and configure library
## Generate symbol export helper header on MSVC
SET(EXPORT_HEADER)
IF(MSVC)
    SET(EXPORT_HEADER "${CMAKE_CURRENT_BINARY_DIR}/include/${LIBRARY_NAME}/${LIBRARY_NAME}_export.h")
    string(TOUPPER ${LIBRARY_NAME} LIBRARY_NAME_CAPS)
    include(GenerateExportHeader)
    GENERATE_EXPORT_HEADER( ${LIBRARY_NAME}
        BASE_NAME ${LIBRARY_NAME_CAPS}
        EXPORT_MACRO_NAME ${LIBRARY_NAME_CAPS}_EXPORT
        EXPORT_FILE_NAME ${EXPORT_HEADER}
        STATIC_DEFINE ${LIBRARY_NAME_CAPS}_BUILT_AS_STATIC
    )
ENDIF()

# install everything
install_package(
    PKG_NAME Calibu
    LIB_NAME calibu
    VERSION 0.2
    DESCRIPTION "Computer vision camera model library."
#    INSTALL_HEADERS ${HEADERS}
    INSTALL_INCLUDE_DIR true
    INSTALL_GENERATED_HEADERS ${CMAKE_CURRENT_BINARY_DIR}/include/calibu/config.h ${EXPORT_HEADER}
    DESTINATION ${CMAKE_INSTALL_PREFIX}
    INCLUDE_DIRS ${USER_INC} ${CMAKE_INSTALL_PREFIX}/include
    LINK_LIBS ${LINK_LIBS}
    LINK_DIRS ${CMAKE_INSTALL_PREFIX}/lib
    )

if( BUILD_APPLICATIONS )
  set( Calibu_DIR ${CMAKE_CURRENT_BINARY_DIR} ) # so subdirs can call find_package( Calibu )
  add_subdirectory( applications )
endif()

###########################
 include(${CMAKE_MODULE_PATH}/cmake_uninstall.cmake.in)

 add_custom_target(uninstall
     COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
 ###########################

