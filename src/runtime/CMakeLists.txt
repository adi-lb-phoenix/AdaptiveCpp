find_package(Threads REQUIRED)


include(CheckIPOSupported)
check_ipo_supported(RESULT is_ipo_supported OUTPUT ipo_error)

set(HIPSYCL_RT_EXTRA_CXX_FLAGS "")
set(HIPSYCL_RT_EXTRA_LINKER_FLAGS "")


list(INSERT CMAKE_MODULE_PATH 0 "${PROJECT_SOURCE_DIR}/cmake/")
find_package(Filesystem REQUIRED "${ACPP_FILESYSTEM_SEARCH_OPTIONS}")
set(CXX_FILESYSTEM_HEADER "${CXX_FILESYSTEM_HEADER}" PARENT_SCOPE)
set(CXX_FILESYSTEM_NAMESPACE "${CXX_FILESYSTEM_NAMESPACE}" PARENT_SCOPE)
list(REMOVE_AT CMAKE_MODULE_PATH 0)

set(HIPSYCL_RT_EXTRA_CXX_FLAGS ${ACPP_RT_SANITIZE_FLAGS})
set(HIPSYCL_RT_EXTRA_LINKER_FLAGS ${ACPP_RT_SANITIZE_FLAGS})


set(HIPSYCL_RT_EXTRA_LINKER_FLAGS ${HIPSYCL_RT_EXTRA_LINKER_FLAGS} ${HIPSYCL_STDPAR_RT_LINKER_FLAGS})

set(CMAKE_INSTALL_RPATH ${base} ${base}/hipSYCL)

add_library(acpp-rt SHARED
  application.cpp
  runtime.cpp
  error.cpp
  backend.cpp
  backend_loader.cpp
  device_id.cpp
  dylib_loader.cpp
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
  adaptivity_engine.cpp
  generic/async_worker.cpp
  hw_model/memcpy.cpp
  serialization/serialization.cpp)

target_compile_options(acpp-rt PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
target_link_libraries(acpp-rt PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS} acpp-common Threads::Threads)

# syclcc already knows about these include directories, but clangd-based tooling does not.
# Specifying them explicitly ensures that IDEs can resolve all hipSYCL includes correctly.
target_include_directories(acpp-rt
  PUBLIC
    $<BUILD_INTERFACE:${HIPSYCL_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
  PRIVATE
    ${HIPSYCL_SOURCE_DIR}
    ${PROJECT_BINARY_DIR}/include
)

# to get the search path next to the rt lib, we need to know the output name, so set it explicitly.
set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "acpp-rt")
if(WIN32)
  set_target_properties(acpp-rt PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS On)

  # ensure actual output and expected are equivalent
  set_target_properties(acpp-rt PROPERTIES RUNTIME_OUTPUT_NAME ${HIPSYCL_RT_LIBRARY_OUTPUT_NAME})
  if(MinGW)
    set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "lib${HIPSYCL_RT_LIBRARY_OUTPUT_NAME}.dll" PARENT_SCOPE)
  else()
    set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "${HIPSYCL_RT_LIBRARY_OUTPUT_NAME}.dll" PARENT_SCOPE)
  endif()
else()
  target_link_libraries(acpp-rt PRIVATE dl std::filesystem)

  # ensure actual output and expected are equivalent
  set_target_properties(acpp-rt PROPERTIES LIBRARY_OUTPUT_NAME ${HIPSYCL_RT_LIBRARY_OUTPUT_NAME})
  if(APPLE)
    set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "lib${HIPSYCL_RT_LIBRARY_OUTPUT_NAME}.so" PARENT_SCOPE)
  else()
    set(HIPSYCL_RT_LIBRARY_OUTPUT_NAME "lib${HIPSYCL_RT_LIBRARY_OUTPUT_NAME}.dylib" PARENT_SCOPE)
  endif()
endif()
if(is_ipo_supported)
  set_property(TARGET acpp-rt PROPERTY INTERPROCEDURAL_OPTIMIZATION True)
endif()

install(TARGETS acpp-rt
        EXPORT install_exports
        INCLUDES DESTINATION include/AdaptiveCpp
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)

set(CMAKE_INSTALL_RPATH ${base}/../ ${base}/llvm-to-backend)

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
  
  target_link_libraries(rt-backend-cuda PRIVATE acpp-rt ${CUDA_LIBS})

  target_compile_options(rt-backend-cuda PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
  target_link_libraries(rt-backend-cuda PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS})

  if(WITH_SSCP_COMPILER)
    target_compile_definitions(rt-backend-cuda PRIVATE -DHIPSYCL_WITH_SSCP_COMPILER)
    target_link_libraries(rt-backend-cuda PRIVATE llvm-to-ptx)
  endif()

  if(is_ipo_supported)
    set_property(TARGET rt-backend-cuda PROPERTY INTERPROCEDURAL_OPTIMIZATION True)
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
    target_link_libraries(rt-backend-hip PRIVATE acpp-rt  hip::host)
  endif()

  target_compile_options(rt-backend-hip PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
  target_link_libraries(rt-backend-hip PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS})

  if(WITH_SSCP_COMPILER)
    target_compile_definitions(rt-backend-hip PRIVATE -DHIPSYCL_WITH_SSCP_COMPILER)
    target_link_libraries(rt-backend-hip PRIVATE llvm-to-amdgpu)
  endif()

  if(is_ipo_supported)
    set_property(TARGET rt-backend-hip PROPERTY INTERPROCEDURAL_OPTIMIZATION True)
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
  target_link_libraries(rt-backend-ze PRIVATE acpp-rt -lze_loader)

  target_compile_options(rt-backend-ze PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
  target_link_libraries(rt-backend-ze PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS})

  if(WITH_SSCP_COMPILER)
    target_compile_definitions(rt-backend-ze PRIVATE -DHIPSYCL_WITH_SSCP_COMPILER)
    target_link_libraries(rt-backend-ze PRIVATE llvm-to-spirv)
  endif()

  if(is_ipo_supported)
    set_property(TARGET rt-backend-ze PROPERTY INTERPROCEDURAL_OPTIMIZATION True)
  endif()

  install(TARGETS rt-backend-ze
        LIBRARY DESTINATION lib/hipSYCL
        ARCHIVE DESTINATION lib/hipSYCL)
