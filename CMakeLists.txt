cmake_minimum_required(VERSION 3.10)
project(IDTECH4-DCF-PLUGIN)

# Find dependencies
find_package(Protobuf REQUIRED)
find_package(gRPC CONFIG REQUIRED)

# Core plugin library
add_library(idtech4_dcf_plugin
    src/dcf/idDCFNetwork.cpp
    src/dcf/dcf_transport.cpp
)
target_include_directories(idtech4_dcf_plugin PRIVATE include)
target_link_libraries(idtech4_dcf_plugin PRIVATE protobuf::libprotobuf gRPC::grpc++)

# WASM-specific flags
if(EMSCRIPTEN)
    target_compile_definitions(idtech4_dcf_plugin PRIVATE EMSCRIPTEN=1)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -s USE_PTHREADS=0")
endif()

# Hub server executable
add_executable(dcf_hub hub/dcf_hub.cpp)
target_include_directories(dcf_hub PRIVATE include)
target_link_libraries(dcf_hub PRIVATE protobuf::libprotobuf gRPC::grpc++)

# Tests
enable_testing()
add_executable(test_network tests/unit/test_network.cpp)
target_link_libraries(test_network PRIVATE idtech4_dcf_plugin gtest gtest_main)
add_test(NAME unit_tests COMMAND test_network)

# Proto generation
add_custom_command(
    OUTPUT ${CMAKE_SOURCE_DIR}/include/dcf/messages.pb.cc ${CMAKE_SOURCE_DIR}/include/dcf/services.grpc.pb.cc
    COMMAND protoc --cpp_out=${CMAKE_SOURCE_DIR}/include/dcf --grpc_out=${CMAKE_SOURCE_DIR}/include/dcf --plugin=protoc-gen-grpc=grpc_cpp_plugin ${CMAKE_SOURCE_DIR}/proto/*.proto
    DEPENDS ${CMAKE_SOURCE_DIR}/proto/messages.proto ${CMAKE_SOURCE_DIR}/proto/services.proto
)