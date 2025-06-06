#hardinfo2 supports CMake version 3.0.2->
if(${CMAKE_VERSION} VERSION_GREATER "3.30.99")
    cmake_minimum_required(VERSION 3.10.0)
    cmake_policy(VERSION 3.10.0)
    SET(CMAKE_EXECUTABLE_ENABLE_EXPORTS 1)
else()
    if(${CMAKE_VERSION} VERSION_GREATER "3.26.99")
        cmake_minimum_required(VERSION 3.5.0)
        cmake_policy(VERSION 3.5.0)
	SET(CMAKE_EXECUTABLE_ENABLE_EXPORTS 1)
    else()
        cmake_minimum_required(VERSION 3.0.2)
        cmake_policy(VERSION 3.0.2)
    endif()
endif()
project(hardinfo2)

#Release settings
set(HARDINFO2_VERSION "2.2.11")
set(HARDINFO2_RELEASE_DATE "2025-03-29")
set(HARDINFO2_RELEASE_NOTE_HTML "<p>Development</p><p>\nImprovements\nLots of minor fixes/maintenance/updates/cleanup\nFor details see homepage news.</p>")

#Build settings
option(HARDINFO2_SERVICE "Build with service support" 1)
option(HARDINFO2_GTK3 "Build for GTK3 (0/off for GTK2)" 1)
option(HARDINFO2_LIBSOUP3 "Build for libsoup-3.0 (0/off for libsoup-2.4)" 1)
option(HARDINFO2_QT5 "Build opengl benchmark for QT5" 1)
option(HARDINFO2_VK "Build vulkan benchmark" 1)
option(HARDINFO2_VK_WAYLAND "Build Vulkan with wayland support" 1)
option(HARDINFO2_VK_X11 "Build Vulkan with X11 support" 1)

SET(CMAKE_INSTALL_PREFIX "/usr")
SET(HARDINFO2_SYSTEMV 0)
SET(HARDINFO2_RUNIT 0)
string(SUBSTRING ${HARDINFO2_RELEASE_DATE} 0 4 HARDINFO2_RELEASE_YEAR)
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/tools/org.hardinfo2.hardinfo2.metainfo.xml" METAINFO_CONTENT)
string(REPLACE "VERSION" "${HARDINFO2_VERSION}" METAINFO_CONTENT "${METAINFO_CONTENT}" )
string(REPLACE "RELEASE_DATE" "${HARDINFO2_RELEASE_DATE}" METAINFO_CONTENT "${METAINFO_CONTENT}" )
string(REPLACE "DESCRIPTION" "${HARDINFO2_RELEASE_NOTE_HTML}" METAINFO_CONTENT "${METAINFO_CONTENT}" )

SET(CMAKE_MODULE_PATH  "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(GNUInstallDirs)

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
	set(HARDINFO2_OS "linux")
else()
	message(FATAL_ERROR "Unsupported operating system: ${CMAKE_SYSTEM_NAME}")
endif()

if(${CMAKE_SYSTEM_PROCESSOR} MATCHES "i[3-6]86")
	set(HARDINFO2_ARCH "x86")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "x86_64")
	set(HARDINFO2_ARCH "x86")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "(ppc|ppc32|ppc64|ppc64le)")
	set(HARDINFO2_ARCH "ppc")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "(mips|mips64el)")
	set(HARDINFO2_ARCH "mips")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "parisc.*")
	set(HARDINFO2_ARCH "parisc")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "sparc32|sparc64")
	set(HARDINFO2_ARCH "sparc")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm.*")
	set(HARDINFO2_ARCH "arm")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64.*")
	set(HARDINFO2_ARCH "arm")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "ia64")
	set(HARDINFO2_ARCH "ia64")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "(loongarch64|loong64)")
	set(HARDINFO2_ARCH "loongarch64")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "alpha")
	set(HARDINFO2_ARCH "alpha")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "s390.*")
	set(HARDINFO2_ARCH "s390")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "m68k")
	set(HARDINFO2_ARCH "m68k")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "sh[3-5]")
	set(HARDINFO2_ARCH "sh")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "(riscv|riscv32|riscv64)")
	set(HARDINFO2_ARCH "riscv")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "e2k")
	set(HARDINFO2_ARCH "e2k")
else()
	message(FATAL_ERROR "Unsupported architecture: ${CMAKE_SYSTEM_PROCESSOR}")
endif()

add_definitions(-DLOCALEDIR="${CMAKE_INSTALL_FULL_LOCALEDIR}")
message(STATUS "LOCALEDIR = ${CMAKE_INSTALL_FULL_LOCALEDIR}")

message(STATUS "Building HardInfo2 for architecture: ${HARDINFO2_OS}-${HARDINFO2_ARCH}")

#add_definitions("-std=gnu89")
#add_definitions("-fanalyzer")
add_definitions("-Wextra")
add_definitions("-Wno-missing-field-initializers")
add_definitions("-Wformat")
add_definitions("-Wformat-security")

set(PACKAGE_LIBSOUP2_MINVERSION 2.42)
set(PACKAGE_LIBGLIB2_MINVERSION 2.24)
SET(RPM_REQ "dmidecode, sysbench, udisks2, glx-utils, lm_sensors, xdg-utils, iperf3, fwupd, xrandr, vulkan-tools, gawk")
SET(DEB_REQ "dmidecode, sysbench, udisks2, mesa-utils, lm-sensors, xdg-utils, iperf3, fwupd, x11-xserver-utils, vulkan-tools, gawk")