endif()

if(WITH_OPENCL_BACKEND)
  include(FetchContent)
  FetchContent_Declare(ocl-headers
    GIT_REPOSITORY https://github.com/KhronosGroup/OpenCL-Headers
    GIT_TAG main
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
  )
  FetchContent_GetProperties(ocl-headers)
  if(NOT ocl-headers_POPULATED)
    FetchContent_Populate(ocl-headers)
    add_subdirectory(${ocl-headers_SOURCE_DIR} ${ocl-headers_BINARY_DIR} EXCLUDE_FROM_ALL)
  endif()
  add_library(ocl-headers INTERFACE)
  target_include_directories(ocl-headers INTERFACE ${ocl-headers_SOURCE_DIR})
  
  FetchContent_Declare(ocl-cxx-headers
    GIT_REPOSITORY https://github.com/KhronosGroup/OpenCL-CLHPP
    GIT_TAG main
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
  )
  FetchContent_GetProperties(ocl-cxx-headers)
  if(NOT ocl-cxx-headers_POPULATED)
    FetchContent_Populate(ocl-cxx-headers)
    add_subdirectory(${ocl-cxx-headers_SOURCE_DIR} ${ocl-cxx-headers_BINARY_DIR} EXCLUDE_FROM_ALL)
  endif()
  add_library(ocl-cxx-headers INTERFACE)
  target_include_directories(ocl-cxx-headers INTERFACE ${ocl-cxx-headers_SOURCE_DIR}/include)

  add_library(rt-backend-ocl SHARED
    ocl/ocl_backend.cpp
    ocl/ocl_code_object.cpp
    ocl/ocl_hardware_manager.cpp
    ocl/ocl_allocator.cpp
    ocl/ocl_usm.cpp
    ocl/ocl_event.cpp
    ocl/ocl_queue.cpp)

  target_include_directories(rt-backend-ocl PRIVATE ${HIPSYCL_SOURCE_DIR}/include)
  target_link_libraries(rt-backend-ocl PRIVATE acpp-rt ${OpenCL_LIBRARIES} ocl-headers ocl-cxx-headers)

  target_compile_options(rt-backend-ocl PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
  target_link_libraries(rt-backend-ocl PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS})
  target_compile_definitions(rt-backend-ocl PRIVATE -DCL_HPP_TARGET_OPENCL_VERSION=210)

  if(WITH_SSCP_COMPILER)
    target_compile_definitions(rt-backend-ocl PRIVATE -DHIPSYCL_WITH_SSCP_COMPILER)
    target_link_libraries(rt-backend-ocl PRIVATE llvm-to-spirv)
  endif()

  if(is_ipo_supported)
    set_property(TARGET rt-backend-ocl PROPERTY INTERPROCEDURAL_OPTIMIZATION True)
  endif()

  install(TARGETS rt-backend-ocl
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

    target_include_directories(rt-backend-omp PRIVATE
      ${HIPSYCL_SOURCE_DIR}/include 
      ${PROJECT_BINARY_DIR}/include)

    if(WITH_SSCP_COMPILER)
      target_sources(rt-backend-omp PRIVATE omp/omp_code_object.cpp)
      target_compile_definitions(rt-backend-omp PRIVATE -DHIPSYCL_WITH_SSCP_COMPILER)
      target_link_libraries(rt-backend-omp PRIVATE llvm-to-host)
    endif()

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

  target_link_libraries(rt-backend-omp PRIVATE acpp-rt  OpenMP::OpenMP_CXX)

  target_compile_options(rt-backend-omp PRIVATE ${HIPSYCL_RT_EXTRA_CXX_FLAGS})
  target_link_libraries(rt-backend-omp PRIVATE ${HIPSYCL_RT_EXTRA_LINKER_FLAGS})

  if(is_ipo_supported)
    set_property(TARGET rt-backend-omp PROPERTY INTERPROCEDURAL_OPTIMIZATION True)
  endif()

  install(TARGETS rt-backend-omp
      RUNTIME DESTINATION bin/hipSYCL
      LIBRARY DESTINATION lib/hipSYCL
      ARCHIVE DESTINATION lib/hipSYCL)
endif()

