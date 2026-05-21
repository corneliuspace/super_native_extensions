# The Flutter tooling requires that developers have a version of Visual Studio
# installed that includes CMake 3.14 or later. You should not increase this
# version, as doing so will cause the plugin to fail to compile for some
# customers of the plugin.
cmake_minimum_required(VERSION 3.14)

# Project-level configuration.
set(PROJECT_NAME "super_native_extensions")
project(${PROJECT_NAME} LANGUAGES CXX)

# This value is used when generating builds using this plugin, so it must
# not be changed
set(PLUGIN_NAME "super_native_extensions_plugin")

# Any new source files that you add to the plugin should be added here.
list(APPEND PLUGIN_SOURCES
  "super_native_extensions_plugin.cpp"
  "super_native_extensions_plugin.h"
)

# Define the plugin library target. Its name must not be changed (see comment
# on PLUGIN_NAME above).
add_library(${PLUGIN_NAME} SHARED
  "include/super_native_extensions/super_native_extensions_plugin_c_api.h"
  "super_native_extensions_plugin_c_api.cpp"
  ${PLUGIN_SOURCES}
)

# Apply a standard set of build settings that are configured in the
# application-level CMakeLists.txt. This can be removed for plugins that want
# full control over build settings.
apply_standard_settings(${PLUGIN_NAME})

# Symbols are hidden by default to reduce the chance of accidental conflicts
# between plugins. This should not be removed; any symbols that should be
# exported should be explicitly exported with the FLUTTER_PLUGIN_EXPORT macro.
set_target_properties(${PLUGIN_NAME} PROPERTIES
  CXX_VISIBILITY_PRESET hidden)
target_compile_definitions(${PLUGIN_NAME} PRIVATE FLUTTER_PLUGIN_IMPL)

# Source include directories and library dependencies. Add any plugin-specific
# dependencies here.
target_include_directories(${PLUGIN_NAME} INTERFACE
  "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_link_libraries(${PLUGIN_NAME} PRIVATE flutter flutter_wrapper_plugin)

# =============================================================================
# BYPASS CARGOKIT: Link directly against your precompiled Rust binary
# Place both files in: windows/bin/
#   - super_native_extensions.dll      (runtime, bundled into your app)
#   - super_native_extensions.dll.lib  (import lib, used only at link time)
# =============================================================================

# 1. Dynamically target architecture (x64 is default for Flutter Windows)
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(ARCH "x64")
else()
    set(ARCH "x86")
endif()

# Recommended: Put your binaries in windows/bin/x64/ to avoid arch conflicts
set(PRECOMPILED_DLL "${CMAKE_CURRENT_SOURCE_DIR}/bin/${ARCH}/super_native_extensions.dll")
set(PRECOMPILED_LIB "${CMAKE_CURRENT_SOURCE_DIR}/bin/${ARCH}/super_native_extensions.dll.lib")

# Link the import library for compile time
target_link_libraries(${PLUGIN_NAME} PRIVATE "${PRECOMPILED_LIB}")

# 2. FORCE Flutter to bundle the DLL into the final executable directory
# We hook into Flutter's native bundle architecture macro
list(APPEND super_native_extensions_bundled_libraries "${PRECOMPILED_DLL}")
set(super_native_extensions_bundled_libraries ${super_native_extensions_bundled_libraries} PARENT_SCOPE)

# 3. Bulletproof fallback: Manually copy the DLL to the build output directory
add_custom_command(TARGET ${PLUGIN_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
    "${PRECOMPILED_DLL}"
    "$<TARGET_FILE_DIR:${PLUGIN_NAME}>"
)