#########################CPack PACKAGING SETUP###############################
if(EXISTS "/etc/os-release")
  file(STRINGS /etc/os-release distro REGEX "^NAME=")
  file(STRINGS /etc/os-release disversion REGEX "^VERSION_ID=")
  file(STRINGS /etc/os-release disversioncode REGEX "^VERSION_CODENAME=")
else()
  set(distro "Linux")
  set(disversion "99")
  set(disversioncode "Linux")
endif()
set(RPM 0)
set(TGZ 0)
string(REGEX REPLACE "NAME=" "" distro "${distro}")
string(REGEX REPLACE "\"" "" distro "${distro}")
string(REGEX REPLACE "/" "" distro "${distro}")
string(REGEX REPLACE " " "" distro "${distro}")
string(REGEX REPLACE "VERSION_ID=" "" disversion "${disversion}")
string(COMPARE EQUAL "${disversion}" "" result)
if(result)
  #sid/rawhide with no version - use project name
  message("-- Found no distro version - Using codename")
  string(REGEX REPLACE "VERSION_CODENAME=" "" disversion "99_${disversioncode}")
endif()
string(REGEX REPLACE "\"" "" disversion "${disversion}")
string(REGEX REPLACE "/" "" disversion "${disversion}")
string(REGEX REPLACE " " "" disversion "${disversion}")
#RPM Distro's
if(${distro} MATCHES "Alma*" OR ${distro} MATCHES "Rocky*" OR ${distro} MATCHES "Fedora*" OR ${distro} MATCHES "RedHat*" OR ${distro} MATCHES "CentOS*" OR ${distro} MATCHES "Oracle*" OR ${distro} MATCHES "openSUSE*" OR ${distro} MATCHES "SUSE*" OR ${distro} MATCHES "PCLinuxOS*" OR ${distro} MATCHES "Ultramarine*" OR ${distro} MATCHES "Nobara*" OR ${distro} MATCHES "OpenMandriva*")
  set(RPM 1)
endif()
#Arch Distro's (tgz)
if(${distro} MATCHES "Arch*" OR ${distro} MATCHES "Endeavour*" OR ${distro} MATCHES "Garuda*" OR ${distro} MATCHES "Reborn*" OR ${distro} MATCHES "SystemRescue*" OR ${distro} MATCHES "Artix*" OR ${distro} MATCHES "Manjaro*")
  set(TGZ 1)
endif()
#Package architecture
if(${RPM})
  set(PACKAGE_DELIMITER "-")
  set(PACKAGE_ARCHITECTURE ".${CMAKE_SYSTEM_PROCESSOR}")
else()
  set(PACKAGE_DELIMITER "_")
  set(PACKAGE_ARCHITECTURE "_${CMAKE_SYSTEM_PROCESSOR}")
  if(${CMAKE_SYSTEM_PROCESSOR} MATCHES "x86_64")
    set(PACKAGE_ARCHITECTURE "_amd64")
  endif()
endif()
set(PACKAGE_DISTRO ${HARDINFO2_VERSION}-${distro}-${disversion}${CPACK_SYSTEM_NAME})
SET(CPACK_PACKAGE_VERSION ${HARDINFO2_VERSION})
#Put it together
string(REPLACE " " "${PACKAGE_DELIMITER}" CPACK_PACKAGE_FILE_NAME "hardinfo2${PACKAGE_DELIMITER}${PACKAGE_DISTRO}${PACKAGE_ARCHITECTURE}")
string(REPLACE " " "${PACKAGE_DELIMITER}" CPACK_SOURCE_PACKAGE_FILE_NAME "hardinfo2${PACKAGE_DELIMITER}${PACKAGE_DISTRO}${PACKAGE_ARCHITECTURE}")
message("-- Found Package Version ${CPACK_PACKAGE_VERSION}")
message("-- Found distro description ${distro}${disversion}")

#flavour output
set(HARDINFO2_OS ${distro})

#fedora ->35
if(${disversion} LESS 35.99)
  if(${distro} MATCHES "Fedora*")
    message("Fedora ->35 - LS24")
    set(HARDINFO2_LIBSOUP3 0)
  endif()
endif()

#centos/redhat/oracle ->9
if(${disversion} LESS 9.99)
  if(${distro} MATCHES "Red*" OR ${distro} MATCHES "CentOS*" OR ${distro} MATCHES "Oracle*" OR ${distro} MATCHES "Alma*" OR ${distro} MATCHES "Rocky*")
    message("Centos/Redhat/oracle/rocky/alma ->9 - LS24")
    set(HARDINFO2_LIBSOUP3 0)
  endif()
endif()

#centos/redhat/oracle/opensuse/suse ->8
if(${disversion} LESS 8.99)
  if(${distro} MATCHES "Red*" OR ${distro} MATCHES "CentOS*" OR ${distro} MATCHES "Oracle*" OR ${distro} MATCHES "openSUSE*" OR ${distro} MATCHES "Alma*" OR ${distro} MATCHES "Rocky*")
    message("Centos/Redhat/oracle/opensuse/suse/rocky/alma ->8 - Old Appstream metainfo VK_WAYLAND=0")
    SET(HARDINFO2_VK_WAYLAND 0)
    string(REPLACE "<url type=\"contact\">https://github.com/hardinfo2/hardinfo2/discussions</url>" "" METAINFO_CONTENT "${METAINFO_CONTENT}" )
  endif()
endif()

