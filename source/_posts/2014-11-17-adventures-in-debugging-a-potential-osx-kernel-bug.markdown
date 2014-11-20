---
layout: post
title: "Chasing an EPROTOTYPE through Rust, sendto, and the OSX Kernel with C-Reduce"
date: 2014-11-19 07:35:04 -0800
comments: true
categories: [apple, debugging, kernel, rust]
---

A slight diversion from my serialization series. Last week, strcat submitted
[#18885](https://github.com/rust-lang/rust/pull/18885) pull request, which adds
support for using a `Vec` as a `Writer`. Over the weekend I submitted
[#18980](https://github.com/rust-lang/rust/pull/18980), which allows a `&[u8]`
to be used as a `Reader`. Overall a pretty simple change. However, when I was
running the test suite, I found that the `std::io::net::tcp::write_close_ip4()`
test was occasionally failing:

```rust
    #[test]
    fn write_close_ip4() {
        let addr = next_test_ip4();
        let mut acceptor = TcpListener::bind(addr).listen();

        spawn(proc() {
            let _stream = TcpStream::connect(addr);
            // Close
        });

        let mut stream = acceptor.accept();
        let buf = [0];
        loop {
            match stream.write(buf) {
                Ok(..) => {}
                Err(e) => {
                    assert!(e.kind == ConnectionReset ||
                            e.kind == BrokenPipe ||
                            e.kind == ConnectionAborted,
                            "unknown error: {}", e);
                    break;
                }
            }
        }
    }
```

The `write` would succeed a few times, then occasionally error with the
`unknown error: Protocol wrong type for socket`, or the `EPROTOTYPE` errno.
This is a really surprising error. As far as I know, the only functions that
return that errno are `socket`, `socketpair`, and `connect`. I searched
everywhere, but I couldn't find any documentation suggesting that `write` would
ever produce that error.

I wasn't the only one who ran into it. bjz opened
[#18900](https://github.com/rust-lang/rust/issues/18900) describing the same
problem. One interesting thing to note is they were also on OSX Yosemite.  So I
took a little bit of time to extract out that test from the Rust test suite
into this [gist](https://gist.github.com/erickt/ac1f35e20834aa5d1972) and got
someone on #rust-internals to run it on linux for me with this little driver
script:

```sh
#!/bin/sh

set -e

rustc test.rs

for i in $(seq 1 1000); do
  ./test tcp::test::write_close_ip4
done
```

and it didn't error out. So it seems to be system dependent. Further
experimentation showed that if we introduced sleeps or a mutex synchronization 
appeared to fix the problem as well. At this point I wasn't sure if this was a
non-issue, a bug in our code, or a bug in the OS. One things for sure though,
if there is a bug, it could be scattered somewhere across the Rust codebase,
which just `std::io` alone is 20 files at around 12522 lines. It'd be a pain to
cut that down to a self contained test case.

Fortunately we've got [C-Reduce](http://embed.cs.utah.edu/creduce/) to help us
out. Back in May [Professor John Regehr](http://www.cs.utah.edu/~regehr/) from
the University of Utah came to our
[Bay Area Rust meetup](http://www.meetup.com/Rust-Bay-Area/events/169434302/)
 to talk about compiler testing and fuzzing. We recorded the talk, so if you
want to watch it, you can find it
[here](https://air.mozilla.org/rust-meetup-may-2014/). One of the things he
talked about was the tool C-Reduce his research group developed to
automatically cut
out unnecessary lines of code that can still reproduce a bug you're looking
for. While it's written to target C files, it turns out Rust is syntatically
close enough to C that it works out pretty well for it too. All you need is a
single source file and driver script that'll report if the compiled source file
reproduced the bug.

Aside 1: By the way, one of the other things I did this weekend was I put
together a Homebrew
[pull request for C-Reduce](https://github.com/Homebrew/homebrew/pull/34220).
It hasn't landed yet, but you want to use it, you can do:

```
% brew install https://raw.githubusercontent.com/erickt/homebrew/delta-and-creduce/Library/Formula/delta.rb
% brew install https://raw.githubusercontent.com/erickt/homebrew/delta-and-creduce/Library/Formula/creduce.rb
```

Hopefully it'll land soon so you'll be able to do:

```
% brew install creduce
```

Anyway, back to the story. So we've got a rather large code base to cover, and
while C-reduce does a pretty good job of trimming away lines, just pointing it
at the entire `std` module is a bit too much for it to handle in a reasonable
amount of time. It probably needs some more semantic information about Rust to
do a better job of cutting out broad swaths of code.

So I needed to do at least a rough pass to slim down the files. I figured the
problem was probably contained in `std::io` or `std::sys`, so I wrote a simple 
`test.rs` that explicitly included those modules as part of the crate (see 
this [gist](https://gist.github.com/erickt/ac1f35e20834aa5d1972) if you are
interested), and used the pretty printer to merge it into one file:

```
% rustc --pretty normal test.rs > test.rs
```

Aside 2: Our pretty printer still has some bugs in it, which I filed:
[19075](https://github.com/rust-lang/rust/issues/19075) and
[19077](https://github.com/rust-lang/rust/issues/19077).  Fortunately it was
pretty simple to fix those cases by hand in the generated `std.rs`, and odds
are good they'll be fixed by the time you might use this process.

Next up, we need a driver script. We can adapt our one from before. The only
special consideration is that we need to make sure to only exit with a return
code of 0 if the version of `std.rs` we're compiling errors with `EPROTOTYPE`:

```bash
#!/bin/sh

rustc \
  -A unused_imports \
  -A unused_unsafe \
  -A non_camel_case_types \
  -A unused_variables \
  --test \
  -o test \
  test.rs >/dev/null 2>&1
if [ "$?" -ne "0" ]; then
  exit 1
fi

root=`dirname $0`

for i in $(seq 1 1000); do
  $root/timeout.sh 5 ./test --nocapture tcp::test::write_close_ip4 2>&1 | tee log
  grep "Protocol wrong type for socket" log
  RET="$?"
  if [ "$RET" -eq "0" ]; then
          exit 0
  elif [ "$RET" -eq "143" ]; then
          exit 1
  fi
  echo
done

exit 1
```

I used the helper script
[timeout.sh](http://www.ict.griffith.edu.au/anthony/software/timeout.sh) to
time out tests in case C-Reduce accidentally made us an infinite loop.

Finally we're ready to start running C-Reduce:

```
% creduce ./driver.sh test.rs
===< 30598 >===
running 8 interestingness test(s) in parallel
===< pass_blank :: 0 >===
(0.0 %, 156170 bytes)
===< pass_clang_binsrch :: replace-function-def-with-decl >===
===< pass_clang_binsrch :: remove-unused-function >===
===< pass_lines :: 0 >===
(-0.6 %, 157231 bytes)
(1.2 %, 154323 bytes)
(3.7 %, 150455 bytes)
(4.6 %, 149074 bytes)
(5.5 %, 147639 bytes)
(6.4 %, 146172 bytes)
(7.3 %, 144881 bytes)
(7.7 %, 144187 bytes)
(9.1 %, 141936 bytes)
(9.9 %, 140706 bytes)
(10.3 %, 140104 bytes)
(10.4 %, 139998 bytes)
...
```

I let it run in the background for sometime in the background while I did some
other things. When I got back, C-Reduce automatically reduced the file from
153KB to a slim 22KB. I then reran rust with the lints enabled to manually cut
out the dead code C-Reduce failed to remove, and flattened away some
unnecessary structs and methods. I was finally left with:

```rust
extern crate libc;

use std::io::net::ip::Ipv4Addr;
use std::io::net::ip::SocketAddr;
use std::io::net::ip::ToSocketAddr;
use std::io::net::ip;
use std::io::test::next_test_ip4;
use std::mem;
use std::num::Int;
use std::os::errno;
use std::os::error_string;
use std::ptr;

fn bind(addr: ip::SocketAddr) -> libc::c_int {
    unsafe {
        let fd = match libc::socket(libc::AF_INET, libc::SOCK_STREAM, 0) {
            -1 => panic!(),
            fd => fd,
        };

        let mut storage = mem::zeroed();
        let len = addr_to_sockaddr(addr, &mut storage);
        let addrp = &storage as *const _ as *const libc::sockaddr;
        match libc::bind(fd, addrp, len) {
            -1 => panic!(),
            _ => fd,
        }
    }
}

fn listen(fd: libc::c_int, backlog: int) {
    unsafe {
        match libc::listen(fd, backlog as libc::c_int) {
            -1 => panic!(),
            _ => {}
        }
    }
}

fn accept(fd: libc::c_int) -> libc::c_int {
    unsafe {
        let x = libc::accept(fd, ptr::null_mut(), ptr::null_mut());
        match x {
            -1 => panic!(),
            fd => fd,
        }
    }
}

fn connect(addr: SocketAddr) -> libc::c_int {
    unsafe {
        let fd = match libc::socket(libc::AF_INET, libc::SOCK_STREAM, 0) {
            -1 => panic!(),
            fd => fd,
        };

        let mut storage = mem::zeroed();
        let len = addr_to_sockaddr(addr, &mut storage);
        let addrp = &storage as *const _ as *const libc::sockaddr;
        let x = libc::connect(fd, addrp, len);
        fd
    }
}

fn write(fd: libc::c_int, buf: &[u8]) -> Result<(), uint> {
    unsafe {
        let len = buf.len();
        let ret = libc::send(fd, buf.as_ptr() as *const _, len as libc::size_t, 0) as i64;
        if ret < 0 {
            Err(errno())
        } else {
            Ok(())
        }
    }
}

fn close(fd: libc::c_int) {
    unsafe {
        let x = libc::close(fd);
        assert_eq!(x, 0);
    }
}

fn addr_to_sockaddr(addr: SocketAddr, storage: &mut libc::sockaddr_storage) -> libc::socklen_t {
    let inaddr = match addr.ip {
        Ipv4Addr(a, b, c, d) => {
            let ip = a as u32 << 24 | b as u32 << 16 | c as u32 << 8 | d as u32 << 0;
            libc::in_addr {
                s_addr: Int::from_be(ip),
            }
        }
        _ => panic!(),
    };
    unsafe {
        let storage = storage as *mut _ as *mut libc::sockaddr_in;
        (*storage).sin_family = libc::AF_INET as libc::sa_family_t;
        (*storage).sin_port = addr.port.to_be();
        (*storage).sin_addr = inaddr;
        let len = mem::size_of::<libc::sockaddr_in>();
        len as libc::socklen_t
    }
}

fn main() {
    let addr = next_test_ip4();

    let listener = bind(addr);
    listen(listener, 128);

    spawn(proc() {
        let addresses = addr.to_socket_addr_all().unwrap();
        for addr in addresses.into_iter() {
            let fd = connect(addr);
            close(fd);
            return;
        }
    });

    let stream = accept(listener);
    loop {
        let x = write(stream, [0]);
        match x {
            Ok(..) => { }
            Err(e) => {
                let e = e as i32;
                assert!(
                    e == libc::ECONNREFUSED ||
                    e == libc::EPIPE ||
                    e == libc::ECONNABORTED,
                    "unknown error: {} {}", e, error_string(e as uint));
                break;
            }
        }
    }

    close(stream);
    close(listener);
}
```

This snippet reproduces the same `EPROTOTYPE` that we started with at the top
of the post. Pretty cool that we got here without much effort?

Now At this point you might say to yourself that couldn't I have extracted this
out myself?  And yeah, you would be right. This is a pretty much a c-in-rust
implementation of this bug. But what's great about using C-Reduce here is that
I only had to make some very rough guesses about what files were and were not
important to include in my `test.rs`. Eventually when we get some rust plugins
written for C-Reduce I probably could just point it at the whole `libstd` let
C-Reduce do it's thing. Doing this by hand can be a pretty long and painful
manual process, especially if we're trying to debug a codegen or runtime bug.
In the past I've spent hours reducing some codegen bugs down into a small
snippet that C-Reduce was also able to produce in a couple minutes.

The last step with this code was to eliminate Rust from the picture, and
translate this code into C:

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <strings.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <pthread.h>
#include <errno.h>

int do_server() {
  int fd;
  struct sockaddr_in server_addr;
 
  fd = socket(AF_INET, SOCK_STREAM, 0);

  if (fd == -1) {
    perror("error socket server");
    exit(1);
  }

  bzero((char*) &server_addr, sizeof(server_addr));
  server_addr.sin_family = AF_INET;
  server_addr.sin_addr.s_addr = INADDR_ANY;
  server_addr.sin_port = htons(9600);

  if (bind(fd, (struct sockaddr *) &server_addr, sizeof(server_addr)) < 0) {
    perror("error binding");
    exit(1);
  }

  return fd;
}

void* do_child_thread(void* unused) {
  struct sockaddr_in client_addr;
  int fd;

  fd = socket(AF_INET, SOCK_STREAM, 0);

  if (fd == -1) {
    perror("error socket client");
    exit(1);
  }

  bzero((char*) &client_addr, sizeof(client_addr));
  client_addr.sin_family = AF_INET;
  client_addr.sin_addr.s_addr = INADDR_ANY;
  client_addr.sin_port = htons(9600);

  if (connect(fd, (struct sockaddr *) &client_addr, sizeof(client_addr)) < 0) {
    perror("error connect");
    exit(1);
  }

  fprintf(stderr, "closing client socket\n");

  if (close(fd) < 0) {
    perror("error close client socket");
    exit(1);
  }

  fprintf(stderr, "closed client socket\n");

  return NULL;
}

int main(int argc, char** argv) {
  int server_fd, client_fd;
  socklen_t client_len;
  struct sockaddr_in client_addr;
  char buf[] = { 'a', '\n' };
  pthread_t child_thread;
  int rc;

  signal(SIGPIPE, SIG_IGN);

  server_fd = do_server();
  rc = listen(server_fd, 5);
  if (rc < 0) {
    perror("error listen");
    return 1;
  }

  rc = pthread_create(&child_thread, NULL, do_child_thread, NULL);
  if (rc != 0) {
    perror("error pthread_create");
    return 1;
  }

  client_len = sizeof(client_addr);
  client_fd = accept(server_fd, (struct sockaddr *) &client_addr, &client_len);
  if (client_fd < 0) {
    perror("error accept");
    return 1;
  }

  while (1) {
    fprintf(stderr, "before send\n");
    rc = send(client_fd, buf, sizeof(buf), 0);
    fprintf(stderr, "after send: %d\n", rc);

    if (rc < 0) {
      if (errno == EPIPE) {
        break;
      } else {
        int so_type;
        socklen_t so_len = sizeof(so_type);
        getsockopt(client_fd, SOL_SOCKET, SO_TYPE, &so_type, &so_len);
        fprintf(stderr, "type: %d %d\n", so_type, SOCK_STREAM);

        perror("error send");
        return 1;
      }
    }
  }

  fprintf(stderr, "before server closing client fd\n");
  if (close(client_fd) < 0) {
    perror("error close client");
    return 1;
  }
  fprintf(stderr, "after server closing client fd\n");


  fprintf(stderr, "before server closing fd\n");
  if (close(server_fd) < 0) {
    perror("error close server");
    return 1;
  }
  fprintf(stderr, "after server closing fd\n");

  rc = pthread_join(child_thread, NULL);
  if (rc != 0 && rc != ESRCH) {
    fprintf(stderr, "error pthread_join: %d\n", rc);
    return 1;
  }

  return 0;
}
```

This also produces `EPROTOTYPE`, so we can eliminate Rust altogther. But lets
keep digging. What exactly is producing this error? If I was on Linux, I'd use
`strace`, but that program isn't on Macs. There's a similar tool called
`dtruss`, but that seemed to slow things down enough that the `EPROTOTYPE`
never happened. Fortunately though there is another program called `errinfo`,
that just prints the `errno` along with every syscall. In one terminal I ran
`while ./test; do sleep 0.1; done`. In the other:

```
% sudo errinfo -n test
...
           a.out           stat64    0
           a.out             open    0
           a.out psynch_mutexwait    0
           a.out   write_nocancel    0
           a.out           sendto    0
           a.out   write_nocancel    0
           a.out   write_nocancel    0
           a.out           sendto   41  Protocol wrong type for socket
           a.out   write_nocancel    0
           a.out       getsockopt    0
...
```

Right there we see our `sendto` syscall is actually returning the `EPROTOTYPE`.
This `errno` then is definitely being created inside the OSX kernel, not in any
userspace code. Fortunately, most of the Apple kernel, XNU, is open sourced, so
we can dig down to what'll be the my last layer. You can find the tarballs at
http://www.opensource.apple.com/. But I'd rather use the
[unoffical GitHub repository](https://github.com/opensource-apple/xnu). Using GitHub's
search tools, We can find all 17 instances of
[EPROTOTYPE](https://github.com/opensource-apple/xnu/search?l=c&q=EPROTOTYPE&utf8=%E2%9C%93)
in the codebase. Now I don't know the XNU codebase, but there are still some
really interesting things we can find. The first is in
[bsd/kern/uipc\_usrreq.c](https://github.com/opensource-apple/xnu/blob/bb7368935f659ada117c0889612e379c97eb83b3/bsd/kern/uipc_usrreq.c#L408):

```c
/*
 * Returns:  0      Success
 *    EINVAL
 *    EOPNOTSUPP
 *    EPIPE
 *    ENOTCONN
 *    EISCONN
 *  unp_internalize:EINVAL
 *  unp_internalize:EBADF
 *  unp_connect:EAFNOSUPPORT  Address family not supported
 *  unp_connect:EINVAL        Invalid argument
 *  unp_connect:ENOTSOCK      Not a socket
 *  unp_connect:ECONNREFUSED  Connection refused
 *  unp_connect:EISCONN       Socket is connected
 *  unp_connect:EPROTOTYPE    Protocol wrong type for socket
 *  unp_connect:???
 *  sbappendaddr:ENOBUFS      [5th argument, contents modified]
 *  sbappendaddr:???          [whatever a filter author chooses]
 */
static int
uipc_send(struct socket *so, int flags, struct mbuf *m, struct sockaddr *nam,
    struct mbuf *control, proc_t p)
{
...
```

Hey look at that! There's handler for the `send` syscall (although for IPC, not
TCP) that actually documents that it can return `EPROTOTYPE`! While it doesn't
explain exactly how this can happen, the fact it mentions `unp_connect` hints
that `uipc_send` may trigger a connect, and that's exactly what we find a
couple lines into the function:

```c
...
    /* Connect if not connected yet. */
    /*
     * Note: A better implementation would complain
     * if not equal to the peer's address.
     */
    if ((so->so_state & SS_ISCONNECTED) == 0) {
      if (nam) {
        error = unp_connect(so, nam, p);
        if (error)
          break;  /* XXX */
      } else {
        error = ENOTCONN;
        break;
      }
    }
...
```

The fact that the comment says the socket might not be connected yet when we're
doing a `send` hints that Apple may have introduced some level of asynchrony
and preemption to sockets. So if we trigger the actual connect here, it could
then return `EPROTOTYPE`, which makes sense. Unfortunately that's still not
quite the behavior we're seeing. We're not getting `EPROTOTYPE` on our first
write, but after we've done a couple.

I believe we find that behavior in the actual TCP syscall file, 
[bsd/netinet/tcp\_usrreq.c](https://github.com/opensource-apple/xnu/blob/bb7368935f659ada117c0889612e379c97eb83b3/bsd/netinet/tcp_usrreq.c#L914-L948):

```c
static int
tcp_usr_send(struct socket *so, int flags, struct mbuf *m,
     struct sockaddr *nam, struct mbuf *control, struct proc *p)
{
  int error = 0;
  struct inpcb *inp = sotoinpcb(so);
  struct tcpcb *tp;
  uint32_t msgpri = MSG_PRI_DEFAULT;
#if INET6
  int isipv6;
#endif
  TCPDEBUG0;

  if (inp == NULL || inp->inp_state == INPCB_STATE_DEAD
#if NECP
    || (necp_socket_should_use_flow_divert(inp))
#endif /* NECP */
    ) {
    /*
     * OOPS! we lost a race, the TCP session got reset after
     * we checked SS_CANTSENDMORE, eg: while doing uiomove or a
     * network interrupt in the non-splnet() section of sosend().
     */
    if (m != NULL)
      m_freem(m);
    if (control != NULL) {
      m_freem(control);
      control = NULL;
    }

    if (inp == NULL)
      error = ECONNRESET;  /* XXX EPIPE? */
    else
      error = EPROTOTYPE;
    tp = NULL;
    TCPDEBUG1();
    goto out;
  }
...
```

I believe that comment explains everything we're seeing. If we trigger a `send`
while the kernel is in the middle of tearing down the socket, it returns
`EPROTOTYPE`. This then looks to be an error we could retry. Once the socket is
fully torn down, it should eventually return the proper `EPIPE`. This is also
pretty easy to test. So I modified the inner loop of our C test:

```
  while (1) {
    fprintf(stderr, "before send\n");
    rc = send(client_fd, buf, sizeof(buf), 0);
    fprintf(stderr, "after send: %d\n", rc);

    if (rc < 0) {
      if (errno == EPIPE) {
        break;
      } else if (errno == EPROTOTYPE) {
        continue;
      } else {
        int so_type;
        socklen_t so_len = sizeof(so_type);
        getsockopt(client_fd, SOL_SOCKET, SO_TYPE, &so_type, &so_len);
        fprintf(stderr, "type: %d %d\n", so_type, SOCK_STREAM);

        perror("error send");
        return 1;
      }
    }
  }
```

And yep, it exits cleanly. After all of this, I think it's pretty clear at this
point that there's no weird kernel corruption bug going on, just a poorly
documented edge case. But it sure was fun chasing this condition through the
system.

To prevent anyone else from tripping over this edge case, I filed a Apple Radar
ticket (number #19012087 for any Apple employees reading this). Hopefully if
anyone runs into this mysterious `EPROTOTYPE` it'll be documented for them, or
at least there's a chance they'll stumble over this blog post and save
themselves a weekend diving through the OS.
