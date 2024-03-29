#cmake 3.12 seems to have a problem finisng BLAS
cmake_minimum_required(VERSION 3.13...3.22)

project(lattice_net)


###   VARIABLES   ##############################################################
set(CMAKE_BUILD_TYPE RelWithDebInfo)
set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH}" "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread -Wall -O3") 
# set(PYTORCH_PATH "/opt/pytorch")
# set(PYTORCH_PATH_2 "/opt/conda/envs/lattice/lib/python3.6/site-packages/torch")
# set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} "/opt/ros/melodic/\;${PYTORCH_PATH}"
#set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} "${PYTORCH_PATH}:${PYTORCH_PATH_2}")
# set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} "${PYTORCH_PATH};${PYTORCH_PATH_2}")
set(CMAKE_CXX_STANDARD 17) #we need c++17 because this solves alignment issues with eigen http://eigen.tuxfamily.org/bz/show_bug.cgi?id=1409
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# set(CMAKE_VERBOSE_MAKEFILE ON)



#pybind attempt 3
# pybind11_add_module(DataLoaderTest ${CMAKE_SOURCE_DIR}/src/pycom/PyCom.cxx )


####   GLOBAL OPTIONS   ###https://stackoverflow.com/questions/15201064/cmake-conditional-preprocessor-define-on-code


######   PACKAGES   ############################################################
# find_package(catkin REQUIRED)
find_package(Eigen3 3.3 REQUIRED NO_MODULE)
# find_package(OpenCV REQUIRED)
find_package(EasyPBR REQUIRED)
# find_package(DataLoaders REQUIRED)
# find_package(Torch REQUIRED)
find_package(CUDA REQUIRED)
add_subdirectory(${EASYPBR_SRC_PATH}/deps/pybind11  [EXCLUDE_FROM_ALL])
# add_subdirectory(${PROJECT_SOURCE_DIR}/deps/pybind11)
# find_package(catkin REQUIRED COMPONENTS roscpp std_msgs cv_bridge pcl_ros image_transport )
#compile with pytorch
# get and append paths for finding dep
execute_process( #do it like this https://github.com/facebookresearch/hanabi_SAD/blob/6e4ed590f5912fcb99633f4c224778a3ba78879b/rela/CMakeLists.txt#L10
  COMMAND python -c "import torch; import os; print(os.path.dirname(torch.__file__), end='')"
  OUTPUT_VARIABLE TorchPath
)
list(APPEND CMAKE_PREFIX_PATH ${TorchPath})
find_package(Torch REQUIRED)


###   INCLUDES   #########################################################
# include_directories(${catkin_INCLUDE_DIRS})
include_directories(${CMAKE_SOURCE_DIR}/include)
include_directories(${CMAKE_SOURCE_DIR}/deps)
include_directories(${EIGEN3_INCLUDE_DIR})
include_directories(${EASYPBR_INCLUDE_DIR})
include_directories(${DATALOADERS_INCLUDE_DIR})
include_directories(${CUDA_INCLUDE_DIRS})
include_directories(${TORCH_INCLUDE_DIRS})
# catkin_package( INCLUDE_DIRS include  CATKIN_DEPENDS roscpp std_msgs cv_bridge pcl_ros image_transport )


###   SOURCES   #################################################################
set(MY_SRC  
    # ${PROJECT_SOURCE_DIR}/src/Frame.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderVolRef.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderImgRos.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderScanNet.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderShapeNetPartSeg.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataTransformer.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderSemanticKitti.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderStanfordIndoor.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderToyExample.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderCloudRos.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderRueMonge.cxx
    # ${PROJECT_SOURCE_DIR}/src/RosBagPlayer.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderPNG.cxx
    # ${PROJECT_SOURCE_DIR}/src/DataLoaderModelNet40.cxx
)
set(DEPS_SRC 
    # ${PROJECT_SOURCE_DIR}/deps/json11/json11.cpp
    )


###   SET ALL THE GLOBAL OPTIONS #########################################

#pybind
pybind11_add_module(latticenet ${PROJECT_SOURCE_DIR}/src/PyBridge.cxx )

#lib 
# add_library( latticenet_cpp SHARED ${MY_SRC}  ${DEPS_SRC}  )

