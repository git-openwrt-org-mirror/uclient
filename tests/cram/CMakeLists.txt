FIND_PACKAGE(PythonInterp 3 REQUIRED)
FILE(GLOB test_cases "test_*.t")

IF(CMAKE_C_COMPILER_ID STREQUAL "Clang")
  FILE(GLOB test_cases_san "test-san_*.t")
ENDIF()

SET(PYTHON_VENV_DIR "${CMAKE_CURRENT_BINARY_DIR}/.venv")
SET(PYTHON_VENV_PIP "${PYTHON_VENV_DIR}/bin/pip")
SET(PYTHON_VENV_CRAM "${PYTHON_VENV_DIR}/bin/cram")

ADD_CUSTOM_COMMAND(
	OUTPUT ${PYTHON_VENV_CRAM}
	COMMAND ${PYTHON_EXECUTABLE} -m venv ${PYTHON_VENV_DIR}
	COMMAND ${PYTHON_VENV_PIP} install cram
)
ADD_CUSTOM_TARGET(prepare-cram-venv ALL DEPENDS ${PYTHON_VENV_CRAM})

ADD_CUSTOM_TARGET(
	http-server-kill ALL
	COMMAND pkill --full -9 "${PYTHON_VENV_DIR}/bin/python3 -m http.server 1922 --bind 127.0.0.1" > /dev/null 2>&1 || true
	DEPENDS ${PYTHON_VENV_CRAM}
)

ADD_CUSTOM_TARGET(
	http-server ALL
	COMMAND ${PYTHON_VENV_DIR}/bin/python3 -m http.server 1922 --bind 127.0.0.1 > /dev/null 2>&1 & ;
			curl --silent --retry 3 --retry-delay 1 --retry-connrefused http://127.0.0.1:1922 > /dev/null
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/server
	DEPENDS http-server-kill
)

ADD_TEST(
	NAME cram
	COMMAND ${PYTHON_VENV_CRAM} ${test_cases} ${test_cases_san}
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

SET_PROPERTY(TEST cram APPEND PROPERTY ENVIRONMENT "BUILD_BIN_DIR=$<TARGET_FILE_DIR:uclient-fetch>")
