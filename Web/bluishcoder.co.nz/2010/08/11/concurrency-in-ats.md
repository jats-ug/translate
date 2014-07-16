# BLUISH CODER: ATSにおける並行プログラミング

(元記事は http://bluishcoder.co.nz/2010/08/11/concurrency-in-ats.html です)

[ATS](http://www.ats-lang.org/)
ではオペレーティングシステムのネイティブスレッドを使った並行プログラミングが可能です。
[Pthreads](http://en.wikipedia.org/wiki/POSIX_Threads)
のラッパーと、ワーカースレッドの生成する高レベル API、そしてそれらを使って
'parworkshop' と呼ばれる並列計算を提供しています。

メーリングリストのアナウンスでは次のように 'parworkshop' を解説しています:

> …'parworkshop' と名付けられたパッケージは、プログラマが 'workshop' を作り、'workers' を借りて、'works' を投稿することなどを可能にします。
> おそらく、もうお分かりでしょう :) 現時点では、それぞれの 'worker' は Pthreads 上に構築されています。
> 長期的には、ハードウェア/アーキティクチャを下層として利用する、より洗練された workshop を構築できるでしょう。

この記事では、'parworkshop' の使い方を学ぶために2つの例を紹介します。1つ目は実際には何も計算しないシンプルなものです。2つ目は
The second is based on a programming example posted on reddit recently.

## 単純な例

'parworkshop' を使うために、2つのファイルをインクルードします:

```ocaml
staload "libats/SATS/parworkshop.sats"
staload "libats/DATS/parworkshop.dats"
```

The SATS file contains the declarations for the functions and datatypes. The DATS file is needed for implementations of some of the template functions.

The basic steps to using ‘parworkshop’ involve:

1. Create a workshop using ‘workshop_make’.
2. Add workers to the workshop using ‘workshop_add_worker’. Each worker is a native thread.
3. Insert one or more units of work to be processed by a worker using ‘workshop_insert_work’.
4. Wait for all units of work to be completed using ‘workshop_wait_blocked_all’.
5. Tell the workers to quit. This can be done by using ‘workshop_insert_work’ with some sentinel to tell the worker to quit.
6. Wait for all workers to quit using ‘workshop_wait_quit_all’.
7. Free the worker using ‘workshop_free’ or ‘workshop_free_vt_exn’.

Each worker created in step 2 above is a native thread. This thread blocks on a queue after being created. ‘workshop_insert_work’ (step 3 above) adds items onto the queue. One of the worker threads will then wake up, call a function that gets the unit of work passed as an argument and then resume blocking on the queue. The function that gets called by the worker thread is one of the parameters in ‘workshop_make’ created in step 1.

The definition of ‘workshop_make is:

```ocaml
fun{a:viewt@ype} workshop_make {n:pos} (
  qsz: size_t n
, fwork: {l:agz} (!WORKSHOPptr (a, l), &a >> a?) -<fun1> int
) : WORKSHOPptr a
```

From this you can see that ‘workshop_make’ is a template function that takes two arguments. The template function is parameterized over the type of the work item (the ‘a’ in the definition). The first of the two arguments is a number for the initial size of the queue that is used for adding work items. The second is a pointer to the function that gets called in a worker thread when a work item is added to the queue. That function must have the definition:

```ocaml
{l:agz} (!WORKSHOPptr (a, l), &a >> a?) -<fun1> int
```

This means the worker thread function receives as an argument the created workshop (ie. the result from ‘workshop_make’) and a reference to the work item. It returns an ‘int’ value. The return value of this worker function tells the worker thread whether it should quit or continue processing queue items. The return values are:

* return status > 0 : the worker is to continue
* return status = 0 : the worker is to quit
* return status = ~1 : the worker is to pause

For example, a worker function that takes an ‘int’ as the unit of work and prints the value of it could be:

```ocaml
fun fwork {l:addr} (
  ws: !WORKSHOPptr(work,l),
  x: &int >> int?
  ): int = let
  val () = printf("x = %d\n", @(x))
in 
 if x < 0 then 0 else 1
end
```

In this example if the value of the ‘int’ is less than zero then the worker thread will quit. To create a workshop and some workers using this function:

```ocaml
val ws = workshop_make<int>(1, fwork)
val _ = workshop_add_worker(ws)
val _ = workshop_add_worker(ws)
```

Notice that the type of the work unit (the ‘int’) is passed as a template parameter to ‘workshop_make’, very similar to C++ syntax for explicit use of template functions. To add work items:

```ocaml
val () = workshop_insert_work(ws, 1)
val () = workshop_insert_work(ws, 2)
val () = workshop_insert_work(ws, 3)
```

This will queue three items which will cause the threads to wake up and start processing them. We can then use ‘workshop_wait_blocked_all’ to wait for them to finish and post work items telling the worker threads to quit:

```ocaml
val () = workshop_wait_blocked_all(ws)
val nworker = workshop_get_nworker(ws)
var i: Nat = 0
val () = while(i < nworker) let
           val () = workshop_insert_work(ws, ~1)
         in i := i + 1 end  
```

This uses ‘workshop_get_nworker’ to get a count of the number of worker threads that exist. Then loops up to this count inserting a work item less than zero to cause those worker threads to exit. The final steps are to wait until they quit and free the workshop:

```ocaml
val () = workshop_wait_quit_all(ws)
val () = workshop_free(ws)
```

The complete program is:

```ocaml
staload "libats/SATS/parworkshop.sats"
staload "libats/DATS/parworkshop.dats"

typedef work = int

fun fwork {l:addr} (
  ws: !WORKSHOPptr(work,l),
  x: &work >> work?
  ): int = let
  val () = printf("x = %d\n", @(x))
in 
  if x < 0 then 0 else 1
end

implement main() = let
  val ws = workshop_make<work>(1, fwork)
  val status = workshop_add_worker(ws)
  val status = workshop_add_worker(ws)
 
  val () = workshop_insert_work(ws, 1)
  val () = workshop_insert_work(ws, 2)
  val () = workshop_insert_work(ws, 3)

  val () = workshop_wait_blocked_all(ws)
  val nworker = workshop_get_nworker(ws)
  var i: Nat = 0
  val () = while(i < nworker) let
             val () = workshop_insert_work(ws, ~1)
           in i := i + 1 end  

  val () = workshop_wait_quit_all(ws)
  val () = workshop_free(ws)
in
  ()
end
```

This can be compiled with (assuming it’s in a file called ‘eg1.dats’):

```
atscc -o eg1 eg1.dats -D_ATS_MULTITHREAD -lpthread
```

Running it gives:

```
$ ./eg1
x = 1
x = 2
x = 3
x = -1
x = -1
```

Note that ‘parworkshop’ makes use of the type system to ensure that resources are used correctly. It is a compile time error for example not to call ‘workshop_free’.

## More complex example

While the previous example is simple it does show the basic steps that need to be done in most ‘parworkshop’ usage. But in most parallel programming tasks you want the worker thread to process some data and return a result somehow. This result is then used for further processing. There are a few examples in the ATS distribution that show how to do this. I’ll cover one approach below.

A thread on programming.reddit.com recently resulted in a number of implementations of a simple programming task in various programming languages. Some of these were implemented in concurrent languages and I tried my hand at a version in ATS using ‘parworkshop’.

The task involves approximating the value of the mathematical constant ‘e’ by taking the following approach (description from here):

> Select a random number between 0 and 1. Now select another and add it to the first. Keep doing this, piling on random numbers. How many random numbers, on average, do you need to make the total greater than 1? The answer is e.

The code to do this is fairly simple in ATS and you can see it in the completed result later. I ended up with a function called ‘n_attempts’ that takes the number of iterations to try and returns the count of the total number of attempts. This can then be divided by the iterations to get the average and therefore the estimation of ‘e’:

```ocaml
fun n_attempts (n:int): int
...
var e = n_attempts(10000000) / double_of_int(10000000)
```

To make use of multiple cores I decided on creatng a worker thread for each core. Each of these worker threads would run a portion of the total number of desired iterations. The unit of work is a data type ‘command’ that can be one of two things:

1. ‘Compute’ which tells the worker thread to perform a number of iterations. The ‘Compute’ contains a pointer to an integer that is used to hold the result produced by the worker thread.
2. ‘Quit’ which tells the worker thread to exit.

This data type is defined as:

```ocaml
dataviewtype command = 
  | {l:agz} Compute of (int @ l | ptr l, int)
  | Quit 

viewtypedef work = command
```

Notice the use of ‘dataviewtype’ instead of ‘datatype’. The latter would require linking against the garbage collector to manage reclamation of the allocated data types. The former results in a ‘linear data type’ which needs to be explicitly free’d and therefore doesn’t need the garbage collector. My goal was to make this example not need the garbage collector.

The worker thread function looks like:

```ocaml
fun fwork {l:addr}
  (ws: !WORKSHOPptr(work,l), x: &work >> work?): int = 
  case+ x of
  | ~Compute (pf_p | p, iterations)  => let 
       val () = !p := n_attempts(iterations)
       extern prfun __unref {l:addr} (pf: int @ l):void
       prval () = __unref(pf_p)
     in 1 end
  | ~Quit () => 0
```

It basically switches on the value of the work unit using a ‘case’ statement. In the case of ‘Quit’ it returns ‘0’ to exit the worker thread. In the case of ‘Compute’ it calls n_attempts with the number of iterations and stores the result in the integer pointer. It then returns ‘1’ for the worker thread to wait for more work units. The use of the ’~’ before the type in the ‘case’ statement tells ATS to release the memory for the linear type (‘Compute’ and ‘Quit’) so we don’t need to do it manually.

When adding the work items to the workshop I create an array of ‘int’ to hold the results. I use a linear array and manually manage the memory:

```ocaml
#define NCPU 2
...
val (pf_gc_arr, pf_arr | arr) = array_ptr_alloc<int>(NCPU)
val () = array_ptr_initialize_elt<int>(!arr, NCPU, 0)
...
val () = array_ptr_free {int} (pf_gc_arr, pf_arr | arr)
```

When inserting the work items I need to create a ‘Compute’ and pass it a pointer to the ‘int’ where the result is stored. This pointer points to a single element of this array. The code to do this is:

```ocaml
fun loop {l,l2:agz} {n:nat} .< n >. (
      pf: !array_v(int, n, l2)
    | ws: !WORKSHOPptr(work, l), 
      p: ptr l2, 
      n: int n, 
      iterations: int)
    : void =
  ...
  prval (pf1, pf2) = array_v_uncons{int}(pf)
  extern prfun __ref {l:addr} (pf: ! int @ l): int @ l
  prval xf = __ref(pf1)
  val () = workshop_insert_work(ws, Compute (xf | p, iterations))
  val () = loop(pf2 | ws, p + sizeof<int>, n - 1, iterations)
  prval () = pf := array_v_cons{int}(pf1, pf2)
  ...
```

This code does some proof manipulations to take the proof that provides access to the entire array, and returns a proof for accessing a particular element. This ensures at compile time that the worker thread function can only ever access the memory for that single element. The magic that does this is ‘array_v_uncons’:

```ocaml
prval (pf1, pf2) = array_v_uncons{int}(pf)
```

‘pf’ is the proof obtained for the entire array (the ‘pf_arr’ returned from ‘array_ptr_alloc’). ‘array_v_uncons’ consumes this and returns two proofs. The first (‘pf1’ here) is an ‘int @ l’ for access to the first item in the array. The second (‘pf2’ here) is for the rest of the array.

We can now pass ‘pf1’ to ‘Compute’ along with the pointer to that integer. This is repeated until there are no more array items to process. Notice the recursive call to ‘loop’ increments the pointer by the size of an ‘int’. ATS is allowing us to efficiently work directly with C memory, but safely by requiring proofs that we can access the memory and that it contains valid data.

The following line occurs after the ‘loop’ recursive call:

```ocaml
prval () = pf := array_v_cons{int}(pf1, pf2)
```

This is needed because ‘array_v_uncons’ consumed our proof held in ‘pf’. But the ‘loop’ definition states that ‘pf’ must not be consumed (that’s the ’!’ in “pf: !array_v(int, n, l2)”). This last line sets the proof of ‘pf’ back to the original by consing together the ‘pf1’ and ‘pf2’ we created when consuming ‘pf’.

The other bit of trickery here is:

```ocaml
extern prfun __ref {l:addr} (pf: ! int @ l): int @ l
prval xf = __ref(pf1)
```

The ‘Compute’ constructor consumes the proof it is passed. If we pass ‘pf1’ it would be consumed and can’t be used in the ‘array_v_cons’ call later. This code creates a new proof ‘xf’ that is the same as the existing ‘pf1’ proof and passes it to the ‘Compute’ constructor. We later consume this proof manually inside the ‘fwork’ function. You probably noticed this code in the ‘fwork’ example earlier:

```ocaml
extern prfun __unref {l:addr} (pf: int @ l):void
prval () = __unref(pf_p)
```

This thread on the ATS mailing list discusses this and a better way of handling the issue using a ‘__borrow’ proof function which I haven’t yet implemented.

The entire program is provided below. Hopefully it’s easy to follow given the previous simple example and the above explanation of the more complicated bits:

```ocaml
staload "libc/SATS/random.sats"
staload "libc/SATS/unistd.sats"
staload "libats/SATS/parworkshop.sats"
staload "libats/DATS/parworkshop.dats"
staload "prelude/DATS/array.dats"

#define ITER 100000000
#define NCPU 2

fn random_double (buf: &drand48_data): double = let
  var r: double
  val _ = drand48_r(buf, r)
in
  r
end

fn attempts (buf: &drand48_data): int = let 
  fun loop (buf: &drand48_data, sum: double, count: int): int = 
    if sum <= 1.0 then loop(buf, sum + random_double(buf), count + 1) else count
in
  loop(buf, 0.0, 0)
end

fun n_attempts (n:int): int = let
  var buf: drand48_data
  val _ = srand48_r(0L, buf)
  fun loop (n:int, count: int, buf: &drand48_data):int =
    if n = 0 then count else loop(n-1, count + attempts(buf), buf)
in
  loop(n, 0, buf)
end

dataviewtype command = 
  | {l:agz} Compute of (int @ l | ptr l, int)
  | Quit 

viewtypedef work = command

fun fwork {l:addr}
  (ws: !WORKSHOPptr(work,l), x: &work >> work?): int = 
  case+ x of
  | ~Compute (pf_p | p, iterations)  => let 
       val () = !p := n_attempts(iterations)
       extern prfun __unref {l:addr} (pf: int @ l):void
       prval () = __unref(pf_p)
     in 1 end
  | ~Quit () => 0

fun insert_all {l,l2:agz}
               {n:nat | n > 0}
    (pf_arr: !array_v(int, n, l2) | ws: !WORKSHOPptr(work, l),
     arr: ptr l2, n: int n, iterations: int):void = let
  fun loop {l,l2:agz} {n:nat} .< n >. (
      pf: !array_v(int, n, l2)
    | ws: !WORKSHOPptr(work, l), 
      p: ptr l2, 
      n: int n, 
      iterations: int)
    : void =
    if n = 0 then () else let
      prval (pf1, pf2) = array_v_uncons{int}(pf)
      extern prfun __ref {l:addr} (pf: ! int @ l): int @ l
      prval xf = __ref(pf1)
      val () = workshop_insert_work(ws, Compute (xf | p, iterations))
      val () = loop(pf2 | ws, p + sizeof<int>, n - 1, iterations)
      prval () = pf := array_v_cons{int}(pf1, pf2)
    in
      // nothing
    end
in
  loop(pf_arr | ws, arr, n, iterations / NCPU)
end

implement main() = let 
 val ws = workshop_make<work>(NCPU, fwork)

  var ncpu: int
  val () = for(ncpu := 0; ncpu < NCPU; ncpu := ncpu + 1) let 
            val _ = workshop_add_worker(ws) in () end
 
  val nworker = workshop_get_nworker(ws)

  val (pf_gc_arr, pf_arr | arr) = array_ptr_alloc<int>(NCPU)
  val () = array_ptr_initialize_elt<int>(!arr, NCPU, 0)
  prval pf_arr  = pf_arr

  val () = insert_all(pf_arr | ws, arr, NCPU, ITER)

  val () = workshop_wait_blocked_all(ws)

  var j: Nat = 0;
  val () = while(j < nworker) let
             val () = workshop_insert_work(ws, Quit ())
           in 
             j := j + 1
           end  

  val () = workshop_wait_quit_all(ws)
  val () = workshop_free_vt_exn(ws)

  var k: Nat = 0;
  var total: int = 0;
  val () = for(k := 0; k < NCPU; k := k + 1) total := total + arr[k]
  val avg = total / double_of_int(ITER)
  val () = printf("total: %d\n", @(total))
  val () = print(avg)
in
  array_ptr_free {int} (pf_gc_arr, pf_arr | arr)
end
```

Hongwei Xi has cleaned this up a bit by replacing ‘__ref’ and ‘__unref’ with ‘__borrow’ and using a stack allocated linear array for the results. This is included in the ATS distribution as randcomp_mt.dats. Looking at that code will hopefully help clarify how these things work when compared to this code I wrote.

Compiling a single threaded version of the code (I posted it in the reddit thread here runs in about 30 seconds on my dual core machine. The ‘parworkshop’ version runs in about 16 seconds. The single threaded version uses ‘drand48’ whereas the ‘parworkshop’ version uses ‘dthread48_r’. The latter is needed otherwise the threads are serialized over a lock in ‘drand48’ for the random number state and the runtime is even slower than the single threaded version. Compiling with ‘-O3’ passed to ‘atscc’ speeds things up quite a bit too.

My first attempt at using ‘parworkshop’ for this task is in this gist paste. Instead of a linear data type to hold the unit of work I used a linear closure and called it in the ‘fwork’ function to get the result. I took this approach originally based on the existing MULTICORE examples in ATS. In the end I opted for the linear datatype approach as I felt it was clearer.