##    CUDA library  ######################
#configure the include paths of jitify so that the kernels can properly include other ones
configure_file(${PROJECT_SOURCE_DIR}/include/lattice_net/jitify_helper/jitify_options.hpp.in ${PROJECT_SOURCE_DIR}/include/lattice_net/jitify_helper/jitify_options.hpp)
install(FILES "${PROJECT_SOURCE_DIR}/include/lattice_net/jitify_helper/jitify_options.hpp" DESTINATION ${PROJECT_SOURCE_DIR}/include/lattice_net/jitify_helper/)
# set(CUDA_NVCC_FLAGS "-gencode arch=compute_30 -code=sm_30" CACHE STRING "nvcc flags" FORCE)
#disabling a certain nvcc warning is done through https://stackoverflow.com/questions/14831051/how-to-disable-a-specific-nvcc-compiler-warnings
# set(CUDA_NVCC_FLAGS "-arch=sm_60 -Xcudafe --diag_suppress=esa_on_defaulted_function_ignored  --expt-relaxed-constexpr --display_error_number " CACHE STRING "nvcc flags" FORCE)
# set(CUDA_NVCC_FLAGS "-arch=sm_60 -Xcudafe --diag_suppress=esa_on_defaulted_function_ignored  --expt-relaxed-constexpr")
# message("NVCC flags are ", ${CUDA_NVCC_FLAGS})
# set(CUDA_VERBOSE_BUILD ON CACHE BOOL "nvcc verbose" FORCE) 
# cuda_add_library(latticenet_cu  SHARED  ${CMAKE_SOURCE_DIR}/src/Lattice.cxx
# add_library(latticenet_cu  SHARED  ${CMAKE_SOURCE_DIR}/src/Lattice.cxx
cuda_add_library(latticenet_cu  SHARED  ${CMAKE_SOURCE_DIR}/src/Lattice.cu
                                        # ${CMAKE_SOURCE_DIR}/src/LatticeFuncs.cu
                                        ${CMAKE_SOURCE_DIR}/src/HashTable.cu  
                                        ${CMAKE_SOURCE_DIR}/src/ModelParams.cxx
                                        ${CMAKE_SOURCE_DIR}/src/TrainParams.cxx
                                        ${CMAKE_SOURCE_DIR}/src/EvalParams.cxx
                                        )
# set(LIBS ${LIBS} latticenet_cu)


##pybind
# pybind11_add_module(dataloaders ${PROJECT_SOURCE_DIR}/src/PyBridge.cxx )

###   EXECUTABLE   #######################################
# add_executable(run_data_loaders ${PROJECT_SOURCE_DIR}/src/main.cxx  )


###   LIBS   ###############################################
if(${TORCH_FOUND})
    set(LIBS ${LIBS} ${TORCH_LIBRARIES} )
    #torch 1.5.0 and above mess with pybind and we therefore need to link against libtorch_python.so also
    find_library(TORCH_PYTHON_LIBRARY torch_python PATHS "${TORCH_INSTALL_PREFIX}/lib")
    message(STATUS "TORCH_PYTHON_LIBRARY: ${TORCH_PYTHON_LIBRARY}")
    if(TORCH_PYTHON_LIBRARY)
        message(STATUS "Linking to torch_python_library")
        set(LIBS ${LIBS} ${TORCH_PYTHON_LIBRARY} )
    endif()
endif()
set(LIBS ${LIBS} ${catkin_LIBRARIES} ${EASYPBR_LIBRARY} ${DATALOADERS_LIBRARY} ${CUDA_LIBRARIES} ${TORCH_LIBRARIES} ${OpenCV_LIBS})






target_link_libraries(latticenet_cu ${LIBS} )
target_link_libraries(latticenet PRIVATE latticenet_cu )
# target_link_libraries(run_data_loaders PRIVATE data_loaders_cpp  )


#definitions for cmake variables that are necesarry during runtime
target_compile_definitions(latticenet_cu PRIVATE PROJECT_SOURCE_DIR="${PROJECT_SOURCE_DIR}") #point to the cmakelist folder of the easy_pbr
target_compile_definitions(latticenet_cu PRIVATE CMAKE_SOURCE_DIR="${CMAKE_SOURCE_DIR}") # points to the CMakeList folder of whichever project included easy_pbr


