#
# This is an example CMakeLists.txt file to compile a program against mathplot
#
# Mathplot provides all its functionality as modules. What you do is
# compile these modules - the mathplot core - into a library. You then
# compile your own program, importing the mathplot features you need
# and linking to this library.
#
cmake_minimum_required(VERSION 3.28.5 FATAL_ERROR)

# Give your project a name. Specify CXX (for maths, mathplot and your code) and also C (for glad)
project(healpixviewer LANGUAGES CXX C)

# mathplot uses c++-20 language features
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

#
# Find the libraries which will be needed by mathplot
#

set(OpenGL_GL_PREFERENCE "GLVND") # Following `cmake --help-policy CMP0072`
find_package(OpenGL REQUIRED)
find_package(glfw3 3.4 REQUIRED) # 3.4 provides glfwGetPlatform which is used by current imgui
find_package(Freetype REQUIRED)

#
# Set compiler flags
#

# Where is mathplot?
set(MPLOT_INCLUDE_PATH "${PROJECT_SOURCE_DIR}/mathplot" CACHE PATH "The path to mathplot")

# Assume a gcc compiler (or a gcc mimic like Clang).
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -g -O3 -Wno-unused-result -Wno-unknown-pragmas -Wfatal-errors")
# Tell clang to be quiet about brace initialisers:
if(CMAKE_CXX_COMPILER_ID MATCHES Clang)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-missing-braces")
endif()

# Tell the program where the mathplot fonts are, to compile them into the binary
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DMPLOT_FONTS_DIR=\"\\\"${PROJECT_SOURCE_DIR}/mathplot/fonts\\\"\"")

# Additional GL compiler flags.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DGL3_PROTOTYPES -DGL_GLEXT_PROTOTYPES")

#
# Define lists of module files to build. We pull in some definitions
# of variables that list the modules in maths and mathplot
#

# Include maths module defs by running the sebsjames/maths macro 'setup_module_variables_for_maths'
include(${PROJECT_SOURCE_DIR}/maths/cmake/module_definitions.cmake)
setup_module_variables_for_maths (${PROJECT_SOURCE_DIR}/maths)
# Include mathplot module defs by running two macros from sebsjames/mathplot
include(${PROJECT_SOURCE_DIR}/mathplot/cmake/module_definitions.cmake)
setup_module_variables_for_mathplot (${PROJECT_SOURCE_DIR}/mathplot ${PROJECT_SOURCE_DIR}/maths)
# sm.config
include(${PROJECT_SOURCE_DIR}/maths_config/cmake/module_definitions.cmake)
setup_module_variables_for_maths_config (${PROJECT_SOURCE_DIR}/maths_config ${PROJECT_SOURCE_DIR}/json ${PROJECT_SOURCE_DIR}/maths)


#
# Compile two 'boilerplate' libraries - glad and the mathplot fontdata
#

# For mathplot, we must build the glad library here
add_library(glad STATIC ${MPLOT_INCLUDE_PATH}/mplot/glad/gl.c)
target_include_directories(glad AFTER PRIVATE ${PROJECT_SOURCE_DIR}/mathplot)

# We have to build the mathplot fonts data library - this packs up
# some TTF fonts into an object file; they're unpacked at runtime.
add_library(mplot_fontdata STATIC ${MPLOT_INCLUDE_PATH}/mplot/VisualFaceAsm.cpp)
target_include_directories(mplot_fontdata AFTER PRIVATE ${PROJECT_SOURCE_DIR}/mathplot)

#
# Compile a library with the mathplot features we want
#

# Define a list of the modules we wish to compile into our library
set(MATHPLOT_MODULES
  ${SM_CONFIG_MODULES}
  ${MPLOT_CORE_MODULES}
  ${PROJECT_SOURCE_DIR}/mathplot/mplot/TxtVisual.cppm
  ${PROJECT_SOURCE_DIR}/mathplot/mplot/ColourBarVisual.cppm
  ${PROJECT_SOURCE_DIR}/mathplot/mplot/HealpixVisual.cppm
  ${PROJECT_SOURCE_DIR}/mathplot/mplot/SphericalProjectionVisual.cppm
)
# De-duplicate the list
list(REMOVE_DUPLICATES MATHPLOT_MODULES)
# Add a library called mathplot_modules
add_library(mathplot_modules SHARED)
# Attach the list of modules to the library
target_sources(mathplot_modules
  PUBLIC FILE_SET CXX_MODULES BASE_DIRS ${PROJECT_SOURCE_DIR}
  FILES ${MATHPLOT_MODULES}
)
# Link the OpenGL, Freetype and GLFW libraries to this internal library
target_link_libraries(mathplot_modules OpenGL::GL Freetype::Freetype glfw glad mplot_fontdata)
# mathplot needs these includes
target_include_directories(mathplot_modules AFTER PRIVATE ${OPENGL_INCLUDE_DIR} ${FREETYPE_INCLUDE_DIRS} ${MPLOT_INCLUDE_PATH}/mplot/glad)
target_include_directories(mathplot_modules AFTER PRIVATE ${PROJECT_SOURCE_DIR}/mathplot ${PROJECT_SOURCE_DIR}/json/include)

#
# Now it's time to define how we will compile the program
#

# The viewer is compiled from a single c++ file
add_executable(viewer viewer.cpp)

# Link our internal library mathplot_modules
target_link_libraries(viewer mathplot_modules chealpix)

#
# All done!
#
