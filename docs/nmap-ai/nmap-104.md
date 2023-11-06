# Nmap源码解析 104

# `libssh2/src/scp.c`

This is a text-based answer to a question about the允许在作品中使用的信息。它解释了版权法中有关如何使用、复制、发行和展示作品的条款。它还说明了在某些情况下，从作品中使用信息是否被认为是合法的。


```cpp
/* Copyright (c) 2009-2019 by Daniel Stenberg
 * Copyright (c) 2004-2008, Sara Golemon <sarag@libssh2.org>
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms,
 * with or without modification, are permitted provided
 * that the following conditions are met:
 *
 *   Redistributions of source code must retain the above
 *   copyright notice, this list of conditions and the
 *   following disclaimer.
 *
 *   Redistributions in binary form must reproduce the above
 *   copyright notice, this list of conditions and the following
 *   disclaimer in the documentation and/or other materials
 *   provided with the distribution.
 *
 *   Neither the name of the copyright holder nor the names
 *   of any other contributors may be used to endorse or
 *   promote products derived from this software without
 *   specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
 * CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
 * INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR
 * CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
 * BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 * SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
 * WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 * NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE
 * USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY
 * OF SUCH DAMAGE.
 */

```

This is a function that discusses how to handle certain sequence of apostrophes in SSH/SCP protocol. The function starts by explaining how it works and what it's used for, and then goes on to explain the problem of it being combined in one pair of quotation marks. It also explains the solution to handle the issue and it's called by name.


```cpp
#include "libssh2_priv.h"
#include <errno.h>
#include <stdlib.h>

#include "channel.h"
#include "session.h"


/* Max. length of a quoted string after libssh2_shell_quotearg() processing */
#define _libssh2_shell_quotedsize(s)     (3 * strlen(s) + 2)

/*
  This function quotes a string in a way suitable to be used with a
  shell, e.g. the file name
  one two
  becomes
  'one two'

  The resulting output string is crafted in a way that makes it usable
  with the two most common shell types: Bourne Shell derived shells
  (sh, ksh, ksh93, bash, zsh) and C-Shell derivates (csh, tcsh).

  The following special cases are handled:
  o  If the string contains an apostrophy itself, the apostrophy
  character is written in quotation marks, e.g. "'".
  The shell cannot handle the syntax 'doesn\'t', so we close the
  current argument word, add the apostrophe in quotation marks "",
  and open a new argument word instead (_ indicate the input
  string characters):
   _____   _   _
  'doesn' "'" 't'

  Sequences of apostrophes are combined in one pair of quotation marks:
  a'''b
  becomes
   _  ___  _
  'a'"'''"'b'

  o  If the string contains an exclamation mark (!), the C-Shell
  interprets it as an event number. Using \! (not within quotation
  marks or single quotation marks) is a mechanism understood by
  both Bourne Shell and C-Shell.

  If a quotation was already started, the argument word is closed
  first:
  a!b

  become
   _  _ _
  'a'\!'b'

  The result buffer must be large enough for the expanded result. A
  bad case regarding expansion is alternating characters and
  apostrophes:

  a'b'c'd'                   (length 8) gets converted to
  'a'"'"'b'"'"'c'"'"'d'"'"   (length 24)

  This is the worst case.

  Maximum length of the result:
  1 + 6 * (length(input) + 1) / 2) + 1

  => 3 * length(input) + 2

  Explanation:
  o  leading apostrophe
  o  one character / apostrophe pair (two characters) can get
  represented as 6 characters: a' -> a'"'"'
  o  String terminator (+1)

  A result buffer three times the size of the input buffer + 2
  characters should be safe.

  References:
  o  csh-compatible quotation (special handling for '!' etc.), see
  http://www.grymoire.com/Unix/Csh.html#toc-uh-10

  Return value:
  Length of the resulting string (not counting the terminating '\0'),
  or 0 in case of errors, e.g. result buffer too small

  Note: this function could possible be used elsewhere within libssh2, but
  until then it is kept static and in this source file.
```

This function appears to compare a path string represented by a combination of a sequence of up to three single-quoted characters, followed by a sequence of zero or more double quotes, to a specified end position of the string. It returns the number of characters that were removed from the destination string by the process.

The function takes into account the different cases of single- or double-quoted paths, and also supports internalsubstrings, which allows it to handle sequences of up to three double-quoted characters at the start or end of the string.

The quality of the output depends on the quality of the input strings. If the input strings are of poor quality, the output may not be what is expected. For example, if the input strings contain characters that are not printable, the function may return an incorrect result. Additionally, if the input strings contain null characters or end quotes incorrectly, the function may also return an incorrect result.

Overall, this function appears to be well-structured and well-suited to handle a wide variety of input strings.


```cpp
*/

static unsigned
shell_quotearg(const char *path, unsigned char *buf,
               unsigned bufsize)
{
    const char *src;
    unsigned char *dst, *endp;

    /*
     * Processing States:
     *  UQSTRING:       unquoted string: ... -- used for quoting exclamation
     *                  marks. This is the initial state
     *  SQSTRING:       single-quoted-string: '... -- any character may follow
     *  QSTRING:        quoted string: "... -- only apostrophes may follow
     */
    enum { UQSTRING, SQSTRING, QSTRING } state = UQSTRING;

    endp = &buf[bufsize];
    src = path;
    dst = buf;
    while(*src && dst < endp - 1) {

        switch(*src) {
            /*
             * Special handling for apostrophe.
             * An apostrophe is always written in quotation marks, e.g.
             * ' -> "'".
             */

        case '\'':
            switch(state) {
            case UQSTRING:      /* Unquoted string */
                if(dst + 1 >= endp)
                    return 0;
                *dst++ = '"';
                break;
            case QSTRING:       /* Continue quoted string */
                break;
            case SQSTRING:      /* Close single quoted string */
                if(dst + 2 >= endp)
                    return 0;
                *dst++ = '\'';
                *dst++ = '"';
                break;
            default:
                break;
            }
            state = QSTRING;
            break;

            /*
             * Special handling for exclamation marks. CSH interprets
             * exclamation marks even when quoted with apostrophes. We convert
             * it to the plain string \!, because both Bourne Shell and CSH
             * interpret that as a verbatim exclamation mark.
             */

        case '!':
            switch(state) {
            case UQSTRING:
                if(dst + 1 >= endp)
                    return 0;
                *dst++ = '\\';
                break;
            case QSTRING:
                if(dst + 2 >= endp)
                    return 0;
                *dst++ = '"';           /* Closing quotation mark */
                *dst++ = '\\';
                break;
            case SQSTRING:              /* Close single quoted string */
                if(dst + 2 >= endp)
                    return 0;
                *dst++ = '\'';
                *dst++ = '\\';
                break;
            default:
                break;
            }
            state = UQSTRING;
            break;

            /*
             * Ordinary character: prefer single-quoted string
             */

        default:
            switch(state) {
            case UQSTRING:
                if(dst + 1 >= endp)
                    return 0;
                *dst++ = '\'';
                break;
            case QSTRING:
                if(dst + 2 >= endp)
                    return 0;
                *dst++ = '"';           /* Closing quotation mark */
                *dst++ = '\'';
                break;
            case SQSTRING:      /* Continue single quoted string */
                break;
            default:
                break;
            }
            state = SQSTRING;   /* Start single-quoted string */
            break;
        }

        if(dst + 1 >= endp)
            return 0;
        *dst++ = *src++;
    }

    switch(state) {
    case UQSTRING:
        break;
    case QSTRING:           /* Close quoted string */
        if(dst + 1 >= endp)
            return 0;
        *dst++ = '"';
        break;
    case SQSTRING:          /* Close single quoted string */
        if(dst + 1 >= endp)
            return 0;
        *dst++ = '\'';
        break;
    default:
        break;
    }

    if(dst + 1 >= endp)
        return 0;
    *dst = '\0';

    /* The result cannot be larger than 3 * strlen(path) + 2 */
    /* assert((dst - buf) <= (3 * (src - path) + 2)); */

    return dst - buf;
}

```

This function appears to be responsible for receiving data from a SCSI device and sending it to the server. It takes in a session object, which contains information about the session, as well as a handle to the SCSI device.

The function first sets up a receive mode for the device, and then enters a loop that listens for incoming data. When data is received, the function checks whether the basename of the file is valid, and if it is, it reads the data into the `session->scpRecv_buffer`.

After the data has been read, the function sets the state of the session object to indicate that the receive operation is complete, and then returns the channel number. If an error occurs (e.g. an I/O error or a closed channel), the function returns NULL to indicate that the operation failed.


```cpp
/*
 * scp_recv
 *
 * Open a channel and request a remote file via SCP
 *
 */
static LIBSSH2_CHANNEL *
scp_recv(LIBSSH2_SESSION * session, const char *path, libssh2_struct_stat * sb)
{
    int cmd_len;
    int rc;
    int tmp_err_code;
    const char *tmp_err_msg;

    if(session->scpRecv_state == libssh2_NB_state_idle) {
        session->scpRecv_mode = 0;
        session->scpRecv_size = 0;
        session->scpRecv_mtime = 0;
        session->scpRecv_atime = 0;

        session->scpRecv_command_len =
            _libssh2_shell_quotedsize(path) + sizeof("scp -f ") + (sb?1:0);

        session->scpRecv_command =
            LIBSSH2_ALLOC(session, session->scpRecv_command_len);

        if(!session->scpRecv_command) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate a command buffer for "
                           "SCP session");
            return NULL;
        }

        snprintf((char *)session->scpRecv_command,
                 session->scpRecv_command_len,
                 "scp -%sf ", sb?"p":"");

        cmd_len = strlen((char *)session->scpRecv_command);
        cmd_len += shell_quotearg(path,
                                  &session->scpRecv_command[cmd_len],
                                  session->scpRecv_command_len - cmd_len);

        /* the command to exec should _not_ be NUL-terminated */
        session->scpRecv_command_len = cmd_len;

        _libssh2_debug(session, LIBSSH2_TRACE_SCP,
                       "Opening channel for SCP receive");

        session->scpRecv_state = libssh2_NB_state_created;
    }

    if(session->scpRecv_state == libssh2_NB_state_created) {
        /* Allocate a channel */
        session->scpRecv_channel =
            _libssh2_channel_open(session, "session",
                                  sizeof("session") - 1,
                                  LIBSSH2_CHANNEL_WINDOW_DEFAULT,
                                  LIBSSH2_CHANNEL_PACKET_DEFAULT, NULL,
                                  0);
        if(!session->scpRecv_channel) {
            if(libssh2_session_last_errno(session) !=
                LIBSSH2_ERROR_EAGAIN) {
                LIBSSH2_FREE(session, session->scpRecv_command);
                session->scpRecv_command = NULL;
                session->scpRecv_state = libssh2_NB_state_idle;
            }
            else {
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block starting up channel");
            }
            return NULL;
        }

        session->scpRecv_state = libssh2_NB_state_sent;
    }

    if(session->scpRecv_state == libssh2_NB_state_sent) {
        /* Request SCP for the desired file */
        rc = _libssh2_channel_process_startup(session->scpRecv_channel, "exec",
                                              sizeof("exec") - 1,
                                              (char *)session->scpRecv_command,
                                              session->scpRecv_command_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block requesting SCP startup");
            return NULL;
        }
        else if(rc) {
            LIBSSH2_FREE(session, session->scpRecv_command);
            session->scpRecv_command = NULL;
            goto scp_recv_error;
        }
        LIBSSH2_FREE(session, session->scpRecv_command);
        session->scpRecv_command = NULL;

        _libssh2_debug(session, LIBSSH2_TRACE_SCP, "Sending initial wakeup");
        /* SCP ACK */
        session->scpRecv_response[0] = '\0';

        session->scpRecv_state = libssh2_NB_state_sent1;
    }

    if(session->scpRecv_state == libssh2_NB_state_sent1) {
        rc = _libssh2_channel_write(session->scpRecv_channel, 0,
                                    session->scpRecv_response, 1);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block sending initial wakeup");
            return NULL;
        }
        else if(rc != 1) {
            goto scp_recv_error;
        }

        /* Parse SCP response */
        session->scpRecv_response_len = 0;

        session->scpRecv_state = libssh2_NB_state_sent2;
    }

    if((session->scpRecv_state == libssh2_NB_state_sent2)
        || (session->scpRecv_state == libssh2_NB_state_sent3)) {
        while(sb && (session->scpRecv_response_len <
                      LIBSSH2_SCP_RESPONSE_BUFLEN)) {
            unsigned char *s, *p;

            if(session->scpRecv_state == libssh2_NB_state_sent2) {
                rc = _libssh2_channel_read(session->scpRecv_channel, 0,
                                           (char *) session->
                                           scpRecv_response +
                                           session->scpRecv_response_len, 1);
                if(rc == LIBSSH2_ERROR_EAGAIN) {
                    _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                   "Would block waiting for SCP response");
                    return NULL;
                }
                else if(rc < 0) {
                    /* error, give up */
                    _libssh2_error(session, rc, "Failed reading SCP response");
                    goto scp_recv_error;
                }
                else if(rc == 0)
                    goto scp_recv_empty_channel;

                session->scpRecv_response_len++;

                if(session->scpRecv_response[0] != 'T') {
                    size_t err_len;
                    char *err_msg;

                    /* there can be
                       01 for warnings
                       02 for errors

                       The following string MUST be newline terminated
                    */
                    err_len =
                        _libssh2_channel_packet_data_len(session->
                                                         scpRecv_channel, 0);
                    err_msg = LIBSSH2_ALLOC(session, err_len + 1);
                    if(!err_msg) {
                        _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                       "Failed to get memory ");
                        goto scp_recv_error;
                    }

                    /* Read the remote error message */
                    (void)_libssh2_channel_read(session->scpRecv_channel, 0,
                                                err_msg, err_len);
                    /* If it failed for any reason, we ignore it anyway. */

                    /* zero terminate the error */
                    err_msg[err_len] = 0;

                    _libssh2_debug(session, LIBSSH2_TRACE_SCP,
                                   "got %02x %s", session->scpRecv_response[0],
                                   err_msg);

                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Failed to recv file");

                    LIBSSH2_FREE(session, err_msg);
                    goto scp_recv_error;
                }

                if((session->scpRecv_response_len > 1) &&
                    ((session->
                      scpRecv_response[session->scpRecv_response_len - 1] <
                      '0')
                     || (session->
                         scpRecv_response[session->scpRecv_response_len - 1] >
                         '9'))
                    && (session->
                        scpRecv_response[session->scpRecv_response_len - 1] !=
                        ' ')
                    && (session->
                        scpRecv_response[session->scpRecv_response_len - 1] !=
                        '\r')
                    && (session->
                        scpRecv_response[session->scpRecv_response_len - 1] !=
                        '\n')) {
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid data in SCP response");
                    goto scp_recv_error;
                }

                if((session->scpRecv_response_len < 9)
                    || (session->
                        scpRecv_response[session->scpRecv_response_len - 1] !=
                        '\n')) {
                    if(session->scpRecv_response_len ==
                        LIBSSH2_SCP_RESPONSE_BUFLEN) {
                        /* You had your chance */
                        _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                       "Unterminated response from "
                                       "SCP server");
                        goto scp_recv_error;
                    }
                    /* Way too short to be an SCP response, or not done yet,
                       short circuit */
                    continue;
                }

                /* We're guaranteed not to go under response_len == 0 by the
                   logic above */
                while((session->
                        scpRecv_response[session->scpRecv_response_len - 1] ==
                        '\r')
                       || (session->
                           scpRecv_response[session->scpRecv_response_len -
                                            1] == '\n'))
                    session->scpRecv_response_len--;
                session->scpRecv_response[session->scpRecv_response_len] =
                    '\0';

                if(session->scpRecv_response_len < 8) {
                    /* EOL came too soon */
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server, "
                                   "too short");
                    goto scp_recv_error;
                }

                s = session->scpRecv_response + 1;

                p = (unsigned char *) strchr((char *) s, ' ');
                if(!p || ((p - s) <= 0)) {
                    /* No spaces or space in the wrong spot */
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server, "
                                   "malformed mtime");
                    goto scp_recv_error;
                }

                *(p++) = '\0';
                /* Make sure we don't get fooled by leftover values */
                session->scpRecv_mtime = strtol((char *) s, NULL, 10);

                s = (unsigned char *) strchr((char *) p, ' ');
                if(!s || ((s - p) <= 0)) {
                    /* No spaces or space in the wrong spot */
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server, "
                                   "malformed mtime.usec");
                    goto scp_recv_error;
                }

                /* Ignore mtime.usec */
                s++;
                p = (unsigned char *) strchr((char *) s, ' ');
                if(!p || ((p - s) <= 0)) {
                    /* No spaces or space in the wrong spot */
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server, "
                                   "too short or malformed");
                    goto scp_recv_error;
                }

                *p = '\0';
                /* Make sure we don't get fooled by leftover values */
                session->scpRecv_atime = strtol((char *) s, NULL, 10);

                /* SCP ACK */
                session->scpRecv_response[0] = '\0';

                session->scpRecv_state = libssh2_NB_state_sent3;
            }

            if(session->scpRecv_state == libssh2_NB_state_sent3) {
                rc = _libssh2_channel_write(session->scpRecv_channel, 0,
                                            session->scpRecv_response, 1);
                if(rc == LIBSSH2_ERROR_EAGAIN) {
                    _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                   "Would block waiting to send SCP ACK");
                    return NULL;
                }
                else if(rc != 1) {
                    goto scp_recv_error;
                }

                _libssh2_debug(session, LIBSSH2_TRACE_SCP,
                               "mtime = %ld, atime = %ld",
                               session->scpRecv_mtime, session->scpRecv_atime);

                /* We *should* check that atime.usec is valid, but why let
                   that stop use? */
                break;
            }
        }

        session->scpRecv_state = libssh2_NB_state_sent4;
    }

    if(session->scpRecv_state == libssh2_NB_state_sent4) {
        session->scpRecv_response_len = 0;

        session->scpRecv_state = libssh2_NB_state_sent5;
    }

    if((session->scpRecv_state == libssh2_NB_state_sent5)
        || (session->scpRecv_state == libssh2_NB_state_sent6)) {
        while(session->scpRecv_response_len < LIBSSH2_SCP_RESPONSE_BUFLEN) {
            char *s, *p, *e = NULL;

            if(session->scpRecv_state == libssh2_NB_state_sent5) {
                rc = _libssh2_channel_read(session->scpRecv_channel, 0,
                                           (char *) session->
                                           scpRecv_response +
                                           session->scpRecv_response_len, 1);
                if(rc == LIBSSH2_ERROR_EAGAIN) {
                    _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                   "Would block waiting for SCP response");
                    return NULL;
                }
                else if(rc < 0) {
                    /* error, bail out*/
                    _libssh2_error(session, rc, "Failed reading SCP response");
                    goto scp_recv_error;
                }
                else if(rc == 0)
                    goto scp_recv_empty_channel;

                session->scpRecv_response_len++;

                if(session->scpRecv_response[0] != 'C') {
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server");
                    goto scp_recv_error;
                }

                if((session->scpRecv_response_len > 1) &&
                    (session->
                     scpRecv_response[session->scpRecv_response_len - 1] !=
                     '\r')
                    && (session->
                        scpRecv_response[session->scpRecv_response_len - 1] !=
                        '\n')
                    &&
                    (session->
                     scpRecv_response[session->scpRecv_response_len - 1]
                     < 32)) {
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid data in SCP response");
                    goto scp_recv_error;
                }

                if((session->scpRecv_response_len < 7)
                    || (session->
                        scpRecv_response[session->scpRecv_response_len - 1] !=
                        '\n')) {
                    if(session->scpRecv_response_len ==
                        LIBSSH2_SCP_RESPONSE_BUFLEN) {
                        /* You had your chance */
                        _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                       "Unterminated response "
                                       "from SCP server");
                        goto scp_recv_error;
                    }
                    /* Way too short to be an SCP response, or not done yet,
                       short circuit */
                    continue;
                }

                /* We're guaranteed not to go under response_len == 0 by the
                   logic above */
                while((session->
                        scpRecv_response[session->scpRecv_response_len - 1] ==
                        '\r')
                       || (session->
                           scpRecv_response[session->scpRecv_response_len -
                                            1] == '\n')) {
                    session->scpRecv_response_len--;
                }
                session->scpRecv_response[session->scpRecv_response_len] =
                    '\0';

                if(session->scpRecv_response_len < 6) {
                    /* EOL came too soon */
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server, "
                                   "too short");
                    goto scp_recv_error;
                }

                s = (char *) session->scpRecv_response + 1;

                p = strchr(s, ' ');
                if(!p || ((p - s) <= 0)) {
                    /* No spaces or space in the wrong spot */
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server, "
                                   "malformed mode");
                    goto scp_recv_error;
                }

                *(p++) = '\0';
                /* Make sure we don't get fooled by leftover values */

                session->scpRecv_mode = strtol(s, &e, 8);
                if(e && *e) {
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server, "
                                   "invalid mode");
                    goto scp_recv_error;
                }

                s = strchr(p, ' ');
                if(!s || ((s - p) <= 0)) {
                    /* No spaces or space in the wrong spot */
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server, "
                                   "too short or malformed");
                    goto scp_recv_error;
                }

                *s = '\0';
                /* Make sure we don't get fooled by leftover values */
                session->scpRecv_size = scpsize_strtol(p, &e, 10);
                if(e && *e) {
                    _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                                   "Invalid response from SCP server, "
                                   "invalid size");
                    goto scp_recv_error;
                }

                /* SCP ACK */
                session->scpRecv_response[0] = '\0';

                session->scpRecv_state = libssh2_NB_state_sent6;
            }

            if(session->scpRecv_state == libssh2_NB_state_sent6) {
                rc = _libssh2_channel_write(session->scpRecv_channel, 0,
                                            session->scpRecv_response, 1);
                if(rc == LIBSSH2_ERROR_EAGAIN) {
                    _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                   "Would block sending SCP ACK");
                    return NULL;
                }
                else if(rc != 1) {
                    goto scp_recv_error;
                }
                _libssh2_debug(session, LIBSSH2_TRACE_SCP,
                               "mode = 0%lo size = %ld", session->scpRecv_mode,
                               session->scpRecv_size);

                /* We *should* check that basename is valid, but why let that
                   stop us? */
                break;
            }
        }

        session->scpRecv_state = libssh2_NB_state_sent7;
    }

    if(sb) {
        memset(sb, 0, sizeof(libssh2_struct_stat));

        sb->st_mtime = session->scpRecv_mtime;
        sb->st_atime = session->scpRecv_atime;
        sb->st_size = session->scpRecv_size;
        sb->st_mode = (unsigned short)session->scpRecv_mode;
    }

    session->scpRecv_state = libssh2_NB_state_idle;
    return session->scpRecv_channel;

  scp_recv_empty_channel:
    /* the code only jumps here as a result of a zero read from channel_read()
       so we check EOF status to avoid getting stuck in a loop */
    if(libssh2_channel_eof(session->scpRecv_channel))
        _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                       "Unexpected channel close");
    else
        return session->scpRecv_channel;
    /* fall-through */
  scp_recv_error:
    tmp_err_code = session->err_code;
    tmp_err_msg = session->err_msg;
    while(libssh2_channel_free(session->scpRecv_channel) ==
           LIBSSH2_ERROR_EAGAIN);
    session->err_code = tmp_err_code;
    session->err_msg = tmp_err_msg;
    session->scpRecv_channel = NULL;
    session->scpRecv_state = libssh2_NB_state_idle;
    return NULL;
}

```

