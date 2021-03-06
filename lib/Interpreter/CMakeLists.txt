#------------------------------------------------------------------------------
# CLING - the C++ LLVM-based InterpreterG :)
#
# This file is dual-licensed: you can choose to license it under the University
# of Illinois Open Source License or the GNU Lesser General Public License. See
# LICENSE.TXT for details.
#------------------------------------------------------------------------------

set(LIBS
  clangDriver
  clangFrontend
  clangParse
  clangSema
  clangAST
  clangLex
  clangSerialization
  clangCodeGen
  clangBasic

  clingUtils
)

find_library(DL_LIBRARY_PATH dl)
if (DL_LIBRARY_PATH)
  list(APPEND LIBS dl)
endif()


set( LLVM_LINK_COMPONENTS
  analysis
  core
  executionengine
  ipo
  mc
  native
  nativecodegen
  object
  option
  orcjit
  runtimedyld
  support
  target
)

add_cling_library(clingInterpreter OBJECT
  AutoSynthesizer.cpp
  AutoloadCallback.cpp
  ASTTransformer.cpp
  BackendPasses.cpp
  CheckEmptyTransactionTransformer.cpp
  CIFactory.cpp
  ClangInternalState.cpp
  ClingCodeCompleteConsumer.cpp
  ClingPragmas.cpp
  DeclCollector.cpp
  DeclExtractor.cpp
  DeclUnloader.cpp
  DynamicLibraryManager.cpp
  DynamicLookup.cpp
  DynamicExprInfo.cpp
  Exception.cpp
  ExceptionRTTI.cpp
  ExternalInterpreterSource.cpp
  ForwardDeclPrinter.cpp
  IncrementalExecutor.cpp
  IncrementalJIT.cpp
  IncrementalParser.cpp
  Interpreter.cpp
  InterpreterCallbacks.cpp
  InvocationOptions.cpp
  LookupHelper.cpp
  NullDerefProtectionTransformer.cpp
  RequiredSymbols.cpp
  Transaction.cpp
  TransactionUnloader.cpp
  ValueExtractionSynthesizer.cpp
  Value.cpp
  ValuePrinter.cpp
  ValuePrinterSynthesizer.cpp

  LINK_LIBS
  ${LIBS}
)

if(NOT WIN32)
set_source_files_properties(ExceptionRTTI.cpp COMPILE_FLAGS "-fexceptions -frtti")
endif()

#set_source_files_properties(Exception.cpp COMPILE_FLAGS " /EHsc ")
# the line above doesn't work, and it gives the following warnings:
# cl : Command line warning D9025: overriding '/EHs' with '/EHs-'
# cl : Command line warning D9025: overriding '/EHc' with '/EHc-'
# so let's change the target compiler flags instead:
if(MSVC)
  add_definitions(-UUNICODE -U_UNICODE)
  get_target_property(CI_COMPILE_FLAGS clingInterpreter COMPILE_FLAGS)
  if(${CI_COMPILE_FLAGS})
    string(REPLACE "/EHs-c-" "" CI_COMPILE_FLAGS ${CI_COMPILE_FLAGS})
    # add /EHsc exception handling flag
    set(CI_COMPILE_FLAGS "${CI_COMPILE_FLAGS} /EHsc")
  else()
    set(CI_COMPILE_FLAGS "/EHsc")
  endif()
  set_target_properties(clingInterpreter PROPERTIES
    COMPILE_FLAGS ${CI_COMPILE_FLAGS})
endif()

#add_dependencies(clangDriver ClangAttrList ClangDiagnosticDriver
#                 ClangDriverOptions ClangCC1Options ClangCC1AsOptions)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h
                   COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h.in 
                                                                 ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h)

add_file_dependencies(${CMAKE_CURRENT_SOURCE_DIR}/CIFactory.cpp ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h)
add_file_dependencies(${CMAKE_CURRENT_SOURCE_DIR}/Interpreter.cpp ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h)

# Remove all -I from CMAKE_CXX_FLAGS
string(REPLACE ";" " " __flags "${CMAKE_CXX_FLAGS}")
string(REGEX REPLACE "-I[^ ]+" "" CMAKE_CXX_FLAGS_NO_I "${__flags}")

# Remove absolute path from CMAKE_CXX_COMPILER
get_filename_component(_path ${CMAKE_CXX_COMPILER} PATH)
get_filename_component(_name ${CMAKE_CXX_COMPILER} NAME)
if("$ENV{PATH}" MATCHES ${_path})
  set(CMAKE_CXX_COMPILER_RELATIVE ${_name})
else()
  set(CMAKE_CXX_COMPILER_RELATIVE ${CMAKE_CXX_COMPILER})
endif()

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/cling-compiledata.h.in
  "#define LLVM_CXX \"${CMAKE_CXX_COMPILER_RELATIVE} ${CMAKE_CXX_FLAGS_NO_I} ${CMAKE_CXX_FLAGS_${uppercase_CMAKE_BUILD_TYPE}}\"
"
)
