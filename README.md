# Linux for Hackers — Workshop Handout

**University Cybersecurity Club — Workshop #1** **For complete beginners. No prior Linux or security experience needed.**

---

## Before You Arrive

You'll need a working Linux environment. Set this up **before the workshop** — we won't have time to troubleshoot installs live.

**Minimum requirements:** a laptop with at least 8 GB RAM and 40 GB free disk space.

**Pick one:**

- **VirtualBox** (free, all platforms) + the **Kali Linux pre-built VirtualBox image** — recommended default
- **VMware Workstation Player** — alternative to VirtualBox
- **WSL2** on Windows (`wsl --install` in an admin PowerShell) — acceptable if you're short on disk space, though some commands later in this guide won't apply (no real GUI, different networking)
- **UTM** with an ARM Ubuntu/Kali image, on Apple Silicon Macs

Boot your VM once before the workshop and confirm you reach a login prompt or desktop.

**Take a snapshot of your VM once it's set up.** This is the single most useful thing you can do before we start: if you break something later, you revert in 30 seconds and lose nothing. Fear of breaking things is the #1 thing that slows beginners down — a snapshot removes that fear completely.

---

## Table of Contents

**Part I — Foundations**

1. [What Is an Operating System?](#module-1--what-is-an-operating-system)
2. [The Kernel, User Space & System Calls](#module-2--the-kernel-user-space--system-calls)
3. [The Boot Process](#module-3--the-boot-process)
4. [History: UNIX, GNU, and Linux](#module-4--history-unix-gnu-and-linux)
5. [Distributions: Kernel vs. Distro](#module-5--distributions-kernel-vs-distro)
6. [Why Hackers Live on Linux](#module-6--why-hackers-live-on-linux)

**Part II — Living in the Shell** 7. [The Shell, the Terminal & the Prompt](#module-7--the-shell-the-terminal--the-prompt) 8. [Anatomy of a Command + Getting Help](#module-8--anatomy-of-a-command--getting-help) 9. [The Filesystem Hierarchy Standard](#module-9--the-filesystem-hierarchy-standard) 10. [Navigation & File Operations](#module-10--navigation--file-operations) 11. [Everything Is a File](#module-11--everything-is-a-file) 12. [Text Processing: grep, sed, awk & Friends](#module-12--text-processing-the-hackers-power-tools) 13. [Redirection, Pipes & Globbing](#module-13--redirection-pipes--globbing) 14. [Finding Things](#module-14--finding-things)

**Part III — The Security-Critical Core** 15. [Users, Groups & Identity](#module-15--users-groups--identity) 16. [Permissions](#module-16--permissions) 17. [SUID, SGID, Sticky Bit & Capabilities](#module-17--suid-sgid-sticky-bit--capabilities) 18. [sudo and Privilege Boundaries](#module-18--sudo-and-privilege-boundaries) 19. [Processes, Signals & Services](#module-19--processes-signals--services) 20. [Scheduled Tasks: cron & systemd timers](#module-20--scheduled-tasks-cron--systemd-timers) 21. [Package Management & Software Supply Chain](#module-21--package-management--software-supply-chain)

**Part IV — Networking & Offensive Context** 22. [Linux Networking Fundamentals](#module-22--linux-networking-fundamentals) 23. [SSH — The Hacker's Front Door](#module-23--ssh--the-hackers-front-door) 24. [netcat & Basic Network Interaction](#module-24--netcat--basic-network-interaction) 25. [Bash Scripting for Automation](#module-25--bash-scripting-for-automation) 26. [Logs, History & Forensic Footprints](#module-26--logs-history--forensic-footprints) 27. [Mapping Linux Skills to the Attack Lifecycle](#module-27--mapping-linux-skills-to-the-attack-lifecycle) 28. [The Blue Side: Hardening Basics](#module-28--the-blue-side-hardening-basics) 29. [Ethics, Law & Responsible Disclosure](#module-29--ethics-law--responsible-disclosure)

**Part V — Practice**

- [Hands-On Labs](#hands-on-labs)
- [Mini CTF](#mini-ctf)
- [Cheat Sheet](#cheat-sheet)
- [Homework & Learning Path](#homework--learning-path)
- [Resources](#resources)

---

# Part I — Foundations: How Computers Actually Work

> Twenty-five minutes of fundamentals before you touch a tool might feel slow. It isn't. Understanding *why* a system behaves the way it does is what separates someone who runs tools from someone who understands what those tools are actually doing — and that difference matters the moment something doesn't go as expected.

---

## Module 1 — What Is an Operating System?

An operating system is the manager and the middleman between your hardware and your programs. Without it, every program would need to know how to talk to every model of disk, network card, and GPU in existence. The OS does five jobs:

1. **Process management** — decides which program runs on which CPU core, and when. The *scheduler* switches between processes thousands of times a second, creating the illusion that everything runs at once.
2. **Memory management** — gives every process its own private *virtual address space*. One process literally cannot read another process's memory, because the addresses mean different things to each. This is enforced by hardware (the MMU).
3. **Filesystem & storage** — turns a spinning disk's raw magnetic regions (or an SSD's raw memory cells) into something you can navigate as `/home/user/notes.txt`.
4. **Device management (drivers)** — translates generic requests ("send these bytes over the network") into vendor-specific hardware commands.
5. **Security & access control** — decides who can do what. Users, permissions, privilege levels.

### Why this matters to a hacker

> **Almost every vulnerability class is a failure of one of these five jobs.**
> 
> - Buffer overflow → a failure of memory isolation
> - Privilege escalation → a failure of access control
> - Race condition / TOCTOU → a failure of process scheduling assumptions
> - Path traversal → a failure of filesystem boundary enforcement
> - Malicious driver / rootkit → a failure of device trust

### Try it

Is the desktop the operating system? No — the graphical desktop (GNOME, KDE, etc.) is just an application running *on top of* the OS. Switch to a raw text console and see the system running with no graphics at all:

```
Ctrl+Alt+F3
```

Log in, run a few commands, then return to your graphical session with `Ctrl+Alt+F1` or `F2`.

---

## Module 2 — The Kernel, User Space & System Calls

The CPU has privilege levels, often drawn as **rings**. Ring 0 is the most privileged (kernel mode); Ring 3 is where your programs run (user mode).

- **Kernel space** — the kernel's code and memory. Full access to hardware, memory, devices.
- **User space** — where your browser, your text editor, and your exploit all run. Restricted. Cannot touch hardware directly.

A user-space program that needs anything real — read a file, open a socket, create a process — must **ask the kernel** via a **system call (syscall)**: a controlled, deliberate doorway through the privilege boundary. Common syscalls: `open()`, `read()`, `write()`, `close()`, `fork()`, `execve()`, `socket()`, `connect()`, `chmod()`, `setuid()`.

> **The user/kernel boundary is the security boundary of the machine.** Everything else — file permissions, user accounts, containers — is built on top of it and enforced by it. A normal exploit gets you code execution in user space, as some user. A **kernel exploit** breaks the boundary itself — game over. **Rootkits** live in kernel space precisely because from there they can lie to every user-space tool you might use to detect them.

### Try it — watch syscalls happen live

```bash
strace ls
strace -e trace=openat ls /etc
```

Every line is `ls` asking the kernel for permission to do something. It cannot read a single byte on its own.

> **Note:** "Linux is an operating system" is slightly imprecise. **Linux is a kernel.** What you install is a kernel + GNU utilities + libraries + package manager + desktop = a *distribution*. More on this in Module 5.

**Hardware detail (optional, good to know):** x86 CPUs actually offer four privilege rings (0–3), not two. Linux only uses Ring 0 and Ring 3 — Rings 1 and 2 sit unused in every x86 chip. Linux chose a simple all-or-nothing trust model instead of a gradient.

---

## Module 3 — The Boot Process

The chain from power-on to login prompt:

1. **Firmware (BIOS / UEFI)** — first code to run, lives on the motherboard. Initialises hardware, finds a bootable device.
2. **Secure Boot** (UEFI feature) — firmware verifies the bootloader's cryptographic signature before running it.
3. **Bootloader (GRUB2)** — loads the kernel image and the *initramfs* into memory.
4. **Kernel initialisation** — kernel decompresses, detects hardware, mounts the root filesystem.
5. **init / PID 1 (systemd)** — first user-space process. Starts every service, mounts filesystems, brings up networking.
6. **Login** — the login prompt appears. PAM handles authentication.

### Why this matters

- **GRUB and physical access:** with physical access and no GRUB password, an attacker can edit the boot entry to drop into a root shell, bypassing login entirely. This is the classic demonstration that **physical access = root access**, and why full-disk encryption matters.
- **Persistence:** malware wants to survive reboot, so it targets init scripts, systemd units, initramfs, or the bootloader itself.

### Try it

```bash
systemctl list-unit-files --state=enabled | head -20   # what starts at boot
systemd-analyze                                          # boot timing
systemd-analyze blame | head -10                         # what's slow
dmesg | head -30                                          # the kernel's own boot log
```

---

## Module 4 — History: UNIX, GNU, and Linux

1. **1969 — UNIX**, Bell Labs, Ken Thompson & Dennis Ritchie. Rewritten in C in 1973 — a radical idea at the time. Before this, every computer needed an operating system hand-written in that specific machine's assembly language; one OS could not run on a different machine's hardware. Writing UNIX in C meant a compiler could translate the same code for different machines, so the OS itself became portable for the first time.
2. **The UNIX philosophy:**
   - Write programs that do one thing and do it well — small, focused, reliable tools instead of one program trying to do everything.
   - Write programs to work together — composability via pipes, so independent tools can be chained into something bigger than any one of them.
   - Write programs that handle text streams, because that is a universal interface — every tool speaks plain text, so any tool can feed any other tool without a custom translator between them.
3. **1983 — GNU project**, Richard Stallman. UNIX had gone proprietary — locked source code, no legal right to read, modify, or share it. GNU's goal: rebuild UNIX's functionality from scratch as free software — keep the proven design, fix the licensing. They wrote the compiler, the shell, the core utilities (`ls`, `cp`, `grep`, etc. — still what's running on your Linux machine today) — everything except a working kernel.
4. **1991 — Linux**, Linus Torvalds. Released under the GPL. The missing kernel GNU never finished.
5. **GNU + Linux = a complete free OS.** Hence the purist name "GNU/Linux."
6. **Today** — Linux runs the majority of web servers, all of the top supercomputers, Android, most cloud infrastructure, and most embedded/IoT devices.

> **Note on UNIX itself:** UNIX was a complete, proprietary operating system — not just a kernel. Linux shares none of its original code; it reimplements the same design as free software. The shared conventions both follow are formalized as **POSIX**, which is why a bash script mostly runs the same on Linux, BSD, and macOS.

---

## Module 5 — Distributions: Kernel vs. Distro

A **distribution** = Linux kernel + GNU coreutils + a package manager + a default software set + (optionally) a desktop environment.

| Family | Package manager | Package format | Examples |
| --- | --- | --- | --- |
| **Debian** | `apt` / `dpkg` | `.deb` | Debian, Ubuntu, **Kali**, Parrot, Mint |
| **Red Hat** | `dnf` / `rpm` | `.rpm` | RHEL, Fedora, CentOS Stream, Rocky, Alma |
| Arch | `pacman` | `.pkg.tar.zst` | Arch, Manjaro, BlackArch |
| SUSE | `zypper` | `.rpm` | openSUSE, SLES |

### Security-focused distributions

- **Kali Linux** — the standard offensive toolkit distro, \~600 pre-installed tools, Debian-based.
- **Parrot Security OS** — Debian-based, lighter, good on weaker hardware.
- **BlackArch** — Arch-based, largest tool count.
- **Tails** — amnesic, Tor-routed, for privacy.
- **REMnux** — malware analysis and reverse engineering.
- **Security Onion** — the defensive counterpart: IDS, log analysis, network monitoring.

> **Important advice: don't daily-drive Kali.** Kali is a tool delivery mechanism, designed to be run as a disposable VM. It historically encouraged running as root, its rolling updates can break things, and using it as your main OS teaches you nothing about administering a normal system. **Learn on Ubuntu or Debian. Attack from Kali in a VM.**

### Try it

```bash
cat /etc/os-release       # which distro am I on
uname -a                  # kernel version, architecture
hostnamectl                # tidy summary
```

---

## Module 6 — Why Hackers Live on Linux

1. **It's what you're attacking.** Most web servers, cloud instances, containers, and network devices run Linux.
2. **Native tooling.** Nmap, Wireshark, Metasploit, Burp, John, Hashcat, sqlmap, Ghidra — all built for or best supported on Linux.
3. **Transparency and control.** You can read the kernel source, inspect any binary, watch every syscall (`strace`), trace every packet.
4. **The shell is a programming environment.** Pipes let you compose tools you didn't write into a tool you did.
5. **Automation and scriptability.** Everything is scriptable, headless, and remotable over SSH.
6. **Networking is first-class.** Raw sockets, packet crafting, custom routing, interface manipulation, traffic interception — all exposed and permitted, with the right privileges.
7. **Free, and reproducible.** Spin up identical VMs, snapshot, and revert with zero licensing friction.

> **A fair caveat:** Linux is not "more secure" in some magic way. It is *differently* secure. It has had severe vulnerabilities too — Shellshock, Heartbleed, Dirty COW, Log4Shell. Its advantage is transparency and control, not invulnerability. Most corporate networks run on Windows/Active Directory, and that is its own, equally real discipline.

---

# Part II — Living in the Shell

---

## Module 7 — The Shell, the Terminal & the Prompt

### Three words people conflate

- **Terminal (emulator)** — the *window*. GNOME Terminal, Konsole, Alacritty, Windows Terminal. It just draws text and captures keystrokes.
- **Shell** — the *program* that interprets what you type. Bash, Zsh, Fish, sh, Dash.
- **Console / TTY** — the raw text channel, no graphics involved (`Ctrl+Alt+F3`).

**The relationship, as an analogy:** the console/TTY is the phone line — it just carries characters back and forth, with no understanding of them. The shell is the person on the other end of the call who actually understands what you're saying and responds. The terminal emulator is the handset you're holding.

### Reading your prompt

```
kali@kali:~$
└┬─┘ └─┬┘ └┬┘└┬┘
 │     │   │  └─ $ = normal user, # = ROOT (danger)
 │     │   └──── current directory (~ = your home)
 │     └──────── hostname
 └────────────── username
```

> **Safety reflex: look at your prompt before you press Enter.** If you see a `#`, every command you type can destroy the system.

### Shells worth knowing

- `bash` — the default nearly everywhere. **Learn this one.**
- `zsh` — Kali's default now, bash-compatible with nicer completion.
- `sh` — the minimal POSIX shell scripts run under.
- `fish` — friendly, but not POSIX-compatible; don't script in it.

```bash
echo $SHELL       # your login shell
cat /etc/shells   # installed shells
```

### Survival keybindings — drill these until they're automatic

| Key | Does |
| --- | --- |
| `Tab` | **Autocomplete.** Press it constantly. Twice for options. |
| `Ctrl+C` | Interrupt the running command |
| `Ctrl+D` | End of input / logout |
| `Ctrl+L` | Clear screen |
| `Ctrl+R` | Search command history |
| `Ctrl+A` / `Ctrl+E` | Jump to start / end of line |
| `Ctrl+U` / `Ctrl+K` | Delete to start / end of line |
| `↑` / `↓` | Previous / next command |
| `Ctrl+Z` then `bg`/`fg` | Suspend, resume in background/foreground |

> `Tab` is the single highest-value habit you can build today. Type `cd /et` and press Tab right now. Get used to it — it's what makes the command line feel fast instead of hostile.

---

## Module 8 — Anatomy of a Command + Getting Help

```
    command   options/flags        arguments
      │            │                   │
    ┌─┴─┐   ┌──────┴───────┐   ┌───────┴────────┐
    ls      -l -a --human-readable   /var/log
```

- **Short flags:** `-l`, single dash, one letter, combinable → `-la` = `-l -a`
- **Long flags:** `--all`, double dash, readable
- Some flags take values: `-o output.txt`, `--output=output.txt`

### Getting help — in this order

```bash
man ls              # the manual. / to search, n for next, q to quit
ls --help           # quick flag summary
tldr ls             # community examples (sudo apt install tldr)
apropos password    # search man pages by keyword
type ls             # is it a binary, builtin, alias, or function?
which nmap          # where does this binary live
```

### Man page sections

| Section | Contains | Example |
| --- | --- | --- |
| 1 | User commands | `man 1 passwd` |
| 2 | System calls | `man 2 open` |
| 3 | Library functions | `man 3 printf` |
| 5 | File formats | `man 5 passwd` ← the `/etc/passwd` file |
| 8 | Admin commands | `man 8 useradd` |

> `man passwd` and `man 5 passwd` are **different documents** — try it. Section 5 is where you learn config file formats.

> Real practitioners type `man` before they type a search engine query. On an isolated network during a real engagement, `man` may be all you have.

---

## Module 9 — The Filesystem Hierarchy Standard

Linux has **one tree**, rooted at `/`. No drive letters. Disks, USB sticks, and network shares are *mounted* into the tree as directories.

```bash
findmnt        # see the tree of mounts
df -h          # disk usage per mount
lsblk          # block devices
```

| Path | Contents | Why it matters |
| --- | --- | --- |
| `/` | Root of everything |  |
| `/usr/bin`, `/usr/sbin` | Most programs | SUID binaries often live here |
| `/usr/local/bin` | Locally installed software | Often writable/misconfigured |
| **`/etc`** | **All system configuration** | `passwd`, `shadow`, `sudoers`, `crontab`, `ssh/`, `hosts` |
| `/home/<user>` | User home directories | SSH keys, `.bash_history`, credentials in dotfiles |
| `/root` | root's home |  |
| **`/var/log`** | **System and app logs** | Evidence, and what attackers tamper with |
| `/var/www` | Web server content | Web shells, config with DB creds |
| `/tmp`, `/var/tmp` | Temporary files, world-writable | Attacker staging ground |
| **`/proc`** | **Virtual: live kernel & process info** | `/proc/<pid>/cmdline`, `environ`, `/proc/version` |
| `/sys` | Virtual: kernel/device parameters | Hardware info |
| `/dev` | Device files | `/dev/null`, `/dev/urandom`, `/dev/tcp` |
| `/opt` | Third-party software | Often poorly permissioned |
| `/boot` | Kernel, initramfs, GRUB config | Bootkits, persistence |
| `/lib`, `/usr/lib` | Shared libraries | Library hijacking, `LD_PRELOAD` |

### Try it — explore `/proc`

```bash
ls /proc | head -20              # every number is a running process
cat /proc/version                # exact kernel — first thing checked in privesc enumeration
cat /proc/cpuinfo | head
cat /proc/self/cmdline; echo
ls -l /proc/self/fd              # open file descriptors of this process
```

These files do not exist on disk — the kernel invents them when you read them. `/proc` is a live window into the kernel, exposed as text.

---

## Module 10 — Navigation & File Operations

### Navigation

```bash
pwd                 # where am I
cd /var/log          # absolute path — starts from /
cd ../..            # relative path — up two levels
cd ~                 # home directory
cd -                 # previous directory (toggle)
```

**Absolute vs relative paths:** `/etc/passwd` always means the same file. `etc/passwd` means "etc inside wherever I currently am." This distinction underpins **path traversal** vulnerabilities (`../../etc/passwd`) that you'll encounter in web security.

### Listing

```bash
ls -la ~             # all files including hidden, long format — run this now
ls -lh               # human-readable sizes
ls -lt               # sort by modification time
```

> **Hidden files are just files whose name starts with a dot** — there's no special "hidden" attribute. `.bashrc`, `.ssh/`, `.git/`, `.env` — the most interesting files on a system are usually hidden, and `ls` without `-a` will never show them.

### File operations

```bash
touch file.txt                  # create empty / update timestamp
mkdir -p a/b/c                  # make nested directories
cp -r dir1 dir2                 # copy directory recursively
mv old new                      # move OR rename
rm -rf dir                      # delete, force, no prompts
ln -s /path/to/target linkname  # symbolic link
```

> ⚠️ **There is no recycle bin.** `rm -rf /` with sufficient privilege destroys the system. A misplaced space — `rm -rf / home/user` — is a classic disaster. Habit: run the `ls` version of a command first, confirm it matches what you intend, then swap `ls` for `rm`. This is exactly why you took a snapshot.

### Reading files

```bash
cat file.txt            # dump whole file
less file.txt            # scrollable pager (/ search, q quit)
head -20 file.txt        # first 20 lines
tail -f /var/log/syslog # follow a log live as it's written
wc -l file.txt           # count lines
file mystery              # what TYPE is this, regardless of extension
strings binary | less     # printable text inside a binary
```

> **Linux does not care about file extensions.** `file` identifies by content, not name — a file named `image.jpg` can secretly be an executable. This is how malicious uploads sneak past extension-only filters.

### Editors

- **nano** — `Ctrl+O` save, `Ctrl+X` exit. Zero friction, use this.
- **vim** — if you ever land in it by accident: press `Esc`, type `:q!`, press `Enter` to escape without saving.

---

## Module 11 — Everything Is a File

Nearly everything in Linux is exposed through the file interface — you can `open`, `read`, `write`, and `close` it: regular files, directories, devices (`/dev/sda`, `/dev/null`), processes (`/proc/1234/`), kernel parameters (`/sys/`), network sockets and pipes.

### File types in `ls -l` — the first character

| Char | Type |
| --- | --- |
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device |
| `b` | Block device |
| `s` | Socket |
| `p` | Named pipe (FIFO) |

### Useful special files

```bash
command > /dev/null 2>&1    # discard all output
head -c 16 /dev/urandom | xxd    # cryptographic randomness, as readable hex
ls -l /dev/sda               # your actual disk, exposed as a file (or /dev/nvme0n1 on NVMe drives)
```

`/dev/urandom` generates an endless stream of random bytes from kernel-gathered noise — this is the mechanism behind generating SSH keys and session tokens. `/dev/sda` (or `nvme0n1` on newer NVMe drives) is your entire physical disk, represented as a file — `dd if=/dev/sda of=backup.img` clones a disk byte-for-byte using nothing but ordinary file tools, which is also why writing to it carelessly can destroy every partition.

### Inodes — how a filename actually connects to data

A filename is just a label inside a directory's table, pointing to an **inode number**. The inode holds everything real about the file — permissions, owner, size, timestamps, and pointers to the actual data blocks on disk. The filename itself is **not** stored in the inode.

```bash
ls -li notes.txt      # first column = inode number
stat notes.txt         # full inode metadata dump
```

Because a name is just a pointer to an inode, one file can have multiple names — a **hard link**:

```bash
ln notes.txt backup.txt
ls -li notes.txt backup.txt    # same inode number — same file, two names
```

This also explains why **delete permission comes from the directory, not the file**: deleting a name just removes an entry from the directory's table and decrements the inode's link count. The data isn't actually erased until the last name pointing to it is gone — which is why deleted files are often recoverable.

---

## Module 12 — Text Processing: The Hacker's Power Tools

Security work is overwhelmingly text wrangling — log files, scan output, config files, wordlists. Being fast at slicing text is being fast at finding the finding.

### grep — search

```bash
grep -i "error" file.log           # case-insensitive
grep -r "password" /etc/ 2>/dev/null   # recursive
grep -n "root" /etc/passwd          # with line numbers
grep -v "debug" file.log            # invert: lines NOT matching
grep -c "404" access.log             # count matches
grep -A 3 -B 3 "failed" auth.log     # 3 lines of context after/before
```

### cut, sort, uniq — the pipeline family

```bash
cut -d: -f1 /etc/passwd    # field 1, colon-delimited = all usernames
sort -u file.txt            # sorted + deduplicated
sort -rn file.txt           # reverse numeric
uniq -c                     # count consecutive duplicates (MUST sort first)
```

### sed — stream editor

```bash
sed 's/old/new/g' file         # replace all occurrences
sed -i 's/old/new/g' file      # edit file IN PLACE — careful
sed -n '5,10p' file             # print only lines 5–10
sed '/^#/d' config               # delete comment lines
```

### awk — field-aware processing

```bash
awk '{print $1}' access.log                 # first field
awk -F: '{print $1, $7}' /etc/passwd        # custom delimiter, two fields
awk '{sum+=$5} END {print sum}' access.log  # sum a column
```

### Build a pipeline — the key skill of this whole module

Build this up one piece at a time on a sample `access.log`, checking the output at each stage:

```bash
cat access.log | awk '{print $1}'
cat access.log | awk '{print $1}' | sort
cat access.log | awk '{print $1}' | sort | uniq -c
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```

**What each stage does:**

- `awk '{print $1}'` — pulls just the first field (the client IP) out of every line.
- `sort` — groups identical IPs next to each other. **Required before `uniq`**, which only merges duplicates that are directly adjacent.
- `uniq -c` — collapses each group into one line, prefixed with a count of how many times it appeared.
- `sort -rn` — re-sorts those counted lines, treating the count as a real **n**umber (not text) and in **r**everse order, so the busiest IP ends up first.
- `head -10` — keeps only the top 10 lines.

That final command is a complete log analyzer built from five independent tools, none of which were written with each other in mind — this is the UNIX philosophy from Module 4, made concrete.

---

## Module 13 — Redirection, Pipes & Globbing

### The three standard streams

| Stream | Number | Default |
| --- | --- | --- |
| stdin | 0 | keyboard |
| stdout | 1 | screen |
| stderr | 2 | screen |

Errors go to a **separate stream** from normal output — which is why `command > file` still prints errors to your screen.

### Redirection

```bash
command > file            # stdout to file (OVERWRITES)
command >> file           # stdout to file (APPENDS)
command 2>/dev/null       # discard errors
command > out 2>&1        # both streams to one file
command < input.txt       # feed a file as stdin
```

### Pipes

```bash
command1 | command2       # stdout of 1 becomes stdin of 2
command | tee file        # display AND save simultaneously
```

### Globbing (wildcards) — done by the shell, not the command

```bash
ls *.txt          # * = any characters
ls file?.txt      # ? = exactly one character
ls file[a-z].txt  # range
```

Prove it's the shell doing the expansion, not `ls`:

```bash
echo *.txt
```

This matters because `&&`, `;`, `|`, and `$()` are exactly the characters that **command injection** vulnerabilities abuse — when a web app passes your input to a shell without sanitising it.

### Command chaining

```bash
echo "Today: $(date)"       # $() runs a command, inserts its output
cmd1 && cmd2      # run cmd2 only if cmd1 SUCCEEDED
cmd1 || cmd2      # run cmd2 only if cmd1 FAILED
```

---

## Module 14 — Finding Things

```bash
find / -name "flag.txt" 2>/dev/null       # by exact name
find / -iname "*config*" 2>/dev/null      # case-insensitive
find / -user root -writable 2>/dev/null   # root-owned and writable by me
find / -perm -4000 -type f 2>/dev/null    # SUID binaries — see Module 17
find / -mtime -1                           # modified in the last 24 hours
find . -name "*.log" -exec rm {} \;       # run a command on each result

locate passwd            # uses a prebuilt database
which nmap               # location of a binary in $PATH
```

> `2>/dev/null` on `find /` is not optional — without it, permission-denied errors from directories you can't read bury your actual results.

Post-compromise enumeration **is** `find` with the right flags: finding a world-writable script owned by root, an SSH key in an unexpected home directory, a config file with plaintext credentials.

---

# Part III — The Security-Critical Core

> Everything so far was learning to move around a system. Everything from here on is directly about security. If you remember one part of this guide, make it this one.

---

## Module 15 — Users, Groups & Identity

Every user has a **UID**. **UID 0 is root** — root is defined by the *number*, not the name. Any account with UID 0 is root. System accounts (UID \< 1000, typically) run services and usually have `/usr/sbin/nologin` as a shell, meaning **they cannot log in interactively.**

```bash
cat /etc/passwd    # world-readable: username:x:UID:GID:comment:home:shell
sudo cat /etc/shadow   # root-only: password HASHES
```

```
root:x:0:0:root:/root:/bin/bash
 │   │ │ │  │     │      └─ login shell (/usr/sbin/nologin = cannot log in)
 │   │ │ │  │     └──────── home directory
 │   │ │ │  └────────────── comment field
 │   │ │ └───────────────── GID
 │   │ └─────────────────── UID  ← 0 means ROOT
 │   └───────────────────── 'x' = password is in /etc/shadow
 └───────────────────────── username
```

> **Important: a line in `/etc/passwd` is not someone logged in.** It just means the account exists. Most entries are system accounts that have never had a human log into them, which is exactly what the `nologin` shell enforces. To see who's actually logged in *right now*, use `who`, `w`, or `whoami` — not `/etc/passwd`.

`/etc/shadow` hash format: `$6$salt$hash` — algorithm ($6 = SHA-512), a salt (defeats rainbow tables), then the hash. Passwords used to be hashed inside `/etc/passwd`, which every user can read — anyone could copy it and crack offline. The shadow file split moved hashes into a root-only file. This is why "can I read `/etc/shadow`?" is one of the first questions in privilege escalation.

### Commands

```bash
whoami           # current username
id               # UID, GID, all groups  ← memorise this
groups           # your groups
who / w          # who is logged in right now
sudo -l          # what can I run with sudo  ← critical
```

> **`id` and `sudo -l` are the first two commands to run on any shell you land on.** Membership in `sudo`, `wheel`, `docker`, `lxd`, `adm`, or `disk` are all effectively paths to root.

---

## Module 16 — Permissions

Permissions are the access-control model of the entire OS. Nearly every Linux privilege escalation is a permissions mistake.

### Reading `ls -l`

```
-rwxr-xr--  1  alice  devs  4096  Sep 17 10:30  script.sh
│└┬┘└┬┘└┬┘     └─┬─┘  └┬─┘
│ │  │  │        │     └──── group owner
│ │  │  └────────┼────────── OTHERS  (everyone else)
│ │  └───────────┼────────── GROUP   (members of 'devs')
│ └──────────────┴────────── OWNER   (alice)
└──────────────────────────── file type
```

| Bit | On a file | On a directory |
| --- | --- | --- |
| **r** (4) | Read contents | **List** the names inside |
| **w** (2) | Modify contents | **Create/delete/rename** files inside |
| **x** (1) | Execute as a program | **Enter/traverse** (`cd` into it) |

> **Two non-obvious facts worth testing yourself:**
> 
> 1. **Delete permission on a file comes from the directory, not the file.** You can delete a read-only file you don't own if you can write to its directory.
> 2. **`x` without `r` on a directory** means you can access `secret/file.txt` if you know the name, but can't list what's in `secret/`.

### Numeric (octal) notation

r=4, w=2, x=1. Add them per group.

| Octal | Symbolic | Meaning |
| --- | --- | --- |
| 777 | `rwxrwxrwx` | Everyone everything ⚠️ almost always wrong |
| 755 | `rwxr-xr-x` | Owner full, others read+execute — standard for programs/dirs |
| 700 | `rwx------` | Owner only — `~/.ssh` |
| 644 | `rw-r--r--` | Owner write, others read — standard for documents |
| 600 | `rw-------` | Owner only — **SSH private keys, credentials** |
| 666 | `rw-rw-rw-` | Everyone can write ⚠️ |

```bash
chmod 755 script.sh
chmod u+x script.sh         # add execute for user
chown alice:devs file       # change owner and group
```

### Security takeaways

- **`chmod 777` is a vulnerability**, not a fix — it means any user, including a compromised service account, can modify the file.
- **SSH enforces strict key permissions on purpose** — a `600` key refuses to be used if others can read it.
- **World-writable files are an enumeration goldmine:**

  ```bash
  find / -type f -perm -002 2>/dev/null      # world-writable files
  ```
- **Root bypasses permission checks entirely.** Permissions protect you from other users. Nothing protects you from yourself when you're root.

### Try it

```bash
mkdir ~/permlab && cd ~/permlab
echo 'echo "I ran!"' > test.sh
./test.sh              # Permission denied — no x bit yet
chmod +x test.sh
./test.sh              # works
chmod 000 test.sh
cat test.sh             # Permission denied — even though you OWN it
```

---

## Module 17 — SUID, SGID, Sticky Bit & Capabilities

### The three special bits

| Bit | Octal | Shows as | Effect |
| --- | --- | --- | --- |
| **SUID** | 4000 | `s` in owner's x | Program runs with the **file owner's** privileges |
| **SGID** | 2000 | `s` in group's x | Runs with the **group's** privileges |
| **Sticky** | 1000 | `t` in others' x | In a shared directory, only the file's owner can delete it |

### SUID, explained properly

```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
```

Changing your password means writing to `/etc/shadow`, which is root-only. But users must be able to change their own passwords. The resolution: `passwd` is owned by root with the SUID bit, so when you run it, it executes as root — long enough to do that one job.

> **Every SUID binary is a deliberate hole in the permission model.** If that program has a bug, or can be made to do something unintended — run a shell, read an arbitrary file — the attacker inherits root. This is why enumerating SUID binaries is step one of privilege escalation.

```bash
find / -perm -4000 -type f 2>/dev/null     # SUID binaries
```

Normal entries: `passwd`, `sudo`, `su`, `mount`, `ping`, `pkexec`. The skill is recognising what does **not** belong on that list. See **GTFOBins** (`gtfobins.github.io`) for a catalogue of standard binaries that become dangerous when misconfigured with SUID or sudo rights.

### Try the classic SUID escalation yourself (on your own VM only)

```bash
cp $(which find) ~/find_demo
sudo chown root:root ~/find_demo
sudo chmod u+s ~/find_demo
ls -l ~/find_demo        # confirm the 's' is there

~/find_demo . -exec /bin/sh \; -quit
whoami                    # should print root
exit
```

**If `whoami` still shows your normal user**, some shells (`sh`, sometimes `dash`) deliberately drop SUID privileges as a protection. Try:

```bash
~/find_demo . -exec /bin/bash -p \; -quit
```

The `-p` flag tells bash not to drop the privileges it was given.

**Clean up afterward — don't leave this lying around:**

```bash
sudo chmod u-s ~/find_demo
rm ~/find_demo
```

This works because `find` was never designed to be a privilege escalation tool — `-exec` is a completely legitimate feature. The vulnerability is entirely in *who was given permission to run it with elevated privileges*. The real system `find` (`ls -l $(which find)`) has no SUID bit — the fix isn't a patch, it's simply never granting SUID to a general-purpose tool.

### Sticky bit

```bash
ls -ld /tmp
# drwxrwxrwt  ← the t
```

`/tmp` is world-writable so any program can drop temp files there. The sticky bit stops one user from deleting another user's files in that shared space.

### Capabilities — a finer-grained alternative

```bash
getcap -r / 2>/dev/null
# e.g. /usr/bin/ping = cap_net_raw+ep
```

Modern Linux can grant fragments of root instead of all of it — `cap_net_raw` lets `ping` craft raw packets without being fully SUID root.

---

## Module 18 — sudo and Privilege Boundaries

- `su` — switch user entirely. `su -` becomes root (needs root's password).
- `sudo` — run **one command** as another user, using **your own** password, if policy allows.

**Why sudo is preferred:** accountability (every use is logged with who did what), granularity (grant one specific command, not a root shell), and revocability.

```bash
sudo command
sudo -l                      # what am I allowed to run — the highest-value command here
sudo visudo                  # safely edit /etc/sudoers
```

`/etc/sudoers` format:

```
alice   ALL=(ALL:ALL) ALL                 # alice can run anything
%sudo   ALL=(ALL:ALL) ALL                 # anyone in group 'sudo' can
bob     ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx
```

> **`sudo -l` is the highest-value single command in Linux privilege escalation.** If a user can run a text editor, a language interpreter, an archiver, or `find` as root, they can almost always get a root shell, because those programs can execute other programs by design. **Granting sudo on a general-purpose program is granting sudo on everything.** The defensive principle: grant sudo on narrowly-scoped actions, never on general-purpose programs.

> `sudo apt install` is fine. `curl https://random-site.com/install.sh | sudo bash` means you just executed unreviewed code from the internet as root. Download and read a script before piping it to a shell.

---

## Module 19 — Processes, Signals & Services

Every running program is a process with a **PID**. Every process has a parent (PPID). PID 1 (`systemd`) is the ancestor of everything. Processes run **as a user**, and that user's permissions define what the process can do.

```bash
ps aux                  # all processes, all users, detailed ← memorise this
ps -ef --forest          # tree view showing parent/child
top / htop                # live view (q to quit)
pgrep -a ssh              # find PIDs by name
```

**Reading `ps aux`:** `USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND`

> 🎯 **`ps aux` is enumeration.** It shows you what services are running, *which user each runs as* (what compromising it would get you), and sometimes **credentials passed as command-line arguments** — because command lines are visible to every user via `/proc/<pid>/cmdline`. Never put a password on a command line; it's visible system-wide.

### Signals and job control

```bash
kill PID             # SIGTERM — polite, "please exit"
kill -9 PID           # SIGKILL — forced, cannot be caught
command &             # run in background
jobs                   # list background jobs
Ctrl+Z                 # suspend current job
nohup command &         # keep running after logout
```

### Services with systemd

```bash
systemctl status ssh
systemctl start|stop|restart ssh
systemctl enable ssh         # start at boot
journalctl -u ssh -n 50      # logs for one service
journalctl -f                # follow all logs live
```

> systemd unit files are a favourite persistence mechanism — an attacker who can write a service file and enable it has access that restarts on every boot. `systemctl list-unit-files --state=enabled` is worth checking on any system you're investigating.

---

## Module 20 — Scheduled Tasks: cron & systemd timers

```bash
crontab -l               # my scheduled jobs
sudo crontab -l          # root's jobs
cat /etc/crontab         # system-wide
ls -la /etc/cron.d/
```

```
* * * * * command
│ │ │ │ └── day of week (0-7)
│ │ │ └──── month (1-12)
│ │ └────── day of month (1-31)
│ └──────── hour (0-23)
└────────── minute (0-59)
```

> **Cron is a canonical persistence mechanism** — a scheduled job re-establishes access on its own. It's also a privilege escalation path: if root runs a script on a schedule and that script (or its directory) is writable by you, you get root execution on the next tick.

```bash
ls -la /etc/cron.d/ && cat /etc/crontab
find / -writable -name "*.sh" 2>/dev/null
```

---

## Module 21 — Package Management & Software Supply Chain

### Debian/Ubuntu/Kali (apt)

```bash
sudo apt update               # refresh the list of available packages — always first
sudo apt upgrade              # actually install the newer versions
sudo apt install nmap
dpkg -L nmap                   # which files did this package install
dpkg -S /usr/bin/nmap          # which package owns this file
```

**`update` vs `upgrade`, clearly:** `apt update` just downloads the current catalog of what's available — it changes nothing on your system. `apt upgrade` reads that catalog and actually installs newer versions of what you have. Running `upgrade` without `update` first means you're acting on a potentially stale list. Think of `update` as refreshing the news feed, and `upgrade` as acting on it.

Other ecosystems: `dnf`/`yum` (Red Hat), `pacman` (Arch), `pip` (Python), `npm` (Node).

### The supply chain

Package managers verify signatures — `apt` checks packages are signed by a trusted repository key, which is why adding a random third-party repository is a real trust decision.

**Supply chain attacks are among the most impactful attacks in modern security:**

- **Typosquatting** — a malicious `pip` package one character off from a popular one
- **Dependency confusion** — a public package shadowing a private internal one
- **Compromised maintainer accounts** — a legitimate package updated with a backdoor
- **The xz/liblzma backdoor (2024)** — a multi-year social engineering campaign placed a backdoor into a compression library used by SSH on major distributions, caught by a developer investigating a small performance anomaly. Trust in open source is a *process*, not an assumption.

**Practical advice:** install from official repositories, read a script before piping it to a shell, and keep systems updated — most compromises exploit known, already-patched vulnerabilities.

---

# Part IV — Networking & Offensive Context

---

## Module 22 — Linux Networking Fundamentals

```bash
ip a                     # interfaces and IPs
ip r                      # routing table
ss -tulpn                 # listening ports — 🎯 the important one
ping -c 4 8.8.8.8          # reachability
dig example.com            # DNS lookup
curl -I https://example.com   # headers only — basic recon
python3 -m http.server 8000   # instant file-transfer web server
```

**Decoding `ss -tulpn`:** `-t` TCP, `-u` UDP, `-l` listening only, `-p` owning process, `-n` numeric. Every listening port is a door. Run it and ask of each result: does this machine actually need this service running? That question is the whole of attack surface reduction.

`cat /etc/hosts` overrides DNS locally — useful for pointing a hostname at your own machine for testing, and equally a place malware uses to silently redirect traffic.

---

## Module 23 — SSH — The Hacker's Front Door

SSH is how essentially every Linux server on the internet is administered — and therefore the most brute-forced service on the internet.

```bash
ssh user@192.168.1.10
ssh-keygen -t ed25519 -C "your@email"   # generate a key pair
ssh-copy-id user@host                     # install your public key on a server
```

The **private key never leaves your machine**. The server holds only the **public** key, in `~/.ssh/authorized_keys`. Authentication proves you hold the private key without ever transmitting it — strictly better than passwords.

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
```

> Finding a private key on a compromised host often means lateral movement to other systems, since keys get reused. An attacker appending their own key to `~/.ssh/authorized_keys` is a classic persistence technique — check that file during incident response.

---

## Module 24 — netcat & Basic Network Interaction

> Everything in this module happens between VMs you own, on a host-only network. Scanning or connecting to systems you don't have written authorisation to test is illegal — see Module 29.

```bash
# Banner grabbing
nc -v target.local 80
# then type:  HEAD / HTTP/1.0   and press Enter twice

# Between two of your own VMs
nc -lvnp 4444          # listen
nc 192.168.56.10 4444  # connect
```

Services announce themselves — `OpenSSH_8.9p1 Ubuntu`, `Apache/2.4.52`. That version string is the start of vulnerability research, looked up against a CVE database. netcat itself is a classic dual-use tool: a legitimate debugging utility and an attacker staple at once — its presence on a server isn't inherently malicious, context is everything.

---

## Module 25 — Bash Scripting for Automation

```bash
#!/bin/bash
echo "Hello, $USER"
```

```bash
chmod +x script.sh
./script.sh
```

**Variables and arguments**

```bash
name="World"
echo "Hello, $name"
$1    # first argument    $@  all arguments    $?  exit code of last command
```

Always quote `"$variables"` — unquoted variables containing spaces are the #1 source of bash bugs.

**Conditionals**

```bash
if [ -f "$1" ]; then
    echo "$1 is a file"
else
    echo "Unknown"
fi
```

**Loops**

```bash
for file in *.txt; do
    echo "Processing $file"
done
```

### Practice script — try building this yourself

```bash
#!/bin/bash
# sysinfo.sh — basic system enumeration
echo "User:     $(whoami)"
echo "UID/GID:  $(id -u)/$(id -g)"
echo "Groups:   $(groups)"
echo "Hostname: $(hostname)"
echo "Kernel:   $(uname -r)"
ip -4 a | grep inet | awk '{print "    " $2, $NF}'
echo "Listening ports:"
ss -tuln | tail -n +2 | awk '{print "    " $1, $5}'
echo "Sudo privileges:"
sudo -n -l 2>/dev/null | tail -n +2 | sed 's/^/    /' || echo "    (password required)"
```

This single script uses almost every concept from this guide: variables, command substitution, pipes, `awk`, identity commands, networking commands. Walk through it line by line and see how the pieces reassemble.

```bash
set -euo pipefail    # exit on error, on undefined variable, and on pipe failure
```

---

## Module 26 — Logs, History & Forensic Footprints

| File | Contains |
| --- | --- |
| `/var/log/syslog` / `/var/log/messages` | General system messages |
| **`/var/log/auth.log`** / `/var/log/secure` | **Authentication, sudo, SSH** |
| `/var/log/apache2/access.log` | Web requests |

```bash
tail -f /var/log/syslog
grep "Failed password" /var/log/auth.log | wc -l
journalctl --since "1 hour ago"
last -20                          # recent logins
```

```bash
history                # your command history
cat ~/.bash_history
```

> **Shell history is a credential leak.** People type passwords into command lines. `mysql -u root -pMyPassword123` lands in `.bash_history` in plaintext forever. On any host, checking every user's `.bash_history` is a five-second check with a high hit rate — which is exactly why you should never put a secret on a command line yourself.

> **In an authorised penetration test, you generally do not destroy logs.** Your client needs them, and log tampering can exceed your rules of engagement. Professional testers document their footprint rather than erase it. Mature organisations also ship logs off-host in real time precisely because an attacker with root can alter anything on that host — evidence shouldn't live only there.

---

## Module 27 — Mapping Linux Skills to the Attack Lifecycle

| Phase | What you're doing | Linux skills from this guide |
| --- | --- | --- |
| **Reconnaissance** | Gathering information | `dig`, `curl -I`, `grep` |
| **Scanning & Enumeration** | Finding hosts, ports, services | `ping`, `ss`, `nc` banner grabbing |
| **Local Enumeration** | "What did I just land on?" | `id`, `sudo -l`, `uname -a`, `ps aux`, `ss -tulpn`, `find` |
| **Privilege Escalation** | Low-privilege user → root | SUID/SGID, sudo misconfiguration, writable cron scripts |
| **Persistence** | Maintaining access | cron, systemd units, `authorized_keys` |
| **Lateral Movement** | Reaching other systems | SSH keys, `.bash_history`, config credentials |
| **Exfiltration** | Moving data out | `scp`, `curl`, `python3 -m http.server` |

None of these are "hacking commands" — they're ordinary system administration commands. **Offensive security is applied systems knowledge.** The people who are good at this are good at understanding how systems work; the tools are secondary.

### The privilege escalation checklist — ten commands worth memorising

```bash
id; whoami; groups
sudo -l
uname -a; cat /proc/version
find / -perm -4000 -type f 2>/dev/null
cat /etc/crontab; ls -la /etc/cron.d/
ps aux | grep root
ss -tulpn
find / -writable -type d 2>/dev/null | grep -v proc
cat ~/.bash_history
ls -la /home/*/.ssh/ 2>/dev/null
```

Automated tools like LinPEAS exist, but run this checklist manually first — running an automated tool before you understand what it's checking means you never actually learn to read its output.

---

## Module 28 — The Blue Side: Hardening Basics

Learning only offense without defense is only half the job — and arguably the less useful half.

| Attack concept | Defence |
| --- | --- |
| Weak permissions | Least privilege; audit world-writable files; sane `umask` |
| SUID abuse | Minimise SUID binaries; use capabilities instead; audit regularly |
| sudo misconfiguration | Narrow, specific sudo rules; never on interpreters/editors |
| Cron privesc | Root-owned, non-writable scripts; audit `/etc/cron.d` |
| Unpatched software | Automated security updates; subscribe to advisories |
| Exposed services | Close unused ports; firewall default-deny |
| SSH brute force | Key-only auth; no root login; fail2ban |
| Credential leaks | Never secrets on command lines; secret managers |
| Log tampering | Centralised off-host logging; file integrity monitoring |
| Persistence | Baseline enabled services & cron; monitor `authorized_keys` |

**Worth knowing by name:** **AppArmor / SELinux** (mandatory access control, confining even root-equivalent processes), **fail2ban** (bans IPs after repeated login failures), **auditd** (kernel-level audit logging), **CIS Benchmarks** (the industry-standard hardening guides organisations audit against).

---

## Module 29 — Ethics, Law & Responsible Disclosure

### The core rule

> **Authorisation is the only thing that separates a security professional from a criminal.** Not intent. Not skill. Not whether you caused damage. **Written authorisation, defined scope, before you touch anything.**
> 
> The exact same command — `nmap -sV 10.0.0.5` — is a billable professional service against a client who hired you and a criminal offence against a system you found interesting. The keystrokes are identical. The authorisation is the only difference.

Unauthorised access to computer systems is criminal in essentially every jurisdiction — relevant frameworks include the **Computer Fraud and Abuse Act** (US), the **Computer Misuse Act** (UK), and the **Budapest Convention on Cybercrime**. **Look up your own country's computer crime law and read it.** "I was just learning" and "I didn't cause any damage" are not legal defences. Scanning itself is often an offence, independent of whether you got in.

### Where to practise legally

- **TryHackMe** — guided, beginner-friendly, structured learning paths
- **Hack The Box** / **HTB Academy** — realistic machines, steeper curve
- **OverTheWire (Bandit)** — a pure Linux/SSH wargame, the perfect next step after this workshop
- **PicoCTF** — beginner-friendly CTF
- **VulnHub** — downloadable vulnerable VMs for your own lab
- **PortSwigger Web Security Academy** — free, excellent, for web security
- Your own VMs, on a host-only network you control
- **Legitimate bug bounty programmes** — HackerOne, Bugcrowd, Intigriti. Read the scope. Obey it exactly.

**Never:** your university's network or systems, your employer's systems, any website you don't own, "just to test." Targeting your own university network specifically can end your club and your enrolment.

### Responsible disclosure — if you find a real vulnerability by accident

1. **Stop.** Don't explore further, don't access data, don't prove it harder than necessary.
2. **Document** minimally and factually.
3. **Report privately** to the vendor: `security@domain`, `/.well-known/security.txt`, or a bug bounty programme.
4. **Give reasonable time** to fix — 90 days is the widely accepted norm.
5. **Do not** publish, sell, share, or use the finding.
6. **Do not** demand payment for silence — that is extortion, not disclosure.

### One closing thought

Skills are neutral. The community you join is not. Security professionals are trusted with extraordinary access, and that trust is the entire product — built slowly over a career, and destroyed permanently by one bad decision made out of curiosity. Learn offense. Practise it in a lab. Apply it with permission. Always.

---

# Part V — Practice

## Hands-On Labs

### Lab 1 — Navigation

```bash
mkdir -p ~/lab/{docs,scripts,logs}
cd ~/lab/docs
touch report.txt notes.md secret.txt
cd ../scripts && touch run.sh
cd ~/lab && ls -laR
```

Tasks: find your absolute path from wherever you are; return home in two different ways; copy `report.txt` into `logs/`; rename it to `old_report.txt`.

### Lab 2 — Text processing

Using a sample `access.log` and `/etc/passwd`:

1. How many total requests are in the log?
2. How many unique IP addresses?
3. Which IP made the most requests?
4. How many 404 responses?
5. List every user in `/etc/passwd` with a real login shell (not `nologin`/`false`).

Answers

```bash
wc -l access.log
awk '{print $1}' access.log | sort -u | wc -l
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -1
grep -c " 404 " access.log
grep -vE "nologin|/false" /etc/passwd | cut -d: -f1
```

### Lab 3 — Permissions

```bash
mkdir ~/permlab && cd ~/permlab
echo "secret data" > confidential.txt
echo 'echo "executed"' > script.sh
```

Tasks:

1. Make `confidential.txt` readable only by you.
2. Make `script.sh` executable and run it.
3. Set a directory so others can enter but not list its contents.
4. Find all SUID binaries on the system.
5. Explain in one sentence why `/usr/bin/passwd` has an `s` in its permissions.

### Lab 4 — Enumeration

Answer these about your own VM:

1. What is your exact kernel version?
2. How many users have UID >= 1000?
3. What services are listening, and on which ports?
4. What can you run with sudo?
5. Are there any cron jobs configured?
6. Which process is using the most memory?

### Lab 5 — Scripting

Build `sysinfo.sh` from Module 25, then extend it with one section of your own choosing.

---

## Mini CTF

Six flags, escalating difficulty, format `flag{...}`:

| # | Skill tested |
| --- | --- |
| 1 | Hidden files — a flag inside a dotfile |
| 2 | grep — buried in a large log file |
| 3 | Permissions — file is `chmod 000`, fix it to read it |
| 4 | find — oddly-named file under `/opt` |
| 5 | Pipes — the flag is the most frequent line in a file (`sort \| uniq -c \| sort -rn`) |
| 6 | Encoding — base64 in a file, `base64 -d` to decode |

---

## Cheat Sheet

```
═══ NAVIGATION ═══════════════════════
pwd              where am I
ls -la           list all, detailed
cd /path         change directory
cd ~  cd ..  cd -    home, up, previous

═══ FILES ════════════════════════════
touch f          create empty file
mkdir -p a/b     create nested dirs
cp -r s d        copy (recursive)
mv s d           move / rename
rm -rf d         delete ⚠️ NO UNDO
cat / less / head -20 / tail -f    read
file f           what type is it
find / -name "x" 2>/dev/null

═══ TEXT ═════════════════════════════
grep -i "x" f        search
grep -rn "x" dir/    recursive + lines
grep -v "x" f        invert match
cut -d: -f1 f        field extraction
sort | uniq -c       count duplicates
sed 's/a/b/g' f      substitute
awk '{print $1}' f   print column 1
wc -l f              count lines

═══ PIPES & REDIRECT ═════════════════
cmd > f          stdout to file
cmd >> f         append
cmd 2>/dev/null  discard errors
cmd1 | cmd2      pipe
cmd | tee f      display AND save
$(cmd)           command substitution

═══ USERS & PERMISSIONS ══════════════
id               who am I, what groups
sudo -l          what can I run as root
chmod 755 f      rwxr-xr-x
chmod +x f       make executable
chown u:g f      change ownership
/etc/passwd      users (world-readable)
/etc/shadow      hashes (root only)
find / -perm -4000 2>/dev/null   SUID

  r=4  w=2  x=1
  755 = rwxr-xr-x   600 = rw-------
  644 = rw-r--r--   700 = rwx------

═══ PROCESSES ════════════════════════
ps aux           all processes
ps aux | grep x  find one
top / htop       live view
kill -9 PID      force kill
systemctl status|start|stop|enable svc
journalctl -u svc -n 50

═══ NETWORK ══════════════════════════
ip a             my IP addresses
ip r             routing table
ss -tulpn        listening ports
ping -c 4 host   reachability
dig host         DNS lookup
curl -I url      HTTP headers
ssh user@host    remote shell
scp f user@h:/p  copy over SSH
python3 -m http.server 8000   quick server

═══ PACKAGES ═════════════════════════
sudo apt update && sudo apt upgrade
sudo apt install pkg
dpkg -L pkg      files in a package

═══ SHORTCUTS ════════════════════════
Tab    AUTOCOMPLETE — use constantly
Ctrl+C interrupt      Ctrl+L clear
Ctrl+R search history Ctrl+D logout
Ctrl+A/E line start/end

═══ HELP ═════════════════════════════
man cmd    cmd --help    apropos word
tldr cmd   which cmd     type cmd
```

---

## Homework & Learning Path

### Right after this workshop

1. **OverTheWire — Bandit, levels 0–15.** The single best next step — a free, self-paced Linux/SSH wargame that directly reinforces everything in this guide.
2. Spend one week using the terminal instead of your file manager for all file operations.
3. Write one bash script that solves a small personal annoyance.

### Next 1–3 months

- **TryHackMe** — Pre-Security path, then Complete Beginner path
- **Linux Journey** (linuxjourney.com) — free, structured
- Build a home lab: Kali VM + Metasploitable 2 on a host-only network
- **PicoCTF** — work through prior-year challenges

### Certifications (a checkpoint, not a shortcut)

- **Linux:** LPIC-1, CompTIA Linux+
- **Security entry:** CompTIA Security+, eJPT
- **Offensive:** OSCP — a serious, 18-month-ish goal, not a beginner target
- **Defensive:** BTL1, CySA+

> Hands-on capability, a GitHub with your own scripts, and CTF experience matter more for a first role than any certificate.

---

## Resources

**Official documentation**

- The Linux Documentation Project — tldp.org
- Arch Wiki — wiki.archlinux.org (useful regardless of distro)
- Kali Linux Docs — kali.org/docs

**Books**

- *The Linux Command Line* — William Shotts (free PDF, the best starting book)
- *How Linux Works* — Brian Ward
- *Linux Basics for Hackers* — OccupyTheWeb

**Practice platforms**

- OverTheWire (Bandit) — overthewire.org/wargames/bandit
- TryHackMe — tryhackme.com
- Hack The Box — hackthebox.com
- PicoCTF — picoctf.org
- PortSwigger Web Security Academy — portswigger.net/web-security
- VulnHub — vulnhub.com

**Reference**

- GTFOBins — gtfobins.github.io
- explainshell.com — paste any command, get every flag explained
- cheat.sh — `curl cheat.sh/find`

**Standards & frameworks**

- MITRE ATT&CK — attack.mitre.org
- OWASP Top 10 — owasp.org
- CIS Benchmarks — cisecurity.org

---

*This guide is a living document — practice the commands, break things on your own VM, and ask questions in the club channel. Good luck, and welcome to Linux.*