这段代码是一个名为 `libssh2_scp_recv` 的函数，它是 SSH2 协议中的一个函口，用于在客户端和服务器之间通过 SCP（Secure Copy）协议传输文件。

具体来说，这段代码的作用是接收一个远程文件，并将其传递给客户端。当客户端通过 SCP 协议请求一个文件时，服务端会使用 `libssh2_struct_stat` 结构体来获取文件的一些信息，如文件大小、修改时间等。然后客户端会根据需要在代码中提供这些信息，以便服务器进行后续处理。

如果客户端提供的文件大小超过了服务端接收文件大小的限制，函数将会抛出错误并返回。否则，函数将返回客户端指针，这样客户端就可以使用传递给他们的信息来获取文件。


```cpp
/*
 * libssh2_scp_recv
 *
 * DEPRECATED
 *
 * Open a channel and request a remote file via SCP.  This receives files
 * larger than 2 GB, but is unable to report the proper size on platforms
 * where the st_size member of struct stat is limited to 2 GB (e.g. windows).
 *
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_recv(LIBSSH2_SESSION *session, const char *path, struct stat * sb)
{
    LIBSSH2_CHANNEL *ptr;

    /* scp_recv uses libssh2_struct_stat, so pass one if the caller gave us a
       struct to populate... */
    libssh2_struct_stat sb_intl;
    libssh2_struct_stat *sb_ptr;
    memset(&sb_intl, 0, sizeof(sb_intl));
    sb_ptr = sb ? &sb_intl : NULL;

    BLOCK_ADJUST_ERRNO(ptr, session, scp_recv(session, path, sb_ptr));

    /* ...and populate the caller's with as much info as fits. */
    if(sb) {
        memset(sb, 0, sizeof(struct stat));

        sb->st_mtime = sb_intl.st_mtime;
        sb->st_atime = sb_intl.st_atime;
        sb->st_size = (off_t)sb_intl.st_size;
        sb->st_mode = sb_intl.st_mode;
    }

    return ptr;
}

```

这段代码是一个用于在支持libssh2库的平台上接收文件的SCP命令的函数。它接受三个参数：libssh2_session表示当前SSH会话，path是要传输的文件的路径，sb是一个包含文件统计信息的结构体。函数首先创建一个名为ptr的LIBSSH2_CHANNEL类型的指针，然后使用scp_recv函数将文件从远程服务器接收。函数使用BLOCK_ADJUST_ERRNO函数来处理可能的错误，最后将ptr指向返回的LIBSSH2_CHANNEL类型的指针，表示成功接收了文件。


```cpp
/*
 * libssh2_scp_recv2
 *
 * Open a channel and request a remote file via SCP.  This supports files > 2GB
 * on platforms that support it.
 *
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_recv2(LIBSSH2_SESSION *session, const char *path,
                  libssh2_struct_stat *sb)
{
    LIBSSH2_CHANNEL *ptr;
    BLOCK_ADJUST_ERRNO(ptr, session, scp_recv(session, path, sb));
    return ptr;
}

```

This is a function that reads a file from a remote host and sends it to the local host. It uses the OpenSSH library to handle the SSH and SCP protocols.

The function takes a session object as its parameter, which contains information about the SSH session. It first sets the state of the session to idle and then enters a loop that reads the file from the remote host and sends it to the local host.

Within the loop, the function first checks if the channel has been closed by calling `libssh2_channel_eof()`. If the channel has been closed, an error is thrown and the loop is exited. Otherwise, the function retrieves the channel from the session object and passes it to the `_libssh2_channel_read()` function to read the file.

After reading the file, the function sets the state of the session to `libssh2_NB_state_value_normal`. This is followed by a call to `_libssh2_debug()` to log any information about the session, such as any errors or file transfer issues.

Finally, the function calls `LIBSSH2_FREE()` to free any resources associated with the session object, such as the file handle or the session ID, and then calls `_libssh2_error()` to log any errors that occurred during the file transfer.

If an error occurs within the loop, the function jumps to the `scp_send_error` label to handle the error and return a value indicating failure. Otherwise, the function continues reading the file and sending it to the local host until the end of the file is reached or an error occurs.


```cpp
/*
 * scp_send()
 *
 * Send a file using SCP
 *
 */
static LIBSSH2_CHANNEL *
scp_send(LIBSSH2_SESSION * session, const char *path, int mode,
         libssh2_int64_t size, time_t mtime, time_t atime)
{
    int cmd_len;
    int rc;
    int tmp_err_code;
    const char *tmp_err_msg;

    if(session->scpSend_state == libssh2_NB_state_idle) {
        session->scpSend_command_len =
            _libssh2_shell_quotedsize(path) + sizeof("scp -t ") +
            ((mtime || atime)?1:0);

        session->scpSend_command =
            LIBSSH2_ALLOC(session, session->scpSend_command_len);

        if(!session->scpSend_command) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate a command buffer for "
                           "SCP session");
            return NULL;
        }

        snprintf((char *)session->scpSend_command,
                 session->scpSend_command_len,
                 "scp -%st ", (mtime || atime)?"p":"");

        cmd_len = strlen((char *)session->scpSend_command);
        cmd_len += shell_quotearg(path,
                                  &session->scpSend_command[cmd_len],
                                  session->scpSend_command_len - cmd_len);

        /* the command to exec should _not_ be NUL-terminated */
        session->scpSend_command_len = cmd_len;

        _libssh2_debug(session, LIBSSH2_TRACE_SCP,
                       "Opening channel for SCP send");
        /* Allocate a channel */

        session->scpSend_state = libssh2_NB_state_created;
    }

    if(session->scpSend_state == libssh2_NB_state_created) {
        session->scpSend_channel =
            _libssh2_channel_open(session, "session", sizeof("session") - 1,
                                  LIBSSH2_CHANNEL_WINDOW_DEFAULT,
                                  LIBSSH2_CHANNEL_PACKET_DEFAULT, NULL, 0);
        if(!session->scpSend_channel) {
            if(libssh2_session_last_errno(session) != LIBSSH2_ERROR_EAGAIN) {
                /* previous call set libssh2_session_last_error(), pass it
                   through */
                LIBSSH2_FREE(session, session->scpSend_command);
                session->scpSend_command = NULL;
                session->scpSend_state = libssh2_NB_state_idle;
            }
            else {
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block starting up channel");
            }
            return NULL;
        }

        session->scpSend_state = libssh2_NB_state_sent;
    }

    if(session->scpSend_state == libssh2_NB_state_sent) {
        /* Request SCP for the desired file */
        rc = _libssh2_channel_process_startup(session->scpSend_channel, "exec",
                                              sizeof("exec") - 1,
                                              (char *)session->scpSend_command,
                                              session->scpSend_command_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block requesting SCP startup");
            return NULL;
        }
        else if(rc) {
            /* previous call set libssh2_session_last_error(), pass it
               through */
            LIBSSH2_FREE(session, session->scpSend_command);
            session->scpSend_command = NULL;
            _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                           "Unknown error while getting error string");
            goto scp_send_error;
        }
        LIBSSH2_FREE(session, session->scpSend_command);
        session->scpSend_command = NULL;

        session->scpSend_state = libssh2_NB_state_sent1;
    }

    if(session->scpSend_state == libssh2_NB_state_sent1) {
        /* Wait for ACK */
        rc = _libssh2_channel_read(session->scpSend_channel, 0,
                                   (char *) session->scpSend_response, 1);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block waiting for response from remote");
            return NULL;
        }
        else if(rc < 0) {
            _libssh2_error(session, rc, "SCP failure");
            goto scp_send_error;
        }
        else if(!rc)
            /* remain in the same state */
            goto scp_send_empty_channel;
        else if(session->scpSend_response[0] != 0) {
            _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                           "Invalid ACK response from remote");
            goto scp_send_error;
        }
        if(mtime || atime) {
            /* Send mtime and atime to be used for file */
            session->scpSend_response_len =
                snprintf((char *) session->scpSend_response,
                         LIBSSH2_SCP_RESPONSE_BUFLEN, "T%ld 0 %ld 0\n",
                         (long)mtime, (long)atime);
            _libssh2_debug(session, LIBSSH2_TRACE_SCP, "Sent %s",
                           session->scpSend_response);
        }

        session->scpSend_state = libssh2_NB_state_sent2;
    }

    /* Send mtime and atime to be used for file */
    if(mtime || atime) {
        if(session->scpSend_state == libssh2_NB_state_sent2) {
            rc = _libssh2_channel_write(session->scpSend_channel, 0,
                                        session->scpSend_response,
                                        session->scpSend_response_len);
            if(rc == LIBSSH2_ERROR_EAGAIN) {
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block sending time data for SCP file");
                return NULL;
            }
            else if(rc != (int)session->scpSend_response_len) {
                _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                               "Unable to send time data for SCP file");
                goto scp_send_error;
            }

            session->scpSend_state = libssh2_NB_state_sent3;
        }

        if(session->scpSend_state == libssh2_NB_state_sent3) {
            /* Wait for ACK */
            rc = _libssh2_channel_read(session->scpSend_channel, 0,
                                       (char *) session->scpSend_response, 1);
            if(rc == LIBSSH2_ERROR_EAGAIN) {
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block waiting for response");
                return NULL;
            }
            else if(rc < 0) {
                _libssh2_error(session, rc, "SCP failure");
                goto scp_send_error;
            }
            else if(!rc)
                /* remain in the same state */
                goto scp_send_empty_channel;
            else if(session->scpSend_response[0] != 0) {
                _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                               "Invalid SCP ACK response");
                goto scp_send_error;
            }

            session->scpSend_state = libssh2_NB_state_sent4;
        }
    }
    else {
        if(session->scpSend_state == libssh2_NB_state_sent2) {
            session->scpSend_state = libssh2_NB_state_sent4;
        }
    }

    if(session->scpSend_state == libssh2_NB_state_sent4) {
        /* Send mode, size, and basename */
        const char *base = strrchr(path, '/');
        if(base)
            base++;
        else
            base = path;

        session->scpSend_response_len =
            snprintf((char *) session->scpSend_response,
                     LIBSSH2_SCP_RESPONSE_BUFLEN, "C0%o %"
                     LIBSSH2_INT64_T_FORMAT " %s\n", mode,
                     size, base);
        _libssh2_debug(session, LIBSSH2_TRACE_SCP, "Sent %s",
                       session->scpSend_response);

        session->scpSend_state = libssh2_NB_state_sent5;
    }

    if(session->scpSend_state == libssh2_NB_state_sent5) {
        rc = _libssh2_channel_write(session->scpSend_channel, 0,
                                    session->scpSend_response,
                                    session->scpSend_response_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block send core file data for SCP file");
            return NULL;
        }
        else if(rc != (int)session->scpSend_response_len) {
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send core file data for SCP file");
            goto scp_send_error;
        }

        session->scpSend_state = libssh2_NB_state_sent6;
    }

    if(session->scpSend_state == libssh2_NB_state_sent6) {
        /* Wait for ACK */
        rc = _libssh2_channel_read(session->scpSend_channel, 0,
                                   (char *) session->scpSend_response, 1);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block waiting for response");
            return NULL;
        }
        else if(rc < 0) {
            _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                           "Invalid ACK response from remote");
            goto scp_send_error;
        }
        else if(rc == 0)
            goto scp_send_empty_channel;

        else if(session->scpSend_response[0] != 0) {
            size_t err_len;
            char *err_msg;

            err_len =
                _libssh2_channel_packet_data_len(session->scpSend_channel, 0);
            err_msg = LIBSSH2_ALLOC(session, err_len + 1);
            if(!err_msg) {
                _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                               "failed to get memory");
                goto scp_send_error;
            }

            /* Read the remote error message */
            rc = _libssh2_channel_read(session->scpSend_channel, 0,
                                       err_msg, err_len);
            if(rc > 0) {
                err_msg[err_len] = 0;
                _libssh2_debug(session, LIBSSH2_TRACE_SCP,
                               "got %02x %s", session->scpSend_response[0],
                               err_msg);
            }
            LIBSSH2_FREE(session, err_msg);
            _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                           "failed to send file");
            goto scp_send_error;
        }
    }

    session->scpSend_state = libssh2_NB_state_idle;
    return session->scpSend_channel;

  scp_send_empty_channel:
    /* the code only jumps here as a result of a zero read from channel_read()
       so we check EOF status to avoid getting stuck in a loop */
    if(libssh2_channel_eof(session->scpSend_channel)) {
        _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                       "Unexpected channel close");
    }
    else
        return session->scpSend_channel;
    /* fall-through */
  scp_send_error:
    tmp_err_code = session->err_code;
    tmp_err_msg = session->err_msg;
    while(libssh2_channel_free(session->scpSend_channel) ==
          LIBSSH2_ERROR_EAGAIN);
    session->err_code = tmp_err_code;
    session->err_msg = tmp_err_msg;
    session->scpSend_channel = NULL;
    session->scpSend_state = libssh2_NB_state_idle;
    return NULL;
}

```

