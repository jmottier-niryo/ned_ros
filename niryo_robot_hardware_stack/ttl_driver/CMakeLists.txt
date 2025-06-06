cmake_minimum_required(VERSION 3.12)
project(ttl_driver)
message( STATUS "\n## Reading ${PROJECT_NAME}/CMakeLists.txt...")

add_compile_definitions(ROSCONSOLE_MIN_SEVERITY=1)

set(HARDWARE_VERSION "ned2" CACHE STRING "Robot model target")
# for preprocessor directives
if ("${HARDWARE_VERSION}" STREQUAL "one")
    add_definitions(-DNIRYO_ONE)
elseif ("${HARDWARE_VERSION}" STREQUAL "ned")
    add_definitions(-DNIRYO_NED)
elseif ("${HARDWARE_VERSION}" STREQUAL "ned2")
    add_definitions(-DNIRYO_NED2)
endif()

#retrieve architecture
execute_process( COMMAND
            uname -m COMMAND tr -d '\n'
        OUTPUT_VARIABLE
            ARCHITECTURE
    )

## c++ options
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

## By adding -Wall and -Werror, the compiler does not ignore warnings anymore,
## enforcing cleaner code.
add_definitions(-Wall  -Werror -Os)

## Find catkin macros and libraries
find_package(catkin REQUIRED
    COMPONENTS
      common
      dynamixel_sdk
      message_generation
      niryo_robot_msgs
      roscpp
      std_msgs
)

if(CATKIN_ENABLE_TESTING AND ENABLE_COVERAGE_TESTING)
  find_package(code_coverage REQUIRED)
  # Add compiler flags for coverage instrumentation before defining any targets
  append_coverage_compiler_flags()
endif()

###############################################
# Declare ROS messages, services and actions ##
###############################################

add_message_files(
  FILES
    ArrayMotorHardwareStatus.msg
    MotorCommand.msg
    MotorHardwareStatus.msg
    CalibrationStatus.msg
)

add_service_files(
  DIRECTORY
    srv
  FILES
    ReadCustomValue.srv
    ReadPIDValue.srv
    ReadVelocityProfile.srv
    WriteCustomValue.srv
    WritePIDValue.srv
    WriteVelocityProfile.srv
)

generate_messages(
  DEPENDENCIES
    niryo_robot_msgs
    std_msgs
)

###################################
## catkin specific configuration ##
###################################

## INCLUDE_DIRS: if your package contains header files
## LIBRARIES: libraries you create in this project that dependent projects also need
## CATKIN_DEPENDS: catkin_packages dependent projects also need
## DEPENDS: system dependencies of this project that dependent projects also need
catkin_package(
    INCLUDE_DIRS
        include
    LIBRARIES
        ${PROJECT_NAME}
    CATKIN_DEPENDS
        common
        dynamixel_sdk
        message_runtime
        niryo_robot_msgs
        roscpp
        std_msgs
)

###########
## Build ##
###########

## Specify additional locations of header files
## Your package locations should be listed before other locations
include_directories(
    include
    ${catkin_INCLUDE_DIRS}
)

## Declare libs and execs
add_library(${PROJECT_NAME}
  src/abstract_dxl_driver.cpp
  src/abstract_end_effector_driver.cpp
  src/abstract_motor_driver.cpp
  src/abstract_stepper_driver.cpp
  src/abstract_ttl_driver.cpp
  src/mock_dxl_driver.cpp
  src/mock_end_effector_driver.cpp
  src/mock_stepper_driver.cpp
  src/ttl_interface_core.cpp
  src/ttl_manager.cpp
)

add_executable(${PROJECT_NAME}_node
  src/ttl_driver_node.cpp
)

## Add dependencies to exported targets, like ROS msgs or srvs
add_dependencies(${PROJECT_NAME}
    niryo_robot_msgs_generate_messages_cpp
    ${PROJECT_NAME}_generate_messages_cpp
)

add_dependencies(${PROJECT_NAME}_node
  ${catkin_EXPORTED_TARGETS}
)

## Specify libraries to link executable targets against
target_link_libraries(${PROJECT_NAME}
  ${catkin_LIBRARIES}
  pthread
)

target_link_libraries(${PROJECT_NAME}_node
  ${PROJECT_NAME}
)

