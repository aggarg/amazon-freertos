set(mchp_dir "${AFR_VENDORS_DIR}/microchip")
set(ecc608a_dir "${mchp_dir}/secure_elements")
set(ECC608_IN_USE TRUE)

set(afr_ports_dir "${CMAKE_CURRENT_LIST_DIR}/ports")
set(board_demos_dir "${CMAKE_CURRENT_LIST_DIR}/aws_demos")
set(board_tests_dir "${CMAKE_CURRENT_LIST_DIR}/aws_tests")

if(AFR_IS_TESTING)
    set(board_dir "${board_tests_dir}")
else()
    set(board_dir "${board_demos_dir}")
endif()

# -------------------------------------------------------------------------------------------------
# FreeRTOS Console metadata
# -------------------------------------------------------------------------------------------------
if("${AFR_BOARD_NAME}" STREQUAL "ecc608a_plus_winsim")
    include("${CMAKE_CURRENT_LIST_DIR}/ecc608a_plus_winsim.cmake")
endif()

# -------------------------------------------------------------------------------------------------
# Compiler settings
# -------------------------------------------------------------------------------------------------
afr_mcu_port(compiler)

target_compile_definitions(
    AFR::compiler::mcu_port
    INTERFACE
        __free_rtos__
        _CONSOLE
        _CRT_SECURE_NO_WARNINGS
)

target_compile_options(
    AFR::compiler::mcu_port
    INTERFACE "/MP" "/wd4210" "/wd4127" "/wd4244" "/wd4310"
)

# -------------------------------------------------------------------------------------------------
# FreeRTOS portable layers
# -------------------------------------------------------------------------------------------------
# Normally the portable layer for kernel should be vendor's driver code.
afr_mcu_port(kernel)


target_sources(
    AFR::kernel::mcu_port
    INTERFACE
        "${AFR_KERNEL_DIR}/portable/MSVC-MingW/port.c"
        "${AFR_KERNEL_DIR}/portable/MSVC-MingW/portmacro.h"
        "${AFR_KERNEL_DIR}/portable/MemMang/heap_4.c"
)
target_include_directories(
    AFR::kernel::mcu_port
    INTERFACE
        "${AFR_KERNEL_DIR}/portable/MSVC-MingW"
        "${board_dir}/config_files"
        "${board_dir}/application_code"
        "${ecc608a_dir}/lib"
        "${ecc608a_dir}/third_party/hidapi/hidapi"
        # Need aws_clientcredential.h
        "$<IF:${AFR_IS_TESTING},${AFR_TESTS_DIR},${AFR_DEMOS_DIR}>/include"
)
target_link_libraries(
    AFR::kernel::mcu_port
    INTERFACE
        3rdparty::tracealyzer_recorder
)

# POSIX
afr_mcu_port(posix)
target_sources(
    AFR::posix::mcu_port
    INTERFACE "${afr_ports_dir}/posix/FreeRTOS_POSIX_portable.h"
)
target_include_directories(
    AFR::posix::mcu_port
    INTERFACE "${afr_ports_dir}/posix"
)
target_link_libraries(
    AFR::posix::mcu_port
    INTERFACE AFR::freertos_plus_posix
)

# PKCS11
afr_mcu_port(pkcs11_implementation DEPENDS AFR::pkcs11_ecc608a)
set(
    ecc608a_hid_src
    "${ecc608a_dir}/third_party/hidapi/windows/hid.c"
)
set(
    ecc608a_hal_src
        "${ecc608a_dir}/lib/hal/atca_hal.c"
        "${ecc608a_dir}/lib/hal/hal_all_platforms_kit_hidapi.c"
        "${ecc608a_dir}/lib/hal/hal_freertos.c"
        "${ecc608a_dir}/lib/hal/hal_windows.c"
        "${ecc608a_dir}/lib/hal/kit_protocol.c"
        "${ecc608a_dir}/lib/hal/atca_hal.h"
        "${ecc608a_dir}/lib/hal/hal_all_platforms_kit_hidapi.h"
        "${ecc608a_dir}/lib/hal/kit_phy.h"
        "${ecc608a_dir}/lib/hal/kit_protocol.h"
)
target_sources(
    AFR::pkcs11_implementation::mcu_port
    INTERFACE
        "${ecc608a_hal_src}"
        "${ecc608a_hid_src}"
)

