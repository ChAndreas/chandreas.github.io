---
layout: post
title:  "Kernel Tracing with Ftrace"
date:   Wed Dec 06 17:15:51 EEST 2017
---

Ftrace is a tracing utility in the the Linux kernel, designed to help out developers of the system to find what is going on inside the kernel.

Ftrace was developed by Steven Rostedt and has been included in the kernel since version 2.6.27.

It can be used for debugging or analyzing latencies and performance issues that take place outside of user-space. 

Ftrace uses the debugfs file system to hold the control files as well as the files to display output. 

To mount this directory, you can add to your /etc/fstab file:

	tracefs       /sys/kernel/debug/tracing       tracefs defaults        0       0

Or you can mount it at run time with:

	mount -t tracefs nodev /sys/kernel/debug/tracing

	cd /sys/kernel/debug/tracing

Some of the key files are
	
	ls /sys/kernel/debug/tracing

	available_tracers – available tracing programs
	tracing_on - 0 into this file to disable the tracer or 1 to enable it
	trace - holds the output of the trace
	current_tracer – display the current tracer that is configured

Current tracers that may be configured
	
	cat /sys/kernel/debug/tracing/available_tracers

	function - Function call tracer to trace all kernel functions.
	function_graph - Similar to the function tracer except that the function tracer probes the functions on their entry
		     whereas the function graph tracer traces on both entry and exit of the functions. It then provides the ability to draw a graph of function calls similar to C code source.
	blk -  The block tracer I/O. The tracer used by the blktrace user application.
	hwlat - The Hardware Latency tracer is used to detect if the hardware produces any latency.
	irqsoff - Traces the areas that disable interrupts and saves the trace with the longest max latency.
	preemptoff - Similar to irqsoff but traces and records the amount of time for which preemption is disabled.
	preemptirqsoff - Similar to irqsoff and preemptoff, but traces and records the largest time for which irqs and/or preemption is disabled.
	wakeup - Traces and records the max latency that it takes for the highest priority task to get scheduled after it has been woken up.
	wakeup_rt - Traces and records the max latency that it takes for just RT tasks (as the current "wakeup" does).
	wakeup_dl - Traces and records the max latency that it takes for a SCHED_DEADLINE task to be woken (as the "wakeup" and "wakeup_rt" does).
	mmiotrace - A special tracer that is used to trace binary module. It will trace all the calls that a module makes to the hardware. Everything it writes and reads from the I/O as well.
	branch - This tracer can be configured when tracing likely/unlikely calls within the kernel. It will trace when a likely and unlikely branch is hit and if it was correct in its prediction of being correct.
	nop - This is the "trace nothing" tracer. A way to disable other tracers and can be use with events

Example how to run ftrace with function tracer

	sysctl kernel.ftrace_enabled=1
	echo function > /sys/kernel/debug/tracing/current_tracer
	echo 1 > /sys/kernel/debug/tracing/tracing_on
	sleep 1
	echo 0 > /sys/kernel/debug/tracing/tracing_on
	cat /sys/kernel/debug/tracing/trace


Limiting the traces with setting up the filters

	set_ftrace_filter - enabling or disabling specific functions to be traced
	set_ftrace_notrace -  Any function that is added here will not be traced
	set_ftrace-pid - only traces functions executed by a task with the given pid
	set_graph_function - Functions listed in this file will cause the function graph tracer to only trace these functions and the functions that they call.
	set_graph_notrace - Disable function graph tracing when the function is hit until it exits the function.

Function name filter is very limited show only when enter or exit the function

	echo schedule > set_ftrace_filter
	
Appending the filter
	
	echo '*rcu*' >> set_ftrace_filter
		
Clear the filter
	
	echo > set_ftrace_filter

Example filter syscalls

	echo SyS_read > set_ftrace_filter

Function stack trace with func_stack_trace option
	
	echo schedule > set_ftrace_filter
	echo 1 > /sys/kernel/debug/tracing/options/func_stack_trace
	echo function > /sys/kernel/debug/tracing/current_tracer
	echo 0 > /sys/kernel/debug/tracing/tracing_on
	ech0 0 > /sys/kernel/debug/tracing/options/func_stack_trace
	 
Tracing Events

The trace event directory. It holds event tracepoints (also known as static tracepoints) that have been compiled into the kernel.
Tracepoints placed in code provides a hook to call a function (probe) that you can provide at runtime can be used
without creating custom kernel modules to register probe functions using the event tracing infrastructure.
	
You can find all events on 
	
	cat /sys/kernel/debug/tracing/available_events
	
Example with events 
	
	echo 0 > /sys/kernel/debug/tracing/tracing_on
	echo 1 > /sys/kernel/debug/tracing/events/syscalls/enable
	echo 1 > /sys/kernel/debug/tracing/events/exceptions/enable
	sh -c 'echo $$ > /sys/kernel/debug/tracing/set_event_pid; echo 1 > /sys/kernel/debug/tracing/tracing_on; exec echo ftrace'
	cat /sys/kernel/debug/tracing/trace
		
set_event_pid Events only trace a task with a PID listed in this file
	
	echo 0 > /sys/kernel/debug/tracing/tracing_on
        echo 1 > /sys/kernel/debug/tracing/events/syscalls/enable
        echo 1 > /sys/kernel/debug/tracing/events/exceptions/enable		
	echo $$ > /sys/kernel/debug/tracing/set_event_pid 
	echo 1 > /sys/kernel/debug/tracing/tracing_on; 
	echo ftrace_events;
	echo 0 > /sys/kernel/debug/tracing/tracing_on;

The new interface trace-cmd 

Change tracer to function
	
	trace-cmd start -p function

See available filters

	trace-cmd list -f

Set ftrace filter

	trace-cmd start -p function -l schedule

Set ftrace pid

	trace-cmd record -p function -F echo ftrace

Syscall tracing

	trace-cmd -p function -g SyS_read

Ftrace no trace (set_ftrace_notrace) 

	trace -p nop -n schedule

Ftrace function stack trace ( options/func_stack_trace )
	
	trace-cmd start -p function -l schedule --func-stack

Ftrace Events list
	
	trace-cmd list -e
	
Set Event 
	
	trace-cmd start -p -e sched

Set Event pid
	
	trace-cmd record -e syscalls -e exceptions -F echo  ftrace	
	
Show output
	
	trace-cmd show
	trace-cmd report

source 

	https://www.kernel.org/doc/Documentation/trace/ftrace.txt	
	https://www.kernel.org/doc/Documentation/trace/events.txt
	https://www.kernel.org/doc/Documentation/trace/tracepoints.txt
	https://www.kernel.org/doc/Documentation/trace/ftrace-design.txt
	https://kernel-recipes.org/en/2017/talks/understanding-the-linux-kernel-via-ftrace/
	