#############
##   Doc   ##
#############

find_program(rosdoc_lite_FOUND rosdoc_lite)
if(rosdoc_lite_FOUND)
    add_custom_target (${PROJECT_NAME}_doc ALL
        rosdoc_lite -o ${CMAKE_CURRENT_BINARY_DIR}/doc ${CMAKE_CURRENT_LIST_DIR}
    )
else()
  message(WARNING "rosdoc_lite not found. Skipping documentation building")
endif()

#############
## Testing ##
#############

if(CATKIN_ENABLE_TESTING)
  find_package(rostest REQUIRED)

  if ("${HARDWARE_VERSION}" STREQUAL "ned" OR "${HARDWARE_VERSION}" STREQUAL "one")
    if ("${ARCHITECTURE}" MATCHES "^(arm.*|aarch64.*|arm64.*)$")
      add_rostest_gtest(${PROJECT_NAME}_unit_tests
        test/unit_tests_ned_one.test
        test/unit_tests_ned_one.cpp
      )
    else()
      add_rostest_gtest(${PROJECT_NAME}_unit_tests
        test/unit_tests_fake_ned_one.test
        test/unit_tests_ned_one.cpp
      )
    endif()
  else()
    if ("${ARCHITECTURE}" MATCHES "^(arm.*|aarch64.*|arm64.*)$")
      add_rostest_gtest(${PROJECT_NAME}_unit_tests
        test/unit_tests_ned2.test
        test/unit_tests_ned2.cpp
      )
    else()
      add_rostest_gtest(${PROJECT_NAME}_unit_tests
        test/unit_tests_fake_ned2.test
        test/unit_tests_ned2.cpp
      )
    endif()
  endif()

  if ("${HARDWARE_VERSION}" STREQUAL "ned" OR "${HARDWARE_VERSION}" STREQUAL "one")
    if ("${ARCHITECTURE}" MATCHES "^(arm.*|aarch64.*|arm64.*)$")
      add_rostest_gtest(${PROJECT_NAME}_integration_tests
        test/service_client_ned_one.test
        test/service_client_ned_one.cpp
      )
    else()
      add_rostest_gtest(${PROJECT_NAME}_integration_tests
        test/service_client_fake_ned_one.test
        test/service_client_ned_one.cpp
      )  
    endif()
  else()
    if ("${ARCHITECTURE}" MATCHES "^(arm.*|aarch64.*|arm64.*)$")
      add_rostest_gtest(${PROJECT_NAME}_integration_tests
        test/service_client_ned2.test
        test/service_client_ned2.cpp
      )
    else()
      add_rostest_gtest(${PROJECT_NAME}_integration_tests
        test/service_client_fake_ned2.test
        test/service_client_ned2.cpp
      )  
    endif()
  endif()

  if(TARGET ${PROJECT_NAME}_unit_tests)
    target_link_libraries(${PROJECT_NAME}_unit_tests
      ${PROJECT_NAME}
      ${catkin_LIBRARIES}
    )
  endif()

  if(TARGET ${PROJECT_NAME}_integration_tests)
    target_link_libraries(${PROJECT_NAME}_integration_tests
      ${PROJECT_NAME}
    )
  endif()

  # Create a target ${PROJECT_NAME}_coverage_report
  if(ENABLE_COVERAGE_TESTING)
    set(COVERAGE_EXCLUDES "*/${PROJECT_NAME}/test*")
    add_code_coverage(
      NAME ${PROJECT_NAME}_coverage_report
      DEPENDENCIES tests
    )
  endif()
  
  ##########################
  ## Static code analysis ##
  ##########################
  find_package(roslint QUIET)

  if(roslint_FOUND)
      roslint_cpp()
  else()
      message(WARNING "roslint not found. Skipping roslint target building")
  endif()
endif()


#############
## Install ##
#############

install(
  TARGETS ${PROJECT_NAME}
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_GLOBAL_BIN_DESTINATION}
)

install(
  TARGETS ${PROJECT_NAME}_node
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

# Mark other files for installation
install(
  DIRECTORY include/${PROJECT_NAME}/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
)


if (rosdoc_lite_FOUND)
  install(
    DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc/
    DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/doc
  )
endif()

install(
  DIRECTORY launch/
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/launch
)

install(
  DIRECTORY config/
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/config
)
