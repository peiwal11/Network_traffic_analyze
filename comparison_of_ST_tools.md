## Python  
- Thread: Uses OS-level threads managed by the operating system.   
- CPU Overhead: High due to the Global Interpreter Lock (GIL) and OS-level context switching.   
- Context Switch Workload: High, as each thread is an OS thread.   
- Drive: Uses an event loop for I/O-bound tasks but relies on threads for concurrency.    
- Capability: Supports HTTP/1.0 and HTTP/1.1.    
- Request: Typically less detailed compared to specialized tools.   
- Weight: Python threads are resource-intensive (heavy).   
-	Stack: Each thread has a 1MB static stack.   
-	Scheduler: Single global queue, harder to load balance due to GIL constraints.

## ApacheBench (ab)   
-	Thread: Single-threaded with non-blocking I/O. It uses a single process to handle multiple connections.   
-	CPU Overhead: Very low because it operates in a single thread without context switching overhead.   
-	Context Switch Workload: None. It uses non-blocking I/O and an event-driven model, avoiding context switching.   
-	Drive: Event loop driven model. It efficiently manages multiple connections within a single thread.   
-	Capability: Supports HTTP/1.0 and HTTP/1.1.   
-	Request: Simple and limited details, mainly focusing on basic performance metrics.  
-	Weight: Very light, as it doesn't create multiple threads or processes.  
-	Stack: Not applicable as it doesn't create multiple threads.  
-	Scheduler: Single queue, event-driven model. It processes requests as they become ready.

     
## go-stress-testing   
-	Thread: Uses goroutines, which are lightweight, managed by the Go runtime.    
-	CPU Overhead: Low due to efficient management of goroutines and OS threads.    
-	Context Switch Workload: Low. The Go runtime handles context switching between goroutines efficiently.   
-	Drive: Go runtime uses a pool of OS threads to execute goroutines.   
-	Capability: Supports HTTP/1.0, HTTP/1.1, and HTTP/2.  
-	Request: Detailed and complex, providing in-depth metrics and analysis.  
-	Weight: Light. Goroutines are lightweight compared to OS threads.   
-	Stack: Goroutines have a 2KB initial stack, which can grow dynamically.   
-	Scheduler: Work-stealing, multiple queues. The Go scheduler handles context switching between goroutines efficiently, with spare threads stealing tasks from busy threads to balance the load.



    

### Overview: go-stress-testing faces significant issues under load, making it almost unusable for more than 10 concurrent requests without keep-alive enabled. In contrast, ApacheBench (ab) successfully handles these conditions. Here are the possible reasons for these failures and the steps taken to investigate them:
1.	Connection Timeouts:      
- Issue: Failures might be caused by connection timeouts. Attempts to set the timeout indicate it does not change the timeout for each connection but rather sets the overall timeout for the task, rendering it nearly useless in preventing connection breaks.   
- Observation: Adjusting the timeout did not prevent failures, indicating the tool might not handle connection timeouts effectively.   
2.	Verification Errors:   
-	Issue: Failures often result in status code 510, which the tool identifies as a parse error. There is no apparent way to disable verification. Changing the verification type to JSON only exacerbates parsing problems.   
-	Observation: The inability to turn off or effectively modify verification settings contributes to the high failure rate.   
3.	Connection Handling (Keep-Alive):   
-	Issue: Enabling keep-alive causes the program to lag significantly and become unresponsive.   
-	Observation: This indicates potential issues in the implementation of keep-alive connections within the tool, leading to severe performance degradation.   
4.	Header Differences:   
-	Issue: Potential differences in request headers were investigated using tcpdump and Wireshark.   
-	Observation: No differences were found between the request headers sent by ab and go-stress-testing, ruling out headers as the cause of the failures.   
5.	Community Feedback:   
-	Issue: Based on issues reported on GitHub, other users have encountered similar problems.   
-	Observation: The tool’s maintainers have not yet addressed these issues, indicating unresolved compatibility and performance problems.

    
Conclusion:   
The go-stress-testing tool may not be suitable for scenarios involving high request rates for large data downloads simultaneously. Additionally, pressing Ctrl-C does not force the requests to stop immediately. Our coworker, [Cobolbaby](https://github.com/cobolbaby), found that instead of stopping the program immediately, the go-stress-testing logic only processes Ctrl-C once a connection is completed. In our case, Ctrl-C failed to stop the process because our connections lasted very long.
