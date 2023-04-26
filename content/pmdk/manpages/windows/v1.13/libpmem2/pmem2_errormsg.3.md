---
draft: false
slider_enable: true
description: ""
disclaimer: "The contents of this web site and the associated <a href=\"https://github.com/pmem\">GitHub repositories</a> are BSD-licensed open source."
aliases: ["pmem2_errormsg.3.html"]
title: "libpmem2 | PMDK"
header: "pmem2 API version 1.0"
---

[comment]: <> (SPDX-License-Identifier: BSD-3-Clause)
[comment]: <> (Copyright 2019, Intel Corporation)

[comment]: <> (pmem2_errormsg.3 -- man page for error handling in libpmem2)

[NAME](#name)<br />
[SYNOPSIS](#synopsis)<br />
[DESCRIPTION](#description)<br />
[RETURN VALUE](#return-value)<br />
[SEE ALSO](#see-also)<br />

# NAME #

**pmem2_errormsgU**()/**pmem2_errormsgW**() - returns last error message

# SYNOPSIS #

```c
#include <libpmem2.h>

const char *pmem2_errormsgU(void);
const wchar_t *pmem2_errormsgW(void);
```


>NOTE: The PMDK API supports UNICODE. If the **PMDK_UTF8_API** macro is
defined, basic API functions are expanded to the UTF-8 API with postfix *U*.
Otherwise they are expanded to the UNICODE API with postfix *W*.

# DESCRIPTION #

If an error is detected during the call to a **libpmem2**(7) function, the
application may retrieve an error message describing the reason of the failure
from **pmem2_errormsgU**()/**pmem2_errormsgW**(). The error message buffer is thread-local;
errors encountered in one thread do not affect its value in
other threads. The buffer is never cleared by any library function; its
content is significant only when the return value of the immediately preceding
call to a **libpmem2**(7) function indicated an error.
The application must not modify or free the error message string.
Subsequent calls to other library functions may modify the previous message.

# RETURN VALUE #

The **pmem2_errormsgU**()/**pmem2_errormsgW**() function returns a pointer to a static buffer
containing the last error message logged for the current thread. If *errno*
was set, the error message may include a description of the corresponding
error code as returned by **strerror**(3).

# SEE ALSO #

**strerror**(3), **libpmem2**(7) and **<https://pmem.io>**