这段代码是一个名为 `libssh2_scp_send_ex` 的函数，它是 SSH2 协议中的一个函数，用于通过 SCP 协议发送一个文件。

具体来说，该函数接受一个 `LIBSSH2_SESSION` 类型的会话，以及一个文件路径名（字符串）和文件大小。它还接受一个表示文件创建时间的 `long mtime` 和一个表示文件修改时间的 `long atime`。函数内部使用 `scp_send` 函数将文件发送到远程服务器，并在函数内部使用 `BLOCK_ADJUST_ERRNO` 函数来处理可能的错误。最后，函数返回一个指向 `LIBSSH2_CHANNEL` 类型的指针，该指针指向发送通道。


```cpp
/*
 * libssh2_scp_send_ex
 *
 * Send a file using SCP. Old API.
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_send_ex(LIBSSH2_SESSION *session, const char *path, int mode,
                    size_t size, long mtime, long atime)
{
    LIBSSH2_CHANNEL *ptr;
    BLOCK_ADJUST_ERRNO(ptr, session,
                       scp_send(session, path, mode, size,
                                (time_t)mtime, (time_t)atime));
    return ptr;
}

```

这段代码定义了一个名为 `libssh2_scp_send64` 的函数，用于通过 SCP 协议发送一个文件。以下是该函数的实现细节：

1. 函数参数：
 - `session`：当前会话句柄，用于与 SCP 服务器通信；
 - `path`：文件路径，用于从服务器下载的文件；
 - `mode`：文件模式，可以是 `0` 或 `1`，表示只读或可读写；
 - `size`：文件大小，以字节为单位；
 - `mtime`：文件修改时间，以秒为单位；
 - `atoime`：文件创建时间，以秒为单位；

2. 函数实现：

a. 首先，创建一个名为 `ptr` 的指向 `LIBSSH2_CHANNEL` 类型的指针；
b. 使用 `BLOCK_ADJUST_ERRNO` 函数，将 `ptr` 指向的用户数据块调整为其要发送的文件大小的倍数；
c. 使用 `scp_send` 函数，将文件从服务器下载到客户端；
d. 使用上面创建的 `ptr` 和 `scp_send` 函数，将文件发送到客户端。

3. 函数返回：

如果没有错误，函数将返回 `libssh2_channel` 类型的指针，指向发送文件的 `LIBSSH2_CHANNEL` 类型的数据块；如果有错误，函数将返回 `NULL`。


```cpp
/*
 * libssh2_scp_send64
 *
 * Send a file using SCP
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_send64(LIBSSH2_SESSION *session, const char *path, int mode,
                   libssh2_int64_t size, time_t mtime, time_t atime)
{
    LIBSSH2_CHANNEL *ptr;
    BLOCK_ADJUST_ERRNO(ptr, session,
                       scp_send(session, path, mode, size, mtime, atime));
    return ptr;
}

```

# `libssh2/src/session.c`

Daniel Stenberg


```cpp
/* Copyright (c) 2004-2007 Sara Golemon <sarag@libssh2.org>
 * Copyright (c) 2009-2015 by Daniel Stenberg
 * Copyright (c) 2010 Simon Josefsson <simon@josefsson.org>
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms,
 * with or without modification, are permitted provided
 * that the following conditions are met:
 *
 *   Redistributions of source code must retain the above
 *   copyright notice, this list of conditions and the
 *   following disclaimer.
 *
 *   Redistributions in binary form must reproduce the above
 *   copyright notice, this list of conditions and the following
 *   disclaimer in the documentation and/or other materials
 *   provided with the distribution.
 *
 *   Neither the name of the copyright holder nor the names
 *   of any other contributors may be used to endorse or
 *   promote products derived from this software without
 *   specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
 * CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
 * INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR
 * CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
 * BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 * SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
 * WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 * NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE
 * USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY
 * OF SUCH DAMAGE.
 */

```

这段代码是一个基于libssh2库的SSH客户端，作用是连接到远程服务器，并执行一系列操作，包括设置公钥、设置本地时间、获取用户信息、下载文件等。

具体来说，代码包含以下几个部分：

1. 引入了libssh2_priv.h、errno.h和<stdbool.h>头文件，这些头文件定义了SSH客户端和系统调用所需的函数和宏。
2. 引入了<errno.h>头文件，用于定义错误处理函数。
3. 包含了#ifdef Have_Unistd_H和#ifdef Have_Gettiemofday的注释，它们用于在编译时检查系统是否支持UNISTD头文件和Gettiemofday函数。
4. 包含了<stdlib.h>和<fcntl.h>头文件，用于定义标准库函数和文件输入输出操作。
5. 包含了所有出入口函数，包括SSH客户端的连接、登录、设置公钥、设置本地时间、获取用户信息、下载文件等功能。
6. 通过<sys/time.h>函数获取当前系统时间，如果当前系统支持UNISTD头文件，则使用UNISTD头文件中的函数获取。
7. 通过<alloca.h>函数分配一个指定大小的内存区域，用于存储获取的文件名和路径等信息。


```cpp
#include "libssh2_priv.h"
#include <errno.h>
#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif
#include <stdlib.h>
#include <fcntl.h>

#ifdef HAVE_GETTIMEOFDAY
#include <sys/time.h>
#endif
#ifdef HAVE_ALLOCA_H
#include <alloca.h>
#endif

```

这段代码是一个 C 语言程序，它包含多个头文件和函数声明。这些头文件和函数声明定义了一个名为 "transport.h" 的头文件。

具体来说，这段代码定义了一系列用于网络传输层的库函数。这些函数用于实现 SSH2 协议的客户端和服务器端功能。主要包括：

1. "libssh2_default_alloc" 函数：负责在运行时分配内存空间，该内存空间用于存储 SSL/TLS 密钥。

2. "ssh2_print_status" 函数：用于打印 SSL/TLS 密钥的使用状态。

3. "ssh2_print_local_address" 函数：用于打印客户端的本地 IP 地址。

4. "ssh2_print_remote_address" 函数：用于打印客户端的远程 IP 地址。

5. "ssh2_print_missing_server_sent_保护" 函数：用于处理客户端发送给服务器的请求，同时实现服务器端的验证。

6. "ssh2_print_response" 函数：用于打印服务器端对客户端的响应，包括 SSL/TLS 密钥。

7. "ssh2_parse_fd_puts" 函数：用于在客户端和服务器端之间传输文件数据。

8. "ssh2_parse_extended_options" 函数：用于解析 SSH2 协议的选项。

9. "ssh2_parse_stats" 函数：用于计算客户端和服务器端的统计信息。

10. "ssh2_main" 函数：用于实现 SSH2 协议的客户端和服务器端功能。

在这段代码中，还通过引入其他头文件的方式引入了 "session.h" 和 "channel.h" 两个头文件，这两个头文件可能用于实现客户端和服务器端的会话管理功能。


```cpp
#include "transport.h"
#include "session.h"
#include "channel.h"
#include "mac.h"
#include "misc.h"

/* libssh2_default_alloc
 */
static
LIBSSH2_ALLOC_FUNC(libssh2_default_alloc)
{
    (void) abstract;
    return malloc(count);
}

```

这两段代码定义了两个名为`libssh2_default_free`和`libssh2_default_realloc`的函数，属于SSH2库中的免费函数和重新分配函数。

`libssh2_default_free`函数用于释放一个指定指针`ptr`所指向的内存空间，它调用了另一个名为`free`的系统函数，将内存空间标记为释放，然后释放内存空间并返回。

`libssh2_default_realloc`函数用于在`ptr`指向的内存空间耗尽时重新分配内存空间，它调用了另一个名为`realloc`的系统函数，在`ptr`所指向的内存空间前面插入新的内存空间，并返回分配的内存空间大小。


```cpp
/* libssh2_default_free
 */
static
LIBSSH2_FREE_FUNC(libssh2_default_free)
{
    (void) abstract;
    free(ptr);
}

/* libssh2_default_realloc
 */
static
LIBSSH2_REALLOC_FUNC(libssh2_default_realloc)
{
    (void) abstract;
    return realloc(ptr, count);
}

```

This is a function in the SSH library that receives a banner from the client and checks it for errors. The banner is a line of text that the client sends to the server, and is typically used to confirm that the connection has been established and that the correct protocol is being used.

The function takes a single parameter, `banner_len`, which is the length of the banner. The function reads the banner character by character and stores them in the `session->remote.banner` buffer.

After the banner has been read, the function checks the length of the banner. If the length is zero, it sets the `session->banner_TxRx_state` to `libssh2_NB_state_idle` and sets the `session->banner_TxRx_total_send` to zero. If the banner length is not zero, the function sets the `session->banner_TxRx_state` to `libssh2_NB_state_idle`, sets the `session->banner_TxRx_total_send` to the total number of bytes to be sent, and then returns the `libssh2_error` value.

Finally, the function returns the `libssh2_error` value if an error occurred, or `LIBSSH2_ERROR_NONE` if no errors occurred.


```cpp
/*
 * banner_receive
 *
 * Wait for a hello from the remote host
 * Allocate a buffer and store the banner in session->remote.banner
 * Returns: 0 on success, LIBSSH2_ERROR_EAGAIN if read would block, negative
 * on failure
 */
static int
banner_receive(LIBSSH2_SESSION * session)
{
    int ret;
    int banner_len;

    if(session->banner_TxRx_state == libssh2_NB_state_idle) {
        banner_len = 0;

        session->banner_TxRx_state = libssh2_NB_state_created;
    }
    else {
        banner_len = session->banner_TxRx_total_send;
    }

    while((banner_len < (int) sizeof(session->banner_TxRx_banner)) &&
           ((banner_len == 0)
            || (session->banner_TxRx_banner[banner_len - 1] != '\n'))) {
        char c = '\0';

        /* no incoming block yet! */
        session->socket_block_directions &= ~LIBSSH2_SESSION_BLOCK_INBOUND;

        ret = LIBSSH2_RECV(session, &c, 1,
                            LIBSSH2_SOCKET_RECV_FLAGS(session));
        if(ret < 0) {
            if(session->api_block_mode || (ret != -EAGAIN))
                /* ignore EAGAIN when non-blocking */
                _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                               "Error recving %d bytes: %d", 1, -ret);
        }
        else
            _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                           "Recved %d bytes banner", ret);

        if(ret < 0) {
            if(ret == -EAGAIN) {
                session->socket_block_directions =
                    LIBSSH2_SESSION_BLOCK_INBOUND;
                session->banner_TxRx_total_send = banner_len;
                return LIBSSH2_ERROR_EAGAIN;
            }

            /* Some kinda error */
            session->banner_TxRx_state = libssh2_NB_state_idle;
            session->banner_TxRx_total_send = 0;
            return LIBSSH2_ERROR_SOCKET_RECV;
        }

        if(ret == 0) {
            session->socket_state = LIBSSH2_SOCKET_DISCONNECTED;
            return LIBSSH2_ERROR_SOCKET_DISCONNECT;
        }

        if(c == '\0') {
            /* NULLs are not allowed in SSH banners */
            session->banner_TxRx_state = libssh2_NB_state_idle;
            session->banner_TxRx_total_send = 0;
            return LIBSSH2_ERROR_BANNER_RECV;
        }

        session->banner_TxRx_banner[banner_len++] = c;
    }

    while(banner_len &&
           ((session->banner_TxRx_banner[banner_len - 1] == '\n') ||
            (session->banner_TxRx_banner[banner_len - 1] == '\r'))) {
        banner_len--;
    }

    /* From this point on, we are done here */
    session->banner_TxRx_state = libssh2_NB_state_idle;
    session->banner_TxRx_total_send = 0;

    if(!banner_len)
        return LIBSSH2_ERROR_BANNER_RECV;

    if(session->remote.banner)
        LIBSSH2_FREE(session, session->remote.banner);

    session->remote.banner = LIBSSH2_ALLOC(session, banner_len + 1);
    if(!session->remote.banner) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Error allocating space for remote banner");
    }
    memcpy(session->remote.banner, session->banner_TxRx_banner, banner_len);
    session->remote.banner[banner_len] = '\0';
    _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Received Banner: %s",
                   session->remote.banner);
    return LIBSSH2_ERROR_NONE;
}

```

这段代码是一个名为 `banner_send` 的函数，属于 LIBSSH2_SESSION 类的成员函数。它的作用是发送默认的 banner，或者通过 libssh2_setopt_string 设置的 banner。

具体来说，函数接受一个 LIBSSH2_SESSION 类型的参数，然后返回一个 int 类型的值。在函数内部，首先定义了一个 `banner` 变量，该变量是一个字符数组，代表默认的 banner，并且通过 libssh2_setopt_string 设置。然后定义了一个 `banner_len` 变量，它存储了 `banner` 变量的长度，减去 1（因为 Banner 字符串中最后一个字符是空字符 '\0'，需要在计算长度时减去）。接下来调用一个名为 `ret` 的变量，该变量存储了发送 Banner 字符串的返回值。如果函数返回 LIBSSH2_ERROR_EAGAIN，说明发送 Banner 字符串会失败，需要再次调用该函数；如果函数返回 0，说明已经发送了 Banner 字符串，并且可以继续发送数据。

总之，该函数的作用是发送默认的或通过设置的 Banner 字符串，并且在发送过程中需要注意是否出错，出错的话需要重新调用函数。


```cpp
/*
 * banner_send
 *
 * Send the default banner, or the one set via libssh2_setopt_string
 *
 * Returns LIBSSH2_ERROR_EAGAIN if it would block - and if it does so, you
 * should call this function again as soon as it is likely that more data can
 * be sent, and this function should then be called with the same argument set
 * (same data pointer and same data_len) until zero or failure is returned.
 */
static int
banner_send(LIBSSH2_SESSION * session)
{
    char *banner = (char *) LIBSSH2_SSH_DEFAULT_BANNER_WITH_CRLF;
    int banner_len = sizeof(LIBSSH2_SSH_DEFAULT_BANNER_WITH_CRLF) - 1;
    ssize_t ret;
```

这段代码的作用是检查当前SSH会话的 banner 字段是否为空，如果为空，则执行一系列操作，然后输出一个包含256个字符的 banner 字符串，并将其复制到名为 banner_dup 的数组中。如果 banner 字符串不为空，则执行以下操作：首先，使用 memcpy 函数将banner字符串复制到banner_dup数组中，然后使用 memcpy 函数将banner_dup数组的最后一个字符复制到banner数组中，并将banner数组的长度设为256，这样就可以在输出的调试信息中避免包含一个CRLF换行符。最后，使用 _libssh2_debug 函数将设置好的banner字符串输出到调试信息中。


