CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
PROJECT(libgraphqlparser C CXX)

SET(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" "${CMAKE_MODULE_PATH}")

INCLUDE(version)

SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11")

FIND_PACKAGE(PythonInterp 2)
IF (NOT PYTHON_VERSION_MAJOR EQUAL 2)
  MESSAGE(WARNING "Python 2 is required.")
ELSE()
  FIND_PROGRAM(CTYPESGEN_FOUND ctypesgen.py)
ENDIF()


FIND_PACKAGE(BISON 3)
FIND_PACKAGE(FLEX)
IF (BISON_FOUND)
  BISON_TARGET(graphqlparser_bison parser.ypp ${CMAKE_CURRENT_BINARY_DIR}/parser.tab.cpp)
  SET(BISON_LOCATION_HEADER ${CMAKE_CURRENT_BINARY_DIR}/location.hh)
ELSE()
  SET(BISON_graphqlparser_bison_OUTPUT_SOURCE ${CMAKE_CURRENT_BINARY_DIR}/parser.tab.cpp)
  SET(BISON_graphqlparser_bison_OUTPUT_HEADER ${CMAKE_CURRENT_BINARY_DIR}/parser.tab.hpp)
  SET(BISON_graphqlparser_bison_OUTPUTS
    ${BISON_graphqlparser_bison_OUTPUT_SOURCE}
    ${BISON_graphqlparser_bison_OUTPUT_HEADER}
    ${CMAKE_CURRENT_BINARY_DIR}/location.hh
    ${CMAKE_CURRENT_BINARY_DIR}/position.hh
    ${CMAKE_CURRENT_BINARY_DIR}/stack.hh)
  SET(BISON_LOCATION_HEADER ${CMAKE_CURRENT_BINARY_DIR}/location.hh)
  FILE(COPY
    ${CMAKE_CURRENT_SOURCE_DIR}/parsergen/parser.tab.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/parsergen/parser.tab.hpp
    ${CMAKE_CURRENT_SOURCE_DIR}/parsergen/location.hh
    ${CMAKE_CURRENT_SOURCE_DIR}/parsergen/position.hh
    ${CMAKE_CURRENT_SOURCE_DIR}/parsergen/stack.hh
    DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
ENDIF()

IF(FLEX_FOUND)
  FLEX_TARGET(GraphQLScanner lexer.lpp ${CMAKE_CURRENT_BINARY_DIR}/lexer.cpp COMPILE_FLAGS "--header-file=${CMAKE_CURRENT_BINARY_DIR}/lexer.h")
  IF (BISON_FOUND)
    ADD_FLEX_BISON_DEPENDENCY(GraphQLScanner graphqlparser_bison)
  ENDIF()
ELSE()
  SET(FLEX_GraphQLScanner_OUTPUTS ${CMAKE_CURRENT_BINARY_DIR}/lexer.cpp)
  FILE(COPY
    ${CMAKE_CURRENT_SOURCE_DIR}/parsergen/lexer.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/parsergen/lexer.h
    DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
ENDIF()

FILE(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/c)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})

SET(graphqlparser_libsrc
  JsonVisitor.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/Ast.h
  ${CMAKE_CURRENT_BINARY_DIR}/Ast.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/AstVisitor.h
  ${CMAKE_CURRENT_BINARY_DIR}/c/GraphQLAst.h
  ${CMAKE_CURRENT_BINARY_DIR}/c/GraphQLAst.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/c/GraphQLAstForEachConcreteType.h
  ${CMAKE_CURRENT_BINARY_DIR}/JsonVisitor.h.inc
  ${CMAKE_CURRENT_BINARY_DIR}/JsonVisitor.cpp.inc
  ${BISON_graphqlparser_bison_OUTPUTS}
  ${FLEX_GraphQLScanner_OUTPUTS}
  c/GraphQLAstNode.cpp
  c/GraphQLAstToJSON.cpp
  c/GraphQLAstVisitor.h
  c/GraphQLAstVisitor.cpp
  c/GraphQLParser.cpp
  GraphQLParser.cpp)


ADD_LIBRARY(graphqlparser_obj    OBJECT ${graphqlparser_libsrc})
ADD_LIBRARY(graphqlparser_shared SHARED $<TARGET_OBJECTS:graphqlparser_obj>)
ADD_LIBRARY(graphqlparser_static STATIC $<TARGET_OBJECTS:graphqlparser_obj>)

# add -fPIC, but only for the .so:
SET_PROPERTY(TARGET graphqlparser_obj    PROPERTY POSITION_INDEPENDENT_CODE ON)
SET_PROPERTY(TARGET graphqlparser_shared PROPERTY POSITION_INDEPENDENT_CODE ON)

SET_TARGET_PROPERTIES(graphqlparser_shared PROPERTIES OUTPUT_NAME graphqlparser CLEAN_DIRECT_OUTPUT 1)
SET_TARGET_PROPERTIES(graphqlparser_static PROPERTIES OUTPUT_NAME graphqlparser CLEAN_DIRECT_OUTPUT 1)

# Enable this and remove CMAKE_CXX_FLAGS fiddle above when we are able
# to upgrade to CMake 2.8.12. Blocker seems to be Travis CI being on
# Ubuntu Precise; Trusty has 2.8.12.
# TARGET_COMPILE_OPTIONS(graphqlparser_shared PUBLIC -std=gnu++11)

ADD_EXECUTABLE(dump_json_ast dump_json_ast.cpp)
TARGET_LINK_LIBRARIES(dump_json_ast graphqlparser_shared)

FUNCTION(GENERATE_AST_FILE FILE_TYPE FILE_RELATIVE_PATH)
  ADD_CUSTOM_COMMAND(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${FILE_RELATIVE_PATH}
    COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.py ${FILE_TYPE} ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.ast > ${CMAKE_CURRENT_BINARY_DIR}/${FILE_RELATIVE_PATH}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.ast ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.py ${CMAKE_CURRENT_SOURCE_DIR}/ast/${FILE_TYPE}.py)
ENDFUNCTION(GENERATE_AST_FILE)

IF(PythonInterp_FOUND)
  GENERATE_AST_FILE(cxx Ast.h)
  GENERATE_AST_FILE(cxx_visitor AstVisitor.h)
  GENERATE_AST_FILE(cxx_impl Ast.cpp)
  GENERATE_AST_FILE(c c/GraphQLAst.h)
  GENERATE_AST_FILE(c_impl c/GraphQLAst.cpp)
  GENERATE_AST_FILE(c_visitor_impl c/GraphQLAstForEachConcreteType.h)
  GENERATE_AST_FILE(cxx_json_visitor_header JsonVisitor.h.inc)
  GENERATE_AST_FILE(cxx_json_visitor_impl JsonVisitor.cpp.inc)
ELSE()
  FILE(COPY
    ${CMAKE_CURRENT_SOURCE_DIR}/pregenerated_ast/Ast.h
    ${CMAKE_CURRENT_SOURCE_DIR}/pregenerated_ast/Ast.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/pregenerated_ast/AstVisitor.h
    ${CMAKE_CURRENT_SOURCE_DIR}/pregenerated_ast/JsonVisitor.h.inc
    ${CMAKE_CURRENT_SOURCE_DIR}/pregenerated_ast/JsonVisitor.cpp.inc
    DESTINATION ${CMAKE_CURRENT_SOURCE_DIR})
  FILE(COPY
    ${CMAKE_CURRENT_SOURCE_DIR}/pregenerated_ast/c/GraphQLAst.h
    ${CMAKE_CURRENT_SOURCE_DIR}/pregenerated_ast/c/GraphQLAst.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/pregenerated_ast/c/GraphQLAstForEachConcreteType.h
    DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/c)
ENDIF()

ADD_SUBDIRECTORY(python)

OPTION(test "Build tests." OFF)

INSTALL(DIRECTORY c ${CMAKE_CURRENT_BINARY_DIR}/c DESTINATION include/graphqlparser
  FILES_MATCHING PATTERN "*.h"
  PATTERN "build" EXCLUDE)

INSTALL(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/Ast.h
  AstNode.h
  ${CMAKE_CURRENT_BINARY_DIR}/AstVisitor.h
  GraphQLParser.h
  JsonVisitor.h
  ${BISON_LOCATION_HEADER}
  DESTINATION include/graphqlparser)
INSTALL(TARGETS
    graphqlparser_shared
    graphqlparser_static
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)

if (UNIX)
  # generate pkgconfig file
  include(FindPkgConfig QUIET)
  if(PKG_CONFIG_FOUND)
    # generate .pc and install
    configure_file("libgraphqlparser.pc.in" "libgraphqlparser.pc" @ONLY)
    install(FILES       "${CMAKE_CURRENT_BINARY_DIR}/libgraphqlparser.pc"
            DESTINATION "${CMAKE_INSTALL_PREFIX}/lib/pkgconfig")
  endif()
endif()

IF (test)
  ADD_SUBDIRECTORY(test)

  if(UNIX)
    # setup valgrind
    ADD_CUSTOM_TARGET(memcheck
      valgrind --leak-check=full --suppressions=./test/valgrind.supp --dsymutil=yes --error-exitcode=1 ./test/runTests  >/dev/null
    )
  endif()

ENDIF()
