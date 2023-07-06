# vimjail[2, 1.5, 2, 2.5] - misc

Writeup for an unintended solution by fslaktern

- [vimjail1](#vimjail1)
  - [Key information](#key-information)
  - [First thoughts](#first-thoughts)
  - [Bruteforcing..?](#bruteforcing)
  - [How-to: Open another file when already in a vim buffer](#how-to-open-another-file-when-already-in-a-vim-buffer)
  - [What happened?](#what-happened)
- [vimjail 1.5](#vimjail-15)
- [vimjail 2](#vimjail-2)
  - [Figuring out why](#figuring-out-why)
  - [What is the effect?](#what-is-the-effect)
- [vimjail 2.5](#vimjail-25)
- [Videos](#videos)
  - [Original solution](#original-solution)
  - [Speedrun](#speedrun)
    - [For vimjail 1 and vimjail 1.5](#for-vimjail-1-and-vimjail-15)
    - [For vimjail 2 and vimjail 2.5](#for-vimjail-2-and-vimjail-25)


## vimjail1

> Connect with `socat file:$(tty),raw,echo=0 tcp:vimjail1.chal.uiuc.tf:1337` 
> You may need to install socat.
> 
> Author: richard

We are given the following 4 files:
`Dockerfile`
```dockerfile=

FROM ubuntu:22.04 as chroot

RUN apt-get update -y
RUN apt-get install -y vim

RUN /usr/sbin/useradd --no-create-home -u 1000 user

COPY flag.txt /
COPY entry.sh /home/user/
COPY vimrc /home/user/

FROM gcr.io/kctf-docker/challenge@sha256:d884e54146b71baf91603d5b73e563eaffc5a42d494b1e32341a5f76363060fb

COPY --from=chroot / /chroot

COPY nsjail.cfg /home/user/

# This is mostly boring kctf and nsjail setup. The important part to pay
# attention to is the "pty,sane" line.
CMD kctf_setup && \
    kctf_drop_privs \
    socat \
      TCP-LISTEN:1337,reuseaddr,fork \
      EXEC:"kctf_pow nsjail --config /home/user/nsjail.cfg -- /home/user/entry.sh",pty,sane
```

`entry.sh`
```shell=
#!/usr/bin/env sh

chmod -r /flag.txt

vim -R -M -Z -u /home/user/vimrc
```

`nsjail.cfg`
```
name: "default-nsjail-configuration"
description: "Default nsjail configuration for pwnable-style CTF task."

mode: ONCE
uidmap {inside_id: "1000"}
gidmap {inside_id: "1000"}
mount_proc: true
rlimit_as_type: HARD
rlimit_cpu_type: HARD
rlimit_nofile_type: HARD
rlimit_nproc_type: HARD

mount: [
  {
    src: "/chroot"
    dst: "/"
    is_bind: true
  },
  {
    dst: "/tmp"
    fstype: "tmpfs"
    options: "size=500000"
    rw: true
  },
  {
    src: "/etc/resolv.conf"
    dst: "/etc/resolv.conf"
    is_bind: true
  }
]
```

`vimrc`
```vim=
set nocompatible
set insertmode

inoremap <c-o> nope
inoremap <c-l> nope
inoremap <c-z> nope
inoremap <c-\><c-n> nope
```

---

### Key information
- The flag is located at /flag.txt
- It is readable by anyone (hence the `chmod +r /flag.txt`)
- We are logged in as `user`
- Upon connecting we are put in a "vimjail"
- The `vimrc` file disables a few keybinds:
    - set insertmode: Forces us to be in insert mode
    - ctrl + o: Exit insert mode, run one command and return to insert mode
    - ctrl + l: Redraw the buffer nad return to normal mode
    - ctrl + z: Sends the foreground process to the background
    - ctrl + \ followed by ctrl + n: Switch to normal mode (alternatice to `Escape`)
- Vim is executed with certain flags that restricts the user even further:
    - -R: Sets read-only mode
    - -M: Sets "restricted" mode - restricts use of shell commands and other potentialy unsafe operations
    - -Z: Disables the use of modelines and the shell temporarily
    - -u /home/user/vimrc: Specifies the config file for Vim to use

### First thoughts

Can we quit vim using `ZZ` (alternative to `:wq`) and return to the shell? Nope.

Can we use netcat instead of socat? It just messes up the UI and certainly doesn't make things easier.

Since we are forced in insert mode, are there any keyboard combinations that we can use? Only one way to find out, I suppose.

### Bruteforcing..?

... and that one way to find out is to try, every single one of them.
Now, you may just do like me and ask ChatGPT for any keyboard combinations that utilize `ctrl`, `alt`, `alt gr`, `shift`, `meta` / `super`, but the response may be insufficient.

As I look back at the conversation I had with ChatGPT it actually sent me a response saying that `ctrl`+`r` was a method of running a command without exiting insert mode, must've been just tired enough to miss that (this is the intended way of solving vimjail*).

> `<C-r>=system('date')<CR>`
> This will execute the date command and insert the output at the current cursor position. The key sequence <C-r> means pressing Ctrl+r, and =system('date') is an expression that runs the date command and returns its output.

Since I somehow missed all of that, the next step was to try all keyboard combinations that include `ctrl`, `alt`, `alt gr`, `shift`, `meta`/`super`. 

A few minutes later I had finished mashing my keyboard. And it seemed to have worked - when I sent `ctrl` + `o` I was able to run (almost) any vim command prefixed with a `:`. Shell access and writing to files were still not possible.

### How-to: Open another file when already in a vim buffer

Now, surely ChatGPT knows how we can open another file in Vim?
As it turns out, we may use both `:n /flag.txt` or `:e /flag.txt` to get the flag. 
\> vimjail1, done.

### What happened?

Why were we suddenly able to run `ctrl` + `o` seemingly unrestricted..?
Something we mashed on the keyboard earlier must've messed with vim and basically enabled the keyboard shortcuts that were initially disabled by the `vimrc` configuration.


## vimjail 1.5

> Fixed unintended solve in vimjail1
>
> Connect with socat file:$(tty),raw,echo=0 tcp:vimjail1-5.chal.uiuc.tf:1337. 
> You may need to install socat.
> 
> Author: richard

Alright, so there was an unintended solution in vimjail 1, let's see if my method of mashing the keys work this time as well.

A few minutes later I concluded with: Yes, it does still work, whatever I had done.

My thoughts at this point were that whatever I had managed to do *was* the intended solution and wasn't patched because of that.
But what did I actually do to get foothold? It was something I had yet to discover.

## vimjail 2

> Connect with socat file:$(tty),raw,echo=0 tcp:vimjail2.chal.uiuc.tf:1337. 
> You may need to install socat.
> 
> Author: richard

Instead of the usual 4 files, we actually get 5 files this time:
- Dockerfile
- entry.sh
- nsjail.cfg
- vimrc
- viminfo

### Figuring out why

That being said, I wasted no time trying to actually figure out the difference between these challenges. Had I not been lucky enough to have found an unintended solution which solves all 4 vimjail challenges, then that would've been my next place to check.

For the third time, bruteforce was sufficient in getting access to the Vim command line.

This time though, I figured out that it had to do with some keyboard combination starting with `ctrl`.

By repeating steps below until I had command line access, I managed to figure out which keyboard shortcut it was that caused magic to happen.

1. Hit `ctrl`+`some key`
2. Hit `ctrl`+`o`
3. Check if the menu line says something like `-- (insert) --` indicating that `ctrl`+`o` succeded nad and let us run Vim commands.

The magic was caused by `ctrl` + `printscreen` and I still have no idea what that combination actually does or why it would mess with Vim. It does not work via TTY, not with netcat, not in a local vim environment, and not in Windows Terminal.

### What is the effect?

However, the effect of it is quite clear. After doing `ctrl`+`printscreen` it seems to just re-enable all the keyboard shortcuts that were disabled in the `vimrc` configuration.

This means that all these shortcuts are now enabled and usable:
- `ctrl`+`o`: run a single Vim command
- `ctrl`+`l`: redraw the buffer and return to normal mode
- `ctrl`+`z`: send the foreground process to the background (it is enabled, but does not work because shell commands are still prohibited by the flags in `entry.sh`)
- `ctrl`+`\` followed by `ctrl`+`n`: switch to normal mode

Very odd, I must say.

## vimjail 2.5

> Fixed unintended solve in vimjail2
>
> Connect with socat file:$(tty),raw,echo=0 tcp:vimjail2-5.chal.uiuc.tf:1337. > You may need to install socat.
> 
> Author: richard

Yet another unintended solve fixed. However, the `prinscreen` magic still seem to work flawlessly.

I needed a bit of help from vim's internal autocompletion for commands and files for this to work, as all lowercase letters (except `q`) and most special symbols were turned into underscores.

## Videos

### Original solution

1. `ctrl` + `printscreen`
2. `return`
3. `ctrl` + `o`
4. `:n flag.txt`

[![asciicast](https://asciinema.org/a/BAJbb6N8MQktlYbLQ2rC82IAB.svg)](https://asciinema.org/a/BAJbb6N8MQktlYbLQ2rC82IAB)

### Speedrun

#### For vimjail 1 and vimjail 1.5

1. `ctrl` + `printscreen`
2. `return`
3. `ctrl` + `l`
4. `:n flag.txt`
5. `ctrl` + `l`
6. `:q`

#### For vimjail 2 and vimjail 2.5

1. `ctrl` + `printscreen`
2. `return`
3. `ctrl` + `l`
4. `:q`
5. `return`

[![asciicast](https://asciinema.org/a/0G8SiBxJ1fFjMVn2RUUNcIdmM.svg)](https://asciinema.org/a/0G8SiBxJ1fFjMVn2RUUNcIdmM)