```cpp
#ifdef LIBSSH2DEBUG
    char banner_dup[256];
#endif

    if(session->banner_TxRx_state == libssh2_NB_state_idle) {
        if(session->local.banner) {
            /* setopt_string will have given us our \r\n characters */
            banner_len = strlen((char *) session->local.banner);
            banner = (char *) session->local.banner;
        }
#ifdef LIBSSH2DEBUG
        /* Hack and slash to avoid sending CRLF in debug output */
        if(banner_len < 256) {
            memcpy(banner_dup, banner, banner_len - 2);
            banner_dup[banner_len - 2] = '\0';
        }
        else {
            memcpy(banner_dup, banner, 255);
            banner_dup[255] = '\0';
        }

        _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Sending Banner: %s",
                       banner_dup);
```

This is a function in the LibSSH2 library that sends a packet over a network connection. The function takes a session object, a banner string, and a send flag, and returns an error code if the packet could not be sent or if there was an error sending the packet. If the packet is sent successfully, the function returns a positive error code, and if the packet is sent with errors, the function returns a negative error code.


```cpp
#endif

        session->banner_TxRx_state = libssh2_NB_state_created;
    }

    /* no outgoing block yet! */
    session->socket_block_directions &= ~LIBSSH2_SESSION_BLOCK_OUTBOUND;

    ret = LIBSSH2_SEND(session,
                        banner + session->banner_TxRx_total_send,
                        banner_len - session->banner_TxRx_total_send,
                        LIBSSH2_SOCKET_SEND_FLAGS(session));
    if(ret < 0)
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                       "Error sending %d bytes: %d",
                       banner_len - session->banner_TxRx_total_send, -ret);
    else
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                       "Sent %d/%d bytes at %p+%d", ret,
                       banner_len - session->banner_TxRx_total_send,
                       banner, session->banner_TxRx_total_send);

    if(ret != (banner_len - session->banner_TxRx_total_send)) {
        if(ret >= 0 || ret == -EAGAIN) {
            /* the whole packet could not be sent, save the what was */
            session->socket_block_directions =
                LIBSSH2_SESSION_BLOCK_OUTBOUND;
            if(ret > 0)
                session->banner_TxRx_total_send += ret;
            return LIBSSH2_ERROR_EAGAIN;
        }
        session->banner_TxRx_state = libssh2_NB_state_idle;
        session->banner_TxRx_total_send = 0;
        return LIBSSH2_ERROR_SOCKET_RECV;
    }

    /* Set the state back to idle */
    session->banner_TxRx_state = libssh2_NB_state_idle;
    session->banner_TxRx_total_send = 0;

    return 0;
}

```

这段代码是一个名为“session_nonblock()”的函数，它用于设置一个套接字的非阻塞或阻塞模式，通过传递一个名为“nonblock”的布尔参数。

首先，函数中使用fcntl函数获取套接字的当前非阻塞状态，然后通过if语句判断传来的非阻塞参数是否为真，如果是，则调用fcntl函数将套接字的非阻塞状态设置为当前状态，否则将非阻塞状态设置为阻塞状态。最后，使用getflag和setflag函数分别获取和设置套接线的非阻塞标志，并将它们组合在一起，以返回非阻塞或阻塞状态的值。


```cpp
/*
 * session_nonblock() sets the given socket to either blocking or
 * non-blocking mode based on the 'nonblock' boolean argument. This function
 * is copied from the libcurl sources with permission.
 */
static int
session_nonblock(libssh2_socket_t sockfd,   /* operate on this */
                 int nonblock /* TRUE or FALSE */ )
{
#undef SETBLOCK
#define SETBLOCK 0
#ifdef HAVE_O_NONBLOCK
    /* most recent unix versions */
    int flags;

    flags = fcntl(sockfd, F_GETFL, 0);
    if(nonblock)
        return fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);
    else
        return fcntl(sockfd, F_SETFL, flags & (~O_NONBLOCK));
```

这段代码定义了一个宏SETBLOCK，其值为1。同时，还定义了一个内联函数UNION,UNION实质上是一个宏，其代码如下：

```cpp
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/types.h>
```

SETBLOCK宏定义用于定义某个系统是否支持I/O非阻塞IO操作。如果是，SETBLOCK的值为1，否则为0。

UNION宏定义则是UNION语言的一种特性，用于将两个C语言源代码区域连接起来，使得程序可以方便地同时使用多个C语言源代码区域。其语法类似于宏定义，但使用的是宏而非函数。

代码中包含两个UNION，第一个UNION将宏SETBLOCK的值定义为1，第二个UNION则是UNION语言的一种特性，用于将两个C语言源代码区域连接起来。这段代码的作用就是定义了一个宏SETBLOCK，并在满足某些条件时将SETBLOCK的值设为1。如果这个系统支持I/O非阻塞IO操作，则会调用非阻塞函数成功，SETBLOCK的值为1；否则，SETBLOCK的值为0。对于Windows，则会调用非阻塞函数失败，SETBLOCK的值为0。


```cpp
#undef SETBLOCK
#define SETBLOCK 1
#endif

#if defined(HAVE_FIONBIO) && (SETBLOCK == 0)
    /* older unix versions and VMS*/
    int flags;

    flags = nonblock;
    return ioctl(sockfd, FIONBIO, &flags);
#undef SETBLOCK
#define SETBLOCK 2
#endif

#if defined(HAVE_IOCTLSOCKET) && (SETBLOCK == 0)
    /* Windows? */
    unsigned long flags;
    flags = nonblock;

    return ioctlsocket(sockfd, FIONBIO, &flags);
```

这段代码定义了一个名为SETBLOCK的宏，其值为3。这个宏被用来定义一个名为SETBLOCK_EXPOSE的函数。同时，还定义了一个名为HAVE_IOCTLSOCKET_CASE的函数指针变量，其值为真（即HAVE_IOCTLSOCKET_CASE函数存在）。接着，代码会根据HAVE_IOCTLSOCKET_CASE的值来判断是否使用SETBLOCK宏，如果HAVE_IOCTLSOCKET_CASE为真，那么就会执行SETBLOCK宏，将其值为4。最后，代码还会根据HAVE_SO_NONBLOCK的值来判断是否使用SETBLOCK宏，如果HAVE_SO_NONBLOCK为真，那么就会执行SETBLOCK宏，其值为1，如果没有设置非阻塞IO，那么就会执行SETBLOCK宏，其值为0。


```cpp
#undef SETBLOCK
#define SETBLOCK 3
#endif

#if defined(HAVE_IOCTLSOCKET_CASE) && (SETBLOCK == 0)
    /* presumably for Amiga */
    return IoctlSocket(sockfd, FIONBIO, (long) nonblock);
#undef SETBLOCK
#define SETBLOCK 4
#endif

#if defined(HAVE_SO_NONBLOCK) && (SETBLOCK == 0)
    /* BeOS */
    long b = nonblock ? 1 : 0;
    return setsockopt(sockfd, SOL_SOCKET, SO_NONBLOCK, &b, sizeof(b));
```

这段代码定义了一个宏SETBLOCK，其值为5。这个宏被用来定义一个区块链SETBLOCK风格的阻塞函数。

如果这段代码所在的目录目录文件系统上有"/dev/null"这个设备，那么SETBLOCK宏定义的阻塞函数会返回0，即成功。否则，如果目录文件系统上没有"/dev/null"，那么SETBLOCK宏定义的阻塞函数会抛出"no non-blocking method was found/used/set"的错误。

最重要的是，这段代码没有对SETBLOCK宏进行任何实际的定义，只是简单地定义了一个宏并将其值为5。因此，如果有人尝试定义SETBLOCK宏，那么编译器不会识别宏定义并抛出错误。


```cpp
#undef SETBLOCK
#define SETBLOCK 5
#endif

#ifdef HAVE_DISABLED_NONBLOCKING
    return 0;                   /* returns success */
#undef SETBLOCK
#define SETBLOCK 6
#endif

#if(SETBLOCK == 0)
#error "no non-blocking method was found/used/set"
#endif
}

```

这段代码是一个用于获取套接字非阻塞状态的函数。函数接收一个整数参数，表示套接字的类型（阻塞或非阻塞），并返回对应的非阻塞状态。

函数首先通过调用fcntl函数获取套接字的非阻塞标志位。如果该函数在尝试获取非阻塞状态时失败，将返回1，表示当前套接字处于阻塞状态。否则，如果套接字已经处于非阻塞状态，函数将返回该状态的值。

根据函数体中的注释，此代码在尝试使用fcntl函数时，如果失败将使用非阻塞套接字。这意味着如果套接字当前处于阻塞状态，该函数将返回1；如果套接字当前处于非阻塞状态，该函数将返回该状态的值。


```cpp
/*
 * get_socket_nonblocking()
 *
 * gets the given blocking or non-blocking state of the socket.
 */
static int
get_socket_nonblocking(int sockfd)
{                               /* operate on this */
#undef GETBLOCK
#define GETBLOCK 0
#ifdef HAVE_O_NONBLOCK
    /* most recent unix versions */
    int flags = fcntl(sockfd, F_GETFL, 0);

    if(flags == -1) {
        /* Assume blocking on error */
        return 1;
    }
    return (flags & O_NONBLOCK);
```

这段代码定义了一个宏 GETBLOCK，其值为 1。该宏定义在 #include <stdint.h> 中。

接下来，代码中定义了一个名为 GETBLOCK 的宏，其值为 1。并通过 #define GETBLOCK 1 来定义了该宏。

接着，代码中通过 if 语句判断了是否定义了名为 GETBLOCK 的宏。如果是，则执行该宏体，即判断调用 getsockopt() 函数时第二个参数（表示发送到套接字中的数据长度）是否为 0。如果是，说明调用者的函数在内尔格式的请求中发送了数据，需要阻塞套接字。因此，代码中返回了 Option_Value 的值，即 1。

最后，代码中使用 SO_ERROR 和 SO_WOULDBLOCK 两个选项，并通过 getsockopt() 函数获取了它们对应的值，并将其存储到 Option_Value 中。


```cpp
#undef GETBLOCK
#define GETBLOCK 1
#endif

#if defined(WSAEWOULDBLOCK) && (GETBLOCK == 0)
    /* Windows? */
    unsigned int option_value;
    socklen_t option_len = sizeof(option_value);

    if(getsockopt
        (sockfd, SOL_SOCKET, SO_ERROR, (void *) &option_value, &option_len)) {
        /* Assume blocking on error */
        return 1;
    }
    return (int) option_value;
```

这段代码定义了一个名为GETBLOCK的宏，其值为2。然后在代码中使用宏定义，将GETBLOCK的值定义为5。接着，代码中使用了一个if语句，该语句检查HAVE_SO_NONBLOCK是否为真，以及GETBLOCK是否为0。如果是，那么代码将执行以下操作：尝试调用getsockopt函数，并获取返回值，如果获取失败，则将阻塞设置为1。最后，代码将返回阻塞状态的值。


```cpp
#undef GETBLOCK
#define GETBLOCK 2
#endif

#if defined(HAVE_SO_NONBLOCK) && (GETBLOCK == 0)
    /* BeOS */
    long b;
    if(getsockopt(sockfd, SOL_SOCKET, SO_NONBLOCK, &b, sizeof(b))) {
        /* Assume blocking on error */
        return 1;
    }
    return (int) b;
#undef GETBLOCK
#define GETBLOCK 5
#endif

```

这段代码的作用是检查服务器套接字是否支持 VMS TCP/IP 服务，并且在服务器套接字上发送一个请求，以获取当前套接字的状态。

具体来说，代码首先检查是否定义了 SO_STATE 标识符，以及是否定义了 __VMS 标识符。如果这两个标识符都被定义了，那么代码会执行下一个操作。

接下来，代码调用 getsockopt 函数来获取服务器套接字的状态。该函数通过套接字套接字指针（sockfd）和套接字状态字（SO_STATE）来获取服务器套接字的状态。套接字状态字包含套接字的阻塞状态(GETBLOCK)的值。如果套接字阻塞状态为 0，那么代码会执行后续操作。

如果调用 getsockopt 函数时出现错误，那么代码会返回一个值。如果服务器套接字的状态无法确定，或者套接字阻塞状态为非 0，那么代码会返回一个值，并且使用上面得到的最大值将套接字状态清零。


```cpp
#if defined(SO_STATE) && defined(__VMS) && (GETBLOCK == 0)

    /* VMS TCP/IP Services */

    size_t sockstat = 0;
    int    callstat = 0;
    size_t size = sizeof(int);

    callstat = getsockopt(sockfd, SOL_SOCKET, SO_STATE,
                                  (char *)&sockstat, &size);
    if(callstat == -1) return 0;
    if((sockstat&SS_NBIO) != 0) return 1;
    return 0;

#undef GETBLOCK
```

这段代码是一个C/C++预处理指令，主要作用是定义了一些符号常量。

首先，定义了一个名为 GETBLOCK 的符号常量，其值为 6。然后通过宏定义的方式，对该符号常量进行了化简，变成了 7。这个符号常量在后续的代码中还会被使用。

接着，通过宏定义的方式，定义了一个名为 GETBLOCK 的符号常量，其值为 0。同时通过头文件的方式，定义了一个名为 GETBLOCK 的符号常量，其值为 6。这个符号常量在后续的代码中也会被使用。

最后，通过条件语句判断，当 GETBLOCK 的值为 0 时，会输出一个错误信息，提示没有找到非阻塞方法。否则，如果 GETBLOCK 的值为 6，则不做任何处理。


```cpp
#define GETBLOCK 6
#endif

#ifdef HAVE_DISABLED_NONBLOCKING
    return 1;                   /* returns blocking */
#undef GETBLOCK
#define GETBLOCK 7
#endif

#if(GETBLOCK == 0)
#error "no non-blocking method was found/used/get"
#endif
}

/* libssh2_session_banner_set
 * Set the local banner to use in the server handshake.
 */
```

这段代码是一个名为`libssh2_session_banner_set`的函数，属于`libssh2_api`库。它的作用是接受一个`LIBSSH2_SESSION`类型的数据作为参数，以及一个字符串`banner`，然后将`banner`设置为客户端和服务器之间的一个新的banner，并将其设置成功。

具体来说，这段代码首先检查`banner`是否为空，如果是，则直接返回0。如果不是，那么它会将`banner`的长度存储到一个名为`session->local.banner`的变量中，并从`libssh2_session_banner_set`函数中分配一些内存来存储`banner`的字符串。然后，它会将`banner`的字符串复制到`session->local.banner`中，并将其末尾三个字符设置为'\0'，这样就可以在输出时明确知道banner的结束符。最后，它使用`_libssh2_debug`函数输出一条调试信息，以便开发人员查看 session 的输出。


```cpp
LIBSSH2_API int
libssh2_session_banner_set(LIBSSH2_SESSION * session, const char *banner)
{
    size_t banner_len = banner ? strlen(banner) : 0;

    if(session->local.banner) {
        LIBSSH2_FREE(session, session->local.banner);
        session->local.banner = NULL;
    }

    if(!banner_len)
        return 0;

    session->local.banner = LIBSSH2_ALLOC(session, banner_len + 3);
    if(!session->local.banner) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for local banner");
    }

    memcpy(session->local.banner, banner, banner_len);

    /* first zero terminate like this so that the debug output is nice */
    session->local.banner[banner_len] = '\0';
    _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Setting local Banner: %s",
                   session->local.banner);
    session->local.banner[banner_len++] = '\r';
    session->local.banner[banner_len++] = '\n';
    session->local.banner[banner_len] = '\0';

    return 0;
}

```

这段代码定义了一个名为`libssh2_banner_set`的函数，属于`libssh2_session`系列的函数。这个函数的作用是设置本地banner，用于在SSH客户端和服务器之间传输消息时显示。

函数的参数包括两个：`session`表示要设置banner的SSH会话，`banner`是一个字符串，表示在SSH客户端和服务器之间传输消息时所显示的本地banner。

函数实现中，首先调用`libssh2_session_banner_set`函数，将`banner`参数设置为`banner`，然后返回`libssh2_session_banner_set`的返回值。


```cpp
/* libssh2_banner_set
 * Set the local banner. DEPRECATED VERSION
 */
LIBSSH2_API int
libssh2_banner_set(LIBSSH2_SESSION * session, const char *banner)
{
    return libssh2_session_banner_set(session, banner);
}

/*
 * libssh2_session_init_ex
 *
 * Allocate and initialize a libssh2 session structure. Allows for malloc
 * callbacks in case the calling program has its own memory manager It's
 * allowable (but unadvisable) to define some but not all of the malloc
 * callbacks An additional pointer value may be optionally passed to be sent
 * to the callbacks (so they know who's asking)
 */
```

这段代码定义了一个名为`LIBSSH2_API`的函数，它的作用是初始化一个`LIBSSH2_SESSION`类型的数据结构。

具体来说，这段代码实现了以下几个步骤：

1. 调用`libssh2_session_init_ex`函数，设置`libssh2_session_init_ex`函数的参数。

2. 如果`my_alloc`函数、`my_free`函数或`my_realloc`函数已存在，则将它们的地址传递给`libssh2_default_alloc`函数、`libssh2_default_free`函数和`libssh2_default_realloc`函数，以便在需要时动态分配内存。

