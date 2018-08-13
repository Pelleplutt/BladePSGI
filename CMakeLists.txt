cmake_minimum_required (VERSION 2.8)
project (bladepsgi)


#
# BladePSGI_Perl library
#

execute_process(COMMAND perl -MConfig -e print\ \$Config{use64bitint} OUTPUT_VARIABLE PERLCONFIG_USE64BITINT RESULT_VARIABLE PERL_FAILED)
if(NOT PERL_FAILED STREQUAL "0")
    message(FATAL_ERROR "Perl not found")
elseif(NOT PERLCONFIG_USE64BITINT STREQUAL "define")
    message(FATAL_ERROR "Your Perl installation has not been configured with use64bitint")
endif()

execute_process(COMMAND perl -MConfig -e print\ \$Config{use64bitall} OUTPUT_VARIABLE PERLCONFIG_USE64BITINT)
if(NOT PERLCONFIG_USE64BITINT STREQUAL "define")
    message(FATAL_ERROR "Your Perl installation has not been configured with use64bitall")
endif()

execute_process(COMMAND perl -MConfig -e print\ \$Config{ivsize} OUTPUT_VARIABLE PERLCONFIG_USE64BITINT)
if(NOT PERLCONFIG_USE64BITINT STREQUAL "8")
    message(FATAL_ERROR "Your Perl installation has not been configured with a 64-bit ivsize")
endif()

execute_process(COMMAND perl -MConfig -e print\ \$Config{ptrsize} OUTPUT_VARIABLE PERLCONFIG_USE64BITINT)
if(NOT PERLCONFIG_USE64BITINT STREQUAL "8")
    message(FATAL_ERROR "Your Perl installation has not been configured with a 64-bit ptrsize")
endif()

add_library(bladepsgi_perl STATIC src/perl/XS.c src/perl/bladepsgi_perl.c src/perl/typemap)

add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/src/perl/XS.c
    COMMAND xsubpp -typemap typemap src/perl/XS.xs > ${CMAKE_CURRENT_BINARY_DIR}/src/perl/XS.c
    DEPENDS src/perl/XS.xs src/perl/typemap
)

execute_process(COMMAND perl -MExtUtils::Embed -e ccopts OUTPUT_VARIABLE PERLXS_CFLAGS)
string(REGEX REPLACE "\n" "" PERLXS_CFLAGS "${PERLXS_CFLAGS}")
execute_process(COMMAND perl -MExtUtils::Embed -e ldopts OUTPUT_VARIABLE PERLXS_LDFLAGS)
string(REGEX REPLACE "\n" "" PERLXS_LDFLAGS "${PERLXS_LDFLAGS}")
# Remove leading/trailing whitespace as well or CMake will complain; see policy
# CMP0004
string(REGEX REPLACE "(^ +)|( +$)" "" PERLXS_LDFLAGS "${PERLXS_LDFLAGS}")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${PERLXS_CFLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${PERLXS_CFLAGS}")


#
# Main binary
#

file(GLOB bladepsgi_SOURCES
    "src/*.hpp"
    "src/*.cpp"
    "src/perl/*.cpp"
)
add_executable(bladepsgi ${bladepsgi_SOURCES})
target_link_libraries(bladepsgi bladepsgi_perl)

# This is horrible, but there doesn't seem to be a better way.  See
# https://cmake.org/pipermail/cmake/2009-February/026790.html.
target_link_libraries(bladepsgi ${PERLXS_LDFLAGS})

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
if(COMPILER_SUPPORTS_CXX11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
else()
    message(FATAL_ERROR "Your compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -g")


#
# Deb files
#

install(FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/bladepsgi
    DESTINATION /usr/sbin
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
)

set(DEB_PACKAGE_NAME ${CMAKE_PROJECT_NAME})
set(DEB_PACKAGE_VERSION "0.1-1")
set(DEB_PACKAGE_SECTION "utils")
set(DEB_PACKAGE_DESRCIPTION "Expose multiple instances of a PSGI application over a FastCGI socket.\\n\
  While PSGI is a nice interface for writing web applications, the options\\n\
  available for actually running these applications are somewhat scarce.\\n\
  BladePSGI was developed when it become apparent that patching our custom\\n\
  Perl-based runner was a dead end.\\n\
  BladePSGI has a process-based architecture, meaning that each instance of the\\n\
  PSGI application runs in its own, single-threaded process. BladePSGI takes care\\n\
  of forking all backends at startup, and then sits in the background making sure\\n\
  that none of the backends crash, or shutting them down if the server\\n\
  administrator asks for that to happen. It also starts a separate monitoring\\n\
  process which can be used to provide statistics to a separate monitoring\\n\
  system, e.g. Prometheus.\\n\
  For communication with other services, BladePSGI exposes a UNIX domain FastCGI\\n\
  socket. Any (for example) HTTP daemon capable of acting as a FastCGI client can\\n\
  be used to then expose the application to the internet.\\n\
  If the PSGI application author chooses they can ask for \"auxiliary processes\"\\n\
  to be started and run with the application, expose the status of each backend\\n\
  process in shared memory, or share atomics-based semaphores or atomic integers\\n\
  between PSGI application backends.")
set(DEB_PACKAGE_MAINTAINER "Marko Tikkaja <marko@trustly.com>")
set(DEB_PACKAGE_ARCH "amd64")

include(deb_packaging.cmake)