#centos/redhat/oracle 10
if((${disversion} GREATER_EQUAL 10) AND (${disversion} LESS 10.99))
  if(${distro} MATCHES "Red*" OR ${distro} MATCHES "CentOS*" OR ${distro} MATCHES "Oracle*" OR ${distro} MATCHES "Alma*" OR ${distro} MATCHES "Rocky*")
    message("Centos/Redhat/oracle/rocky/alma 10 - Missing xrandr+sysbench!!!")
    SET(RPM_REQ "dmidecode, udisks2, glx-utils, lm_sensors, xdg-utils, iperf3, fwupd, vulkan-tools, gawk") #miss sysbench, xrandr
  endif()
endif()

#centos/redhat/oracle/opensuse/suse 7
if(${disversion} LESS 7.99)
  if((${distro} MATCHES "Red*" OR ${distro} MATCHES "CentOS*" OR ${distro} MATCHES "Oracle*" OR ${distro} MATCHES "openSUSE*" OR ${distro} MATCHES "SUSE*"))
      SET(RPM_REQ "dmidecode, sysbench, udisks2, glx-utils, lm_sensors, xdg-utils, iperf3, fwupd, xrandr, gawk") #miss vulkan-tools
      set(HARDINFO2_VK 0)
  endif()
endif()

#centos/redhat/oracle/opensuse/suse 6
if(${disversion} LESS 6.99)
  if((${distro} MATCHES "Red*" OR ${distro} MATCHES "CentOS*" OR ${distro} MATCHES "Oracle*" OR ${distro} MATCHES "openSUSE*" OR ${distro} MATCHES "SUSE*"))
    message("RPM EL6 - Too old to have run directory - creating at install")
    #create run directory owned by root
    install(DIRECTORY DESTINATION /run)
    message("RPM EL6 - GTK2 - deprecated - support will end at any time")
    set(HARDINFO2_GTK3 0)
    message("RPM EL6 - LS24 - deprecated version 2.34.3 - but ok")
    set(PACKAGE_LIBSOUP2_MINVERSION 2.34.3)
    set(HARDINFO2_SYSTEMV 1)
    message("RPM EL6 - no QT5 - Disabling opengl benchmarks")
    set(HARDINFO2_QT5 0)
    SET(RPM_REQ "dmidecode, sysbench, glx-utils, lm_sensors, xdg-utils, xrandr, gawk") #miss vulkan-tools, udisks2, fwupd, iperf3
    set(HARDINFO2_VK 0)
  endif()
endif()

#opensuse
if(${distro} MATCHES "openSUSE*")
  SET(RPM_REQ "dmidecode, sysbench, udisks2, xdg-utils, fwupd, xrandr,vulkan-tools,  Mesa-demo-x, iperf, sensors, gawk")
  #opensuse tumbeweed
  if(${disversion} GREATER 20000101)
  else()
    #opensuse leap 16
    if(${disversion} GREATER 15.99)
    endif()
  endif()
endif()


#Fedora 23
if(${distro}${disversion} MATCHES "Fedora23")
    message("FC23 - no QT5 - Disabling opengl benchmarks")
    set(HARDINFO2_QT5 0)
    set(HARDINFO2_VK 0)
    SET(RPM_REQ "dmidecode, sysbench, udisks2, glx-utils, lm_sensors, xdg-utils, iperf3, fwupd, xrandr, gawk") #miss vulkan-tools
    message("Fedora 23 - Too Old for Appstream metainfo")
    string(REPLACE "<component type=\"desktop-application\">" "<component>" METAINFO_CONTENT "${METAINFO_CONTENT}" )
    string(REPLACE "<url type=\"contact\">https://github.com/hardinfo2/hardinfo2/discussions</url>" "" METAINFO_CONTENT "${METAINFO_CONTENT}" )
    string(REPLACE "<url type=\"contact\">https://github.com/hardinfo2/hardinfo2/discussions</url>" "" METAINFO_CONTENT "${METAINFO_CONTENT}" )
    string(REPLACE "<url type=\"translate\">https://github.com/hardinfo2/hardinfo2/tree/master/po</url>" "" METAINFO_CONTENT "${METAINFO_CONTENT}" )
endif()

#debian 7
if(${distro}${disversion} MATCHES "DebianGNULinux7")
    message("DEB 7 - GTK2 to avoid c11 for ubergraph - deprecated - support will end at any time")
    message("DEB 7 - LS24 - deprecated version 2.34.3 - but ok")
    message("DEB 7 - no QT5 - Disabling opengl benchmarks")
    set(HARDINFO2_GTK3 0)
    set(PACKAGE_LIBSOUP2_MINVERSION 2.34.3)
    set(HARDINFO2_SYSTEMV 1)
    set(HARDINFO2_QT5 0)
    set(HARDINFO2_VK 0)
    SET(DEB_REQ "dmidecode, sysbench, mesa-utils, lm-sensors, xdg-utils, x11-xserver-utils, libsigsegv2, gawk") #miss vulkan-tools,fwupd, udisks2, iperf3 +libsigsegv2 for gawk
endif()

#debian 8
if(${distro}${disversion} MATCHES "DebianGNULinux8")
    message("DEB 8 - GTK3 version 3.14.5 is low - Themes disabled")
    message("DEB 8 - no QT5 - Disabling opengl benchmarks")
    set(HARDINFO2_QT5 0)
    set(HARDINFO2_VK 0)
    SET(DEB_REQ "dmidecode, sysbench, udisks2, mesa-utils, lm-sensors, xdg-utils, iperf3, x11-xserver-utils, libsigsegv2, gawk") #miss vulkan-tools,fwupd +libsigsegv2 for gawk
