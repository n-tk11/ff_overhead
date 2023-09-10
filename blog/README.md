Fastfreeze(FF) is a turn-key solution for containerized checkpoint and restore.FF uses CRIU as its core engine to do most of the jobs. It is easy to use through a simple CLI interface with very few commands and complications and is also non-privileged to provide a more secure way to do things.   
But there there is a problem when using it to checkpoint applications with multiple processes, particularly the restore time.  
Let's look at the restoration time of a checkpoint Memhog with 1-8 processes  
![[Picture1.png]]
Fastfreeze take so much time when there is more than 1 process. ~50x compare to 1 process and ~ 500x with 8 processes. These huge delays are obviously problematic for many use cases eg. Using FF in migration.  

Firstly, This is our setup doing all things further.  
- Virtual Machine (managed with VirtualBox) Ubuntu 22.04
- Docker 
- Container Image: Base on docker.io/ubuntu:20.04
- Application: UMS's Evaluation Memhog app (https://github.com/hpcclab/NIMS/tree/main/evaluation/memhog)
- Fastfreeze v1.3.0 using source from the main branch (including its dependencies) (https://github.com/twosigma/fastfreeze)
  
Let's investigate more where the cause resides in. First, we did it very simply: use the fastfreeze verbose option when restoring.  
```eg. fastfreeze checkpoint -vvv ```
as it gives a lot of information including the timestamp.
![[Pasted image 20230910230917.png]]
We focused at the point where the restore process hang for about 40 s. 
At fist glance, It looks like the hang was coming from when CRIU try to restore a process to a pgid. Is it??

To be sure we added more debugging code into the CRIU source and built the whole FF again to get more problem understanding. We built like >20 versions of FF but here we will  only show you some interesting ones.

![[Pasted image 20230910231713.png]]
As we added more fine-grain debug messages, The hang happen ed after some process scheduling/synchronization, which is very weird. 
Lets look at the point where the last line before hang print out.
```C
static inline void futex_wait_while(futex_t *f, uint32_t v)
{
	while ((uint32_t)atomic_read(&f->raw) == v) {
		int ret = sys_futex((uint32_t *)&f->raw.counter, FUTEX_WAIT, v, NULL, NULL, 0);
		LOCK_BUG_ON(ret < 0 && ret != -EWOULDBLOCK);
	}
}
```
To be said, at the point of delay , The process 1002 just sleep!! 
What about another process?

Before we go further, This is a summarized diagram of how FF restores 2 processes Memhog. Note here that the pid will always be the same through out the investigation as FF set root_pid with a constant number(default 1000).
![[Pasted image 20230910233602.png]]
As you can see here Memhog processes(1001,1002) should be restored concurrently(as seen in the code login). 

From the above knowledge,we assumed that the problem may not come from the process that active before the hang(1002) but because another child process(1001) stuck somewhere and not do any work even its sibling sleep! 

Let's look at the next logs from another source modification. 
![[Pasted image 20230910234839.png]]
Mainly, we took a broader look this time.  From here we saw that the restore procedure continued at the point where a step relating to a function call far before our hang happened ended and it was also related to another sibling process(1001) restoration as we assumed.

As we saw that the hang might related to "set_next_pid", we also added some debug messages to a Fastfreeze utility called "set_ns_last_pid".
![[Pasted image 20230911000908.png]]
Thats great! At this point, we guaranteed that the problem was from the set_ns_last_pid thing.

The problem arises from a process called the fork_hack (or fork_bomb or process_spinning) which is a very simple but genius way that Fastfreeze used to control what the spawning process pID should be by frequently fork process and kill it till we get the desire PID.
You may wonder why Fastfreeze wants to control what the PID is, isn't that seem against how Linux works? Can we solve this problem by changing this?  No, not an easy thing to do, it is a very weird requirement but this is not what Fastfreeze developer intended. It is what CRIU decides how to checkpoint/restore an app, things should be the same before and after restoring even the PID.  

So, To change the PID fixing logic to solve our problem seems to be a bit overdoing. We have to find another way around. 
Take a look at set_ns_last_pid logic
 set_ns_last_pid (by 2sigma)  
	→ If /proc/sys/kernel/ns_last_pid is writable, just write it  
	→ else fork bomb(process spinning)
Something new here!  ```/proc/sys/kernel/ns_last_pid```! This file may lead to our solution as it is where the kernel reads what the last PID in the namespace is and will fork with the next PID.

Our problem happens because we always run into the second case as we cannot directly write the /proc/.../ns_last_pid file because we may want to run our app as a non-privileged(also avoid the CAP_SYS_ADMIN) to serve production scenarios related to security risks.

The target now is to find a way to let FF's set_ns_last_pid write to /proc/../ns_last_pid while still achieving security requirement by not run the container as privileged and not give any process/app the CAP_SYS_ADMIN capability. 

To achieve this, First, we gave the CAP_CHECKPOINT_RESTORE , one of the new Linux capability added to Linux since kernel v5.9 which also allow writing to /proc/.../ns_last_pid!, to set_ns_last_pid executable eg. ```setcap 40+eip ./set_ns_last_pid```  (40 is a cap_id for CAP_CHECKPOINT_RESTORE).  
That did not solve our problem!! Why??  As we researched more, we knew that by default docker will mount /proc as a read-only filesystem so even if the binary has the capability to write it still cannot write.  A simple way to do this is to disable the docker default security options by adding some parameters when starting the image.
```
eg. docker run .... --security-opt systempaths=unconfined --security-opt apparmor=unconfined ...

```
the *systempaths* option will make docker no more mount /proc as a read-only fs and *apparmor* option is related to security rules which by the default docker apparmor profile also block writing to /proc files. 
Disable all default security behavior seem a bit dangerous and risky. You can also write your own custom apparmor profile to only allow writing to /proc/.../ns_last_pid not the whold /proc filesystem. 

For example in our case, we modified the docker default profile with a tiny added globbing pattern.
```
...snip...

deny @{PROC}/sys/kernel/{?,??,[^s][^h][^m]**}-@{PROC}/sys/kernel/ns_last_pid  w,  # deny everything except shm* and ns*(ns_last_pid) in /proc/sys/kernel/

...snip...
```
the ```-@{PROC}/sys/kernel/ns_last_pid``` will except our /proc/.../ns_last_pid in a rule that denies writing to /proc/sys/kernel files.

Here the result
![[Pasted image 20230911010948.png]]
Walah! We reduced the restore time from ~50s to ~1s that's a fantastic improvement! and as you can see no more set_next_pid hanging anymore! 
Not only for 2 processes, we also tested with 8 processed and look at it.
![[Pasted image 20230911011331.png]]
The delay decreased from ~500s to ~5s. That's 100 times improvement!! 

For more detail about the solution and example running commands/codes, see: https://github.com/n-tk11/ff_overhead