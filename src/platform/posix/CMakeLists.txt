#
# Copyright 2023 Staysail Systems, Inc. <info@staystail.tech>
#
# This software is supplied under the terms of the MIT License, a
# copy of which should be located in the distribution where this
# file was obtained (LICENSE.txt).  A copy of the license may also be
# found online at https://opensource.org/licenses/MIT.
#

#  POSIX.

# We cannot use nng_sources_if because these tests don't go into
# the static library unless they also go into the dynamic.
if (NNG_PLATFORM_POSIX)

    nng_find_package(Threads)
    nng_link_libraries(Threads::Threads)

    # Unconditionally declare the following feature test macros.  These are
    # needed for some platforms (glibc and SunOS/illumos) and are harmless
    # on the others.
    nng_defines(_GNU_SOURCE)
    nng_defines(_REENTRANT)
    nng_defines(_THREAD_SAFE)
    nng_defines(_POSIX_PTHREAD_SEMANTICS)

    # Hardcoded feature flags below need to be visible both as compile
    # definitions (for source files) and as CMake variables (for poller /
    # entropy selection later in this file).  nng_have does both.
    macro(nng_have DEF)
        set(${DEF} ON)
        nng_defines(${DEF})
    endmacro()

    # Features mandated by POSIX.1 on every supported target.
    nng_have(NNG_HAVE_RECVMSG)
    nng_have(NNG_HAVE_SENDMSG)
    nng_have(NNG_HAVE_MSG_CONTROL)
    nng_have(NNG_HAVE_UNIX_SOCKETS)
    nng_have(NNG_HAVE_SOCKETPAIR)
    nng_have(NNG_HAVE_INET6)
    nng_have(NNG_HAVE_POLL)
    nng_have(NNG_HAVE_SELECT)

    # C11 / modern compiler+libc baselines (GCC >=4.9, Clang >=3.3,
    # glibc >=2.17, musl, macOS >=10.15, BSD).
    nng_have(NNG_HAVE_STDATOMIC)
    nng_have(NNG_HAVE_TIMESPEC_GET)
    nng_have(NNG_HAVE_CLOCK_GETTIME_LIBC)
    nng_have(NNG_HAVE_CLOCK_GETTIME)

    # Per-OS feature matrix.  These are hardcoded rather than probed so cold
    # cmake configure stays cheap; the underlying platforms do not vary on
    # these features within their supported release ranges.

    # Common BSD features (Darwin, FreeBSD, NetBSD, OpenBSD).
    if (NNG_PLATFORM_BSD)
        nng_have(NNG_HAVE_LOCKF)
        nng_have(NNG_HAVE_FLOCK)
        nng_have(NNG_HAVE_ARC4RANDOM)
        nng_have(NNG_HAVE_GETENTROPY)
        nng_have(NNG_HAVE_SYS_RANDOM)
        nng_have(NNG_HAVE_KQUEUE)
        nng_have(NNG_HAVE_GETPEEREID)
        nng_have(NNG_HAVE_BACKTRACE)
    endif()

    if (NNG_PLATFORM_LINUX)
        nng_have(NNG_HAVE_EVENTFD)
        nng_have(NNG_HAVE_EPOLL)
        nng_have(NNG_HAVE_EPOLL_CREATE1)
        nng_have(NNG_HAVE_SOPEERCRED)
        nng_have(NNG_HAVE_LOCKF)
        nng_have(NNG_HAVE_PTHREAD_SETNAME_NP)
        # backtrace(3) is glibc-only on Linux; musl ships <execinfo.h> as a
        # stub or omits it. Android's bionic gained it in NDK r23. Probe.
        nng_check_sym(backtrace_symbols_fd execinfo.h NNG_HAVE_BACKTRACE)
    elseif (NNG_PLATFORM_DARWIN)
        nng_have(NNG_HAVE_LOCALPEERCRED)
        nng_have(NNG_HAVE_LOCALPEERPID)
        nng_have(NNG_HAVE_PTHREAD_SETNAME_NP)
    elseif (NNG_PLATFORM_FREEBSD)
        nng_have(NNG_HAVE_LOCALPEERCRED)
        # FreeBSD has both pthread_set_name_np (always) and pthread_setname_np
        # (since 12.2). Use the older form so we keep working on FreeBSD < 12.2.
        nng_have(NNG_HAVE_PTHREAD_SET_NAME_NP)
    elseif (NNG_PLATFORM_NETBSD)
        nng_have(NNG_HAVE_PTHREAD_SETNAME_NP)
    elseif (NNG_PLATFORM_OPENBSD)
        nng_have(NNG_HAVE_SOCKPEERCRED)
        nng_have(NNG_HAVE_PTHREAD_SET_NAME_NP)
    elseif (NNG_PLATFORM_SUNOS)
        nng_have(NNG_HAVE_LOCKF)
        nng_have(NNG_HAVE_GETPEERUCRED)
        nng_have(NNG_HAVE_BACKTRACE)
        nng_have(NNG_HAVE_PTHREAD_SETNAME_NP)
        # SunOS / illumos requires linking -lsocket for socket(2) and friends.
        nng_link_libraries(socket)
    elseif (NNG_PLATFORM_QNX)
        nng_have(NNG_HAVE_LOCKF)
        nng_have(NNG_HAVE_PTHREAD_SETNAME_NP)
    else()
        # Unknown POSIX target (Haiku, DragonFly, Hurd, AIX, ...).  Fall back
        # to the original probe-based detection so the build still selects a
        # working poller, entropy source, file lock and thread-name primitive.
        nng_check_func(lockf NNG_HAVE_LOCKF)
        nng_check_func(flock NNG_HAVE_FLOCK)
        nng_check_func(arc4random_buf NNG_HAVE_ARC4RANDOM)
        nng_check_sym(kqueue sys/event.h NNG_HAVE_KQUEUE)
        nng_check_sym(eventfd sys/eventfd.h NNG_HAVE_EVENTFD)
        nng_check_sym(epoll_create sys/epoll.h NNG_HAVE_EPOLL)
        nng_check_sym(epoll_create1 sys/epoll.h NNG_HAVE_EPOLL_CREATE1)
        nng_check_sym(getpeereid unistd.h NNG_HAVE_GETPEEREID)
        nng_check_sym(SO_PEERCRED sys/socket.h NNG_HAVE_SOPEERCRED)
        nng_check_sym(LOCAL_PEERCRED sys/un.h NNG_HAVE_LOCALPEERCRED)
        nng_check_sym(backtrace_symbols_fd execinfo.h NNG_HAVE_BACKTRACE)
        nng_check_lib(pthread pthread_setname_np NNG_HAVE_PTHREAD_SETNAME_NP)
        nng_check_lib(pthread pthread_set_name_np NNG_HAVE_PTHREAD_SET_NAME_NP)
    endif()

    # getentropy availability depends on glibc (>=2.25) / bionic (>=API 28);
    # also probed in the unknown-POSIX fallback above.
    if (NNG_PLATFORM_LINUX)
        nng_check_func(getentropy NNG_HAVE_GETENTROPY)
        nng_check_func(getrandom NNG_HAVE_GETRANDOM)
        nng_check_sym(getentropy sys/random.h NNG_HAVE_SYS_RANDOM)
    elseif (NOT (NNG_PLATFORM_BSD OR NNG_PLATFORM_SUNOS OR NNG_PLATFORM_QNX))
        nng_check_func(getentropy NNG_HAVE_GETENTROPY)
        nng_check_func(getrandom NNG_HAVE_GETRANDOM)
        nng_check_sym(getentropy sys/random.h NNG_HAVE_SYS_RANDOM)
    endif()

    # libnsl was historically required on SunOS and very old glibc for
    # gethostbyname.  Probe only on SunOS where it still matters.
    if (NNG_PLATFORM_SUNOS)
        nng_check_lib(nsl gethostbyname NNG_HAVE_LIBNSL)
    endif()

    # GCC needs libatomic on some architectures (ARM, RISC-V, s390x, sparc,
    # loongarch, mips, 32-bit x86, ...) where the architecture lacks atomic
    # primitives for some operand sizes. Denylist x86_64/amd64 only since
    # those are the only arches where libatomic is provably never needed.
    if (NNG_PLATFORM_LINUX AND NOT CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
        nng_check_lib(atomic __atomic_load_1 NNG_HAVE_LIBATOMIC)
    endif()

    nng_sources(
            posix_impl.h
            posix_aio.h
            posix_ipc.h
            posix_config.h
            posix_pollq.h
            posix_tcp.h

            posix_alloc.c
            posix_atomic.c
            posix_clock.c
            posix_debug.c
            posix_file.c
            posix_ipcconn.c
            posix_ipcdial.c
            posix_ipclisten.c
            posix_peerid.c
            posix_pipe.c
            posix_resolv_gai.c
            posix_sockaddr.c
            posix_socketpair.c
            posix_sockfd.c
            posix_tcpconn.c
            posix_tcpdial.c
            posix_tcplisten.c
            posix_thread.c
            posix_udp.c
    )

    set(NNG_POLLQ_POLLER "auto" CACHE STRING "Poller used for multiplexing I/O")
    set_property(CACHE NNG_POLLQ_POLLER PROPERTY STRINGS auto ports kqueue epoll poll select)
    mark_as_advanced(NNG_POLLQ_POLLER)
    if (NNG_POLLQ_POLLER STREQUAL "ports")
        set(NNG_POLLQ_PORTS ON)
    elseif (NNG_POLLQ_POLLER STREQUAL "kqueue")
        set(NNG_POLLQ_KQUEUE ON)
    elseif (NNG_POLLQ_POLLER STREQUAL "epoll")
        set(NNG_POLLQ_EPOLL ON)
    elseif (NNG_POLLQ_POLLER STREQUAL "poll")
        set(NNG_POLLQ_POLL ON)
    elseif (NNG_POLLQ_POLLER STREQUAL "select")
        set(NNG_POLLQ_SELECT ON)
    elseif (NNG_HAVE_PORT_CREATE)
        set(NNG_POLLQ_PORTS ON)
    elseif (NNG_HAVE_KQUEUE)
        set(NNG_POLLQ_KQUEUE ON)
    elseif (NNG_HAVE_EPOLL AND NNG_HAVE_EVENTFD)
        set(NNG_POLLQ_EPOLL ON)
    elseif (NNG_HAVE_POLL)
        set(NNG_POLLQ_POLL ON)
    elseif (NNG_HAVE_SELECT)
        set(NNG_POLLQ_SELECT ON)
    endif()

    if (NNG_POLLQ_PORTS)
        message(STATUS "Using port events for multiplexing I/O.")
        nng_defines(NNG_POLLQ_PORTS)
        nng_sources(posix_pollq_port.c)
    elseif (NNG_POLLQ_KQUEUE)
        message(STATUS "Using kqueue for multiplexing I/O.")
        nng_defines(NNG_POLLQ_KQUEUE)
        nng_sources(posix_pollq_kqueue.c)
    elseif (NNG_POLLQ_EPOLL)
        message(DEBUG "Using epoll for multiplexing I/O.")
        nng_defines(NNG_POLLQ_EPOLL)
        nng_sources(posix_pollq_epoll.c)
    elseif (NNG_POLLQ_POLL)
        message(STATUS "Using poll for multiplexing I/O.")
        nng_defines(NNG_POLLQ_POLL)
        nng_sources(posix_pollq_poll.c)
    elseif (NNG_POLLQ_SELECT)
        message(STATUS "Using select for multiplexing I/O.")
        nng_defines(NNG_POLLQ_SELECT)
        nng_sources(posix_pollq_select.c)
    else()
        message(FATAL_ERROR "No suitable poller found for multiplexing I/O.")
    endif ()

    if (NNG_HAVE_ARC4RANDOM)
        nng_sources(posix_rand_arc4random.c)
    elseif (NNG_HAVE_GETENTROPY)
        nng_sources(posix_rand_getentropy.c)
    elseif (NNG_HAVE_GETRANDOM)
        nng_sources(posix_rand_getrandom.c)
    else ()
        nng_sources(posix_rand_urandom.c)
    endif ()

    nng_test(posix_ipcwinsec_test)

endif ()