endif()

#debian 9
if(${distro}${disversion} MATCHES "DebianGNULinux99")
else()
    if(${distro}${disversion} MATCHES "DebianGNULinux9")
        message("DEB 9 - QT5 too low - Disabling opengl benchmarks")
        set(HARDINFO2_QT5 0)
        set(HARDINFO2_VK 0)
	SET(DEB_REQ "dmidecode, sysbench, udisks2, mesa-utils, lm-sensors, xdg-utils, iperf3, fwupd, x11-xserver-utils, gawk") #miss vulkan-tools
    endif()
endif()

#debian 10
if(${distro}${disversion} MATCHES "DebianGNULinux10")
  SET(DEB_REQ "dmidecode, udisks2, mesa-utils, lm-sensors, xdg-utils, iperf3, fwupd, x11-xserver-utils, vulkan-tools, gawk") #miss sysbench
endif()

#debian ->11
if(${disversion} LESS 11.99)
  if(${distro} MATCHES "DebianGNULinux")
    message("DEB ->11 - LS24 VK_WAYLAND=0")
    set(HARDINFO2_LIBSOUP3 0)
    set(HARDINFO2_VK_WAYLAND 0)
  endif()
  if(${distro} MATCHES "RaspbianGNULinux")
    message("DEB ->11 - LS24 VK_WAYLAND=0")
    set(HARDINFO2_LIBSOUP3 0)
    set(HARDINFO2_VK_WAYLAND 0)
  endif()
endif()

#pureos ->10 (debian 11 or less)
if(${disversion} LESS 10.99)
  if(${distro} MATCHES "PureOS")
    message("PureOS ->10 - LS24")
    set(HARDINFO2_LIBSOUP3 0)
  endif()
endif()

#ubuntu 16
if(${distro}${disversion} MATCHES "Ubuntu16")
    message("Ubuntu 16 - no QT5 - Disabling opengl+vulkan benchmarks")
    set(HARDINFO2_VK 0)
    set(HARDINFO2_QT5 0)
    SET(DEB_REQ "dmidecode, sysbench, udisks2, mesa-utils, lm-sensors, xdg-utils, iperf3, fwupd, x11-xserver-utils, gawk") #miss vulkan-tools
    message("ubuntu 16 - Old Appstream metainfo")
    string(REPLACE "<url type=\"contact\">https://github.com/hardinfo2/hardinfo2/discussions</url>" "" METAINFO_CONTENT "${METAINFO_CONTENT}" )
    string(REPLACE "<url type=\"translate\">https://github.com/hardinfo2/hardinfo2/tree/master/po</url>" "" METAINFO_CONTENT "${METAINFO_CONTENT}" )
endif()

#ubuntu 18
if(${distro}${disversion} MATCHES "Ubuntu18")
    SET(DEB_REQ "dmidecode, sysbench, udisks2, mesa-utils, lm-sensors, xdg-utils, iperf3, fwupd, x11-xserver-utils, gawk") #miss vulkan-tools
    message("ubuntu 18 - Old Appstream metainfo + noVulkan")
    set(HARDINFO2_VK 0)
    string(REPLACE "<url type=\"contact\">https://github.com/hardinfo2/hardinfo2/discussions</url>" "" METAINFO_CONTENT "${METAINFO_CONTENT}" )
endif()

#ubuntu ->20.xx
if(${disversion} LESS 20.99)
  if(${distro} MATCHES "Ubuntu")
    message("Ubuntu ->20.xx - LS24 VK_WAYLAND=0")
    set(HARDINFO2_VK_WAYLAND 0)
    set(HARDINFO2_LIBSOUP3 0)
  endif()
endif()

#slackware
if(${distro} MATCHES "Slackware")
    message("Slackware - LS24 VK_WAYLAND=0")
    set(HARDINFO2_LIBSOUP3 0)
    set(HARDINFO2_VK_WAYLAND 0)
    message("FIXME: cannot find libdecor in slackware!")
endif()

#slint
if(${distro} MATCHES "Slint")
    message("Slint VK_WAYLAND=0")
    set(HARDINFO2_VK_WAYLAND 0)
    message("FIXME: cannot find libdecor in slint!")
endif()

#OpenMandriva
if(${distro} MATCHES "OpenMandriva")
    SET(RPM_REQ "dmidecode, udisks2, sysbench, fwupd, lm_sensors, xdg-utils, xrandr, vulkan-tools,  glxinfo, iperf, gawk")
    set(HARDINFO2_VK_WAYLAND 0)
    message("FIXME: cannot find libdecor in mandriva!")
endif()

#PCLinuxOS
if(${distro} MATCHES "PCLinux")
    SET(CPACK_RPM_PACKAGE_AUTOREQPROV "no")  
    SET(RPM_REQ "dmidecode, udisks2, lm_sensors, xdg-utils, xrandr, vulkan-tools,  glxinfo, iperf, gawk") #miss sysbench,fwupd
    set(HARDINFO2_SYSTEMV 1)
endif()

#Devuan
if(${distro} MATCHES "Devuan")
    set(HARDINFO2_SYSTEMV 1)
endif()

#MX
if(${distro} MATCHES "DebianGNULinux")
    if(EXISTS "/etc/initrd_release")
        file(STRINGS /etc/initrd_release mxdistro REGEX "^NAME=")
        if(${mxdistro} MATCHES "MX")
            message("-- Found MX")
            set(HARDINFO2_SYSTEMV 1)
	    set(distro "MXLinux")
            set(HARDINFO2_OS ${distro})
        endif()
   endif()
