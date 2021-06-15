cmake_minimum_required(VERSION 3.10.0)
project(fast_gicp_cuda)

find_package(cpp_config REQUIRED)
cpp_conf(
  -Wno-zero-as-null-pointer-constant -Wno-unused-variable -Wno-sign-compare -Wno-shadow -Wno-suggest-override -Wno-redundant-decls -Wno-ctor-dtor-privacy
  -Wno-extra-semi -Wno-unused-parameter -Wno-deprecated-copy -Wno-float-equal -Wno-float-conversion -Wno-duplicated-branches
)

set(CMAKE_BUILD_TYPE "Release")
set(CUDA_SOURCES "cuda_sources")

find_package(PCL REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(ament_cmake REQUIRED)

find_package(OpenMP)
if (OPENMP_FOUND)
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

find_package(CUDA REQUIRED)
include_directories(${CUDA_INCLUDE_DIRS})
link_directories(${CUDA_LIBRARY_DIRS})


add_library(${PROJECT_NAME} SHARED
  src/fast_gicp/gicp/fast_vgicp_cuda.cpp
  src/fast_gicp/ndt/ndt_cuda.cpp
)

ament_target_dependencies(
  ${PROJECT_NAME}
  PCL
  Eigen3
  CUDA
)

target_link_libraries(${PROJECT_NAME}
  ${PCL_LIBRARIES}
)

target_include_directories(${PROJECT_NAME} PUBLIC
${PCL_INCLUDE_DIRS}
${EIGEN3_INCLUDE_DIR}
)

target_include_directories(
  ${PROJECT_NAME} PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
                                 $<INSTALL_INTERFACE:include>
)



### CUDA ###
set(CUDA_NVCC_FLAGS "--expt-relaxed-constexpr")
add_definitions(-DUSE_VGICP_CUDA)

cuda_add_library(${CUDA_SOURCES} SHARED
  src/fast_gicp/cuda/fast_vgicp_cuda.cu
  src/fast_gicp/cuda/brute_force_knn.cu
  src/fast_gicp/cuda/covariance_estimation.cu
  src/fast_gicp/cuda/covariance_estimation_rbf.cu
  src/fast_gicp/cuda/covariance_regularization.cu
  src/fast_gicp/cuda/gaussian_voxelmap.cu
  src/fast_gicp/cuda/find_voxel_correspondences.cu
  src/fast_gicp/cuda/compute_derivatives.cu
  src/fast_gicp/cuda/compute_mahalanobis.cu
  src/fast_gicp/cuda/ndt_cuda.cu
  src/fast_gicp/cuda/ndt_compute_derivatives.cu
)
target_include_directories(${CUDA_SOURCES} PRIVATE
  include
  thirdparty/Eigen
  thirdparty/nvbio
)
cuda_add_cublas_to_target(${CUDA_SOURCES})

target_link_libraries(${PROJECT_NAME}
${CUDA_SOURCES}
)
add_dependencies(${PROJECT_NAME} ${CUDA_SOURCES})

ament_export_targets(export_${PROJECT_NAME} HAS_LIBRARY_TARGET)

install(DIRECTORY include/ DESTINATION include)

install(
    TARGETS ${PROJECT_NAME} ${CUDA_SOURCES}
    EXPORT export_${PROJECT_NAME}
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES
    DESTINATION include
)

ament_package()
