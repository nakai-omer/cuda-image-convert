
cmake_minimum_required(VERSION 3.8)

# determine if cuda-image-format is being built as a submodule inside another repo,
# or if it's being build standalone (if the later, we need to do some configuration)
get_directory_property(hasParent PARENT_DIRECTORY)

if(hasParent)
	message("-- cuda-image-format:  building as submodule, ${hasParent}")
else()
	message("-- cuda-image-format:  building as standalone")
	
	# standalone project
	project(cuda-image-format)
  
	# -std=gnu++11
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wno-write-strings")	
  
	# setup CUDA
	set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cuda")
	find_package(CUDA)
	message("-- CUDA version: ${CUDA_VERSION}")

	set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -O3)

	if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64")
		message("-- CUDA ${CUDA_VERSION} detected (${CMAKE_SYSTEM_PROCESSOR}), enabling SM_53 SM_62")
		set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_53,code=sm_53 -gencode arch=compute_62,code=sm_62)

		if(CUDA_VERSION_MAJOR GREATER 9)
			message("-- CUDA ${CUDA_VERSION} detected (${CMAKE_SYSTEM_PROCESSOR}), enabling SM_72")
			set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_72,code=sm_72)
		endif()

		if(CUDA_VERSION_MAJOR GREATER 10)
			message("-- CUDA ${CUDA_VERSION} detected (${CMAKE_SYSTEM_PROCESSOR}), enabling SM_87")
			set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_87,code=sm_87)
		endif()
	endif()
	
	# setup project output paths
	set(PROJECT_OUTPUT_DIR  ${PROJECT_BINARY_DIR}/${CMAKE_SYSTEM_PROCESSOR})
	set(PROJECT_INCLUDE_DIR ${PROJECT_OUTPUT_DIR}/include)

	file(MAKE_DIRECTORY ${PROJECT_INCLUDE_DIR})
	file(MAKE_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)

	message("-- system arch:  ${CMAKE_SYSTEM_PROCESSOR}")
	message("-- output path:  ${PROJECT_OUTPUT_DIR}")

	set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)
	set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)
	set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)
	
	# build C/C++ interface
	include_directories(${PROJECT_INCLUDE_DIR})

endif()

# option for enabling/disabling NVMM memory in multimedia stack
if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64")
	set(ENABLE_NVMM_DEFAULT ON)
else()
	set(ENABLE_NVMM_DEFAULT OFF)
endif()

option(ENABLE_NVMM "Enable use of NVMM zero-copy memory in video and camera streaming" ${ENABLE_NVMM_DEFAULT})
message("-- NVMM zero-copy memory:  ENABLE_NVMM=${ENABLE_NVMM}")

if(ENABLE_NVMM)
	add_definitions(-DENABLE_NVMM)
endif()

# additional paths for includes and libraries
include_directories(${PROJECT_INCLUDE_DIR}/cuda-image-format)

if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64")
	link_directories(/usr/lib/aarch64-linux-gnu/tegra)
endif()

# build library
file(GLOB cudaImageFormatSources *.cpp cuda/*.cu cuda/*.cpp image/*.cpp)
file(GLOB cudaImageFormatIncludes *.h *.hpp cuda/*.h cuda/*.cuh image/*.h image/*.inl)

cuda_add_library(cuda-image-format SHARED ${cudaImageFormatSources})
target_link_libraries(cuda-image-format GL GLU GLEW ${CUDA_nppicc_LIBRARY})	

if(ENABLE_NVMM)
	target_link_libraries(cuda-image-format nvbuf_utils)
endif()

# transfer all headers to the include directory 
file(MAKE_DIRECTORY ${PROJECT_INCLUDE_DIR}/cuda-image-format)

foreach(include ${cudaImageFormatIncludes})
	message("-- Copying ${include}")
	configure_file(${include} ${PROJECT_INCLUDE_DIR}/cuda-image-format COPYONLY)
endforeach()
	
# install headers
foreach(include ${cudaImageFormatIncludes})
	install(FILES "${include}" DESTINATION include/cuda-image-format)
endforeach()

# install the shared library
install(TARGETS cuda-image-format DESTINATION lib EXPORT cuda-image-formatConfig)

# install the cmake project, for importing
install(EXPORT cuda-image-formatConfig DESTINATION share/cuda-image-format/cmake)