# Native Inkling 2 engine from Project Maya (MIT). The host expert kernels use x86 intrinsics.
if(NOT CMAKE_SYSTEM_NAME STREQUAL "Linux" OR NOT CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
  message(FATAL_ERROR "strata-ik2 requires Linux x86-64")
endif()

if(STRATA_ENABLE_CUDA)
  set(ik2_archs)
  foreach(arch IN LISTS CMAKE_CUDA_ARCHITECTURES)
    if(arch MATCHES "^90(-real|-virtual)?$")
      list(APPEND ik2_archs "90a${CMAKE_MATCH_1}")
    elseif(arch MATCHES "^([0-9]+)a?(-real|-virtual)?$")
      if(CMAKE_MATCH_1 GREATER_EQUAL 70)
        list(APPEND ik2_archs "${arch}")
      endif()
    else()
      message(FATAL_ERROR "For strata-ik2, specify numeric CUDA architectures (e.g. 86)")
    endif()
  endforeach()
  if(NOT ik2_archs)
    message(FATAL_ERROR "strata-ik2 requires Volta (7.0) or newer CUDA compute capability")
  endif()
  set(CMAKE_CUDA_ARCHITECTURES ${ik2_archs})
  set(CMAKE_CUDA_STANDARD 17)
  set(CMAKE_CUDA_STANDARD_REQUIRED ON)
endif()

find_package(Threads REQUIRED)
if(STRATA_ENABLE_HIP)
  set_source_files_properties(
    ik2_kernels.cu ik2_off.cu ik2_main.cu ik2_draft.cu compat/ds4/ds4_lt.cu
    PROPERTIES LANGUAGE HIP)
endif()

add_library(ik2_kernels STATIC ik2_kernels.cu ik2_off.cu)
target_include_directories(ik2_kernels PUBLIC
  ${_strata_gpu_include_directories}
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${PROJECT_SOURCE_DIR}/include
  ${PROJECT_SOURCE_DIR}/third_party/ggml)
target_link_libraries(ik2_kernels PUBLIC ${_strata_gpu_runtime_target})

set(_ik2_lane_sources
  ik2_lane.cpp
  compat/hy4/hy4_lane.cpp
  compat/ds4/ds4_iq_cpu.cpp
  compat/ds4/ds4_gguf_cpu.cpp)
set_source_files_properties(ik2_lane.cpp PROPERTIES COMPILE_OPTIONS "-ffp-contract=off")

add_executable(strata-ik2 ik2_main.cu ik2_draft.cu ik2_cpu.cpp ${_ik2_lane_sources})
target_include_directories(strata-ik2 PRIVATE
  ${_strata_gpu_include_directories}
  ${PROJECT_SOURCE_DIR}/include
  ${CMAKE_CURRENT_SOURCE_DIR}/compat/ds4
  ${CMAKE_CURRENT_SOURCE_DIR}/compat/hy4
  ${CMAKE_CURRENT_SOURCE_DIR}/compat/mm3
  ${PROJECT_SOURCE_DIR}/third_party/ggml)
target_link_libraries(strata-ik2 PRIVATE
  ik2_kernels
  strata_kernels_cpu
  strata_spec
  ${_strata_gpu_runtime_target}
  ${_strata_gpu_blas_target}
  Threads::Threads)
target_compile_definitions(strata-ik2 PRIVATE MAYA_DS4_IQ_ONLY DS4_IQ_MIN_FORMS)

if(STRATA_ENABLE_HIP)
  target_sources(strata-ik2 PRIVATE compat/ds4/ds4_lt.cu)
  if(STRATA_HIPBLASLT_AVAILABLE)
    set_source_files_properties(compat/ds4/ds4_lt.cu PROPERTIES COMPILE_DEFINITIONS STRATA_HIPBLASLT_AVAILABLE=1)
    target_link_libraries(strata-ik2 PRIVATE roc::hipblaslt)
  endif()
endif()

find_package(OpenMP QUIET COMPONENTS CXX)
if(OpenMP_CXX_FOUND)
  target_sources(strata-ik2 PRIVATE ik2_cpu_deep.cpp)
  set_source_files_properties(ik2_cpu_deep.cpp PROPERTIES COMPILE_OPTIONS "-mavx2;-mfma;-mf16c;-ffp-contract=off")
  target_compile_definitions(strata-ik2 PRIVATE IK2_HOST_AVX2=1)
  target_link_libraries(strata-ik2 PRIVATE OpenMP::OpenMP_CXX)
endif()

if(STRATA_BUILD_TESTS)
  if(STRATA_ENABLE_HIP)
    set_source_files_properties(../../tests/ik2_kernels_test.cu PROPERTIES LANGUAGE HIP)
  endif()
  add_executable(ik2_kernels_test ../../tests/ik2_kernels_test.cu)
  target_link_libraries(ik2_kernels_test PRIVATE ik2_kernels)
  add_test(NAME ik2_kernels_test COMMAND ik2_kernels_test)
  set_tests_properties(ik2_kernels_test PROPERTIES TIMEOUT 300)
endif()
