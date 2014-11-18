---
layout: post
title: "Adventures debugging a potential OSX kernel bug with creduce"
date: 2014-11-17 07:35:04 -0800
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

The `write` would succeed a few times, then occasionally error with `unknown
error: Protocol wrong type for socket`, or the `EPROTOTYPE` error. This is a
really surprising error. As far as I know, the only functions that return that
error type are `socket`, `socketpair`, and `connect`. I searched everywhere,
but I couldn't find any documentation suggesting that `write` would ever
produce that error.

I wasn't the only one who ran into it. bjz opened
[#18900](https://github.com/rust-lang/rust/issues/18900) describing the same
problem. One interesting thing to note is they were also on OSX Yosemite.
So I took a little bit of time to extract out that test from the Rust test
suite into this [gist](https://gist.github.com/erickt/ac1f35e20834aa5d1972) and
got someone on #rust-internals to run it on linux for me with this little
driver script:

```sh
#!/bin/sh

set -e

rustc test.rs

for i in $(seq 1 1000); do
	./test tcp::test::write_close_ip4
done
```

and it didn't error out. So it seems to be isolated to a particular OS. So I
could think of a couple potential sources of errors here:

* We're using threads. Maybe the standard requires us locking some socket
	structure that we're not doing? I added some sleeps inside the thread, and
	that seemed to help the problems go away. Alex Crichton added some locks to
  the test, and that also seemed to make the `EPROTOTYPE` go away.
* We're relying on RAII to close the sockets. I think our implementation still
	zeros out data when we've left the scope. Maybe we're overwriting something
  we shouldn't be?
* This is a new OS, we could be tripping over a bug inside the kernel.
* This a new behavior, but it's just not documented.

One things for sure though, while the test case I extract was pretty small, if
there's a bug it could be scattered across the std::io files. That's 59 files
at about 10790 lines. Fortunately there's a tool out there called
[C-Reduce](http://embed.cs.utah.edu/creduce/) that can help with this. creduce
is an amazing tool that [Professor John
Regehr](http://www.cs.utah.edu/~regehr/) from the University of Utah introduced
us to back in the [May
meetup](http://www.meetup.com/Rust-Bay-Area/events/169434302/). With the help
of a driver scipt that compiles, runs, and reports if a file is "interesting",
creduce can automatically cut out the unnecessary lines of a file that still
reproduce that "interesting" effect. While it's designed to run on C files, our
syntax is close enough to C that it still works out pretty well for us too.

Unfortunately it can be a bit of a pain to install, so if you're on a Mac, I
took some time this weekend to submit a PR to
[Homebrew](https://github.com/Homebrew/homebrew/pull/34220). To install it, just do:

```
# If it's already been merged in:
% brew install creduce

# Otherwise, try installing it from my PR:
% brew install https://raw.githubusercontent.com/erickt/homebrew/delta-and-creduce/Library/Formula/delta.rb
% brew install https://raw.githubusercontent.com/erickt/homebrew/delta-and-creduce/Library/Formula/creduce.rb
```

Anyway, creduce can only work on a single file, so we need to concatenate all
of the files into one uber file. This can be done by copying all the `io` files
into our test directory and include them into our crate and get everything to
compile. See this [gist](https://gist.github.com/erickt/ac1f35e20834aa5d1972)
if you're interested. Finally use the pretty printer to convert it into a
single file:

```
% rustc --pretty normal test.rs
```

Next up, we need to adapt our driver from earlier to be a bit more thorough.
creduce considers the result of a driver interesting if it exits with a `0`
return code. So we need to make sure that miscompilation, the programming
falling into an infinite loop (using the
[timeout.sh](http://www.ict.griffith.edu.au/anthony/software/timeout.sh), or
the test program not erroring out with `EPROTOTYPE` to exit with a `1` status.
Also, just so the output is readable, we filter out a lot of Rust's lints:

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
	echo $i
	$root/timeout.sh 5 ./test --nocapture tcp::test::write_close_ip4 2>&1 | tee log
	grep "Protocol wrong type for socket" log
	RET="$?"
	if [ "$RET" -eq "0" ]; then
		echo "found a bad one" 1>&2
		exit 0
	elif [ "$RET" -eq "143" ]; then
		echo "timed out" 1>&2
		exit 1
	fi
	echo
done

exit 1
```

It's a shame creduce isn't aware of Rust lints. It could really help speeding
up the process of cutting out code if it was able to take advantage of the
compiler saying this function or block of code is dead. But that's another
project for another day.

So we start out with a file with about `153 KB`. To reduce, we then run:

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

This may take a while to run. I found in this particular instance I needed to
creduce a bunch of code, then rerun `rust --test test.rs` and manually remove
the dead code. Then back to run creduce some more. Finally, creduce finished, I
cleaned up the code, and after reformatting, ended up with this
[gist](https://gist.github.com/erickt/41170b8e6f1dd2e8baa4). Only 406 lines! So
much better. It's also pretty clear what's going on at this point, and so it's
pretty straightforward to get rid of the structs, implementations, and etc to
turn it into:

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

Which still produces the `EPROTOTYPE`. So we can safely remove RAII being the
problem. This is also small enough that we can convert it into C and eliminate
Rust altogether:

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

So not Rust's fault at all! So this leads us to the OS. The next step is to try
to observe what syscalls are happening when we get the error.  If I was on
Linux, I'd use `strace`, but that program isn't on Macs. There's a similar tool
called `dtruss`, but that seemed to slow things down enough that the
`EPROTOTYPE` never happened. Fortunately though there is another program called
`errinfo`, that just prints the `errno` along with every syscall.  Exactly what
I want. In one terminal I ran `while ./test; do sleep 0.1; done`. In the other:

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

So we can clearly see that it is definitely the `write`-by-way-of-`sendto`
that's getting the error. So this is definitely happening inside the OSX
kernel, not in any userspace code.

Most of the Apple kernel, XNU, is open sourced. You can find it at
http://www.opensource.apple.com/. Or even better, there is an [unoffical GitHub
repository](https://github.com/opensource-apple/xnu), so we can use GitHub's
native search interface for
[EPROTOTYPE](https://github.com/opensource-apple/xnu/search?l=c&q=EPROTOTYPE&utf8=%E2%9C%93)
and we find only 17 instances. I don't know the XNU codebase, but I was able to
still find some pretty interesting things. The first is in
[bsd/kern/uipc\_usrreq.c](https://github.com/opensource-apple/xnu/blob/bb7368935f659ada117c0889612e379c97eb83b3/bsd/kern/uipc_usrreq.c#L408):

```c
/*
 * Returns:	0			Success
 *		EINVAL
 *		EOPNOTSUPP
 *		EPIPE
 *		ENOTCONN
 *		EISCONN
 *	unp_internalize:EINVAL
 *	unp_internalize:EBADF
 *	unp_connect:EAFNOSUPPORT	Address family not supported
 *	unp_connect:EINVAL		Invalid argument
 *	unp_connect:ENOTSOCK		Not a socket
 *	unp_connect:ECONNREFUSED	Connection refused
 *	unp_connect:EISCONN		Socket is connected
 *	unp_connect:EPROTOTYPE		Protocol wrong type for socket
 *	unp_connect:???
 *	sbappendaddr:ENOBUFS		[5th argument, contents modified]
 *	sbappendaddr:???		[whatever a filter author chooses]
 */
static int
uipc_send(struct socket *so, int flags, struct mbuf *m, struct sockaddr *nam,
    struct mbuf *control, proc_t p)
{
...
```

Ah ha! Even though this file suggests it's IPC, not TCP related, this is the
first piece of documentation that mentions `send` and `EPROTOTYPE` at the same
time. It also hints at how the error can occur by metioning
`unp_connect::EPROTOTYPE`. Looking through the function, we can then see that
it is in fact using the `uipc_connect`:

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
					break;	/* XXX */
			} else {
				error = ENOTCONN;
				break;
			}
		}
...
```

The `connect` call is documented as being able to return `EPROTOTYPE`. I'm
guessing Apple introduced asynchronous connection, where if a user is really
fast, they can call `sendto` before the `connect` finished connecting. If that
connection would have `EPROTOTYPE`ed, then the `sendto` will have to too.

Unfortunately though, that's still not quite the behavior we're seeing. We're
managing to do a couple successful writes before we get the `EPROTOTYPE`, so
that's not quite yet the smoking gun.

What might be is in what I think the code that implements the TCP syscall,
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
			error = ECONNRESET;	/* XXX EPIPE? */
		else
			error = EPROTOTYPE;
		tp = NULL;
		TCPDEBUG1();
		goto out;
	}
...
```

I think that comment may be the actual source of the error. If I'm reading it
correctly, it looks like this is yet another case where we could be `send`ing
while another operation hasn't completed yet. In this case, tearing down the
socket. I think that matches our symptoms exactly. This also suggests that this
could be a retryable error, just like `EINTR`. To prove this out, I modified my
`test.c` to ignore the error, and sure enough, the next error was the expected
`EPIPE`:

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

I think that's far enough down this rabbit hole. I think it's pretty clear at
this point that there's no weird kernel corruption bug going on, just a poorly
documented edge case. I've filed a Apple Radar ticket (number #19012087 for any
Apple employees reading this). Hopefully if anyone else runs into this weird
edge case, Apple will have updated the documentation, or they'll stumble across
this blog post and save themselves a weekend.
