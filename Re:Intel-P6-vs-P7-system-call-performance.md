          
# Re: Intel P6 vs P7 system call performance

\[Posted December 18, 2002 by corbet\]


- From:	 	Linus Torvalds <torvalds@transmeta.com>
- To:	 	"H. Peter Anvin" <hpa@transmeta.com>
- Subject:	 	Re: Intel P6 vs P7 system call performance
- Date:	 	Tue, 17 Dec 2002 22:38:09 -0800 (PST)
- Cc:	 	Ulrich Drepper <drepper@redhat.com>, Matti Aarnio <matti.aarnio@zmailer.org>, Hugh Dickins <hugh@veritas.com>, Dave Jones <davej@codemonkey.org.uk>, Ingo Molnar <mingo@elte.hu>, <linux-kernel@vger.kernel.org>

```
On Tue, 17 Dec 2002, Linus Torvalds wrote:
>
> Which is ok for a regular fast system call (ebp will get restored
> immediately), but it is NOT ok for the system call restart case, since in
> that case we want %ebp to contain the old stack pointer, not the sixth
> argument.
I came up with an absolutely wonderfully _disgusting_ solution for this.

The thing to realize on how to solve this is that since "sysenter" loses
track of EIP, there's really no real reason to try to return directly
after the "sysenter" instruction anyway. The return point is really
totally arbitrary, after all.

Now, couple this with the fact that system call restarting will always
just subtract two from the "return point" aka saved EIP value (that's the
size of an "int 0x80" instruction), and what you can do is to make the
kernel point the sysexit return point not at just past the "sysenter", but
instead make it point to just past a totally unrelated 2-byte jump
instruction.

With that in mind, I made the sysentry trampoline look like this:

        static const char sysent\[\] = {
                0x51,                   /\* push %ecx \*/
                0x52,                   /\* push %edx \*/
                0x55,                   /\* push %ebp \*/
                0x89, 0xe5,             /\* movl %esp,%ebp \*/
                0x0f, 0x34,             /\* sysenter \*/
        /\* System call restart point is here! (SYSENTER\_RETURN - 2) \*/
                0xeb, 0xfa,             /\* jmp to "movl %esp,%ebp" \*/
        /\* System call normal return point is here! (SYSENTER\_RETURN in entry.S) \*/
                0x5d,                   /\* pop %ebp \*/
                0x5a,                   /\* pop %edx \*/
                0x59,                   /\* pop %ecx \*/
                0xc3                    /\* ret \*/
        };

which does the right thing for a "restarted" system call (ie when it
restarts, it won't re-do just the sysenter instruction, it will really
restart at the backwards jump, and thus re-start the "movl %esp,%ebp"
too).

Which means that now the kernel can happily trash %ebp as part of the
sixth argument setup, since system call restarting will re-initialize it
to point to the user-level stack that we need in %ebp because otherwise it
gets totally lost.

I'm a disgusting pig, and proud of it to boot.

			Linus

-
To unsubscribe from this list: send the line "unsubscribe linux-kernel" in
the body of a message to majordomo@vger.kernel.org
More majordomo info at  [http://vger.kernel.org/majordomo-info.html](http://vger.kernel.org/majordomo-info.html)
Please read the FAQ at  [http://www.tux.org/lkml/](http://www.tux.org/lkml/)
```

### Disassembly of __kernel_vsyscall in linux-gate.so.1
```
(gdb) disas __kernel_vsyscall
Dump of assembler code for function __kernel_vsyscall:
   0xf7fdc420 <+0>:    push   %ecx
   0xf7fdc421 <+1>:    push   %edx
   0xf7fdc422 <+2>:    push   %ebp
   0xf7fdc423 <+3>:    mov    %esp,%ebp
   0xf7fdc425 <+5>:    sysenter
   0xf7fdc427 <+7>:    nop
   0xf7fdc428 <+8>:    nop
   0xf7fdc429 <+9>:    nop
   0xf7fdc42a <+10>:    nop
   0xf7fdc42b <+11>:    nop
   0xf7fdc42c <+12>:    nop
   0xf7fdc42d <+13>:    nop
   0xf7fdc42e <+14>:    int    $0x80
=> 0xf7fdc430 <+16>:    pop    %ebp
   0xf7fdc431 <+17>:    pop    %edx
   0xf7fdc432 <+18>:    pop    %ecx
   0xf7fdc433 <+19>:    ret   
End of assembler dump.
(gdb)
```
