### Overview
유저에서 프로세스 관리, device 제어 등 기능을 수행할 때, System call을 사용하여 커널로 요청을 보낸다.
[[Kernel vs. User]]

이때 `Trap`이라는 software Interrupt을 걸어 진행중인 flow를 커널 mode로 변경한다.
kernel에서 작업이 끝나면 레지스터를 통해 return을 반환하고 user mode로 변경된다.

Pintos를 기준으로 system call을 알아본다.
### User Invoke
print와 같이 stdout으로 출력 또는 stdin 입력은 standard library 내부에서  `write` system call wrapper로 구현되어 있다.

해당 system call은 실제로 다음과 같은 어셈블리 코드로 구현된 매크로로 구현되어있다.

```C
#define syscall0(NUMBER) \

({ \

	int retval; \
	
	asm volatile \
	
		("pushl %[number]; int $0x30; addl $4, %%esp" \
		
			: "=a" (retval) \
			
			: [number] "i" (NUMBER) \
			
			: "memory"); \
	
	retval; \

})
```

1. Push system call number
2. Set parameter
3. Interrupt 0x30
4. Pop stack (revert stack pointer)

같은 매크로로 여러의 system call을 구분하기 위해 Number를 stack에 push하여 커널에서 읽도록 한다.

해당 예제는 parameter가 없는 매크로로, 아키텍처에 따라 calling convention에 따라 stack에 push 혹은 레지스터에 push하도록 한다. 

이후 interrupt을 발생시키는데, 커널 Interrupt handler에서 0x30 index에 해당하는 system call interrupt임을 나타낸다.

커널에서 작업이 끝나면 다시 해당 asm 코드로 flow가 넘어와 stack을 원래대로 돌려놓는다.

### Kernel Process
user에서 interrupt를 걸면 해당 index에 해당하는 handler가 호출된다.

```C
void exception_init(void)
{
   /* These exceptions can be raised explicitly by a user program,
      e.g. via the INT, INT3, INTO, and BOUND instructions.  Thus,
      we set DPL==3, meaning that user programs are allowed to
      invoke them via these instructions. */
   intr_register_int(3, 3, INTR_ON, kill, "#BP Breakpoint Exception");
   intr_register_int(4, 3, INTR_ON, kill, "#OF Overflow Exception");
   intr_register_int(5, 3, INTR_ON, kill,
                     "#BR BOUND Range Exceeded Exception");

   /* These exceptions have DPL==0, preventing user processes from
      invoking them via the INT instruction.  They can still be
      caused indirectly, e.g. #DE can be caused by dividing by
      0.  */
   intr_register_int(0, 0, INTR_ON, kill, "#DE Divide Error");
   intr_register_int(1, 0, INTR_ON, kill, "#DB Debug Exception");
   intr_register_int(6, 0, INTR_ON, kill, "#UD Invalid Opcode Exception");
   intr_register_int(7, 0, INTR_ON, kill,
                     "#NM Device Not Available Exception");
   intr_register_int(11, 0, INTR_ON, kill, "#NP Segment Not Present");
   intr_register_int(12, 0, INTR_ON, kill, "#SS Stack Fault Exception");
   intr_register_int(13, 0, INTR_ON, kill, "#GP General Protection Exception");
   intr_register_int(16, 0, INTR_ON, kill, "#MF x87 FPU Floating-Point Error");
   intr_register_int(19, 0, INTR_ON, kill,
                     "#XF SIMD Floating-Point Exception");

   /* Most exceptions can be handled with interrupts turned on.
      We need to disable interrupts for page faults because the
      fault address is stored in CR2 and needs to be preserved. */
   intr_register_int(14, 0, INTR_OFF, page_fault, "#PF Page-Fault Exception");
}

void syscall_init(void)
{
  intr_register_int(0x30, 3, INTR_ON, syscall_handler, "syscall");
}

```

커널 시작시에 위와 같은 interrupt과 handler가 등록된다. 이후 exception과 같은 예외상황에서 커널의 작업들이 수행되도록 한다. ex) page fault

user에서 `int $0x30` 을 호출하면 해당 sysca