endif()

#Void
if(${distro} MATCHES "Void")
    set(HARDINFO2_RUNIT 1)
endif()

if(${distro} MATCHES "Alpine")
    set(HARDINFO2_SYSTEMV 1)
endif()

if(${distro} MATCHES "ClearLinux*")
  set(HARDINFO2_VK_WAYLAND 0)
  message("FIXME: libdecor for wayland on clear linux")
endif()

if(${distro} MATCHES "SystemRescue*")
  set(HARDINFO2_VK 0)
endif()

if(${distro} MATCHES "Puppy")
    set(HARDINFO2_SYSTEMV 1)
endif()

if(${distro} MATCHES "Artix")
    set(HARDINFO2_SYSTEMV 1)
endif()

#RiscV
if(${HARDINFO2_ARCH} MATCHES "riscv")
    SET(DEB_REQ "dmidecode, udisks2, mesa-utils, lm-sensors, xdg-utils, iperf3, fwupd, x11-xserver-utils, vulkan-tools, gawk") #miss sysbench
    SET(RPM_REQ "dmidecode, udisks2, glx-utils, lm_sensors, xdg-utils, iperf3, fwupd, xrandr, vulkan-tools, gawk") #miss sysbench
endif()

#E2K - was probably code error in proxy libsoup3 handling
#if(${HARDINFO2_ARCH} MATCHES "e2k")
#    message("Arch E2K - Using too old GCC for libsoup3 - LS24")
#    set(HARDINFO2_LIBSOUP3 0)
#endif()

#WSL 1/2
if(EXISTS "/bin/wslinfo")
    message("WSL Detected")
endif()


#Fedora 39 - bug with man pages - upgrade cmake to 3.28.x

################################################################################

if(${CMAKE_BUILD_TYPE} MATCHES [Dd]ebug)
	set(HARDINFO2_DEBUG 1)
        if(${distro}${disversion} MATCHES "DebianGNULinux7")
        else()
	   set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address")
        endif()
endif()

add_subdirectory(po)

if(HARDINFO2_QT5)
  add_subdirectory(deps/qgears2)
endif()
if(HARDINFO2_VK)
  add_subdirectory(deps/vkgears)
endif()

include(FindPkgConfig)
if (HARDINFO2_GTK3)
    message(STATUS "Building for GTK3")
    pkg_check_modules(GTK REQUIRED gtk+-3.0>=3.0 cairo>=1.0 cairo-png>=1.0 gthread-2.0>=2.10 gmodule-export-2.0>=2.10)
    add_definitions(-DGTK_DISABLE_SINGLE_INCLUDES)
else()
    message(STATUS "Building for GTK2 - (Deprecated - Please update your distro!!)")
    pkg_check_modules(GTK REQUIRED gtk+-2.0>=2.10 gthread-2.0>=2.10 gmodule-export-2.0>=2.10)
endif()

if (${HARDINFO2_LIBSOUP3})
    pkg_check_modules(LIBSOUP REQUIRED libsoup-3.0>=3.00)
else()
    pkg_check_modules(LIBSOUP REQUIRED libsoup-2.4>=${PACKAGE_LIBSOUP2_MINVERSION})
endif()
pkg_check_modules(GLIB2 REQUIRED glib-2.0>=${PACKAGE_LIBGLIB2_MINVERSION})
pkg_check_modules(JSON_GLIB REQUIRED json-glib-1.0>=0.14.2)
pkg_check_modules(X11 REQUIRED x11)

include(FindZLIB REQUIRED)

include_directories(
	${CMAKE_SOURCE_DIR}
	${CMAKE_SOURCE_DIR}/includes
	${CMAKE_SOURCE_DIR}/includes/${HARDINFO2_ARCH}
	${CMAKE_SOURCE_DIR}/deps/uber-graph
	${CMAKE_SOURCE_DIR}/deps/sysobj_early/include
	${CMAKE_SOURCE_DIR}/deps/sysobj_early/gui
	${CMAKE_BINARY_DIR}
	${GTK_INCLUDE_DIRS}
	${LIBSOUP_INCLUDE_DIRS}
	${ZLIB_INCLUDE_DIRS}
	${X11_INCLUDE_DIRS}
	${JSON_GLIB_INCLUDE_DIRS}
)
link_directories(
	${GTK_LIBRARY_DIRS}
	${LIBSOUP_LIBRARY_DIRS}
	${X11_LIBRARY_DIRS}
	${JSON_GLIB_LIBRARY_DIRS}
)

