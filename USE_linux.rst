CPU
  utilization
    system-wide:
      `vmstat 1`, "us" + "sy" + "st"
      `sar -u`, sum fields except "%idle" and "%iowait"
      `dstat -c`, sum fields except "idl" and "wai"
    per-cpu:
      `mpstat -P ALL 1`, sum fields except "%idle" and "%iowait"
      `sar -P ALL`, same as `mpstat`
    per-process:
      `top`, "%CPU"
      `htop`, "CPU%"
      `ps -o pcpu`
      `pidstat 1`, "%CPU"
    per-kernel-thread:
      `top`/`htop` ("K" to toggle), where VIRT == 0 (heuristic). [1]_
  saturation
    system-wide:
      `vmstat 1`, "r" > CPU count [2]_
      `sar -q`, "runq-sz" > CPU count
      `dstat -p`, "run" > CPU count 
    per-process:
      `/proc/PID/schedstat` 2nd field (sched_info.run_delay)
      `perf sched latency` (shows "Average" and "Maximum" delay per-schedule)
      dynamic tracing, eg, SystemTap schedtimes.stp "queued(us)" [3]_
  errors
    	`perf` (LPE) if processor specific error events (CPC) are available, eg AMD64's "04Ah Single-bit ECC Errors Recorded by Scrubber" [4]_

Memory capacity
  utilization
    system-wide:
      `free -m`, "Mem:" (main memory), "Swap:" (virtual memory)
      `vmstat 1`, "free" (main memory), "swap" (virtual memory)
      `sar -r`, "%memused"
      `dstat -m`, "free"
      `slabtop -s c` for kmem slab usage
    per-process:
    	`top`/`htop`, "RES" (resident main memory), "VIRT" (virtual memory), "Mem" for system-wide summary
  saturation
    system-wide:
      `vmstat 1`, "si"/"so" (swapping)
      `sar -B`, "pgscank" + "pgscand" (scanning)
      `sar -W`
    per-process:
    	10th field (min_flt) from /proc/PID/stat for minor-fault rate, or dynamic tracing [5]_
    	OOM killer: `dmesg | grep killed`
  errors
    	`dmesg` for physical failures; dynamic tracing, eg, SystemTap uprobes for failed malloc()s

Network Interfaces
  utilization
    `sar -n DEV 1`, "rxKB/s"/max "txKB/s"/max
    `ip -s link`, RX/TX tput / max bandwidth
    `/proc/net/dev`, "bytes" RX/TX tput/max
    nicstat "%Util" [6]_
  saturation
    `ifconfig`, "overruns", "dropped"
    `netstat -s`, "segments retransmited"
    `sar -n EDEV`, *drop and *fifo metrics
    `/proc/net/dev`, RX/TX "drop"
    nicstat "Sat" [6]_
    dynamic tracing for other TCP/IP stack queueing [7]_
  errors
	  `ifconfig`, "errors", "dropped"
	  `netstat -i`, "RX-ERR"/"TX-ERR"
	  `ip -s link`, "errors"
	  `sar -n EDEV`, "rxerr/s" "txerr/s"
	  `/proc/net/dev`, "errs", "drop"
	  extra counters may be under /sys/class/net/...
	  dynamic tracing of driver function returns 76

Storage device I/O
  utilization
    system-wide:
    	`iostat -xz 1`, "%util"
    	`sar -d`, "%util" 
  	per-process:
  		iotop
  		`pidstat -d`
  		`/proc/PID/sched` "se.statistics.iowait_sum"
  saturation
    `iostat -xnz 1`, "avgqu-sz" > 1, or high "await"
    `sar -d` same
    LPE block probes for queue length/latency
    dynamic/static tracing of I/O subsystem (incl. LPE block probes)
  errors
    `/sys/devices/.../ioerr_cnt`
    `smartctl`
    dynamic/static tracing of I/O subsystem response codes [8]_

Storage capacity
  utilization
    swap:
    	`swapon -s`
    	`free`
    	`/proc/meminfo` "SwapFree"/"SwapTotal"
  	file systems:
  		`df -h`
  saturation
    	not sure this one makes sense - once it's full, ENOSPC
  errors
	    `strace` for ENOSPC
	    dynamic tracing for ENOSPC
	    `/var/log/messages` errs, depending on FS

Storage controller
  utilization
    `iostat -xz 1`, sum devices and compare to known IOPS/tput limits per-card
  saturation
    see storage device saturation, ...
  errors
    see storage device errors, ...

Network controller
  utilization
    infer from `ip -s link` (or /proc/net/dev) and known controller max tput for its interfaces
  saturation
    see network interface saturation, ...
  errors
    see network interface errors, ...

