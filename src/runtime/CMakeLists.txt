find_package(Threads REQUIRED)

set(HIPSYCL_RT_EXTRA_CXX_FLAGS "")
set(HIPSYCL_RT_EXTRA_LINKER_FLAGS "")
set(HIPSYCL_RT_SANITIZE "" CACHE STRING
    "Enable building hipSYCL runtime with specified sanitizers")

list(INSERT CMAKE_MODULE_PATH 0 "${PROJECT_SOURCE_DIR}/cmake/")
find_package(Filesystem REQUIRED Final Experimental Boost)
set(CXX_FILESYSTEM_HEADER "${CXX_FILESYSTEM_HEADER}" PARENT_SCOPE)
set(CXX_FILESYSTEM_NAMESPACE "${CXX_FILESYSTEM_NAMESPACE}" PARENT_SCOPE)
list(REMOVE_AT CMAKE_MODULE_PATH 0)

if(NOT ${HIPSYCL_RT_SANITIZE} STREQUAL "")
  set(HIPSYCL_RT_EXTRA_CXX_FLAGS "-fsanitize=${HIPSYCL_RT_SANITIZE}")
  set(HIPSYCL_RT_EXTRA_LINKER_FLAGS "-fsanitize=${HIPSYCL_RT_SANITIZE}")
endif()

add_library(hipSYCL-rt SHARED
  application.cpp
  runtime.cpp
  error.cpp
  backend.cpp
  backend_loader.cpp
  hints.cpp
  device_id.cpp
  operations.cpp
  data.cpp
  inorder_executor.cpp
  kernel_cache.cpp
  multi_queue_executor.cpp
  dag.cpp
  dag_node.cpp
  dag_builder.cpp
  dag_direct_scheduler.cpp
  dag_unbound_scheduler.cpp
  dag_manager.cpp
  dag_submitted_ops.cpp
  settings.cpp
  generic/async_worker.cpp
  hw_model/memcpy.cpp
  serialization/serialization.cpp)

target_compile_options(hipSYCL-rt PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
target_link_libraries(hipSYCL-rt PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS} opensycl-common Threads::Threads)

# syclcc already knows about these include directories, but clangd-based tooling does not.
# Specifying them explicitly ensures that IDEs can resolve all hipSYCL includes correctly.
target_include_directories(hipSYCL-rt
  PUBLIC
    $<BUILD_INTERFACE:${HIPSYCL_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
  PRIVATE
    ${HIPSYCL_SOURCE_DIR}
    ${PROJECT_BINARY_DIR}/include
)

# to get the search path next to the rt lib, we need to know the output name, so set it explicitly.
set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "hipSYCL-rt")
if(WIN32)
  set_target_properties(hipSYCL-rt PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS On)

  # ensure actual output and expected are equivalent
  set_target_properties(hipSYCL-rt PROPERTIES RUNTIME_OUTPUT_NAME ${HIPSYCL_RT_LIBRARY_OUTPUT_NAME})
  if(MinGW)
    set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "${HIPSYCL_RT_LIBRARY_OUTPUT_NAME}.dll" PARENT_SCOPE)
  else()
    set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "lib${HIPSYCL_RT_LIBRARY_OUTPUT_NAME}.dll" PARENT_SCOPE)
  endif()
else()
  target_link_libraries(hipSYCL-rt PRIVATE dl std::filesystem)

  # ensure actual output and expected are equivalent
  set_target_properties(hipSYCL-rt PROPERTIES LIBRARY_OUTPUT_NAME ${HIPSYCL_RT_LIBRARY_OUTPUT_NAME})
  if(APPLE)
    set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "lib${HIPSYCL_RT_LIBRARY_OUTPUT_NAME}.so" PARENT_SCOPE)
  else()
    set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "lib${HIPSYCL_RT_LIBRARY_OUTPUT_NAME}.dylib" PARENT_SCOPE)
  endif()
endif()

install(TARGETS hipSYCL-rt
        EXPORT install_exports
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)


if(WITH_CUDA_BACKEND)
  add_library(rt-backend-cuda SHARED
    cuda/cuda_event.cpp
    cuda/cuda_event_pool.cpp
    cuda/cuda_queue.cpp
    cuda/cuda_instrumentation.cpp
    cuda/cuda_allocator.cpp
    cuda/cuda_device_manager.cpp
    cuda/cuda_hardware_manager.cpp
    cuda/cuda_backend.cpp
    cuda/cuda_code_object.cpp)

  target_include_directories(rt-backend-cuda PRIVATE
    ${HIPSYCL_SOURCE_DIR}/include
    ${CUDA_TOOLKIT_ROOT_DIR}/include)
  
  target_link_libraries(rt-backend-cuda PRIVATE hipSYCL-rt ${CUDA_LIBRARIES} ${CUDA_DRIVER_LIBRARY})

  target_compile_options(rt-backend-cuda PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
  target_link_libraries(rt-backend-cuda PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS})

  if(WITH_SSCP_COMPILER)
    target_compile_definitions(rt-backend-cuda PRIVATE -DHIPSYCL_WITH_SSCP_COMPILER)
    target_link_libraries(rt-backend-cuda PRIVATE llvm-to-ptx)
  endif()

  install(TARGETS rt-backend-cuda
        RUNTIME DESTINATION bin/hipSYCL
        LIBRARY DESTINATION lib/hipSYCL
        ARCHIVE DESTINATION lib/hipSYCL)
endif()