3. 创建一个`LIBSSH2_SESSION`类型的数据结构，其大小为`sizeof(LIBSSH2_SESSION)`，同时设置其几个成员变量：

- `session`：存储`LIBSSH2_SESSION`类型数据的指针。
- `local_alloc`：存储分配内存的函数的地址。
- `local_free`：存储释放内存的函数的地址。
- `local_realloc`：存储重新分配内存的函数的地址。
- `send`：存储`libssh2_send`函数的地址。
- `recv`：存储`libssh2_recv`函数的地址。
- `abstract`：存储传递给`libssh2_api_timeout`和`libssh2_api_block_mode`函数的用户定义的抽象。

4. 设置`session`的几个成员函数的默认值：

- `api_timeout`：设置为0，表示使用默认的无超时时间限制的API。
- `api_block_mode`：设置为1，表示阻塞API的使用。

5. 在`libssh2_session_init_ex`函数中，记录调用日志，以便在需要时打印。

6. 初始化函数的返回值，如果初始化成功则返回`session`指针。


```cpp
LIBSSH2_API LIBSSH2_SESSION *
libssh2_session_init_ex(LIBSSH2_ALLOC_FUNC((*my_alloc)),
                        LIBSSH2_FREE_FUNC((*my_free)),
                        LIBSSH2_REALLOC_FUNC((*my_realloc)), void *abstract)
{
    LIBSSH2_ALLOC_FUNC((*local_alloc)) = libssh2_default_alloc;
    LIBSSH2_FREE_FUNC((*local_free)) = libssh2_default_free;
    LIBSSH2_REALLOC_FUNC((*local_realloc)) = libssh2_default_realloc;
    LIBSSH2_SESSION *session;

    if(my_alloc) {
        local_alloc = my_alloc;
    }
    if(my_free) {
        local_free = my_free;
    }
    if(my_realloc) {
        local_realloc = my_realloc;
    }

    session = local_alloc(sizeof(LIBSSH2_SESSION), &abstract);
    if(session) {
        memset(session, 0, sizeof(LIBSSH2_SESSION));
        session->alloc = local_alloc;
        session->free = local_free;
        session->realloc = local_realloc;
        session->send = _libssh2_send;
        session->recv = _libssh2_recv;
        session->abstract = abstract;
        session->api_timeout = 0; /* timeout-free API by default */
        session->api_block_mode = 1; /* blocking API by default */
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "New session resource allocated");
        _libssh2_init_if_needed();
    }
    return session;
}

```

The `libssh2_session_callback_set()` function takes a pointer to a `void` function and a `void` pointer as its arguments. However, this function reliance on a type cast to `void` pointer which is not allowed in ISO C.

The `libssh2_debug()` function is used to log information about the session, such as connection errors or message debugging. The `LIBSSH2_DEBUG()` macro generates this debug message if an error occurs.

The `session` argument is of type `LIBSSH2_SESSION`, which is an pointer to an `LIBSSH2_SESSION` object. It is passed to the function by the `libssh2_init()` function when it creates a new `LIBSSH2_SESSION` object.

The `cbtype` argument is an integer that is used to determine the type of the callback function. The values for `cbtype` are `LIBSSH2_CALLBACK_IGNORE`, `LIBSSH2_CALLBACK_DEBUG`, `LIBSSH2_CALLBACK_DISCONNECT`, `LIBSSH2_CALLBACK_MACERROR`, `LIBSSH2_CALLBACK_SEND`, and `LIBSSH2_CALLBACK_RECV`.

The `callback` argument is a `void` pointer that is passed to the function by the `libssh2_session_callback_set()` function. It is passed to the function by the `libssh2_set_session_callback()` function when it sets the callback function for the session.


```cpp
/*
 * libssh2_session_callback_set
 *
 * Set (or reset) a callback function
 * Returns the prior address
 *
 * ALERT: this function relies on that we can typecast function pointers
 * to void pointers, which isn't allowed in ISO C!
 */
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wpedantic"
LIBSSH2_API void *
libssh2_session_callback_set(LIBSSH2_SESSION * session,
                             int cbtype, void *callback)
{
    void *oldcb;

    switch(cbtype) {
    case LIBSSH2_CALLBACK_IGNORE:
        oldcb = session->ssh_msg_ignore;
        session->ssh_msg_ignore = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_DEBUG:
        oldcb = session->ssh_msg_debug;
        session->ssh_msg_debug = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_DISCONNECT:
        oldcb = session->ssh_msg_disconnect;
        session->ssh_msg_disconnect = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_MACERROR:
        oldcb = session->macerror;
        session->macerror = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_X11:
        oldcb = session->x11;
        session->x11 = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_SEND:
        oldcb = session->send;
        session->send = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_RECV:
        oldcb = session->recv;
        session->recv = callback;
        return oldcb;
    }
    _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Setting Callback %d",
                   cbtype);

    return NULL;
}
```

This function appears to be the main function for keeping an SSH connection open and alive. It sets various variables and performs some basic error handling.

The function first sets the error code to LIBSSH2_ERROR_NONE, since SSH2 does not use this error code by default.

It then sets the timeout to the value of the API timeout, and sets a flag indicating whether the connection has been closed or the time to the next timeout has expired.

Next, it sets up a loop that waits for either a call to `libssh2_keepalive_send()` or a timeout. It performs some additional error handling to avoid the user seeing an error message if the API timeout expires. If the timeout does not occur, it waits for a call to `libssh2_keepalive_send()`.

Finally, it sets the `has_timeout` flag based on whether the connection has been closed or the timeout has occurred.


```cpp
#pragma GCC diagnostic pop

/*
 * _libssh2_wait_socket()
 *
 * Utility function that waits for action on the socket. Returns 0 when ready
 * to run again or error on timeout.
 */
int _libssh2_wait_socket(LIBSSH2_SESSION *session, time_t start_time)
{
    int rc;
    int seconds_to_next;
    int dir;
    int has_timeout;
    long ms_to_next = 0;
    long elapsed_ms;

    /* since libssh2 often sets EAGAIN internally before this function is
       called, we can decrease some amount of confusion in user programs by
       resetting the error code in this function to reduce the risk of EAGAIN
       being stored as error when a blocking function has returned */
    session->err_code = LIBSSH2_ERROR_NONE;

    rc = libssh2_keepalive_send(session, &seconds_to_next);
    if(rc)
        return rc;

    ms_to_next = seconds_to_next * 1000;

    /* figure out what to wait for */
    dir = libssh2_session_block_directions(session);

    if(!dir) {
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                       "Nothing to wait for in wait_socket");
        /* To avoid that we hang below just because there's nothing set to
           wait for, we timeout on 1 second to also avoid busy-looping
           during this condition */
        ms_to_next = 1000;
    }

    if(session->api_timeout > 0 &&
        (seconds_to_next == 0 ||
         ms_to_next > session->api_timeout)) {
        time_t now = time(NULL);
        elapsed_ms = (long)(1000*difftime(now, start_time));
        if(elapsed_ms > session->api_timeout) {
            return _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                                  "API timeout expired");
        }
        ms_to_next = (session->api_timeout - elapsed_ms);
        has_timeout = 1;
    }
    else if(ms_to_next > 0) {
        has_timeout = 1;
    }
    else
        has_timeout = 0;

```

这段代码是一个使用Poll的网络安全工具，用于在Linux系统上监听套接字连接。

它首先定义了一个名为sockets的1结构体数组，用于存储所有的套接字。然后，它创建了一个指向套接字文件的fd变量，并将其初始化为服务器套接字的fd。

接下来，它枚举了与服务器套接字相关的Poll事件，并设置可用的事件列表为POLLIN和POLLOUT。如果dir的值与LIBSSH2_SESSION_BLOCK_INBOUND或LIBSSH2_SESSION_BLOCK_OUTBOUND相对应，它将设置相应事件的值。

最后，它调用poll函数并传递sockets数组和fd变量作为参数，得到一个rc值，用于判断是否成功监听套接字。如果rc值为0，则表示成功监听套接字，如果有任何事件发生，则rc值为-1，并返回下一个事件编号。


```cpp
#ifdef HAVE_POLL
    {
        struct pollfd sockets[1];

        sockets[0].fd = session->socket_fd;
        sockets[0].events = 0;
        sockets[0].revents = 0;

        if(dir & LIBSSH2_SESSION_BLOCK_INBOUND)
            sockets[0].events |= POLLIN;

        if(dir & LIBSSH2_SESSION_BLOCK_OUTBOUND)
            sockets[0].events |= POLLOUT;

        rc = poll(sockets, 1, has_timeout?ms_to_next: -1);
    }
```

这段代码是一个 C 语言程序，它是一个用于在 Linux 系统上监听文件描述符 (如 TCP 连接) 的 select 函数。select 函数接受三个参数：

1. 指向文件描述符 (如 TCP 连接) 的指针，通常是一个整数类型。
2. 指向文件描述符的指针，通常是一个整数类型。
3. 指向时间戳 (timeval) 的结构体，它存储了当前时间的大小。

该程序的作用是监控服务器套接字上的文件描述符，以便在有数据可读或写时通知客户端。

具体来说，程序首先创建了两个文件描述符的指针 (fd_set *writefd 和 fd_set *readfd)，并将它们初始化为 NULL。然后，程序设置了一个名为 tv 的结构体，它存储了一个当前时间的定时器，以每秒 1000 毫秒的频率更新。

接下来，程序通过调用 select 函数来监控服务器套接字上的文件描述符。这个函数接受四个参数：

1. 一个指向文件描述符的指针，通常是一个整数类型。
2. 一个指向文件描述符的指针，通常是一个整数类型。
3. 一个指向时间戳的指针，它存储了当前时间的大小。
4. 一个指向文件描述符的指针，通常是一个整数类型。

如果其中任何一个是非零，函数就会返回一个状态码，指示套接字是否应该继续监听该文件描述符。如果所有参数都为非零，那么 select 函数就会一直运行，直到被中断或套接字中没有文件描述符可以监听为止。


```cpp
#else
    {
        fd_set rfd;
        fd_set wfd;
        fd_set *writefd = NULL;
        fd_set *readfd = NULL;
        struct timeval tv;

        tv.tv_sec = ms_to_next / 1000;
        tv.tv_usec = (ms_to_next - tv.tv_sec*1000) * 1000;

        if(dir & LIBSSH2_SESSION_BLOCK_INBOUND) {
            FD_ZERO(&rfd);
            FD_SET(session->socket_fd, &rfd);
            readfd = &rfd;
        }

        if(dir & LIBSSH2_SESSION_BLOCK_OUTBOUND) {
            FD_ZERO(&wfd);
            FD_SET(session->socket_fd, &wfd);
            writefd = &wfd;
        }

        rc = select(session->socket_fd + 1, readfd, writefd, NULL,
                    has_timeout ? &tv : NULL);
    }
```

This is a function in the libssh2 library that handles the startup phase of SSH. It is responsible for asking the server for the ssh-userauth service, and if it is not available, it will return an error.

It works by first checking the current startup state of the session. If it is in the state_sent4, it means that it has sent the ssh-userauth service request and it will wait for the server's response. If it is not, it starts to send the request and if it's successful it sets the state to idle.

If the startup state is not correct, it will return an error. If the startup state is correct, it will send the ssh-userauth service request and wait for the server's response.


```cpp
#endif
    if(rc == 0) {
        return _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                              "Timed out waiting on socket");
    }
    if(rc < 0) {
        return _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                              "Error waiting on socket");
    }

    return 0; /* ready to try again */
}

static int
session_startup(LIBSSH2_SESSION *session, libssh2_socket_t sock)
{
    int rc;

    if(session->startup_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "session_startup for socket %d", sock);
        if(LIBSSH2_INVALID_SOCKET == sock) {
            /* Did we forget something? */
            return _libssh2_error(session, LIBSSH2_ERROR_BAD_SOCKET,
                                  "Bad socket provided");
        }
        session->socket_fd = sock;

        session->socket_prev_blockstate =
            !get_socket_nonblocking(session->socket_fd);

        if(session->socket_prev_blockstate) {
            /* If in blocking state change to non-blocking */
            rc = session_nonblock(session->socket_fd, 1);
            if(rc) {
                return _libssh2_error(session, rc,
                                      "Failed changing socket's "
                                      "blocking state to non-blocking");
            }
        }

        session->startup_state = libssh2_NB_state_created;
    }

    if(session->startup_state == libssh2_NB_state_created) {
        rc = banner_send(session);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
        else if(rc) {
            return _libssh2_error(session, rc,
                                  "Failed sending banner");
        }
        session->startup_state = libssh2_NB_state_sent;
        session->banner_TxRx_state = libssh2_NB_state_idle;
    }

    if(session->startup_state == libssh2_NB_state_sent) {
        do {
            rc = banner_receive(session);
            if(rc == LIBSSH2_ERROR_EAGAIN)
                return rc;
            else if(rc)
                return _libssh2_error(session, rc,
                                      "Failed getting banner");
        } while(strncmp("SSH-", (char *)session->remote.banner, 4));

        session->startup_state = libssh2_NB_state_sent1;
    }

    if(session->startup_state == libssh2_NB_state_sent1) {
        rc = _libssh2_kex_exchange(session, 0, &session->startup_key_state);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
        else if(rc)
            return _libssh2_error(session, rc,
                                  "Unable to exchange encryption keys");

        session->startup_state = libssh2_NB_state_sent2;
    }

    if(session->startup_state == libssh2_NB_state_sent2) {
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "Requesting userauth service");

        /* Request the userauth service */
        session->startup_service[0] = SSH_MSG_SERVICE_REQUEST;
        _libssh2_htonu32(session->startup_service + 1,
                         sizeof("ssh-userauth") - 1);
        memcpy(session->startup_service + 5, "ssh-userauth",
               sizeof("ssh-userauth") - 1);

        session->startup_state = libssh2_NB_state_sent3;
    }

    if(session->startup_state == libssh2_NB_state_sent3) {
        rc = _libssh2_transport_send(session, session->startup_service,
                                     sizeof("ssh-userauth") + 5 - 1,
                                     NULL, 0);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
        else if(rc) {
            return _libssh2_error(session, rc,
                                  "Unable to ask for ssh-userauth service");
        }

        session->startup_state = libssh2_NB_state_sent4;
    }

    if(session->startup_state == libssh2_NB_state_sent4) {
        rc = _libssh2_packet_require(session, SSH_MSG_SERVICE_ACCEPT,
                                     &session->startup_data,
                                     &session->startup_data_len, 0, NULL, 0,
                                     &session->startup_req_state);
        if(rc)
            return rc;

        if(session->startup_data_len < 5) {
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Unexpected packet length");
        }

        session->startup_service_length =
            _libssh2_ntohu32(session->startup_data + 1);


        if((session->startup_service_length != (sizeof("ssh-userauth") - 1))
            || strncmp("ssh-userauth", (char *) session->startup_data + 5,
                       session->startup_service_length)) {
            LIBSSH2_FREE(session, session->startup_data);
            session->startup_data = NULL;
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Invalid response received from server");
        }
        LIBSSH2_FREE(session, session->startup_data);
        session->startup_data = NULL;

        session->startup_state = libssh2_NB_state_idle;

        return 0;
    }

    /* just for safety return some error */
    return LIBSSH2_ERROR_INVAL;
}

```

这段代码是一个名为 `libssh2_session_handshake()` 的函数，它是 LibSSH2 协议中的一个函数。它的作用是接受一个 `LIBSSH2_SESSION` 类型的参数 `session` 和一个 `libssh2_socket_t` 类型的参数 `sock`。

具体来说，这个函数接受一个开始会话的 `LIBSSH2_SESSION` 类型的对象和一个已连接的 `libssh2_socket_t` 类型的指针 `sock`。它使用 `session_startup()` 函数来初始化会话对 `session` 和 `sock`，然后使用 `BLOCK_ADJUST()` 函数来确保递归调用过程中的 `rc` 不会被其他调用中断。最后，它返回一个整数，成功则返回 0，失败则返回一个非零值。

这段代码定义了一个名为 `libssh2_session_handshake()` 的函数，用于初始化安全套接字（SSH）会话。这个函数接受两个参数：一个 `LIBSSH2_SESSION` 类型的会话对象 `session` 和一个 `libssh2_socket_t` 类型的已连接的套接字 `sock`。它通过调用 `session_startup()` 函数来建立会话对，然后使用 `BLOCK_ADJUST()` 函数来确保递归调用过程中的 `rc` 不会被其他调用中断。最后，它返回一个整数，成功则返回 0，失败则返回一个非零值。


```cpp
/*
 * libssh2_session_handshake()
 *
 * session: LIBSSH2_SESSION struct allocated and owned by the calling program
 * sock:    *must* be populated with an opened and connected socket.
 *
 * Returns: 0 on success, or non-zero on failure
 */
LIBSSH2_API int
libssh2_session_handshake(LIBSSH2_SESSION *session, libssh2_socket_t sock)
{
    int rc;

    BLOCK_ADJUST(rc, session, session_startup(session, sock) );

    return rc;
}

```

