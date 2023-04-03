# How to fix the program from Coherency misses
The struct used to store the thread info is 32bytes (on my machine)
```c
typedef struct
{
        int thread_id; // 4 bytes (the compiler will align to 8 bytes)
        pthread_t thread; // 8 bytes
        double error; // 8 bytes
        atomic_uintmax_t progress_counter; // 8 bytes
} thread_info_t;
```
If we run the program, we see that the tests of the parallel implementation are slower respect to the sequential version
```
**********************************************************************
Starting sequential reference run...
**********************************************************************
./gs_seq -s 2048 -o /tmp/tmp.QTDwwkAvr4.gs_seq.test.out
Reached maximum number of iterations. Solution did NOT converge.
Note: This is normal if you are using the default settings.
**** Summary ****
   Execution time: 0.252152 s
*****************

**********************************************************************
Starting parallel run...
**********************************************************************
./gs_pth -s 2048 -o /tmp/tmp.YrXjvXx8Lj.gs_pth.test.out
Reached maximum number of iterations. Solution did NOT converge.
Note: This is normal if you are using the default settings.
**** Summary ****
   Execution time: 0.391656 s
*****************

Test results: 
OK
```
What is happening in our program? Why the parallel version is slower? Let's check the cache.<br>
In order to check our cache we can use two different commands:
- `getconf -a`
- `lscpu -C`

Both gives some useful information about our system.
```
 getconf -a | grep CACHE
LEVEL1_ICACHE_SIZE                 32768
LEVEL1_ICACHE_ASSOC                
LEVEL1_ICACHE_LINESIZE             64
LEVEL1_DCACHE_SIZE                 32768
LEVEL1_DCACHE_ASSOC                8
LEVEL1_DCACHE_LINESIZE             64
LEVEL2_CACHE_SIZE                  262144
LEVEL2_CACHE_ASSOC                 4
LEVEL2_CACHE_LINESIZE              64
LEVEL3_CACHE_SIZE                  8388608
LEVEL3_CACHE_ASSOC                 16
LEVEL3_CACHE_LINESIZE              64
LEVEL4_CACHE_SIZE                  0
LEVEL4_CACHE_ASSOC                 
LEVEL4_CACHE_LINESIZE     
```
```
 lscpu -C
NAME ONE-SIZE ALL-SIZE WAYS TYPE        LEVEL SETS PHY-LINE COHERENCY-SIZE
L1d       32K     128K    8 Data            1   64        1             64
L1i       32K     128K    8 Instruction     1   64        1             64
L2       256K       1M    4 Unified         2 1024        1             64
L3         8M       8M   16 Unified         3 8192        1             64
```
Here it is! From both we can conclude that our cachelines over all the 3 levels are 64bytes! (`lscpu` call it **COHERENCY_SIZE**). Considering how we counted the bytes before, two instances of the `thread_info_t` struct will fit in a single cache line.
<br>
Let's imagine to have 4 threads, when a thread will update its local error value
```c
threads[tid].error += fabs(gs_matrix[GS_INDEX(row, col)] - new_value);
```
It will bring 2 elements of the `threads` array in the cacheline. Modifying a field of one, this will trigger the coherence protocol that will need to invalidate the other cacheline. During the whole computation there is an invalidation storm between the adiacent threads creating a phenomenon called `false sharing`. <br>
Since we detected the dimension of our cacheline, we want that this is invalidated as less as possible. The solution: PAD the struct to let this fits the whole cacheline.
```c
typedef struct
{
        int thread_id;
        pthread_t thread;
        double error;
        atomic_uintmax_t progress_counter;
        char __padding[32];
} thread_info_t;
``` 
Our struct now is 64 bytes, the dimension of the cacheline. Running the tests again...
```
**********************************************************************
Starting sequential reference run...
**********************************************************************
./gs_seq -s 2048 -o /tmp/tmp.2LrA77nhY8.gs_seq.test.out
Reached maximum number of iterations. Solution did NOT converge.
Note: This is normal if you are using the default settings.
**** Summary ****
   Execution time: 0.282462 s
*****************

**********************************************************************
Starting parallel run...
**********************************************************************
./gs_pth -s 2048 -o /tmp/tmp.ZqofHNupsZ.gs_pth.test.out
Reached maximum number of iterations. Solution did NOT converge.
Note: This is normal if you are using the default settings.
**** Summary ****
   Execution time: 0.154456 s
*****************

Test results: 
OK
```
We've been able to optimize our program just adding some bytes.