set(HARDINFO2_MODULES
	computer
	devices
	benchmark
	network
)
set(HARDINFO2_RESOURCES
	"data/benchmark.data"
	"data/benchmark.json"
	"data/blobs-update-version.json"
	"data/vendor.ids"
	"data/sdcard.ids"
	"data/usb.ids"
	"data/arm.ids"
	"data/edid.ids"
	"data/ieee_oui.ids"
	"data/pci.ids"
	"data/kernel-module-icons.json"
)
set(HARDINFO2_MANPAGES
	"hardinfo2.1"
)
set(MODULE_computer_SOURCES
	modules/computer.c
	modules/computer/alsa.c
	modules/computer/boots.c
	modules/computer/display.c
	modules/computer/environment.c
	modules/computer/filesystem.c
	modules/computer/languages.c
	modules/computer/loadavg.c
	modules/computer/memory.c
	modules/computer/memory_usage.c
	modules/computer/modules.c
	modules/computer/os.c
	modules/computer/uptime.c
	modules/computer/users.c
	modules/computer/groups.c
)
set(MODULE_devices_SOURCES
	modules/devices.c
	modules/devices/${HARDINFO2_ARCH}/processor.c
	modules/devices/gpu.c
	modules/devices/monitors.c
	modules/devices/battery.c
	modules/devices/dmi.c
	modules/devices/dmi_memory.c
	modules/devices/firmware.c
	modules/devices/devicetree.c
	modules/devices/inputdevices.c
	modules/devices/spd-decode.c
	modules/devices/pci.c
	modules/devices/printers.c
	modules/devices/resources.c
	modules/devices/sensors.c
	modules/devices/storage.c
	modules/devices/usb.c
)
set(MODULE_network_SOURCES
	modules/network.c
	modules/network/net.c
	modules/network/nfs.c
	modules/network/samba.c
)
if(HARDINFO2_QT5)
  set(HARDINFO2_QT5_FILE "modules/benchmark/opengl.c")
else()
  set(HARDINFO2_QT5_FILE "")
endif()
if(HARDINFO2_VK)
  set(HARDINFO2_VK_FILE "modules/benchmark/vulkan.c")
else()
  set(HARDINFO2_VK_FILE "")
endif()
set(MODULE_benchmark_SOURCES
	modules/benchmark.c
	modules/benchmark/bench_util.c
	modules/benchmark/blowfish.c
	modules/benchmark/blowfish2.c
	modules/benchmark/cryptohash.c
	modules/benchmark/fbench.c
	modules/benchmark/fftbench.c
	modules/benchmark/fft.c
	modules/benchmark/fib.c
	modules/benchmark/md5.c
	modules/benchmark/nqueens.c
	modules/benchmark/raytrace.c
	modules/benchmark/sha1.c
	modules/benchmark/zlib.c
	modules/benchmark/sysbench.c
	modules/benchmark/iperf3.c
	${HARDINFO2_QT5_FILE}
	${HARDINFO2_VK_FILE}
	modules/benchmark/storage.c
	modules/benchmark/cachemem.c
	modules/benchmark/drawing.c
	modules/benchmark/guibench.c
)

set_source_files_properties(
	modules/benchmark/blowfish.c
	modules/benchmark/fftbench.c
	modules/benchmark/md5.c
	modules/benchmark/sha1.c
	PROPERTIES
	COMPILE_FLAGS "-O0 -Wall -Wextra -Werror=implicit-function-declaration"
)

set_source_files_properties(
	modules/devices/monitors.c
	modules/devices/dmi_memory.c
	hardinfo2/problem_marker.c
	hardinfo2/hinote_util.c
	PROPERTIES
	COMPILE_FLAGS "-std=c99 -Wall -Wextra -Wno-switch -Werror=implicit-function-declaration"
)

foreach (_module ${HARDINFO2_MODULES})
	add_library(${_module} MODULE ${MODULE_${_module}_SOURCES})
	set_target_properties(${_module} PROPERTIES PREFIX "")
	set_target_properties(${_module} PROPERTIES COMPILE_FLAGS "-Wall -Wextra -Wno-unused-parameter -Wno-missing-braces -Wno-switch -Werror=implicit-function-declaration")
	target_link_libraries(${_module} ${JSON_GLIB_LIBRARIES})
endforeach()

find_library(LIBSENSORS_LIBRARY NAMES libsensors.so)
if (LIBSENSORS_LIBRARY)
	set(HAS_LIBSENSORS 1)
	target_link_libraries(devices ${LIBSENSORS_LIBRARY})
endif ()

add_library(sysobj_early STATIC
	deps/sysobj_early/src/gg_slist.c
	deps/sysobj_early/src/strstr_word.c
	deps/sysobj_early/src/util_ids.c
	deps/sysobj_early/src/util_sysobj.c
	deps/sysobj_early/src/appf.c
	deps/sysobj_early/src/nice_name.c
	deps/sysobj_early/gui/uri_handler.c
	deps/sysobj_early/src/util_edid.c
	deps/sysobj_early/src/format_early.c
	deps/sysobj_early/src/cpubits.c
)
set_target_properties(sysobj_early PROPERTIES COMPILE_FLAGS "-std=c99 -Wall -Wextra -Wno-parentheses -Werror=implicit-function-declaration")
target_link_libraries(sysobj_early m)

if (HARDINFO2_GTK3)
add_library(uber-graph STATIC
	deps/uber-graph/g-ring.c
	deps/uber-graph/uber-frame-source.c
	deps/uber-graph/uber-graph.c
	deps/uber-graph/uber-heat-map.c
	deps/uber-graph/uber-label.c
	deps/uber-graph/uber-line-graph.c
	deps/uber-graph/uber-range.c
	deps/uber-graph/uber-scale.c
	deps/uber-graph/uber-scatter.c
	deps/uber-graph/uber-timeout-interval.c
	deps/uber-graph/uber-window.c
)
set_target_properties(uber-graph PROPERTIES COMPILE_FLAGS "-Wall -Wextra -Wno-deprecated-declarations -Wno-unused-parameter -Werror=implicit-function-declaration")
endif()

set_source_files_properties(
	hardinfo2/usb_util.c
	hardinfo2/pci_util.c
	hardinfo2/gpu_util.c
	hardinfo2/cpu_util.c
	hardinfo2/x_util.c
	hardinfo2/dt_util.c
	PROPERTIES
	COMPILE_FLAGS "-Wall -Wextra -Wno-parentheses -Werror=implicit-function-declaration"
)

