#
# Copyright 2025 Staysail Systems, Inc. <info@staystail.tech>
#
# This software is supplied under the terms of the MIT License, a
# copy of which should be located in the distribution where this
# file was obtained (LICENSE.txt).  A copy of the license may also be
# found online at https://opensource.org/licenses/MIT.
#

#  Core.
nng_directory(core)

# All POSIX targets provide strnlen, strcasecmp, strncasecmp, localtime_r,
# and syslog -- these are POSIX.1-mandated, so no probe is needed.
# strlcpy is universally available on BSD/Darwin/Android/musl/QNX/SunOS but
# only appeared in glibc 2.38 (Aug 2023), so we still probe on Linux.
if (NNG_PLATFORM_POSIX)
    nng_defines(NNG_HAVE_STRNLEN)
    nng_defines(NNG_HAVE_STRCASECMP)
    nng_defines(NNG_HAVE_STRNCASECMP)
    nng_defines(NNG_HAVE_LOCALTIME_R)
    nng_defines(NNG_HAVE_SYSLOG)
endif()
if (NNG_PLATFORM_LINUX AND NOT NNG_PLATFORM_ANDROID)
    nng_check_sym(strlcpy string.h NNG_HAVE_STRLCPY)
elseif (NNG_PLATFORM_POSIX)
    nng_defines(NNG_HAVE_STRLCPY)
endif()

nng_sources(
        defs.h

        aio.c
        aio.h
        device.c
        device.h
        dialer.c
        dialer.h
        sockfd.c
        sockfd.h
        file.c
        file.h
        idhash.c
        idhash.h
        init.c
        init.h
        list.c
        list.h
        listener.c
        listener.h
        lmq.c
        lmq.h
        log.c
        message.c
        message.h
        msgqueue.c
        msgqueue.h
        nng_impl.h
        options.c
        options.h
        pollable.c
        pollable.h
        panic.c
        panic.h
        pipe.c
        pipe.h
        platform.h
        protocol.h
        reap.c
        reap.h
        refcnt.c
        refcnt.h
        sockaddr.c
        socket.c
        socket.h
        sockimpl.h
        stats.c
        stats.h
        stream.c
        stream.h
        strs.c
        strs.h
        taskq.c
        taskq.h
        tcp.c
        tcp.h
        thread.c
        thread.h
        url.c
        url.h
)

nng_test(aio_test)
nng_test(args_test)
nng_test(buf_size_test)
nng_test(errors_test)
nng_test(id_test)
nng_test(idhash_test)
nng_test(init_test)
nng_test(list_test)
nng_test(log_test)
nng_test(message_test)
nng_test(reconnect_test)
nng_test(sock_test)
nng_test(sockaddr_test)
nng_test(synch_test)
nng_test(stats_test)
nng_test(url_test)