CPU interconnect
  utilization
    LPE (CPC) for CPU interconnect ports, tput / max
  saturation
    LPE (CPC) for stall cycles
  errors
    LPE (CPC) for whatever is available

Memory interconnect
  utilization
    LPE (CPC) for memory busses, tput / max; or CPI greater than, say, 5; CPC may also have local vs remote counters
  saturation
    LPE (CPC) for stall cycles
  errors
    LPE (CPC) for whatever is available

I/O interconnect
  utilization
    LPE (CPC) for tput / max if available; inference via known tput from iostat/ip/...
  saturation
    LPE (CPC) for stall cycles
  errors
    LPE (CPC) for whatever is available

  .. [1] There can be some oddities with the %CPU from top/htop in virtualized environments; I'll update with details later when I can.
  CPU utilization: a single hot CPU can be caused by a single hot thread, or mapped hardware interrupt.  Relief of the bottleneck usually involves tuning to use more CPUs in parallel.
  `uptime` "load average" (or /proc/loadavg) wasn't included for CPU metrics since Linux load averages include tasks in the uninterruptable state (usually I/O).
  .. [2] The man page for vmstat describes "r" as "The number of processes waiting for run time", which is either incorrect or misleading (on recent Linux distributions it's reporting those threads that are waiting, <i>and</i> threads that are running on-CPU; it's just the wait threads in other OSes).
  .. [3] There may be a way to measure per-process scheduling latency with perf's sched:sched_process_wait event, otherwise `perf probe` to dynamically trace the scheduler functions, although, the overhead under high load to gather and post-process many (100s of) thousands of events per second may make this prohibitive.  SystemTap can aggregate per-thread latency in-kernel to reduce overhead, although, last I tried schedtimes.stp (on FC16) it produced thousands of "unknown transition:" warnings.
  LPE == <a href="../perf.html">Linux Performance Events</a>, aka perf_events. This is a powerful observability toolkit that reads CPC and can also use static and dynamic tracing.  Its interface is the `perf` command.
  CPC == CPU Performance Counters (aka "Performance Instrumentation Counters" (PICs) or "Performance Monitoring Counters" (PMCs), or "Performance Monitoring Unit" (PMU) Hardware Events), read via programmable registers on each CPU by `perf` (which it was originally designed to do).  These have traditionally been hard to work with due to differences between CPUs.  LPE `perf` makes life easier by providing aliases for commonly used counters.  Be aware that there are usually many more made available by the processor, accessible by providing their hex values to `perf stat -e`.  Expect to spend some quality time (days) with the processor vendor manuals when trying to use these.  (My short <a href="http://www.beginningwithi.com/comments/2010/04/30/performance-instrumentation-counters/">video</a> about CPC may be useful, despite not being on Linux).
  .. [4] There aren't many error-related events in the recent Intel and AMD processor manuals; be aware that the public manuals may not show a complete list of events.
  .. [5] The goal is a measure of memory capacity saturation - the degree to which a process is driving the system beyond its ability (and causing paging/swapping).  High fault latency works well, but there isn't a standard LPE probe or existing SystemTap example of this (roll your own using dynamic tracing).  Another metric that may serve a similar goal is minor-fault rate by process, which could be watched from /proc/PID/stat.  This should be available in `htop` as MINFLT.
  .. [6] Tim Cook ported nicstat to Linux; it can be found on <a href="http://sourceforge.net/projects/nicstat/">sourceforge</a> or his <a href="https://blogs.oracle.com/timc/nicstat-the-solaris-and-linux-network-monitoring-tool-you-did-not-know-you-needed">blog</a>.
  .. [7] Dropped packets are included as both saturation and error indicators, since they can occur due to both types of events.
  .. [8] This includes tracing functions from different layers of the I/O subsystem: block device, SCSI, SATA, IDE, ...  Some static probes are available (LPE "scsi" and "block" tracepoint events), else use dynamic tracing.
  CPI == Cycles Per Instruction (others use IPC == Instructions Per Cycle).
  I/O interconnect: this includes the CPU to I/O controller busses, the I/O controller(s), and device busses (eg, PCIe).
  Dynamic Tracing: Allows custom metrics to be developed, live in production.  Options on Linux include: LPE's "perf probe", which has some basic functionality (function entry and variable tracing), although in a trace-n-dump style that can cost performance; SystemTap (in my <a href="http://dtrace.org/blogs/brendan/2011/10/15/using-systemtap/">experience</a>, almost unusable on CentOS/Ubuntu, but much more stable on Fedora); DTrace-for-Linux, either the Paul Fox port (which I've tried) or the OEL port (which Adam has <a href="http://dtrace.org/blogs/ahl/2012/02/23/dtrace-oel-update/">tried</a>), both projects very much in beta.