if (HARDINFO2_GTK3)
add_executable(hardinfo2
	hardinfo2/usb_util.c
	hardinfo2/pci_util.c
	hardinfo2/expr.c
        hardinfo2/hardinfo.c
	hardinfo2/socket.c
	hardinfo2/util.c
	hardinfo2/gg_key_file_parse_string_as_value.c
	hardinfo2/gg_strescape.c
	hardinfo2/problem_marker.c
	hardinfo2/hinote_util.c
	hardinfo2/vendor.c
	hardinfo2/info.c
	hardinfo2/cpu_util.c
	hardinfo2/dmi_util.c
	hardinfo2/dt_util.c
	hardinfo2/x_util.c
	hardinfo2/gpu_util.c
	hardinfo2/udisks2_util.c
	hardinfo2/storage_util.c
	shell/callbacks.c
	shell/iconcache.c
	shell/menu.c
	shell/report.c
	shell/shell.c
	shell/stock.c
	shell/syncmanager.c
	shell/loadgraph-uber.c
)
target_link_libraries(hardinfo2
	-Wl,-whole-archive sysobj_early -Wl,-no-whole-archive
	uber-graph
	${GTK_LIBRARIES}
	${LIBSOUP_LIBRARIES}
	m
	${ZLIB_LIBRARIES}
	${X11_LIBRARIES}
	${JSON_GLIB_LIBRARIES}
)
set_target_properties(hardinfo2 PROPERTIES COMPILE_FLAGS "-Wno-deprecated-declarations -Werror=implicit-function-declaration")
else()
add_executable(hardinfo2
	hardinfo2/usb_util.c
	hardinfo2/pci_util.c
	hardinfo2/expr.c
        hardinfo2/hardinfo.c
	hardinfo2/socket.c
	hardinfo2/util.c
	hardinfo2/gg_key_file_parse_string_as_value.c
	hardinfo2/gg_strescape.c
	hardinfo2/problem_marker.c
	hardinfo2/hinote_util.c
	hardinfo2/vendor.c
	hardinfo2/info.c
	hardinfo2/cpu_util.c
	hardinfo2/dmi_util.c
	hardinfo2/dt_util.c
	hardinfo2/x_util.c
	hardinfo2/gpu_util.c
	hardinfo2/udisks2_util.c
	hardinfo2/storage_util.c
	shell/callbacks.c
	shell/iconcache.c
	shell/menu.c
	shell/report.c
	shell/shell.c
	shell/stock.c
	shell/syncmanager.c
	shell/loadgraph.c
)
target_link_libraries(hardinfo2
	-Wl,-whole-archive sysobj_early -Wl,-no-whole-archive
	${GTK_LIBRARIES}
	${LIBSOUP_LIBRARIES}
	m
	${ZLIB_LIBRARIES}
	${X11_LIBRARIES}
	${JSON_GLIB_LIBRARIES}
)
endif()

if(${TGZ})
    SET(PACK_REQ "dmidecode,iperf3,lm_sensors,mesa-utils,sysbench,udisks2,vulkan-tools,xdg-utils,xorg-xrandr,fwupd,qt5-base")
endif()
if(${RPM})
    SET(PACK_REQ ${RPM_REQ})
endif()
if(${DEB})
    SET(PACK_REQ ${DEB_REQ})
endif()
configure_file(config.h.cmake ${CMAKE_BINARY_DIR}/config.h @ONLY)
configure_file(hardinfo2.desktop.cmake ${CMAKE_BINARY_DIR}/hardinfo2.desktop @ONLY)

if(HARDINFO2_SERVICE)
    #cannot use CMAKE_INSTALL_LIBDIR due to distros with dual lib (arch)-gnu-linux,lib64
    set(CMAKE_INSTALL_SERVICEDIR "/usr/lib" CACHE PATH "Path where the systemd service file gets installed")
    message("ServiceDir=" ${CMAKE_INSTALL_SERVICEDIR})

    if(${HARDINFO2_RUNIT})
        install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/tools/run DESTINATION /etc/sv/hardinfo2/
        PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
    elseif(${HARDINFO2_SYSTEMV})
        install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/tools/hardinfo2 DESTINATION /etc/init.d/
        PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
    else()
        install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/tools/hardinfo2.service DESTINATION ${CMAKE_INSTALL_SERVICEDIR}/systemd/system/)
    endif()
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/tools/postinst ${CMAKE_CURRENT_BINARY_DIR}/packaging/postinst)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/tools/prerm ${CMAKE_CURRENT_BINARY_DIR}/packaging/prerm)
endif()


#AppStream MetaInfo
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/org.hardinfo2.hardinfo2.metainfo.xml" "${METAINFO_CONTENT}")
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/org.hardinfo2.hardinfo2.metainfo.xml DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/metainfo/)
execute_process(COMMAND appstream-util validate-relax --nonet ${CMAKE_CURRENT_BINARY_DIR}/org.hardinfo2.hardinfo2.metainfo.xml)
execute_process(COMMAND appstreamcli validate --no-net ${CMAKE_CURRENT_BINARY_DIR}/org.hardinfo2.hardinfo2.metainfo.xml)

install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/tools/hwinfo2_fetch_sysdata DESTINATION ${CMAKE_INSTALL_BINDIR}
        PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

install(TARGETS hardinfo2 ${HARDINFO2_MODULES}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}/hardinfo2/modules
	PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
)
install(FILES ${CMAKE_BINARY_DIR}/hardinfo2.desktop
	DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)