if(WITH_ROCM_BACKEND)
  set(ENABLE_ROCM_UNIFIED_MEMORY_API ON CACHE BOOL "Utilize unified memory API in ROCm. Older ROCm versions may not support this.")

  add_library(rt-backend-hip SHARED
    hip/hip_event.cpp
    hip/hip_event_pool.cpp
    hip/hip_queue.cpp
    hip/hip_instrumentation.cpp
    hip/hip_allocator.cpp
    hip/hip_device_manager.cpp
    hip/hip_hardware_manager.cpp
    hip/hip_backend.cpp
    hip/hip_code_object.cpp)

  target_compile_definitions(rt-backend-hip PRIVATE HIPSYCL_RT_HIP_TARGET_ROCM=1)
  if(ENABLE_ROCM_UNIFIED_MEMORY_API)
    target_compile_definitions(rt-backend-hip PRIVATE HIPSYCL_RT_HIP_SUPPORTS_UNIFIED_MEMORY=1)
  endif()
  target_include_directories(rt-backend-hip PRIVATE ${HIPSYCL_SOURCE_DIR}/include)
  if(NOT HIP_FOUND)
    target_include_directories(rt-backend-hip PRIVATE ${ROCM_PATH}/include)
    target_link_libraries(rt-backend-hip PRIVATE ${ROCM_LIBS})
  else()
    # Supress warnings because wrongly set CXX arguments
    target_compile_options(rt-backend-hip PRIVATE -Wno-unused-command-line-argument)
    target_link_libraries(rt-backend-hip PRIVATE hipSYCL-rt  hip::host)
  endif()

  target_compile_options(rt-backend-hip PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
  target_link_libraries(rt-backend-hip PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS})

  if(WITH_SSCP_COMPILER)
    target_compile_definitions(rt-backend-hip PRIVATE -DHIPSYCL_WITH_SSCP_COMPILER)
    target_link_libraries(rt-backend-hip PRIVATE llvm-to-amdgpu)
  endif()

  install(TARGETS rt-backend-hip
        RUNTIME DESTINATION bin/hipSYCL
        LIBRARY DESTINATION lib/hipSYCL
        ARCHIVE DESTINATION lib/hipSYCL)
endif()

if(WITH_LEVEL_ZERO_BACKEND)
  add_library(rt-backend-ze SHARED
    ze/ze_backend.cpp
    ze/ze_hardware_manager.cpp
    ze/ze_allocator.cpp
    ze/ze_event.cpp
    ze/ze_queue.cpp
    ze/ze_code_object.cpp)
  
  target_include_directories(rt-backend-ze PRIVATE ${HIPSYCL_SOURCE_DIR}/include)
  target_link_libraries(rt-backend-ze PRIVATE hipSYCL-rt -lze_loader)

  target_compile_options(rt-backend-ze PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
  target_link_libraries(rt-backend-ze PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS})

  if(WITH_SSCP_COMPILER)
    target_compile_definitions(rt-backend-ze PRIVATE -DHIPSYCL_WITH_SSCP_COMPILER)
    target_link_libraries(rt-backend-ze PRIVATE llvm-to-spirv)
  endif()

  install(TARGETS rt-backend-ze
        LIBRARY DESTINATION lib/hipSYCL
        ARCHIVE DESTINATION lib/hipSYCL)
endif()

if(WITH_CPU_BACKEND)
  add_library(rt-backend-omp SHARED
    omp/omp_allocator.cpp
    omp/omp_backend.cpp
    omp/omp_event.cpp
    omp/omp_hardware_manager.cpp
    omp/omp_queue.cpp)

    find_package(OpenMP REQUIRED)

    target_include_directories(rt-backend-omp PRIVATE ${HIPSYCL_SOURCE_DIR}/include)
    if(APPLE)
      if(CMAKE_VERSION VERSION_LESS "3.16")
        message(FATAL_ERROR "CMake 3.16.0+ is required for macOS OpenMP support!")
      endif()
      target_include_directories(rt-backend-omp PRIVATE ${OpenMP_CXX_INCLUDE_DIRS})
      string(JOIN " " hipSYCL_OpenMP_CXX_LIBRARIES ${OpenMP_CXX_LIBRARIES})
      set(hipSYCL_OpenMP_CXX_LIBRARIES ${hipSYCL_OpenMP_CXX_LIBRARIES} PARENT_SCOPE)
    endif()
  
  list(LENGTH OpenMP_CXX_LIBRARIES OpenMP_CXX_LIBRARIES_LENGTH)
  if(WIN32 AND ${OpenMP_CXX_LIBRARIES_LENGTH} EQUAL 0)
    # FindOpenMP does a bad job here, finding any library.. so add some more hints..
    find_library(hipSYCL_OpenMP_libomp_LIBRARY
      NAMES libomp libgomp libiomp5 HINTS
      ${CMAKE_${LANG}_IMPLICIT_LINK_DIRECTORIES}
      ${LLVM_PREFIX_DIR}/lib
    )
    target_link_libraries(rt-backend-omp PRIVATE ${hipSYCL_OpenMP_libomp_LIBRARY})
  endif()

  target_link_libraries(rt-backend-omp PRIVATE hipSYCL-rt  OpenMP::OpenMP_CXX)

  target_compile_options(rt-backend-omp PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
  target_link_libraries(rt-backend-omp PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS})

  install(TARGETS rt-backend-omp
      RUNTIME DESTINATION bin/hipSYCL
      LIBRARY DESTINATION lib/hipSYCL
      ARCHIVE DESTINATION lib/hipSYCL)
endif()

