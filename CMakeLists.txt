cmake_minimum_required(VERSION 3.16)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/scc/cmake)

include(ConanInline)

project(ecosystem-vp LANGUAGES C CXX VERSION 1.0.0)

option(ENABLE_SCV "Enable the use of SCV" ON)
option(LIBS_ONLY "Just build the shared libraries needed to build the VP" OFF)
option(NO_FW_BUILD "Disable the automatic rebuild of firmware running on the VP" OFF)
option(EN_EXT_DEBUG "Enable extended debug output at runtime" OFF)
option(ENABLE_COVERAGE "Enable code coverage" OFF)
option(ENABLE_SANITIZER "Enable address sanitizer" OFF)
option(ENABLE_CLANGTIDY "Enable static analysis with clang-tidy." OFF)

set(SCC_LIB_ONLY ON)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
include(CheckCXXCompilerFlag)
include(GNUInstallDirs)

# add address sanitizer
if(ENABLE_SANITIZER)
  set(CMAKE_C_FLAGS             "${CMAKE_C_FLAGS} -fsanitize=address")
  set(CMAKE_CXX_FLAGS           "${CMAKE_CXX_FLAGS} -fsanitize=address")
  set(CMAKE_EXE_LINKER_FLAGS    "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fsanitize=address")
endif()
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
    add_compile_options(/vmg /wd26812) # suppress Warnung C26812: "enum class" gegenüber "enum" (Enum.3) bevorzugen
endif()
set(CONAN_BOOST_OPTIONS
boost:header_only=False 
boost:without_contract=True
boost:without_fiber=True
boost:without_graph=True
boost:without_graph_parallel=True
boost:without_iostreams=True
boost:without_json=True
boost:without_locale=True
boost:without_log=True
boost:without_math=True
boost:without_mpi=True
boost:without_nowide=True
boost:without_python=True
boost:without_random=True
boost:without_regex=True
boost:without_stacktrace=True
boost:without_test=True
boost:without_timer=True
boost:without_type_erasure=True
boost:without_wave=True
)
if (UNIX)
    list(APPEND CONAN_BOOST_OPTIONS boost:fPIC=True boost:shared=True)
endif (UNIX)

set(CONAN_PACKAGE_LIST fmt/8.0.1 zlib/1.2.11 boost/1.75.0 gsl-lite/0.37.0 elfio/3.8)

if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
     list(APPEND CONAN_BOOST_OPTIONS boost:fPIC=True lua:compile_as_cpp=False)
     list(APPEND CONAN_PACKAGE_LIST tcc/0.9.27 seasocks/1.4.4 lua/5.4.3)
endif()
set(CONAN_PACKAGE_OPTIONS fmt:header_only=True ${CONAN_BOOST_OPTIONS})
if(NOT USE_CWR_SYSTEMC AND NOT USE_NCSC_SYSTEMC AND NOT DEFINED ENV{SYSTEMC_HOME})
	set(CONAN_PACKAGE_LIST ${CONAN_PACKAGE_LIST} systemc/2.3.3 systemc-cci/1.0.0) 
	set(CONAN_PACKAGE_OPTIONS ${CONAN_PACKAGE_OPTIONS} systemc:shared=True systemc-cci:shared=True systemc:disable_virtual_bind=False)
endif()

conan_check() 
conan_add_remote(NAME minres URL https://artifactory.minres.com/artifactory/api/conan/oss)
conan_cmake_configure(REQUIRES ${CONAN_PACKAGE_LIST}
                      GENERATORS cmake_find_package
                      OPTIONS ${CONAN_PACKAGE_OPTIONS}
                      )
conan_cmake_autodetect(settings)
conan_install()
# needed when using CentOS devenv-7 or -8 and boost does not build because of missing GLIBCXX version                
#conan_install(BUILD b2 missing)                    
find_package(tcc)
find_package(elfio)
find_package(fmt)
find_package(gsl-lite)
set(Boost_NO_BOOST_CMAKE ON) #  Don't do a find_package in config mode before searching for a regular boost install.


find_package(Boost REQUIRED COMPONENTS program_options QUIET)

set(CLANG_FORMAT_EXCLUDE_PATTERNS "scc")
find_package(ClangFormat)

if(ENABLE_COVERAGE)
    include(CodeCoverage)
    append_coverage_compiler_flags()
    set(COVERAGE_EXCLUDES "${tgfs_verif_SOURCE_DIR}/scc/*" "${tgfs_verif_SOURCE_DIR}/build/*" "$ENV{HOME}/.conan/*" "/opt/*")
endif()

if(ENABLE_CLANGTIDY)
    find_program(CLANG_TIDY_EXE NAMES "clang-tidy")
    if (CLANG_TIDY_EXE)
      	message(STATUS "clang-tidy found: ${CLANG_TIDY_EXE}")
        set(CLANG_TIDY_CHECKS 
        "-*,modernize-*,-modernize-use-trailing-return-type,clang-analyzer-*,concurrency-*,cppcoreguidelines-*,boost-*,bugprone-*,performance-*,portability-*,readability-*")
        #set(CLANG_TIDY_CHECKS "*")
        set(CMAKE_CXX_CLANG_TIDY 
                ${CLANG_TIDY_EXE};
                -header-filter=${tgfs_verif_SOURCE_DIR};
                -checks=${CLANG_TIDY_CHECKS};)
    else()
      	message(AUTHOR_WARNING "clang-tidy not found!")
      	set(CMAKE_CXX_CLANG_TIDY "" CACHE STRING "" FORCE) # delete it
    endif()
endif ()

add_subdirectory(scc)
add_subdirectory(tgc-iss/dbt-rise-core)
add_subdirectory(tgc-iss/dbt-rise-tgc)
if(NOT USE_CWR_SYSTEMC)
	add_subdirectory(vpvper)
endif()
if(NOT LIBS_ONLY)
	add_subdirectory(src)
endif()
if(NOT NO_FW_BUILD)
	add_custom_target(fw-hello-world ALL COMMAND make -C fw/hello-world
                   USES_TERMINAL WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
endif()

# CTest is a testing tool that can be used to test your project.
enable_testing()
add_test(NAME tgc-vp-hello-world
         #WORKING_DIRECTORY ${ecosystem-vp_DIR}/bin
         COMMAND tgc-vp -f ${CMAKE_CURRENT_LIST_DIR}/fw/hello-world/prebuilt/hello.elf)
