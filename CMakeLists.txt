cmake_minimum_required(VERSION 2.8)

message(STATUS "Project: respeakerd")
message(STATUS "Project Directory: ${PROJECT_SOURCE_DIR}")

SET(RESPEAKERD_VERSION 1.1.0)
SET(RESPEAKER_SDK_VERSION 2.0.0)
SET(RESPEAKER_SDK_SOVERSION 2)
SET(RESPEAKER_SDK_MINOR_VERSION 2.0)

# the RPATH to be used when installing
SET(CMAKE_INSTALL_RPATH "")

# don't add the automatically determined parts of the RPATH
# which point to directories outside the build tree to the install RPATH
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH FALSE)

include("TargetArch.cmake")
target_architecture(arch)
message("the arch is ${arch}")
string(FIND ${arch} "86" is_x86)
string(FIND ${arch} "arm" is_arm)

ADD_SUBDIRECTORY(src)