这段代码定义了一个名为 `libssh2_session_startup()` 的函数，它是 `libssh2_session_handshake()` 的一个别名，用于在 `session` 和 `sock` 分别传入一个 `LIBSSH2_SESSION` 结构和一个已打开的、连接的 `socket` 参数的情况下，返回 `libssh2_session_handshake()` 的返回值。

具体来说，这个函数首先检查传入的 `session` 是否已经被初始化，如果是，就执行 `libssh2_session_handshake()`，如果已经初始化，就调用 `libssh2_session_handshake()`，传递给 `session` 的 `session` 和 `socket` 参数，然后返回 `libssh2_session_handshake()` 的返回值。如果 `session` 没有被初始化，或者 `session` 传来的 `session` 参数没有被正确使用 `libssh2_session_new()` 初始化，函数将会抛出错误。


```cpp
/*
 * libssh2_session_startup()
 *
 * DEPRECATED. Use libssh2_session_handshake() instead! This function is not
 * portable enough.
 *
 * session: LIBSSH2_SESSION struct allocated and owned by the calling program
 * sock:    *must* be populated with an opened and connected socket.
 *
 * Returns: 0 on success, or non-zero on failure
 */
LIBSSH2_API int
libssh2_session_startup(LIBSSH2_SESSION *session, int sock)
{
    return libssh2_session_handshake(session, (libssh2_socket_t) sock);
}

```

13165141234567890123456789


```cpp
/*
 * libssh2_session_free
 *
 * Frees the memory allocated to the session
 * Also closes and frees any channels attached to this session
 */
static int
session_free(LIBSSH2_SESSION *session)
{
    int rc;
    LIBSSH2_PACKET *pkg;
    LIBSSH2_CHANNEL *ch;
    LIBSSH2_LISTENER *l;
    int packets_left = 0;

    if(session->free_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "Freeing session resource",
                       session->remote.banner);

        session->free_state = libssh2_NB_state_created;
    }

    if(session->free_state == libssh2_NB_state_created) {
        while((ch = _libssh2_list_first(&session->channels))) {

            rc = _libssh2_channel_free(ch);
            if(rc == LIBSSH2_ERROR_EAGAIN)
                return rc;
        }

        session->free_state = libssh2_NB_state_sent;
    }

    if(session->free_state == libssh2_NB_state_sent) {
        while((l = _libssh2_list_first(&session->listeners))) {
            rc = _libssh2_channel_forward_cancel(l);
            if(rc == LIBSSH2_ERROR_EAGAIN)
                return rc;
        }

        session->free_state = libssh2_NB_state_sent1;
    }

    if(session->state & LIBSSH2_STATE_NEWKEYS) {
        /* hostkey */
        if(session->hostkey && session->hostkey->dtor) {
            session->hostkey->dtor(session, &session->server_hostkey_abstract);
        }

        /* Client to Server */
        /* crypt */
        if(session->local.crypt && session->local.crypt->dtor) {
            session->local.crypt->dtor(session,
                                       &session->local.crypt_abstract);
        }
        /* comp */
        if(session->local.comp && session->local.comp->dtor) {
            session->local.comp->dtor(session, 1,
                                      &session->local.comp_abstract);
        }
        /* mac */
        if(session->local.mac && session->local.mac->dtor) {
            session->local.mac->dtor(session, &session->local.mac_abstract);
        }

        /* Server to Client */
        /* crypt */
        if(session->remote.crypt && session->remote.crypt->dtor) {
            session->remote.crypt->dtor(session,
                                        &session->remote.crypt_abstract);
        }
        /* comp */
        if(session->remote.comp && session->remote.comp->dtor) {
            session->remote.comp->dtor(session, 0,
                                       &session->remote.comp_abstract);
        }
        /* mac */
        if(session->remote.mac && session->remote.mac->dtor) {
            session->remote.mac->dtor(session, &session->remote.mac_abstract);
        }

        /* session_id */
        if(session->session_id) {
            LIBSSH2_FREE(session, session->session_id);
        }
    }

    /* Free banner(s) */
    if(session->remote.banner) {
        LIBSSH2_FREE(session, session->remote.banner);
    }
    if(session->local.banner) {
        LIBSSH2_FREE(session, session->local.banner);
    }

    /* Free preference(s) */
    if(session->kex_prefs) {
        LIBSSH2_FREE(session, session->kex_prefs);
    }
    if(session->hostkey_prefs) {
        LIBSSH2_FREE(session, session->hostkey_prefs);
    }

    if(session->local.kexinit) {
        LIBSSH2_FREE(session, session->local.kexinit);
    }
    if(session->local.crypt_prefs) {
        LIBSSH2_FREE(session, session->local.crypt_prefs);
    }
    if(session->local.mac_prefs) {
        LIBSSH2_FREE(session, session->local.mac_prefs);
    }
    if(session->local.comp_prefs) {
        LIBSSH2_FREE(session, session->local.comp_prefs);
    }
    if(session->local.lang_prefs) {
        LIBSSH2_FREE(session, session->local.lang_prefs);
    }

    if(session->remote.kexinit) {
        LIBSSH2_FREE(session, session->remote.kexinit);
    }
    if(session->remote.crypt_prefs) {
        LIBSSH2_FREE(session, session->remote.crypt_prefs);
    }
    if(session->remote.mac_prefs) {
        LIBSSH2_FREE(session, session->remote.mac_prefs);
    }
    if(session->remote.comp_prefs) {
        LIBSSH2_FREE(session, session->remote.comp_prefs);
    }
    if(session->remote.lang_prefs) {
        LIBSSH2_FREE(session, session->remote.lang_prefs);
    }

    /*
     * Make sure all memory used in the state variables are free
     */
    if(session->kexinit_data) {
        LIBSSH2_FREE(session, session->kexinit_data);
    }
    if(session->startup_data) {
        LIBSSH2_FREE(session, session->startup_data);
    }
    if(session->userauth_list_data) {
        LIBSSH2_FREE(session, session->userauth_list_data);
    }
    if(session->userauth_pswd_data) {
        LIBSSH2_FREE(session, session->userauth_pswd_data);
    }
    if(session->userauth_pswd_newpw) {
        LIBSSH2_FREE(session, session->userauth_pswd_newpw);
    }
    if(session->userauth_host_packet) {
        LIBSSH2_FREE(session, session->userauth_host_packet);
    }
    if(session->userauth_host_method) {
        LIBSSH2_FREE(session, session->userauth_host_method);
    }
    if(session->userauth_host_data) {
        LIBSSH2_FREE(session, session->userauth_host_data);
    }
    if(session->userauth_pblc_data) {
        LIBSSH2_FREE(session, session->userauth_pblc_data);
    }
    if(session->userauth_pblc_packet) {
        LIBSSH2_FREE(session, session->userauth_pblc_packet);
    }
    if(session->userauth_pblc_method) {
        LIBSSH2_FREE(session, session->userauth_pblc_method);
    }
    if(session->userauth_kybd_data) {
        LIBSSH2_FREE(session, session->userauth_kybd_data);
    }
    if(session->userauth_kybd_packet) {
        LIBSSH2_FREE(session, session->userauth_kybd_packet);
    }
    if(session->userauth_kybd_auth_instruction) {
        LIBSSH2_FREE(session, session->userauth_kybd_auth_instruction);
    }
    if(session->open_packet) {
        LIBSSH2_FREE(session, session->open_packet);
    }
    if(session->open_data) {
        LIBSSH2_FREE(session, session->open_data);
    }
    if(session->direct_message) {
        LIBSSH2_FREE(session, session->direct_message);
    }
    if(session->fwdLstn_packet) {
        LIBSSH2_FREE(session, session->fwdLstn_packet);
    }
    if(session->pkeyInit_data) {
        LIBSSH2_FREE(session, session->pkeyInit_data);
    }
    if(session->scpRecv_command) {
        LIBSSH2_FREE(session, session->scpRecv_command);
    }
    if(session->scpSend_command) {
        LIBSSH2_FREE(session, session->scpSend_command);
    }
    if(session->sftpInit_sftp) {
        LIBSSH2_FREE(session, session->sftpInit_sftp);
    }

    /* Free payload buffer */
    if(session->packet.total_num) {
        LIBSSH2_FREE(session, session->packet.payload);
    }

    /* Cleanup all remaining packets */
    while((pkg = _libssh2_list_first(&session->packets))) {
        packets_left++;
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
            "packet left with id %d", pkg->data[0]);
        /* unlink the node */
        _libssh2_list_remove(&pkg->node);

        /* free */
        LIBSSH2_FREE(session, pkg->data);
        LIBSSH2_FREE(session, pkg);
    }
    _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
         "Extra packets left %d", packets_left);

    if(session->socket_prev_blockstate) {
        /* if the socket was previously blocking, put it back so */
        rc = session_nonblock(session->socket_fd, 0);
        if(rc) {
            _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
             "unable to reset socket's blocking state");
        }
    }

    if(session->server_hostkey) {
        LIBSSH2_FREE(session, session->server_hostkey);
    }

    /* error string */
    if(session->err_msg &&
       ((session->err_flags & LIBSSH2_ERR_FLAG_DUP) != 0)) {
        LIBSSH2_FREE(session, (char *)session->err_msg);
    }

    LIBSSH2_FREE(session, session);

    return 0;
}

```

这段代码定义了一个名为`libssh2_session_free`的函数，它属于`libssh2_session`类的成员函数。

这个函数的主要作用是释放分配给`session`会话的内存，并关闭与`session`会话相关的通道。具体实现如下：

1. 首先，在函数内部调用一个名为`session_free`的函数，它接受`session`参数，并使用`libssh2_session_free`函数的内部参数作为它的`session`参数。
2. 然后，使用`BLOCK_ADJUST`函数来处理任何剩余的空间，确保所有分配给会话的内存都被释放。
3. 最后，返回释放成功时返回的`rc`，通常情况下`rc`会是一个表示状态的整数，例如0表示成功，负数表示错误。

这段代码的作用是确保`libssh2_session`会话的内存被正确释放，以便应用程序可以继续使用它们。


```cpp
/*
 * libssh2_session_free
 *
 * Frees the memory allocated to the session
 * Also closes and frees any channels attached to this session
 */
LIBSSH2_API int
libssh2_session_free(LIBSSH2_SESSION * session)
{
    int rc;

    BLOCK_ADJUST(rc, session, session_free(session) );

    return rc;
}

```



This function appears to be part of the SSH client library, and is used to handle the description and destination of a disconnect message.

It takes two arguments: a pointer to a character string `description` and a pointer to a character string `lang`. The function compares the length of the `description` to 256, and extracts the packet type (1), reason code (4), and description length (4).

If the `description` is too long, the function returns an error. Otherwise, it constructs the disconnect data and sends it with the `libssh2_transport_send` function.

Finally, the function sets the state of the SSH client session to `libssh2_NB_state_idle`.


```cpp
/*
 * libssh2_session_disconnect_ex
 */
static int
session_disconnect(LIBSSH2_SESSION *session, int reason,
                   const char *description,
                   const char *lang)
{
    unsigned char *s;
    unsigned long descr_len = 0, lang_len = 0;
    int rc;

    if(session->disconnect_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "Disconnecting: reason=%d, desc=%s, lang=%s", reason,
                       description, lang);
        if(description)
            descr_len = strlen(description);

        if(lang)
            lang_len = strlen(lang);

        if(descr_len > 256)
            return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                                  "too long description");

        /* 13 = packet_type(1) + reason code(4) + descr_len(4) + lang_len(4) */
        session->disconnect_data_len = descr_len + lang_len + 13;

        s = session->disconnect_data;

        *(s++) = SSH_MSG_DISCONNECT;
        _libssh2_store_u32(&s, reason);
        _libssh2_store_str(&s, description, descr_len);
        /* store length only, lang is sent separately */
        _libssh2_store_u32(&s, lang_len);

        session->disconnect_state = libssh2_NB_state_created;
    }

    rc = _libssh2_transport_send(session, session->disconnect_data,
                                 session->disconnect_data_len,
                                 (unsigned char *)lang, lang_len);
    if(rc == LIBSSH2_ERROR_EAGAIN)
        return rc;

    session->disconnect_state = libssh2_NB_state_idle;

    return 0;
}

```

这段代码定义了一个名为`libssh2_session_disconnect_ex`的函数，属于`libssh2_session_private`类别。

这个函数的作用是处理在SSH会话中，当一个用户按照正常程序关闭会话时，会调用这个函数。它会执行以下操作：

1. 将`LIBSSH2_STATE_EXCHANGING_KEYS`位设置为0，以取消当前正在进行的键盘协商。
2. 如果已经没有其他阻塞操作在进行，使用`session_disconnect`函数处理会话的关闭。
3. 使用`BLOCK_ADJUST`函数来允许或阻止进一步的阻塞。

它进一步说明，该函数仅在`session`参数被正确初始化，并且传入了有效的`desc`和`lang`参数时才起作用。


```cpp
/*
 * libssh2_session_disconnect_ex
 */
LIBSSH2_API int
libssh2_session_disconnect_ex(LIBSSH2_SESSION *session, int reason,
                              const char *desc, const char *lang)
{
    int rc;
    session->state &= ~LIBSSH2_STATE_EXCHANGING_KEYS;
    BLOCK_ADJUST(rc, session,
                 session_disconnect(session, reason, desc, lang));

    return rc;
}

```

This function appears to determine the method type for a given `method_type` and return the corresponding method name, if applicable. The function takes a single parameter of type `LIBSSH2_METHOD_TYPE`, and uses a case-based comparison to determine the method type based on the `method_type` parameter. If the `method_type` is not recognized, the function returns `LIBSSH2_ERROR_INVAL`.


```cpp
/* libssh2_session_methods
 *
 * Return the currently active methods for method_type
 *
 * NOTE: Currently lang_cs and lang_sc are ALWAYS set to empty string
 * regardless of actual negotiation Strings should NOT be freed
 */
LIBSSH2_API const char *
libssh2_session_methods(LIBSSH2_SESSION * session, int method_type)
{
    /* All methods have char *name as their first element */
    const LIBSSH2_KEX_METHOD *method = NULL;

    switch(method_type) {
    case LIBSSH2_METHOD_KEX:
        method = session->kex;
        break;

    case LIBSSH2_METHOD_HOSTKEY:
        method = (LIBSSH2_KEX_METHOD *) session->hostkey;
        break;

    case LIBSSH2_METHOD_CRYPT_CS:
        method = (LIBSSH2_KEX_METHOD *) session->local.crypt;
        break;

    case LIBSSH2_METHOD_CRYPT_SC:
        method = (LIBSSH2_KEX_METHOD *) session->remote.crypt;
        break;

    case LIBSSH2_METHOD_MAC_CS:
        method = (LIBSSH2_KEX_METHOD *) session->local.mac;
        break;

    case LIBSSH2_METHOD_MAC_SC:
        method = (LIBSSH2_KEX_METHOD *) session->remote.mac;
        break;

    case LIBSSH2_METHOD_COMP_CS:
        method = (LIBSSH2_KEX_METHOD *) session->local.comp;
        break;

    case LIBSSH2_METHOD_COMP_SC:
        method = (LIBSSH2_KEX_METHOD *) session->remote.comp;
        break;

    case LIBSSH2_METHOD_LANG_CS:
        return "";

    case LIBSSH2_METHOD_LANG_SC:
        return "";

    default:
        _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                       "Invalid parameter specified for method_type");
        return NULL;
    }

    if(!method) {
        _libssh2_error(session, LIBSSH2_ERROR_METHOD_NONE,
                       "No method negotiated");
        return NULL;
    }

    return method->name;
}

```

这段代码定义了一个名为libssh2_session_abstract的函数，属于libssh2_session_abstract库。它的作用是获取一个名为session的libssh2_session对象的抽象属性的指针。

具体来说，libssh2_session_abstract函数的实现如下：

1. 它返回一个名为session_abstract的指针，如果有，则返回；
2. 如果没有，它创建一个名为session_error的指针，并将libssh2_last_error函数的返回值作为其值；
3. 它将错误信息作为errmsg参数传递给libssh2_last_error函数；
4. 错误信息字符串由errmsg参数指定，如果需要内存释放，必须在调用程序中释放它。

这段代码的作用是向libssh2_session_abstract库提供了一个名为session的libssh2_session对象的抽象属性的指针，以便用户使用libssh2_last_error函数获取错误信息。


```cpp
/* libssh2_session_abstract
 * Retrieve a pointer to the abstract property
 */
LIBSSH2_API void **
libssh2_session_abstract(LIBSSH2_SESSION * session)
{
    return &session->abstract;
}

/* libssh2_session_last_error
 *
 * Returns error code and populates an error string into errmsg If want_buf is
 * non-zero then the string placed into errmsg must be freed by the calling
 * program. Otherwise it is assumed to be owned by libssh2
 */
```

这段代码是一个名为`libssh2_session_last_error`的函数，它是`libssh2`库中的一个函数，用于报告SSH会话的最后一个错误。该函数接受三个参数：一个指向`LIBSSH2_SESSION`结构的指针`session`，一个指向字符数组的指针`errmsg`，以及一个整数`want_buf`，用于确定是否想要从`errmsg`缓冲区中复制错误消息。函数的主要作用是在错误报告和错误消息缓冲区中分配资源，并检查是否需要从`errmsg`缓冲区中复制错误消息。

