
cmake_minimum_required(VERSION 3.8)

project(cuda_image_convert LANGUAGES CUDA CXX)

# Default to C99
set(CMAKE_C_STANDARD 99)

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(CUDA REQUIRED) 

include(FindCUDA/select_compute_arch)
CUDA_DETECT_INSTALLED_GPUS(INSTALLED_GPU_CCS_1)
string(STRIP "${INSTALLED_GPU_CCS_1}" INSTALLED_GPU_CCS_2)
string(REPLACE " " ";" INSTALLED_GPU_CCS_3 "${INSTALLED_GPU_CCS_2}")
string(REPLACE "." "" CUDA_ARCH_LIST "${INSTALLED_GPU_CCS_3}")
set(CMAKE_CUDA_ARCHITECTURES ${CUDA_ARCH_LIST})

# build library
file(GLOB cudaImageConvertSources *.cpp cuda/*.cu cuda/*.cpp image/*.cpp)

add_library(cuda_image_convert SHARED ${cudaImageConvertSources})
target_include_directories(cuda_image_convert PUBLIC ${CUDA_INCLUDE_DIRS} $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>)
target_link_libraries(cuda_image_convert ${CUDA_nppicc_LIBRARY})	

set_target_properties(cuda_image_convert PROPERTIES CUDA_SEPARABLE_COMPILATION ON)

# Export 
include(GNUInstallDirs)

install( 
  TARGETS cuda_image_convert
  EXPORT cudaImageConvertTargets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(
    DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

include(CMakePackageConfigHelpers)

# generate the config file that is includes the exports
configure_package_config_file(cmake/Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/cuda_image_convertConfig.cmake"
  INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake"
  NO_SET_AND_CHECK_MACRO
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

# generate the version file for the config file
write_basic_package_version_file(
  "cuda_image_convertConfigVersion.cmake"
  VERSION "${cuda_image_convert_VERSION_MAJOR}.${cuda_image_convert_VERSION_MINOR}"
  COMPATIBILITY AnyNewerVersion
)

# install the configuration file
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/cuda_image_convertConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/cuda_image_convertConfigVersion.cmake
    DESTINATION share/cuda_image_convert/cmake)

install(
    EXPORT cudaImageConvertTargets
    DESTINATION share/cuda_image_convert/cmake
)