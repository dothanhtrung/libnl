cmake_minimum_required(VERSION 3.22)

project(libnl C)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

set(CMAKE_INSTALL_RPATH "$ORIGIN")
set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)

# Just for generating config files for libnl
set(AUTOTOOLS_HOST aarch64-linux-android33)
if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/lib/defs.h)
    execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/autogen.sh --host=${HOST}
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
    execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/configure --host=${HOST}
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
endif()

include(GNUInstallDirs)
add_definitions(
        -DNDK_BUILD=1
        -DSYSCONFDIR="${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_SYSCONFDIR}/libnl"
        -DPKGLIBDIR="${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}"
)

find_package(FLEX REQUIRED)
find_package(BISON REQUIRED)

FLEX_TARGET(ematch_grammar ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/cls/ematch_grammar.l
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/cls/ematch_grammar.c
        DEFINES_FILE ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/cls/ematch_grammar.h)
FLEX_TARGET(pktloc_grammar ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/pktloc_grammar.l
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/pktloc_grammar.c
        DEFINES_FILE ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/pktloc_grammar.h)
BISON_TARGET(ematch_syntax ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/cls/ematch_syntax.y
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/cls/ematch_syntax.c
        DEFINES_FILE ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/cls/ematch_syntax.h)
BISON_TARGET(pktloc_syntax ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/pktloc_syntax.y
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/pktloc_syntax.c
        DEFINES_FILE ${CMAKE_CURRENT_SOURCE_DIR}/lib/route/pktloc_syntax.h)
ADD_FLEX_BISON_DEPENDENCY(ematch_grammar ematch_syntax)

include_directories(
        include
        include/linux-private
        lib
        lib/route
)

file(GLOB NL_SOURCES lib/*.c)
add_library(nl SHARED ${NL_SOURCES})

file(GLOB_RECURSE NL_ROUTE_SOURCES lib/route/*.c)
list(FILTER NL_ROUTE_SOURCES EXCLUDE REGEX ".*lib/route/cls/ematch_.*\.c$|.*lib/route/pktloc_.*\.c")
add_library(nl-route SHARED
        lib/fib_lookup/lookup.c
        lib/fib_lookup/request.c
        ${NL_ROUTE_SOURCES}
        ${FLEX_ematch_grammar_OUTPUTS}
        ${FLEX_pktloc_grammar_OUTPUTS}
        ${BISON_ematch_syntax_OUTPUTS}
        ${BISON_pktloc_syntax_OUTPUTS}
)
target_link_libraries(nl-route nl)

file(GLOB NL_NF_SOURCES lib/netfilter/*.c)
add_library(nl-nf SHARED ${NL_NF_SOURCES})
target_link_libraries(nl-nf nl nl-route)

add_library(nl-cli SHARED src/lib/addr.c
        src/lib/class.c
        src/lib/cls.c
        src/lib/ct.c
        src/lib/exp.c
        src/lib/link.c
        src/lib/neigh.c
        src/lib/qdisc.c
        src/lib/route.c
        src/lib/rule.c
        src/lib/tc.c
        src/lib/utils.c
)
target_link_libraries(nl-cli nl nl-route nl-nf)