具体来说，这段代码首先检查`session`是否已经包含错误代码，如果没有，则表示没有错误发生，直接返回0。如果`session`包含错误代码，则检查`err_msg`参数是否已经包含错误消息，如果是，则使用`LIBSSH2_ALLOC`函数将错误消息复制到`errmsg`缓冲区中，同时检查`want_buf`参数是否为真，如果是，则使用`memcpy`函数将`errmsg`缓冲区中的错误消息复制到`error`缓冲区中，并使用`strlen`函数获取错误消息的长度，最后将`msglen`变量初始化为错误消息长度，`errmsg_len`变量初始化为0。如果`want_buf`参数为假，则直接将`error`缓冲区中的错误消息复制到`errmsg`缓冲区中。

如果`err_msg`参数包含错误消息，则首先使用`strlen`函数获取错误消息的长度，然后使用`LIBSSH2_ALLOC`函数将错误消息复制到`errmsg`缓冲区中，最后使用`memcpy`函数将`errmsg`缓冲区中的错误消息复制到`error`缓冲区中，并使用`strlen`函数获取错误消息的长度，最后将`msglen`变量初始化为错误消息长度，`errmsg_len`变量初始化为0。如果`want_buf`参数为真，则与上面类似，使用`memcpy`函数将`errmsg`缓冲区中的错误消息复制到`error`缓冲区中，并使用`strlen`函数获取错误消息的长度，最后将`msglen`变量初始化为错误消息长度，`errmsg_len`变量初始化为0。


```cpp
LIBSSH2_API int
libssh2_session_last_error(LIBSSH2_SESSION * session, char **errmsg,
                           int *errmsg_len, int want_buf)
{
    size_t msglen = 0;

    /* No error to report */
    if(!session->err_code) {
        if(errmsg) {
            if(want_buf) {
                *errmsg = LIBSSH2_ALLOC(session, 1);
                if(*errmsg) {
                    **errmsg = 0;
                }
            }
            else {
                *errmsg = (char *) "";
            }
        }
        if(errmsg_len) {
            *errmsg_len = 0;
        }
        return 0;
    }

    if(errmsg) {
        const char *error = session->err_msg ? session->err_msg : "";

        msglen = strlen(error);

        if(want_buf) {
            /* Make a copy so the calling program can own it */
            *errmsg = LIBSSH2_ALLOC(session, msglen + 1);
            if(*errmsg) {
                memcpy(*errmsg, error, msglen);
                (*errmsg)[msglen] = 0;
            }
        }
        else
            *errmsg = (char *)error;
    }

    if(errmsg_len) {
        *errmsg_len = msglen;
    }

    return session->err_code;
}

```

这段代码定义了一个名为`libssh2_session_last_errno`的函数，它是`libssh2_session_set_last_error`的别名。这两个函数的作用是相同的，都是用来设置SSH会话的错误代码。

具体来说，这两个函数接受一个`LIBSSH2_SESSION`类型的参数，分别返回一个整数类型的错误代码。如果函数成功执行，则返回原来的错误代码；如果函数失败，则会执行错误处理程序，并返回一个表示错误信息的错误代码。

错误代码的值通常是一个常数，定义在`libssh2_session_err_ codes`数组中。这个数组包含了SSH会话可能出现的各种错误代码及其对应的错误信息。


```cpp
/* libssh2_session_last_errno
 *
 * Returns error code
 */
LIBSSH2_API int
libssh2_session_last_errno(LIBSSH2_SESSION * session)
{
    return session->err_code;
}

/* libssh2_session_set_last_error
 *
 * Sets the internal error code for the session.
 *
 * This function is available specifically to be used by high level
 * language wrappers (i.e. Python or Perl) that may extend the library
 * features while still relying on its error reporting mechanism.
 */
```

这段代码是一个名为`libssh2_session_set_last_error`的函数，属于`libssh2_session`库。它接受两个参数：`session`和`errcode`，分别表示会话和错误码。函数内部使用`_libssh2_error_flags`函数对传入的错误码和错误消息进行处理，并将结果返回。

具体来说，这段代码的作用是设置或获取一个`libssh2_session`对象中的会话标志，以便在会话过程中处理错误。通过设置或获取这些标志，用户可以更好地控制会话的行为，例如记录错误信息以便 later 处理，或者在需要时重试操作。


```cpp
LIBSSH2_API int
libssh2_session_set_last_error(LIBSSH2_SESSION* session,
                               int errcode,
                               const char *errmsg)
{
    return _libssh2_error_flags(session, errcode, errmsg,
                                LIBSSH2_ERR_FLAG_DUP);
}

/* Libssh2_session_flag
 *
 * Set/Get session flags
 *
 * Return error code.
 */
```

这段代码是一个名为`libssh2_session_flag`的函数，属于`libssh2_session`类的成员函数。它的作用是接收一个`LIBSSH2_SESSION`类型的会话对象`session`，以及一个`int`类型的标志`flag`和一个`int`类型的值`value`，然后根据`flag`的值执行相应的操作，并将结果返回。

具体来说，这段代码实现了一个基于switch语句的if语句，会根据`flag`的值来执行相应的操作，如果`flag`的值不在`LIBSSH2_FLAG_SIGPIPE`和`LIBSSH2_FLAG_COMPRESS`中，则会返回`LIBSSH2_ERROR_INVAL`，否则会将`value`设置为对应的`flag`的值，并将结果返回为`0`。


```cpp
LIBSSH2_API int
libssh2_session_flag(LIBSSH2_SESSION * session, int flag, int value)
{
    switch(flag) {
    case LIBSSH2_FLAG_SIGPIPE:
        session->flag.sigpipe = value;
        break;
    case LIBSSH2_FLAG_COMPRESS:
        session->flag.compress = value;
        break;
    default:
        /* unknown flag */
        return LIBSSH2_ERROR_INVAL;
    }

    return LIBSSH2_ERROR_NONE;
}

```

这段代码是一个名为`_libssh2_session_set_blocking`的函数，属于libssh2库。它接受一个libssh2_session类型的参数，和一个int类型的blocking参数。

函数的作用是设置session的blocking模式，即设置session是否阻塞。如果设置为ON，那么阻塞当前会话，直到函数返回；如果设置为OFF，那么恢复当前会话，不会阻塞。

函数内部首先获取session原来设置的blocking模式，然后将其与传入的blocking参数进行比较，最后将session的blocking模式修改为新的blocking参数。

函数会输出一条日志信息，以便开发人员了解问题的调试。


```cpp
/* _libssh2_session_set_blocking
 *
 * Set a session's blocking mode on or off, return the previous status when
 * this function is called. Note this function does not alter the state of the
 * actual socket involved.
 */
int
_libssh2_session_set_blocking(LIBSSH2_SESSION *session, int blocking)
{
    int bl = session->api_block_mode;
    _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                   "Setting blocking mode %s", blocking?"ON":"OFF");
    session->api_block_mode = blocking;

    return bl;
}

```

这段代码定义了一个名为libssh2_session_set_blocking的函数，它的作用是设置一个套路的阻塞模式。阻塞模式与套口的fcntl函数类似，当设置为O_NONBLOCK时，会阻塞调用进程对套片的输入输出操作，类似于套口的非阻塞输入输出。

libssh2_session_set_blocking函数的参数包括一个libssh2_session指针和一个阻塞模式，它会调用内部函数_libssh2_session_set_blocking，将阻塞模式设置为传入的值。如果传入的阻塞模式为0，则会执行内部函数，设置阻塞模式为OFF。


```cpp
/* libssh2_session_set_blocking
 *
 * Set a channel's blocking mode on or off, similar to a socket's
 * fcntl(fd, F_SETFL, O_NONBLOCK); type command
 */
LIBSSH2_API void
libssh2_session_set_blocking(LIBSSH2_SESSION * session, int blocking)
{
    (void) _libssh2_session_set_blocking(session, blocking);
}

/* libssh2_session_get_blocking
 *
 * Returns a session's blocking mode on or off
 */
```

这两段代码是使用libssh2库实现的SSH会话管理API。

libssh2_session_get_blocking(LIBSSH2_SESSION *session) 函数用于获取当前session的API块模式，它返回的是int类型的session->api_block_mode，即是否开启了API块模式。

libssh2_session_set_timeout(LIBSSH2_SESSION *session, long timeout) 函数用于设置session的timeout，它是long类型的函数，可以将timeout设置为0来禁用定时器，timeout的单位是毫秒。这个函数会将timeout设置为 session->api_timeout，然后更新session->api_timeout，使得所有的libssh2_session_peer_address_strings和libssh2_session_service_丙将使用这个新的timeout。


```cpp
LIBSSH2_API int
libssh2_session_get_blocking(LIBSSH2_SESSION * session)
{
    return session->api_block_mode;
}


/* libssh2_session_set_timeout
 *
 * Set a session's timeout (in msec) for blocking mode,
 * or 0 to disable timeouts.
 */
LIBSSH2_API void
libssh2_session_set_timeout(LIBSSH2_SESSION * session, long timeout)
{
    session->api_timeout = timeout;
}

```

这段代码定义了一个名为`libssh2_session_get_timeout`的函数，它属于`libssh2_session`类型。

这个函数的作用是获取libssh2会话的超时时间，如果没有设置这个时间，那么它的值就是0。它返回这个会话的`api_timeout`成员，这个成员是存储在`session`结构中的。

接下来定义了一个名为`libssh2_poll_channel_read`的函数，它也属于`libssh2_session`类型。这个函数的作用是返回一个整数，表示当前正在等待的通道是否有数据可用。如果当前没有数据可用，那么它的值就是0，否则，它的值就是返回值。


```cpp
/* libssh2_session_get_timeout
 *
 * Returns a session's timeout, or 0 if disabled
 */
LIBSSH2_API long
libssh2_session_get_timeout(LIBSSH2_SESSION * session)
{
    return session->api_timeout;
}

/*
 * libssh2_poll_channel_read
 *
 * Returns 0 if no data is waiting on channel,
 * non-0 if data is available
 */
```

这段代码是一个名为`libssh2_poll_channel_read`的函数，它是`libssh2_poll`函数的子函数。它用于在SSH会话中等待数据包的到达，并在接收到数据包时通知客户端。

具体来说，这段代码的作用如下：

1. 检查输入的`channel`是否为空，如果是，返回`LIBSSH2_ERROR_BAD_USE`错误。
2. 获取`session`对象，它是`channel`的会话。
3. 获取第一个加入`session`的`packet`对象。
4. 循环遍历输入的`packet`对象。
5. 如果`packet`的数据长度小于5字节，返回`LIBSSH2_ERROR_BUFFER_TOO_SMALL`错误。
6. 如果当前正在读取的`packet`包含了`SSH_MSG_CHANNEL_EXTENDED_DATA`或`SSH_MSG_CHANNEL_DATA`消息，则返回`1`，表示已经接收到主机的数据。
7. 如果`extended`为0且`packet`的数据包含主机的数据，则返回`1`。
8. 如果读取任何数据，返回`0`。

由于数据包可能包含错误的格式或缺乏数据，因此这段代码并不能保证它的正确性。


```cpp
LIBSSH2_API int
libssh2_poll_channel_read(LIBSSH2_CHANNEL *channel, int extended)
{
    LIBSSH2_SESSION *session;
    LIBSSH2_PACKET *packet;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    session = channel->session;
    packet = _libssh2_list_first(&session->packets);

    while(packet) {
        if(packet->data_len < 5) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Packet too small");
        }

        if(channel->local.id == _libssh2_ntohu32(packet->data + 1)) {
            if(extended == 1 &&
                (packet->data[0] == SSH_MSG_CHANNEL_EXTENDED_DATA
                 || packet->data[0] == SSH_MSG_CHANNEL_DATA)) {
                return 1;
            }
            else if(extended == 0 &&
                    packet->data[0] == SSH_MSG_CHANNEL_DATA) {
                return 1;
            }
            /* else - no data of any type is ready to be read */
        }
        packet = _libssh2_list_next(&packet->node);
    }

    return 0;
}

```

这两个函数定义在poll_channel_write.c文件中，它们的作用是关于通道的。

1. poll_channel_write函数定义了一个名为"poll_channel_write"的静态内部函数，它接收一个名为"channel"的输入参数，并返回一个整数。函数的作用是判断在给定的通道上进行写入操作是否会影响其他连接的操作。如果写入操作会影响其他连接的操作，那么函数将返回0；如果不会影响其他连接的操作，那么函数将返回1。

2. poll_listener_queued函数定义了一个名为"poll_listener_queued"的静态内部函数，它接收一个或多个名为"channel_name"的输入参数，并返回一个整数。函数的作用是判断当前是否有连接正在等待接受。如果没有连接正在等待接受，那么函数将返回0；如果有连接正在等待接受，那么函数将返回1。


```cpp
/*
 * poll_channel_write
 *
 * Returns 0 if writing to channel would block,
 * non-0 if data can be written without blocking
 */
static inline int
poll_channel_write(LIBSSH2_CHANNEL * channel)
{
    return channel->local.window_size ? 1 : 0;
}

/* poll_listener_queued
 *
 * Returns 0 if no connections are waiting to be accepted
 * non-0 if one or more connections are available
 */
```

这段代码定义了一个名为 `poll_listener_queued` 的函数，属于 `libssh2_listener` 系列的函数。其作用是获取一个 `LIBSSH2_LISTENER` 类型的 `listener` 对象，并返回一个整数，表示是否成功调用 `libssh2_list_first` 函数。如果调用成功，则返回 1，否则返回 0。

接下来是另一个名为 `libssh2_poll` 的函数，属于 `libssh2_pollfd` 系列的函数。其作用是检查多个套接字（socket）、通道（channel）和监听器（listener）是否活跃，并在指定的时间内等待它们的响应。如果有套接字活跃，函数将返回它们的 `LIBSSH2_SELECTED` 标志，否则返回 0。

整段代码的作用是实现一个高性能的网络编程界面，用于在远程主机上监听连接和数据传输。`libssh2_listener` 用于监听套接字，`libssh2_pollfd` 用于检测网络活动。通过 `libssh2_poll` 函数，可以实时检测网络中的连接和事件，并据此触发其他底层网络协议的行为。


```cpp
static inline int
poll_listener_queued(LIBSSH2_LISTENER * listener)
{
    return _libssh2_list_first(&listener->queue) ? 1 : 0;
}

/*
 * libssh2_poll
 *
 * Poll sockets, channels, and listeners for activity
 */
LIBSSH2_API int
libssh2_poll(LIBSSH2_POLLFD * fds, unsigned int nfds, long timeout)
{
    long timeout_remaining;
    unsigned int i, active_fds;
```

This function appears to be setting up a connection to an SSH server using the libssh2 library, and is using the pollfd data structure to handle the input/output streams of the connection. The function takes in an optional pointer to an SSH2 session, and uses it to set up a socket or a channel for polling. If an invalid poll type is passed to poll(), the function will return -1.


```cpp
#ifdef HAVE_POLL
    LIBSSH2_SESSION *session = NULL;
#ifdef HAVE_ALLOCA
    struct pollfd *sockets = alloca(sizeof(struct pollfd) * nfds);
#else
    struct pollfd sockets[256];

    if(nfds > 256)
        /* systems without alloca use a fixed-size array, this can be fixed if
           we really want to, at least if the compiler is a C99 capable one */
        return -1;
#endif
    /* Setup sockets for polling */
    for(i = 0; i < nfds; i++) {
        fds[i].revents = 0;
        switch(fds[i].type) {
        case LIBSSH2_POLLFD_SOCKET:
            sockets[i].fd = fds[i].fd.socket;
            sockets[i].events = fds[i].events;
            sockets[i].revents = 0;
            break;

        case LIBSSH2_POLLFD_CHANNEL:
            sockets[i].fd = fds[i].fd.channel->session->socket_fd;
            sockets[i].events = POLLIN;
            sockets[i].revents = 0;
            if(!session)
                session = fds[i].fd.channel->session;
            break;

        case LIBSSH2_POLLFD_LISTENER:
            sockets[i].fd = fds[i].fd.listener->session->socket_fd;
            sockets[i].events = POLLIN;
            sockets[i].revents = 0;
            if(!session)
                session = fds[i].fd.listener->session;
            break;

        default:
            if(session)
                _libssh2_error(session, LIBSSH2_ERROR_INVALID_POLL_TYPE,
                               "Invalid descriptor passed to libssh2_poll()");
            return -1;
        }
    }
```

This code appears to be using the `libssh2_poll()` function to monitor multiple file descriptors for events of the desired persuasion. It is using a combination of `FD_SET` system calls to update the file descriptors and `libssh2_error()` function to handle errors.

The function is iterating over all file descriptors with the desired type and setting the `revents` field of each to 0. This is done in the case of LIBSSH2_POLLFD_SOCKET, LIBSSH2_POLLFD_CHANNEL, LIBSSH2


