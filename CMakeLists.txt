set(_PROTOBUF_PROTOC protoc)
set(_GRPC_CPP_PLUGIN_EXECUTABLE /usr/bin/grpc_cpp_plugin)

file(MAKE_DIRECTORY "generated")
file(MAKE_DIRECTORY "generated/python")

set(GRPC_GENERATED_DIR "${CMAKE_CURRENT_SOURCE_DIR}/generated/")
set(GRPC_PYTHON_DIR "${CMAKE_CURRENT_SOURCE_DIR}/generated/python")

file(GLOB interfaces "*.proto")


include(../cmake/piradio.cmake)

piradio_project("piradio-interfaces" INTERFACES)


foreach(interface IN ITEMS ${interfaces})
  get_filename_component(_proto "${interface}" ABSOLUTE)
  get_filename_component(_proto_path "${_proto}" DIRECTORY)
  get_filename_component(_proto_name "${_proto}" NAME_WLE)

  list(APPEND _protocols ${_proto_name})
  
  # Generated sources
  set(${_proto_name}_proto_srcs "${GRPC_GENERATED_DIR}/${_proto_name}.pb.cc")
  set(${_proto_name}_proto_hdrs "${GRPC_GENERATED_DIR}/${_proto_name}.pb.h")
  set(${_proto_name}_grpc_srcs "${GRPC_GENERATED_DIR}/${_proto_name}.grpc.pb.cc")
  set(${_proto_name}_grpc_hdrs "${GRPC_GENERATED_DIR}/${_proto_name}.grpc.pb.h")

  set(${_proto_name}_library ${_proto_name}_grpc_proto CACHE INTERNAL "${_proto_name}_library")

  set(${_proto_name}_python_source ${GRPC_PYTHON_DIR}/${_proto_name}_pb2.py ${GRPC_PYTHON_DIR}/${_proto_name}_pb2_grpc.py CACHE INTERNAL "${_proto_name}_python_source")
  
  message("GRPC Protocol Definition: ${_proto_name}")
  message("Python sources: ${${_proto_name}_python_source}")

  add_custom_command(
    OUTPUT "${${_proto_name}_proto_srcs}" "${${_proto_name}_proto_hdrs}" "${${_proto_name}_grpc_srcs}" "${${_proto_name}_grpc_hdrs}"
    COMMAND ${_PROTOBUF_PROTOC}
    ARGS --grpc_out "${GRPC_GENERATED_DIR}"
    --cpp_out "${GRPC_GENERATED_DIR}"
    -I "${_proto_path}"
    --plugin=protoc-gen-grpc="${_GRPC_CPP_PLUGIN_EXECUTABLE}"
    "${_proto}"
    DEPENDS "${_proto}"
    )

  
  #link_libraries("grpc++_reflection" "grpc" "protobuf")
  add_library(${${_proto_name}_library} STATIC
    ${${_proto_name}_grpc_srcs}
    ${${_proto_name}_grpc_hdrs}
    ${${_proto_name}_proto_srcs}
    ${${_proto_name}_proto_hdrs})

  add_custom_target(${_proto_name}_python_source ALL
    python3 -m grpc_tools.protoc -I "${_proto_path}" --python_out=${GRPC_PYTHON_DIR} --grpc_python_out=${GRPC_PYTHON_DIR} ${_proto}
    BYPRODUCTS ${${_proto_name}_python_source}
  )
  
  target_link_options(${${_proto_name}_library} PRIVATE "LINKER:-no-as-needed")
  target_include_directories(${${_proto_name}_library} PUBLIC ${GRPC_GENERATED_DIR})

  #link_libraries("${_proto_name}_grpc_proto")
endforeach()

set(protocols ${_protocols} CACHE INTERNAL "protocols")

macro(target_add_protocol target proto_name)
  message("Defined protocols: ${protocols}")
  message("Adding protocol ${proto_name} to ${target}: library ${${proto_name}_library}")
  target_link_libraries(${target} ${${proto_name}_library})
endmacro()
  
macro(python_add_protocol proto_name)
  get_cmake_property(_variableNames VARIABLES)
  list (SORT _variableNames)
  foreach (_variableName ${_variableNames})
    message(STATUS "${_variableName}=${${_variableName}}")
  endforeach()
  set(_tmpvar ${${proto_name}_python_source})
  message("Copying: ${_tmpvar} to ${CMAKE_CURRENT_SOURCE_DIR}")
  file(COPY ${${proto_name}_python_source} DESTINATION "${CMAKE_CURRENT_SOURCE_DIR}")
endmacro()
