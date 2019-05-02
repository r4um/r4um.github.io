---
layout: post
title: "bash: no clear screen"
comments: false
description: "bash: no clear"
keywords: "bash, readline, termcap, gdb, linux"
---

Switched a linuxbrew installation from the source based to bottled one (got tired of source builds failing most of the time). Installed bash bottle and switched to it as shell, everything works fine except pressing `CTRL-l` doesn't clear screen anymore, looks like it's just sending a newline. How is it something so trivial that has always worked stopped working?

Next to make sure the `CTRL-l` key is actually bound to clear screen.

```shell
$ bind -P | grep clear
clear-screen can be found on "\C-l".
```

So the key is bound, just not doing what it is supposed to.

Inspecting the [bash formula](https://github.com/Homebrew/linuxbrew-core/blob/master/Formula/bash.rb) there are no extra build options passed, inspecting the library dependencies


```shell
$ ldd `which bash`
       linux-vdso.so.1 (0x00007ffc05bd1000)
       libdl.so.2 => /home/linuxbrew/.linuxbrew/lib/libdl.so.2 (0x00007faf29ae0000)
       libc.so.6 => /home/linuxbrew/.linuxbrew/lib/libc.so.6 (0x00007faf29741000)
       /home/linuxbrew/.linuxbrew/lib/ld.so (0x00007faf29ce4000)
```

There is no external dependency on readline or ncurses, so it is using the readline shipped with the bash source. 

Looking at the [documentation](https://tiswww.case.edu/php/chet/readline/readline.html#SEC53), searching for a clear screen function doesn't yield anything :/.

Digging through source lands at [rl\_clear\_screen](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/readline/text.c#L591) function, the function of interest would be [\_rl\_clear\_screen](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/readline/display.c#L2874:1), looks like `_rl_term_clrpag` is `NULL` that would explain the newline being sent by call to [rl\_crlf](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/readline/terminal.c#L669:1). Confirming via gdb

```shell
# clear screen funtion is present
$ objdump --dynamic-syms `which bash` | grep clear.screen
00000000004c0970 g    DF .text  0000000000000025  Base        _rl_clear_screen
00000000004c6e20 g    DF .text  000000000000003c  Base        rl_clear_screen

$ gdb -q `which bash`
Reading symbols from /home/linuxbrew/.linuxbrew/bin/bash...
warning: Loadable section ".dynstr" outside of ELF segments
(no debugging symbols found)...done.
(gdb) b _rl_clear_screen
Breakpoint 1 at 0x4c0970
(gdb) r
$
Breakpoint 1, 0x00000000004c0970 in _rl_clear_screen ()
(gdb) bt
#0  0x00000000004c0970 in _rl_clear_screen ()
#1  0x00000000004c6e33 in rl_clear_screen ()
#2  0x00000000004ab059 in _rl_dispatch_subseq ()
#3  0x00000000004ab5b8 in readline_internal_char ()
#4  0x00000000004abd45 in readline ()
#5  0x0000000000424741 in yy_readline_get ()
#6  0x0000000000426d06 in shell_getc ()
#7  0x000000000042a251 in read_token.constprop ()
#8  0x000000000042db74 in yyparse ()
#9  0x0000000000423eef in parse_command ()
#10 0x0000000000423fe8 in read_command ()
#11 0x00000000004241e6 in reader_loop ()
#12 0x0000000000422e70 in main ()
(gdb) p (char*) _rl_term_clrpag
$1 = 0x0
(gdb) x/3s $rdi
0x7723a8:       "screen-256color"
0x7723b8:       "\020"
0x7723ba:       ""
(gdb) # We can see above the term name being passed as screen-256color

```


Now to dig into why `_rl_term_clrpag` is `NULL`, it is a [global](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/readline/terminal.c#L114) variable initialised via the [\_rl\_init\_terminal\_io](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/readline/terminal.c#L436) function. The call to [tgetent](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/readline/terminal.c#L483) looks of interest
if it doesn't [succeed](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/readline/terminal.c#L486) `_rl_term_clrpag` will stay [NULL](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/readline/terminal.c#L443) as initialised. Confirming via gdb.

```shell
$ gdb -q `which bash`
Reading symbols from /home/linuxbrew/.linuxbrew/bin/bash...
warning: Loadable section ".dynstr" outside of ELF segments
(no debugging symbols found)...done.
(gdb) b _rl_init_terminal_io
Breakpoint 1 at 0x4c4f30
(gdb) r
Starting program: /home/linuxbrew/.linuxbrew/bin/bash
Breakpoint 1, 0x00000000004c4f30 in _rl_init_terminal_io ()
(gdb) b tgetent
Breakpoint 2 at 0x4cf600
(gdb) c
Continuing.

Breakpoint 2, 0x00000000004cf600 in tgetent ()
(gdb) n
Single stepping until exit from function tgetent,
which has no line number information.
0x00000000004c5431 in _rl_init_terminal_io ()
(gdb) p $eax
$2 = -1
```

As confirmed `tgetent` is returning `-1` [[^1]], this function is part of inbuilt [termcap](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/termcap/termcap.c#L1) functionality in bash.

Reading through the [tgetent](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/termcap/termcap.c#L451:1) function it is [looking for](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/termcap/termcap.c#L526) [/etc/termcap](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/termcap/termcap.c#L106:9). Confirming via gdb

```shell
$ gdb -q `which bash`
Reading symbols from /home/linuxbrew/.linuxbrew/bin/bash...
warning: Loadable section ".dynstr" outside of ELF segments
(no debugging symbols found)...done.
(gdb) b tgetent
Breakpoint 1 at 0x4cf600
(gdb) r
Starting program: /home/linuxbrew/.linuxbrew/bin/bash

Breakpoint 1, 0x00000000004cf600 in tgetent ()
(gdb) s
Single stepping until exit from function tgetent,
which has no line number information.
open64 () at ../sysdeps/unix/syscall-template.S:84
84      ../sysdeps/unix/syscall-template.S: No such file or directory.
(gdb) x/3s $rdi
0x4efef5:       "/etc/termcap"
0x4eff02:       ""
0x4eff03:       ""
(gdb) n
86      in ../sysdeps/unix/syscall-template.S
(gdb) s
0x00000000004cf65c in tgetent ()
(gdb) p $eax
$1 = -1
```

As expected there is no `/etc/termcap`, also as per manpages termcap is deprecated, applications should use terminfo, which is part of ncurses.

Now either we rebuild bash with ncurses support or build this file. A quick search on converting terminfo files to termcap points to `tic` utility. It needs a source terminfo file, which is present in the [ncurses sources](https://raw.githubusercontent.com/mirror/ncurses/master/misc/terminfo.src), going through the tic manpage, selecting the right options

```shell
$ tic -K -C -q -r terminfo.src > termcap
``` 

Moving this to `/etc/termcap` (or point the [TERMCAP](https://github.com/bminor/bash/blob/d233b485e83c3a784b803fb894280773f16f2deb/lib/termcap/termcap.c#L487) environment variable to this file) and restarting bash, clear screen on pressing `CTRL-l` now works.

___
Footnote:

[^1]: 1: If you are wondering on the usage of `rdi` and `eax` registers, these conventions are part of the [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) on [x86](https://github.molgen.mpg.de/git-mirror/glibc/blob/glibc-2.15/sysdeps/unix/sysv/linux/x86_64/syscall.S#L32). Return code from function calls is stored in `eax`.