install(FILES ${HARDINFO2_RESOURCES}
	DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/hardinfo2
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)
install(DIRECTORY pixmaps
	DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/hardinfo2
	PATTERN "*.{png,svg,jpg}"
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)
install(FILES ${HARDINFO2_MANPAGES}
	DESTINATION ${CMAKE_INSTALL_MANDIR}/man1
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)
install(FILES pixmaps/hardinfo2.svg
    DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/scalable/apps
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)


########################CPack PACKAGE BUILDING###########################
SET(CPACK_PACKAGE_HOMEPAGE_URL "https://www.hardinfo2.org")
if(${TGZ})
    SET(CPACK_PACKAGING_INSTALL_PREFIX "/usr")
    SET(CPACK_SOURCE_IGNORE_FILES "/build/;/.git/;/.github/;.gitignore;.gitmodules;*.po.old;*~;*.mo;")
    SET(CPACK_SOURCE_GENERATOR "TGZ")
    SET(CPACK_GENERATOR "External")
    SET(CPACK_EXTERNAL_PACKAGE_SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/tools/archcpack")
elseif(${RPM})
    set(CPACK_PACKAGING_INSTALL_PREFIX "/usr")
    SET(CPACK_SOURCE_IGNORE_FILES "/build/;/.git/;/.github/;.gitignore;.gitmodules;*.po.old;*~;*.mo;")
    SET(CPACK_SOURCE_GENERATOR "RPM;TGZ")
    SET(CPACK_GENERATOR "RPM")
    SET(PACKAGE_GTK2 "libgtk-2-0")
else()
    SET(CPACK_SOURCE_IGNORE_FILES "/build/;/.git/;/.github/;.gitignore;.gitmodules;*.po.old;*~;*.mo")
    SET(CPACK_SOURCE_GENERATOR "DEB;TGZ")
    SET(CPACK_GENERATOR "DEB")
    SET(PACKAGE_GTK2 "libgtk2.0-0")
endif()

#TGZ
SET(CPACK_ARCHIVE_hardinfo2_FILE_NAME "hardinfo2")

#RPM
SET(CPACK_RPM_PACKAGE_OBSOLETES "hardinfo")
SET(CPACK_RPM_PACKAGE_MAINTAINER "hwspeedy <hardinfo2@bigbear.dk>")
SET(CPACK_RPM_PACKAGE_DESCRIPTION "Hardinfo2 - System Information and Benchmark")
SET(CPACK_RPM_PACKAGE_REQUIRES ${RPM_REQ})
SET(CPACK_RPM_PACKAGE_LICENSE "GPL2+")
SET(CPACK_RPM_PACKAGE_HOMEPAGE "https://github.com/hardinfo2/hardinfo2")
SET(CPACK_RPM_PACKAGE_URL "https://www.hardinfo2.org")
if(${HARDINFO2_LIBSOUP3})
    SET(PACKAGE_LS "libsoup (>=3.0)")
else()
    SET(PACKAGE_LS "libsoup (>=${PACKAGE_LIBSOUP2_MINVERSION})")
endif()
if(HARDINFO2_GTK3)
    SET(PACKAGE_GTK "libgtk-3-0 (>=3.0)")
else()
    SET(PACKAGE_GTK "${PACKAGE_GTK2} (>=2.20)")
endif()
SET(CPACK_RPM_PACKAGE_DEPENDS "libjson-glib-1.0-0, zlib, ${PACKAGE_LS}, ${PACKAGE_GTK}, libglib2.0-0 (>=2.10)")
#RPM-SERVICE
if(HARDINFO2_SERVICE)
    SET(CPACK_RPM_POST_INSTALL_SCRIPT_FILE "${CMAKE_CURRENT_BINARY_DIR}/packaging/postinst")
    SET(CPACK_RPM_PRE_UNINSTALL_SCRIPT_FILE "${CMAKE_CURRENT_BINARY_DIR}/packaging/prerm")
endif()

#DEB
SET(CPACK_DEBIAN_PACKAGE_REPLACES "hardinfo")
SET(CPACK_DEBIAN_PACKAGE_BREAKS "hardinfo")
SET(CPACK_DEBIAN_PACKAGE_MAINTAINER "hwspeedy <hardinfo2@bigbear.dk>")
SET(CPACK_DEBIAN_PACKAGE_DESCRIPTION "Hardinfo2 - System Information and Benchmark")
SET(CPACK_DEBIAN_PACKAGE_HOMEPAGE "https://github.com/hardinfo2/hardinfo2")
if(${HARDINFO2_LIBSOUP3})
    SET(PACKAGE_LS "libsoup-3.0-0 (>=3.0)")
else()
    SET(PACKAGE_LS "libsoup2.4-1 (>=${PACKAGE_LIBSOUP2_MINVERSION})")
endif()
if(HARDINFO2_GTK3)
    SET(PACKAGE_GTK "libgtk-3-0 (>=3.0)")
else()
    SET(PACKAGE_GTK "${PACKAGE_GTK2} (>=2.20)")
endif()
SET(CPACK_DEBIAN_PACKAGE_DEPENDS "${DEB_REQ}, libjson-glib-1.0-0, zlib1g, ${PACKAGE_LS}, ${PACKAGE_GTK}")
#DEB-SERVICE
if(HARDINFO2_SERVICE)
    SET(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_BINARY_DIR}/packaging/postinst;${CMAKE_CURRENT_BINARY_DIR}/packaging/prerm")
endif()

INCLUDE(CPack)
