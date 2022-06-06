# Copyright (C) 1995-2019, Rene Brun and Fons Rademakers.
# All rights reserved.
#
# For the licensing terms see $ROOTSYS/LICENSE.
# For the list of contributors see $ROOTSYS/README/CREDITS.


set(
  FIT_HEADERS
  ParameterSettings.h
)

set(
  MATH_HEADERS
  Error.h
  FitMethodFunction.h
  Functor.h
  GenAlgoOptions.h
  IFunction.h
  IFunctionfwd.h
  IOptions.h
  Minimizer.h
  MinimizerOptions.h
  Util.h
  WrappedFunction.h
  WrappedParamFunction.h
)

set(
  MATH_SOURCES
  GenAlgoOptions.cxx
  MinimizerOptions.cxx
)

copy_standalone(
  SOURCE ../../../mathcore/inc/Fit DESTINATION ../../inc/Fit
  OUTPUT FIT_HEADERS
  FILES ${FIT_HEADERS}
)
copy_standalone(
  SOURCE ../../../mathcore/inc/Math DESTINATION ../../inc/Math
  OUTPUT MATH_HEADERS
  FILES ${MATH_HEADERS}
)
copy_standalone(
  SOURCE ../../../mathcore/src DESTINATION .
  OUTPUT MATH_SOURCES
  FILES ${MATH_SOURCES}
)

# Adding the headers helps IDEs show the correct headers on targets
add_library(
  FermiMinuit2Math
  ${MATH_SOURCES}
  ${MATH_HEADERS}
  ${FIT_HEADERS}
)

# Add alias for direct inclusion
add_library(FermiMinuit2::Math ALIAS FermiMinuit2Math)

# Build and install directories are different, using CMake Generator expression
target_include_directories(
  FermiMinuit2Math
  PUBLIC
  $<BUILD_INTERFACE:${FermiMinuit2_SOURCE_DIR}/inc>
  $<INSTALL_INTERFACE:include/fermi/Minuit2>
)

# We need to add the ROOT mathcore directories if build inside of ROOT without standalone)
if(minuit2_inroot AND NOT minuit2_standalone)
  target_include_directories(
    FermiMinuit2Math
    PUBLIC
    $<BUILD_INTERFACE:${FermiMinuit2_SOURCE_DIR}/../mathcore/inc>
  )
endif()

target_compile_definitions(
  FermiMinuit2Math
  PRIVATE
  MATH_NO_PLUGIN_MANAGER
)

target_compile_definitions(
  FermiMinuit2Math
  PUBLIC
  ROOT_Math_VecTypes
  MATHCORE_STANDALONE
)

target_link_libraries(FermiMinuit2Math PUBLIC FermiMinuit2Common)

target_compile_features(FermiMinuit2Math PUBLIC cxx_auto_type cxx_static_assert)
set_target_properties(FermiMinuit2Math PROPERTIES CXX_EXTENSIONS OFF)

install(
  TARGETS FermiMinuit2Math
  EXPORT FermiMinuit2Targets
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)

install(FILES ${FIT_HEADERS} DESTINATION include/fermi/Minuit2/Fit)
install(FILES ${MATH_HEADERS} DESTINATION include/fermi/Minuit2/Math)