```cpp
#elif defined(HAVE_SELECT)
    LIBSSH2_SESSION *session = NULL;
    libssh2_socket_t maxfd = 0;
    fd_set rfds, wfds;
    struct timeval tv;

    FD_ZERO(&rfds);
    FD_ZERO(&wfds);
    for(i = 0; i < nfds; i++) {
        fds[i].revents = 0;
        switch(fds[i].type) {
        case LIBSSH2_POLLFD_SOCKET:
            if(fds[i].events & LIBSSH2_POLLFD_POLLIN) {
                FD_SET(fds[i].fd.socket, &rfds);
                if(fds[i].fd.socket > maxfd)
                    maxfd = fds[i].fd.socket;
            }
            if(fds[i].events & LIBSSH2_POLLFD_POLLOUT) {
                FD_SET(fds[i].fd.socket, &wfds);
                if(fds[i].fd.socket > maxfd)
                    maxfd = fds[i].fd.socket;
            }
            break;

        case LIBSSH2_POLLFD_CHANNEL:
            FD_SET(fds[i].fd.channel->session->socket_fd, &rfds);
            if(fds[i].fd.channel->session->socket_fd > maxfd)
                maxfd = fds[i].fd.channel->session->socket_fd;
            if(!session)
                session = fds[i].fd.channel->session;
            break;

        case LIBSSH2_POLLFD_LISTENER:
            FD_SET(fds[i].fd.listener->session->socket_fd, &rfds);
            if(fds[i].fd.listener->session->socket_fd > maxfd)
                maxfd = fds[i].fd.listener->session->socket_fd;
            if(!session)
                session = fds[i].fd.listener->session;
            break;

        default:
            if(session)
                _libssh2_error(session, LIBSSH2_ERROR_INVALID_POLL_TYPE,
                               "Invalid descriptor passed to libssh2_poll()");
            return -1;
        }
    }
```

This is a C function that defines the behavior of an fsDNAd\_channel\_pollfd\_event handler. It is a part of an fsDNAd library that provides apoll file descriptor functionality to the Linux \<stream\_io\> ecosystem.

The function takes in a single file descriptor (fd) from the input stream, and handles the\_pollfd\_event\_call function, which is called by the fpDNAd library.

The function first checks if the file descriptor is already active. If it is not active, it adds it to the active\_fds queue and returns.

If the file descriptor is active, it checks for the revents bit set by the input stream, and updates the revents variable accordingly. If the file descriptor is a socket, it checks for the session state and updates the revents variable accordingly.

If the file descriptor is a listener, it checks for the session state and updates the revents variable accordingly.

The function also defines a timeout\_remaining variable, which resets to zero if any of the file descriptors have an active session. This variable is used to reset the timeout counter in case the connection is closed by the server.

The last修枝是为了不阻塞正在连接的文件描述符，即使有正在活动的文件描述符。


```cpp
#else
    /* No select() or poll()
     * no sockets structure to setup
     */

    timeout = 0;
#endif /* HAVE_POLL or HAVE_SELECT */

    timeout_remaining = timeout;
    do {
#if defined(HAVE_POLL) || defined(HAVE_SELECT)
        int sysret;
#endif

        active_fds = 0;

        for(i = 0; i < nfds; i++) {
            if(fds[i].events != fds[i].revents) {
                switch(fds[i].type) {
                case LIBSSH2_POLLFD_CHANNEL:
                    if((fds[i].events & LIBSSH2_POLLFD_POLLIN) &&
                        /* Want to be ready for read */
                        ((fds[i].revents & LIBSSH2_POLLFD_POLLIN) == 0)) {
                        /* Not yet known to be ready for read */
                        fds[i].revents |=
                            libssh2_poll_channel_read(fds[i].fd.channel,
                                                      0) ?
                            LIBSSH2_POLLFD_POLLIN : 0;
                    }
                    if((fds[i].events & LIBSSH2_POLLFD_POLLEXT) &&
                        /* Want to be ready for extended read */
                        ((fds[i].revents & LIBSSH2_POLLFD_POLLEXT) == 0)) {
                        /* Not yet known to be ready for extended read */
                        fds[i].revents |=
                            libssh2_poll_channel_read(fds[i].fd.channel,
                                                      1) ?
                            LIBSSH2_POLLFD_POLLEXT : 0;
                    }
                    if((fds[i].events & LIBSSH2_POLLFD_POLLOUT) &&
                        /* Want to be ready for write */
                        ((fds[i].revents & LIBSSH2_POLLFD_POLLOUT) == 0)) {
                        /* Not yet known to be ready for write */
                        fds[i].revents |=
                            poll_channel_write(fds[i].fd. channel) ?
                            LIBSSH2_POLLFD_POLLOUT : 0;
                    }
                    if(fds[i].fd.channel->remote.close
                        || fds[i].fd.channel->local.close) {
                        fds[i].revents |= LIBSSH2_POLLFD_CHANNEL_CLOSED;
                    }
                    if(fds[i].fd.channel->session->socket_state ==
                        LIBSSH2_SOCKET_DISCONNECTED) {
                        fds[i].revents |=
                            LIBSSH2_POLLFD_CHANNEL_CLOSED |
                            LIBSSH2_POLLFD_SESSION_CLOSED;
                    }
                    break;

                case LIBSSH2_POLLFD_LISTENER:
                    if((fds[i].events & LIBSSH2_POLLFD_POLLIN) &&
                        /* Want a connection */
                        ((fds[i].revents & LIBSSH2_POLLFD_POLLIN) == 0)) {
                        /* No connections known of yet */
                        fds[i].revents |=
                            poll_listener_queued(fds[i].fd. listener) ?
                            LIBSSH2_POLLFD_POLLIN : 0;
                    }
                    if(fds[i].fd.listener->session->socket_state ==
                        LIBSSH2_SOCKET_DISCONNECTED) {
                        fds[i].revents |=
                            LIBSSH2_POLLFD_LISTENER_CLOSED |
                            LIBSSH2_POLLFD_SESSION_CLOSED;
                    }
                    break;
                }
            }
            if(fds[i].revents) {
                active_fds++;
            }
        }

        if(active_fds) {
            /* Don't block on the sockets if we have channels/listeners which
               are ready */
            timeout_remaining = 0;
        }
```

这段代码是用来在Linux系统上检测网络套接字（socket）是否支持使用libssh2_gettimeofday函数来获取当前时间戳（timestamp），如果libssh2_gettimeofday函数可用，则使用该函数获取当前时间，否则就等待一段时间后再次尝试。

具体来说，代码首先通过if语句判断当前系统是否支持libssh2_gettimeofday函数。如果支持，则使用该函数获取当前时间，并将其存储在struct timeval类型的变量tv_begin和tv_end中。接着，代码使用poll函数对当前套接字进行超时，并设置一个名为timeout_remaining的变量来减去当前时间戳和超时时间戳之间的时间。如果libssh2_gettimeofday函数不可用，则代码将使用poll函数等待一段时间，然后将timeout_remaining设置为0。

总的来说，这段代码的作用是检测网络套接字是否支持libssh2_gettimeofday函数，并使用该函数获取当前时间戳。如果libssh2_gettimeofday函数不可用，则等待一段时间后再次尝试获取当前时间戳。


```cpp
#ifdef HAVE_POLL

#ifdef HAVE_LIBSSH2_GETTIMEOFDAY
        {
            struct timeval tv_begin, tv_end;

            _libssh2_gettimeofday((struct timeval *) &tv_begin, NULL);
            sysret = poll(sockets, nfds, timeout_remaining);
            _libssh2_gettimeofday((struct timeval *) &tv_end, NULL);
            timeout_remaining -= (tv_end.tv_sec - tv_begin.tv_sec) * 1000;
            timeout_remaining -= (tv_end.tv_usec - tv_begin.tv_usec) / 1000;
        }
#else
        /* If the platform doesn't support gettimeofday,
         * then just make the call non-blocking and walk away
         */
        sysret = poll(sockets, nfds, timeout_remaining);
        timeout_remaining = 0;
```

This is a C function that reads the contents of a file and sends it to a Redis server. It uses the Redis server to store the data that it reads from the file.

The function takes two parameters:

- `sfds`: A pointer to an array of `SSH2_FSM_FileDesc` structures. This array contains information about the file that we want to read.
- `redis_string`: A pointer to a Redis string. This string will be used as the key for the data that we read from the file.

The function reads each file descriptor in the `sfds` array and sends the contents of the file to the Redis server using the `Redis` library. It does this by repeatedly calling the `Redis` library's `LPUSH` or `LPCAV` method to send the data from the file to the Redis server.

The function also handles the case where the Redis server is not available. In this case, the function will keep trying to send the data to the Redis server, until the connection is closed or a timeout occurs.


```cpp
#endif /* HAVE_GETTIMEOFDAY */

        if(sysret > 0) {
            for(i = 0; i < nfds; i++) {
                switch(fds[i].type) {
                case LIBSSH2_POLLFD_SOCKET:
                    fds[i].revents = sockets[i].revents;
                    sockets[i].revents = 0; /* In case we loop again, be
                                               nice */
                    if(fds[i].revents) {
                        active_fds++;
                    }
                    break;
                case LIBSSH2_POLLFD_CHANNEL:
                    if(sockets[i].events & POLLIN) {
                        /* Spin session until no data available */
                        while(_libssh2_transport_read(fds[i].fd.
                                                      channel->session)
                              > 0);
                    }
                    if(sockets[i].revents & POLLHUP) {
                        fds[i].revents |=
                            LIBSSH2_POLLFD_CHANNEL_CLOSED |
                            LIBSSH2_POLLFD_SESSION_CLOSED;
                    }
                    sockets[i].revents = 0;
                    break;
                case LIBSSH2_POLLFD_LISTENER:
                    if(sockets[i].events & POLLIN) {
                        /* Spin session until no data available */
                        while(_libssh2_transport_read(fds[i].fd.
                                                      listener->session)
                              > 0);
                    }
                    if(sockets[i].revents & POLLHUP) {
                        fds[i].revents |=
                            LIBSSH2_POLLFD_LISTENER_CLOSED |
                            LIBSSH2_POLLFD_SESSION_CLOSED;
                    }
                    sockets[i].revents = 0;
                    break;
                }
            }
        }
```

这段代码的作用是设置一个定时器，在一定时间内超时后执行一些操作。

具体来说，它首先检查是否定义了`HAVE_SELECT`函数。如果是，就执行以下操作：

1. 将`timeout_remaining`除以1000，得到剩余的`timeout_remaining`时间，然后将其乘以1000，得到当前时间的微秒数。
2. 如果定义了`HAVE_LIBSSH2_GETTIMEOFDAY`函数，就执行以下操作：

  1. 获取当前时间的微秒数和微秒数组。
  2. 使用`select`函数等待1000个毫秒，然后将`timeout_remaining`减去当前时间的微秒数和微秒数组中的值。

如果不是定义了`HAVE_SELECT`函数，或者是由于某些原因无法使用`HAVE_LIBSSH2_GETTIMEOFDAY`函数，那么代码会执行以下操作：

1. 如果当前平台不支持`gettimeofday`函数，那么直接返回一个非阻塞I/O状态码，并退出定时器。
2. 设置`timeout_remaining`为0，然后退出定时器。


```cpp
#elif defined(HAVE_SELECT)
        tv.tv_sec = timeout_remaining / 1000;
        tv.tv_usec = (timeout_remaining % 1000) * 1000;
#ifdef HAVE_LIBSSH2_GETTIMEOFDAY
        {
            struct timeval tv_begin, tv_end;

            _libssh2_gettimeofday((struct timeval *) &tv_begin, NULL);
            sysret = select(maxfd + 1, &rfds, &wfds, NULL, &tv);
            _libssh2_gettimeofday((struct timeval *) &tv_end, NULL);

            timeout_remaining -= (tv_end.tv_sec - tv_begin.tv_sec) * 1000;
            timeout_remaining -= (tv_end.tv_usec - tv_begin.tv_usec) / 1000;
        }
#else
        /* If the platform doesn't support gettimeofday,
         * then just make the call non-blocking and walk away
         */
        sysret = select(maxfd + 1, &rfds, &wfds, NULL, &tv);
        timeout_remaining = 0;
```

这段代码是一个 C 语言代码，定义了一个头文件 #ifdef，用于检查系统是否支持某种特定功能。

如果系统支持该功能，则执行以下代码。如果不支持，则什么都不做。

具体来说，这段代码的作用是检查系统是否支持 libssh2_pollfd 函数。如果系统支持该函数，则使用该函数对指定文件描述符的数据输入流进行 poll 操作，即轮询是否有数据可用。如果文件描述符允许数据输入流，则通过调用 _libssh2_transport_read 函数从文件中读取数据。如果文件描述符不允许数据输入流，则什么都不做。

通过这种机制，可以确保在支持 libssh2_pollfd 函数的系统上，对指定文件描述符的数据输入流进行 poll 操作，以获取是否有数据可用。在不支持 libssh2_pollfd 函数的系统上，不会执行该函数，因此不会对指定文件描述符的数据输入流进行 poll 操作。


```cpp
#endif

        if(sysret > 0) {
            for(i = 0; i < nfds; i++) {
                switch(fds[i].type) {
                case LIBSSH2_POLLFD_SOCKET:
                    if(FD_ISSET(fds[i].fd.socket, &rfds)) {
                        fds[i].revents |= LIBSSH2_POLLFD_POLLIN;
                    }
                    if(FD_ISSET(fds[i].fd.socket, &wfds)) {
                        fds[i].revents |= LIBSSH2_POLLFD_POLLOUT;
                    }
                    if(fds[i].revents) {
                        active_fds++;
                    }
                    break;

                case LIBSSH2_POLLFD_CHANNEL:
                    if(FD_ISSET(fds[i].fd.channel->session->socket_fd,
                                &rfds)) {
                        /* Spin session until no data available */
                        while(_libssh2_transport_read(fds[i].fd.
                                                      channel->session)
                              > 0);
                    }
                    break;

                case LIBSSH2_POLLFD_LISTENER:
                    if(FD_ISSET
                        (fds[i].fd.listener->session->socket_fd, &rfds)) {
                        /* Spin session until no data available */
                        while(_libssh2_transport_read(fds[i].fd.
                                                      listener->session)
                              > 0);
                    }
                    break;
                }
            }
        }
```

这段代码是一个C库函数，它是用来在SSH2会话中获取当前受阻塞的 direction。该函数在以下情况下将被调用：

1. 如果函数在接收数据时被阻塞，它将返回LIBSSH2_SOCKET_BLOCK_INBOUND。
2. 如果函数在发送数据时被阻塞，它将返回LIBSSH2_SOCKET_BLOCK_OUTBOUND。
3. 如果函数在等待客户端连接时被阻塞，它将导致函数无限期地等待客户端连接，直到超时（timeout_remaining）用完，此时函数将返回0并输出一条错误消息。

该函数的核心是while循环，它会在每次尝试连接客户端时检查当前的timeout_remaining是否大于0。如果timeout_remaining大于0，说明客户端连接还没有成功，函数会继续等待，直到客户端连接成功或者超时。如果当前连接没有成功，函数将继续等待，直到超时（timeout_remaining）用完。

如果函数在等待客户端连接的过程中，连接成功或者超时（timeout_remaining）用完，它将返回当前受阻塞的direction，该方向可以是LIBSSH2_SOCKET_BLOCK_INBOUND或LIBSSH2_SOCKET_BLOCK_OUTBOUND。


```cpp
#endif /* else no select() or poll() -- timeout (and by extension
        * timeout_remaining) will be equal to 0 */
    } while((timeout_remaining > 0) && !active_fds);

    return active_fds;
}

/*
 * libssh2_session_block_directions
 *
 * Get blocked direction when a function returns LIBSSH2_ERROR_EAGAIN
 * Returns LIBSSH2_SOCKET_BLOCK_INBOUND if recv() blocked
 * or LIBSSH2_SOCKET_BLOCK_OUTBOUND if send() blocked
 */
LIBSSH2_API int
```

这两段代码属于SSH客户端库中的函数，作用如下：

1. libssh2_session_block_directions(LIBSSH2_SESSION *session)：
  this function returns session->socket_block_directions，它是一个指针，指向了在SSH会话中使用的套接字块方向（例如STREAM_REUSE_做到了1，那么它所使用的套接字块方向就是STREAM_REUSE_1）。

2. libssh2_session_banner_get(LIBSSH2_SESSION *session)：
  this function返回远程 banner（服务器 ID 字符串），远程 banner是远程服务器发送的包含服务器 ID 的字符串。

  if(NULL == session)
      return NULL；这里，如果session为空，那么函数返回NULL，这样可以避免在空指针上执行内存操作。

  if(NULL == session->remote.banner)
      return NULL；如果session->remote.banner为空，那么函数返回NULL，同样可以避免在空指针上执行内存操作。


```cpp
libssh2_session_block_directions(LIBSSH2_SESSION *session)
{
    return session->socket_block_directions;
}

/* libssh2_session_banner_get
 * Get the remote banner (server ID string)
 */

LIBSSH2_API const char *
libssh2_session_banner_get(LIBSSH2_SESSION *session)
{
    /* to avoid a coredump when session is NULL */
    if(NULL == session)
        return NULL;

    if(NULL == session->remote.banner)
        return NULL;

    return (const char *) session->remote.banner;
}

```