# CS354

## Three basic rules about interrupt processing  
1. Interrupt code must not leave interrupts disabled for too long. Length of time interrupt can be delayed depends on attached devices 
2. Interrupt handler must never call a function that will move the executing process out of current or ready 
3. Interrupt handler must use special function to return, must never enable interrupts explicitly 

12.1 Suppose an interrupt handler contains an error that explicitly enables interrupts. Describe how the system might fail. 
Ans - While executing handler code - interrupt generated - jump to that handler - control may never come back to original handler & original state may be corrupted or overwritten by the nested handlers

(only one interrupt in progress at a time per process)

12.4 Imagine a processor where the hardware automatically switches context
to a special “interrupt process” whenever an interrupt occurs. The only
purpose of the interrupt process is to run interrupt code. Does such a
design make an operating system easier or more difficult to design? Explain. Hint: will the interrupted process be permitted to reschedule?
Ans - It's more difficult. The code for handlers are usually set during initialization of the system. Complex to determine where to return after completion of process. If resched called, will go back to ready queue, the executing process may call interrupt which will run the same interrupt process - how will you keep track of where you left off for each process. Creates issues with ready queue too. 