# FreeRTOS Plus TCP
afr_mcu_port(freertos_plus_tcp)
target_sources(
    AFR::freertos_plus_tcp::mcu_port
    INTERFACE
        "${AFR_MODULES_FREERTOS_PLUS_DIR}/standard/freertos_plus_tcp/portable/BufferManagement/BufferAllocation_2.c"
        "${AFR_MODULES_FREERTOS_PLUS_DIR}/standard/freertos_plus_tcp/portable/NetworkInterface/WinPCap/NetworkInterface.c"
        # Header files added to the target so that these are available in code downloaded from the FreeRTOS console.
        "${AFR_MODULES_FREERTOS_PLUS_DIR}/standard/freertos_plus_tcp/portable/Compiler/MSVC/pack_struct_end.h"
        "${AFR_MODULES_FREERTOS_PLUS_DIR}/standard/freertos_plus_tcp/portable/Compiler/MSVC/pack_struct_start.h"
)

target_include_directories(
    AFR::freertos_plus_tcp::mcu_port
    INTERFACE
        "${AFR_MODULES_FREERTOS_PLUS_DIR}/standard/freertos_plus_tcp/portable/Compiler/MSVC"
)
target_link_libraries(
    AFR::freertos_plus_tcp::mcu_port
    INTERFACE 3rdparty::win_pcap
)

# Secure sockets
afr_mcu_port(secure_sockets)
target_link_libraries(
    AFR::secure_sockets::mcu_port
    INTERFACE AFR::secure_sockets_freertos_plus_tcp
)

# Over-the-air Updates
afr_mcu_port(ota)
target_sources(
    AFR::ota::mcu_port
    INTERFACE
        "${afr_ports_dir}/ota_pal_for_aws/ota_pal.c"
        "${afr_ports_dir}/ota_pal_for_aws/ota_pal.h"
)
target_include_directories(
    AFR::ota::mcu_port
    INTERFACE
        "${afr_ports_dir}/ota_pal_for_aws"
)
target_link_libraries(
    AFR::ota::mcu_port
    INTERFACE
        AFR::crypto
        AFR::pkcs11
        AFR::ota
)

if(AFR_ENABLE_TESTS)
   target_include_directories(
       AFR::ota::mcu_port
       INTERFACE "${PROJECT_SOURCE_DIR}/tests/integration_test/ota_pal"
   )
endif()

# -------------------------------------------------------------------------------------------------
# FreeRTOS demos and tests
# -------------------------------------------------------------------------------------------------
# TODO, remove network manager src.
afr_glob_src(network_manager_src DIRECTORY "${AFR_DEMOS_DIR}/network_manager")

if(AFR_IS_TESTING)
    set(exe_target aws_tests)
else()
    set(exe_target aws_demos)
    set(extra_src ${network_manager_src})
endif()

add_executable(
    ${exe_target}
    "${board_dir}/application_code/main.c"
    "${board_demos_dir}/application_code/aws_demo_logging.c"
    "${board_demos_dir}/application_code/aws_entropy_hardware_poll.c"
    "${board_demos_dir}/application_code/aws_run-time-stats-utils.c"
    ${extra_src}
)
target_include_directories(
    ${exe_target}
    PRIVATE
        "${board_demos_dir}/application_code"
)
target_link_libraries(
    ${exe_target}
    PRIVATE
        AFR::freertos_plus_tcp
        AFR::utils
        AFR::dev_mode_key_provisioning
        setupapi.lib
)
