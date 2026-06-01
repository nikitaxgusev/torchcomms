# Copyright (c) Meta Platforms, Inc. and affiliates.
# Extension: torchcomms._comms_rcclx
file(GLOB TORCHCOMMS_RCCLX_SOURCES "comms/torchcomms/rcclx/*.cpp")

set(CONDA_INCLUDE "${CONDA_PREFIX}/include")
set(ROCM_HOME $ENV{ROCM_HOME})
if(NOT ROCM_HOME)
    set(ROCM_HOME "/opt/rocm")
endif()
set(ROCM_INCLUDE "${ROCM_HOME}/include")
set(RCCLX_INCLUDE $ENV{RCCLX_INCLUDE})
message(STATUS "RCCLX_INCLUDE is set to ${RCCLX_INCLUDE}")
set(RCCLX_SHARED_LIB $ENV{RCCLX_LIB}/librccl.so)
message(STATUS "RCCLX_SHARED_LIB is set to ${RCCLX_SHARED_LIB}")

# RcclxApiShardedRelay.cpp uses ncclx::Hints, ncclx::allGatherInit,
# ncclx::allGatherExec, ncclx::pFree and ncclShardedRelayMultiGroupAllReduce —
# these are only declared by the rcclx-dev variant of <rccl.h>. The stub TU
# (RcclxApiShardedRelayStub.cpp) provides ncclInternalError fallbacks for the
# frozen rcclx-stable / rcclx-last-stable / rccl-dev snapshots. BUCK picks
# exactly one of the two via select() on the active rccl_constraint; mirror
# that here by detecting the rcclx-dev variant via a marker header it
# installs (nccl_device/core.h, the same marker the NCCLX extension already
# uses to gate its dev-only sources). rcclx-dev ships both the nccl_device/*
# headers and the inline ncclx::* declarations in rccl.h together, so the
# marker also gates the sharded-relay symbols correctly.
if(EXISTS "${RCCLX_INCLUDE}/nccl_device/core.h")
    message(STATUS
        "rcclx-dev API headers found, linking RcclxApiShardedRelay.cpp")
    list(FILTER TORCHCOMMS_RCCLX_SOURCES
        EXCLUDE REGEX "RcclxApiShardedRelayStub\\.cpp$")
else()
    message(STATUS
        "rcclx-dev API headers NOT found,"
        " linking RcclxApiShardedRelayStub.cpp")
    list(FILTER TORCHCOMMS_RCCLX_SOURCES
        EXCLUDE REGEX "RcclxApiShardedRelay\\.cpp$")
endif()

add_library(torchcomms_comms_rcclx MODULE ${TORCHCOMMS_RCCLX_SOURCES})
set_target_properties(torchcomms_comms_rcclx PROPERTIES
    PREFIX ""
    OUTPUT_NAME "_comms_rcclx"
    SUFFIX ".${Python3_SOABI}.so"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/comms/torchcomms"
)

target_include_directories(torchcomms_comms_rcclx PRIVATE
    ${ROOT}
    ${RCCLX_INCLUDE}
    ${CONDA_INCLUDE}
    ${Python3_INCLUDE_DIRS}
    ${ROCM_INCLUDE}
    ${ROCM_INCLUDE}/hip
)
target_compile_features(torchcomms_comms_rcclx PRIVATE cxx_std_20)
target_link_directories(torchcomms_comms_rcclx PRIVATE ${CONDA_LIB})
target_link_libraries(torchcomms_comms_rcclx PRIVATE
    torchcomms
    ${TORCH_LIBRARIES}
    ${TORCH_PYTHON_LIB}
    ${RCCLX_SHARED_LIB}
)

set(CMAKE_CXX_COMPILE_OBJECT "${CMAKE_CXX_COMPILE_OBJECT} -std=c++20")

install(TARGETS torchcomms_comms_rcclx
        LIBRARY DESTINATION .
)
