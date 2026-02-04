## Lecture 3: Concurrency and Parallelism

Up until this point we have built up some basic primitives needed for implementing a distributed system– threads for computation and rpcs for communication. However, we have not tackled the question of **why** one might want to build a distributed system.

One primary reason is **parallelism.** Parallelism refers to doing work **simultaneously.** Intuitively, if we have an application that runs at a slow speed on a single computer, we may think, running that same application with the power of 100 computers should surely be faster? Right? However, if we can achieve such performance benefits is dependent on the design of our system.

Designing a distributed system such that we can leverage parallelism and achieve major speedups will be the focus of the next few sections.

### Concurrency vs. Parallelism

One important terminology distinction is that **concurrency** and **parallelism** refer to two different concepts**.** 

**Concurrency** is the ability to manage multiple running processes. However, it may be that these *processes never actually run simultaneously,* but instead these processes are *interleaved* on a single resource.

**Parallelism** is when multiple processes run simultaneously on multiple resources.

![][image1]

*A system is said to be concurrent if it can support two or more actions in progress at the same time. A system is said to be parallel if it can support two or more actions executing simultaneously.*

- *The Art of Concurrency, Clay Breshears*

Let’s consider, for example, a simple machine with a single cpu core. Though it is not possible to achieve parallel execution (at any one time exactly one process may be running on the single core), it is possible to achieve concurrency. We may interleave execution of one process with another.

In the section on threads, we were actually discussing the idea of **concurrent programming.** Concurrent programming is structuring your program in such a way that smaller units of the program (in our case threads or goroutines) can be run in an interleaved or simultaneous fashion.

## Understanding Concurrency

With parallelism, it is clear that running or more processes simultaneously on two cpus will complete faster than running them one after the other on a single cpu. But why is concurrency useful?

There will be no performance benefit if the two tasks require only CPU time (and no other resource, like i/o or memory). However, commonly, processes require some resources. For example, they may be blocked waiting for a response over the network.

![][image2]  
Now, consider if we run these two tasks (with io time requirements) sequentially.

![][image3]  
As you may notice, there is a lot of time where the CPU is idle as the running task waits for io.  
However, what happens when we run these two tasks concurrently? Our operating system notices for example, that task A is blocked on IO and runs one time unit of Task B. Then Task A is ready to resume and we continue from there. There is only one instance in time where, unfortunately, both tasks are using io resources, and the CPU is idle.  
![][image4]  
However, with concurrent execution– interleaving running tasks –we complete both tasks in 8 time units as compared to the 11.5 time units of the sequential execution.

In fact, we’ve explored this idea already– remember our example of a command center orchestrating satellite averages? Our command center exhibited concurrency. Though the command center had only one CPU, it was able to achieve speedup by processing one satellite’s data when we were blocked waiting for other satellites to respond.

## Understanding Parallelism

### Data Parallelism

**Data Parallelism** refers to the same task executing on different data at the same time.  
Oftentimes data parallelism is achieved via partitioning a dataset into chunks and running the task on each chunk simultaneously.

### Task Parallelism

**Task Parallelism** refers to different tasks executing at the same time, operating on the same or different data.

### Pipeline Parallelism

**Pipeline Parallelism** occurs when different tasks operate in parallel forming a pipeline. The output of each task is passed as input to the next task in the pipeline. Each task can continue processing the next input by the time it pushes the result to the output.

Consider the below 3-stage pipelined task. Naively, we only complete a task once each of the 3 stages finished. Now, we can complete a task every time a single stage completes\! 

If each stage takes 3 seconds, without pipelining we complete one task every 9 seconds. With pipelining we complete one task every 3 seconds. From the perspective of a single task, it still takes 9 seconds for our system to output a response from the time it arrives in stage 1 to the time it exits stage 3\. However, we can service *more requests per unit of time*. **Pipelining improves our system’s throughput.**

Notice, pipeline parallelism is most effective when each stage can run in parallel. Thus, it is important that stages use separate resources. If two stages, for example, are both competing for CPU resources, we will not see the full impact of running these stages in parallel.

## Appendix: Parallelism Examples in Go Code

Consider the unparalleled section of go code. Can we add **task parallelism**?

```golang
import "stats" 
func main() { ... 
    mean, _ := stats.Mean(*s1) 
    median, _ := stats.Median(*s1) 
    p99th, _ := stats.Percentile(s2, 99) 
}
```

One solution would resemble the implementation below. This solution uses channels to grab the results from each of the goroutines.

Task parallelism implementation with channels:

```golang
func main() {  
    meanCh := make(chan float64)
	medianCh := make(chan float64)
	p99Ch := make(chan float64)

	go func(input *[]float64) {
		mean, _ := stats.Mean(input)
		meanCh <- mean
	}(&data)

	go func(input *[]float64) {
		median, _:= stats.Median(input)
		medianCh <- median
	}(&data)

	go func(input *[]float64) {
		p99th, _ := stats.Percentile(input, 99)
		p99Ch <- p99th
	}(&data)

	mean := <-meanCh
	median := <-medianCh
	p99 := <-p99Ch
}

```

Can we add **data parallelism** to the go code which normalizes our satellite data readings?

Serial implementation of normalizing 100,000 readings:

```golang
for i := 0; i < 100000; i++ {
    data[i] = normalizeReading(data[i])
}
```

The example below breaks up our data into 12 batches and normalizes the data in parallel. Notice, because the goroutines work on non-overlapping sections of the input data, there is no data race\!

Implementation with data parallelism and waitgroups:

```golang
batch := len(data) / 12 // Divide up data into 12 chunks, where 12 is # cores
var wg sync.WaitGroup

for start := 0; start < len(data); start += batch {
	end := start + batch
	if end > len(data) {
		end = len(data)
	}
	wg.Add(1)
	go func(s, e int) {
		defer wg.Done()
		for i := s; i < e; i++ {
			data[i] = scaleReading(data[i])
		}
	}(start, end)
}

wg.Wait()
```


Finally, can we add **pipeline parallelism** to our entire command center code?

Implementation with pipelining via channels:

```golang
readingQueue := []string{"temp", "dist",...}
const N = 100000

for _, q := range readingQueue {
	var data []float64

	// Collect readings
	for i := 0; i < N; i++ {
		data = append(data, performReading(q))
		time.Sleep(1 * time.Second)
	}

	// Normalize readings
	for i := 0; i < len(data); i++ {
		data[i] = normalizeReading(data[i])
	}

	// Compute stats
	mean, _ := stats.Mean(data)
	p99, _ := stats.Percentile(data, 99)
	median, _ := stats.Median(data)

	// Send to Earth
	sendToEarth(q, data, mean, p99, median)
}
```

The below implementation uses channels between a go routine responsible for each of a 4 stage pipeline.

```golang
readingQueue := []string{"temp", "dist",...}

// Channels between stages
readCh := make(chan ReadingBatch)
statsCh := make(chan StatsResult)

// Stage 1: Reading goroutine(s)
go func() {
	for _, q := range readingQueue {
		var data []float64
		for i := 0; i < 1000000; i++ {
			data = append(data, performReading(q))
			time.Sleep(1 * time.Second)
		}
		readCh <- ReadingBatch{q, data}
	}
	close(readCh)
}()

// Stage 2: Normalization goroutine
go func() {
	for batch := range readCh {
		for i := range batch.data {
			batch.data[i] = normalizeReading(batch.data[i])
		}

		mean, _ := stats.Mean(batch.data)
		p99, _ := stats.Percentile(batch.data, 99)
		median, _ := stats.Median(batch.data)
		statsCh <- StatsResult{...}
     }
     close(statsCh)
}()

// Stage 3: Send results to Earth
for result := range statsCh {
	sendToEarth(result.q, result.data, result.mean, result.p99, result.median)
}

```

The implementation with pipeline parallelism will improve our system performance best cases where stages primarily use different system resources. 

### WaitGroups

Another technique for coordinating goroutines is the go synchronization package’s WaitGroup primitive. A WaitGroup allows a main or coordinating goroutine to wait for all worker threads to complete or mark themselves as “done”.

`var wg sync.WaitGroup;` is how you make a waitgroup. 

`wg.Add(n)` is how you add n to the “waiting” counter of the waitgroup.

`wg.Done()` is how you mark one worker as “done” on the waitgroup. Effectively, this reduces the “waiting” counter by 1\. Oftentimes, you will see `defer wg.Done()` at the top of a function. The `defer` keyword specifies a line to run right before the function returns. This is useful so we do not have to remember to put `wg.Done()` before every `return` statement of a long function.

`wg.Wait()` blocks until the “waiting” counter is zero and all workers have completed.

### Go Bugs

#### Deadlock

What happens when we run the following two snippets of worker code simultaneously?

```golang
func workerA() {
	ALock.Lock()
	defer ALock.Unlock()

	for i := 0; i < 1000; i++ {
		if(A[i] == 0) {
			BLock.Lock()
			A[i] = B[i]
			BLock.Unlock()
		}
	}
	...
}
```

```golang

func workerB() {
	BLock.Lock()
	defer BLock.Unlock()

	for i := 0; i < 1000; i++ {
		if(B[i] == 0) {
			ALock.Lock()
			B[i] = A[i]
			ALock.Unlock()
		}
	}
	...
}

```

It is possible that we have the following pattern: workerA acquires Alock. workerB acquires Block. workerA tries to acquire BLock but can’t as workerB already holds the lock, and blocks. workerB tries to acquire ALock but can’t as workerA already holds the lock, and blocks. This problem is known as **deadlock.**  
**Deadlock** is when two or more threads cannot progress because each is waiting for a resource the other holds. 

A key feature of **deadlock** is that it will never be resolved– neither thread will ever give up their resource. Deadlock is defined by a circular dependency in resource requirements between who holds a resource and who requests a resource.

#### Anonymous Function Scope

When running the following code we get:

```
func do_processing(batch int) {
    log.Println(“Working on Batch: ”, batch)
}

func main() {
	batch := 0
	for i := 0; i< 5; i+= 1 {
		batch += 1
		go func() {
			do_processing(batch)
		}()
	}
}
```

Upon running the code we get the following buggy output. Instead of having one call of do\_processing per batch, we see something to the effect of:

```
Working on Batch: 5
Working on Batch: 5
Working on Batch: 5
Working on Batch: 5
Working on Batch: 5
```

Why is this?

Well, this is because `batch` is a **shared variable** across all threads, even as the threads start to execute. `batch` is in scope for all of our anonymous functions. Also, don’t forget, that threads may be scheduled any way the goruntime wishes. So what happens is:

1. The go runtime increments batch, calls the anonymous function, no body code of the thread executes yet.  
2. The go runtime increments batch, calls the anonymous function, no body code of the thread executes yet.  
3. The go runtime increments batch, calls the anonymous function, no body code of the thread executes yet.  
4. The go runtime increments batch, calls the anonymous function, no body code of the thread executes yet.  
5. The go runtime increments batch, calls the anonymous function, no body code of the thread executes yet.

Then, all the body code of each of the five threads executes\! And we see, 5, the shared value of `batch` get printed out five times.

How can we fix this? We should instead pass in batch as a parameter to the anonymous function:


```golang
func do_processing(batch int) {
    log.Println(“Working on Batch: ”, batch)
}

func main() {
	batch := 0
	for i := 0; i< 5; i+= 1 {
		batch += 1
		go func(batch int) {
			do_processing(batch)
		}(batch)
	}
}
```

Since go is pass-by-value for integers, each thread will be started with whatever the value of `batch` was **at the time of calling.**

#### Slices

We notice a bug when running the below code:

```golang
func add(slice []int, val int) {
	slice = append(slice, val)
	log.Println(slice)
}

func main() {
	slice := []int{1,2,3,4}
	log.Println(slice)
	add(slice, 5)
	log.Println(slice)
}
```

For some reason, we get the output:

```
[1,2,3,4]
[1,2,3,4,5]
[1,2,3,4]
```

This is because go is pass-by-value for the slice header. Within the body of `add`, we reassign that copy of the slice header’s data point to a newly allocated array: \[1,2,3,4,5\]. However, when add returns, the original slice header in `main` remains unchanged, still pointing to an array of \[1,2,3,4\].

![][image5]  
*What the slice header in `add` resembles at the start of function call.*  
![][image6]  
*What the slice header in `add` resembles at the end of function call.*

To address this, we must instead pass the slice header **by reference, using a pointer.**

```golang
func add(slice *[]int, val int) {
	*slice = append(*slice, val)
	log.Println(slice)
}

func main() {
	slice := []int{1,2,3,4}
	log.Println(slice)
	add(&slice, 5)
	log.Println(slice)
}
```

The syntax for go pointers is similar to C: 

* We use `*` as an operator to dereference (follow that pointer to the data object it refers to).   
* We use `&` to get a pointer to a data object  
* We use `*type` (.i.e*.* `*int`*),* to indicate the type of this expression is pointer-to-whatever-type or pointer-to-int 

#### Select and Timers

Select statements are useful programming structure for waiting on the result of several channels.

```golang
func main() {
    …
    for {
        select {
            case <-quit:
            …
            case <-ticker.C: //every 500ms 
                …
            case <- msg_ready:
                …
            default:
                log.Println(“Nothing to do!”)
        }
    }
}
```

However, if nothing is ready on any channel, the `default` case will run. If there is no `default` case,  then the select statement will block. ((The above code causes a lot of messages to be printed\!))

**If more than one channel is ready,** go will select at random a branch of the select statement to execute.

`time.Ticker` creates a ticker object with `ticker.C` being a channel which fires (sending a time object) every specified period.

**Beware of using `time.After`** in a select statement**.** Each time `time.After` is called, a new channel is created which will send a message after the specified time.

```golang
func main() {
    …
    for {
        select {
            case <-quit:
            …
            case <-time.After(500*time.Millisecond): //every 500ms 
                … //will never run!
            default:
                log.Println(“Nothing to do!”)
        }
    }
}
```

The above code will never enter the second select case\! This is because we continually loop into the default case, which re-runs the `time.After` line of the code, which **recreates** the channel, effectively **restarting the timer.**

Instead, it is preferable to use `time.Ticker` in these cases.  


[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,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KT2dJGkIf/vb3+Lss8+OW265Jf70pz/F7bffHueff34MGTIkJk6cSJLUydPRHn/88XHFFVfEbbfdFqeffnpcfvnlQUdM53rPPfek/GftXMnzkpKNpKNjP/XUU1Me5H/ZZZfF9ddfH2PGjIkzzzwzCZLBgwen8//617/i448/jmuuuSbVkzrccMMNcdxxx8Udd9yRbMI48iUf0l111VXJrnJdqBtpCAgaRNGRRx6Z8r/55puTvRdeeGFi9vbbb6drSUP6ckDkYDPii3qW4/2UgAQkUCQCCocitUYBbWFkf91118XKK68chx9+ePzyl7+MH//4xzFq1Kh44oknAs/AXXfdFXT2u+22Wxx99NFx7LHHxu677x508HgoqBYdOSNzOmhG2gMHDoxjjjkmfvCDH6QR+4QJE0gWL774Ytx4442xwQYbxM9//vM46qijYtttt42HHnooeTrolOlgl1lmmZh//vnTNfzAg4C3oRxPOXgEymkPPPDA2G+//ZL4oCxs32abbeKggw6KPn36JBFz7733Rr9+/ZrrsPPOOwd1w8NCGQgVPAd4KLp37x6HHXZY/OIXv4gNN9wweR5IQ4ePZ+Pqq69Ofz3ziCOOSHXYYYcdEqORI0dGjx49Yt5554233nqLS1LgOgQLf311k002iXnmmSfF+0MCEpBA0QgoHNqjReo4DwRB165d44c//GGstNJKscQSS8T3vve96N+/fyy++OLJ7U8a4ugcF1tssVh44YVjq622iiWXXDJNb9Dh4oKnM+/Zs2fKa7XVVotFF100ll566SD/Ll26pOmCYcOGpet23HHHWH755VMnu+WWWwaihM4a7wZ5rbLKKjNRf/311wNxgsDhxGuvvcZHrLHGGoEAWGeddWLVVVdN9iJ2tttuuyRI1l577UBI4HVAxGA39VpkkUVi6623jizLAg8BmVE2QoS8dtppp1hxxRWT/aQtd/TkxVTNcsstlxhR36WWWioJpF122SXxo94wqhQOCDGmR77//e8H11KeQQISkEARCSgcitgqBbGJUTCdGZ0vHXzZLDo+RAIdKOsPWKOAcEAAlNMwoqZD5RwdNR075zbaaKPUefKd/OmUERjzzTdfmhJ45plngs6cONIQKBvh8q1vfSvwLHyTcMALgPgoCweO6czp/BEc5IOwwGOw+uqrx1prrZVEATY899xzaf0FaxeY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>

[image4]: <data:image/png;base64,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>

[image5]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAQ0AAADOCAYAAAAkGVemAAAQAElEQVR4AeydB7hUxRXHzwAGJYqICijFBzaIilgixooxajTE8pnEaCxYgl+MBaMRsMYajWKwxBJ7iYqxJHaNGv0sWCNqgkYEH0VAioIUjaDm/eZlX3Zf3WXntt0/H/N2d+7cM3P/s/u/Z86cc267r/VPCAgBIVACAu1M/4SAEBACJSAg0igBLDUVAkLATKShb4EQiBCBShQt0qjEWdU1CYEIESiKNKZNm2Zjxoyxiy66qKrLJZdcYh988EGE0yHRQiD9CBRFGkuWLLFNNtnEhg8fXtVl0KBBtnjx4vTPqkYoBCJEoCjS+OqrryIcgkQLgXoE9DcbCBRFGnVbuNm4Go1SCAiByBEoijTat28f+UDUgRAQAtlAoCjS0PIkG5OpUQqBOBAoijS0PIljKkrrQ62FQFIIFEUaSQ1O/QoBIZA+BEQa6ZsTjUgIpBoBkUaqp0eDEwLpQ6AqSCN9sGtEQiC7CIg0sjt3GrkQSASBREmDXZnZs2fb008/bffcc4+NHz/e5s+f74H47LPPbMKECbZw4UL797//bZMmTfL1Sf9Zvnx50kNQ/0IgUQQSJY3JkyfbCSecYOPGjbM33njDrrrqKjvvvPNswYIFnjz4PGXKFHv99df98USRUudCQAh4BJqQhq+N6c8TTzxhBMNdfvnldv755xtRpJ9//rnV1tYWjGDzzTe3zTbbzNdxHIJ55JFHvCbiK+v+LKgjmhdeeMGQOX369Loa/RcCQiAKBBIljQ022MD4gd91111++dG1a1cbOXKk1dTUFFzrc889Zy+++KItW7bM7rvvPrvhhhvsX//6lw/Xf/TRR+3jjz+2P/7xj/bYY4/ZxIkTbfTo0fb2228XyNAHISAEwiCQKGnsvPPOdtZZZ9n9999vP/7xj23fffe1l156yTp37mz5/7BzQAxLly61xx9/3Pbee287+uijbf/99/fkAUG8++67duSRR9oRRxxhQ4YM8YSSL0PvhYAQCINAoqThnPNEgfbw0EMP2fe//30755xz7N5772326iCNefPm2cCBAz2xQDInnXSSvfPOO/bmm2/ab37zGzv++OMN7QMSQTNpVpAqhUBSCFRAv4mSxplnnmnYJpxz1qtXLzv22GO9xvHUU0+1CC07Lv/5z3/8cXZWIIuOHTvagAEDfIKgY445xiinn366tWuX6OX5MeqPEKg0BBL9VfXp08euvfZae+CBB+zll1+2v/3tb/b+++/bjjvu2CzOq666qmHvwNiJdnHdddfZ2LFjfVYxCIKsWqussoqXxXGF9DcLoyqFQFkIJEoaBx98sF+ePPnkk3b33Xd7f41dd93V9tlnH1t55ZX9jsnqq69uGEz79etnnTp1skMOOcTmzJnj7SCQxKhRozxpDB061NtDWOqwI8PnspDRyUJACDSLQKKkgcHz0EMP9TsmLCmOO+44+9nPfmZoFGussYYddNBBftmCrWO33XYzNIdtttnGL0MOPPBAo33//v3tm9/8picaDKGQCnYOlivNXrEqs4OARppKBBIlDRBZaaWVPDGsv/761rNnT8M+QT0Eseaaa9o3vvENQ9uAYHL1a6+9tvXt29c4Th2F8zgfjYStW+pUhIAQCI9A4qQR/pIkUQgIgSgREGlEia5kC4EKRCAW0iDIC/+KKPD78ssvvUcoW7FRyM+MTA1UCMSEQCykMWvWLGOXI4prmjt3rl1wwQXexTwK+ZIpBIRAIQKxkAZbowSTFXYd5hMh9K+++qopY3oYPCVFCLSFQCykkRvEK6+84jUOnLLQEKjnx/7888/7bdff/e53hlaSq8fbk9iUE0880YfP55YgU6dONdqedtpplv9sVZYquJCPGDHCP3OWYDhkEdyG5+lll11mV199tS1atIhqFSEgBFYAgdhIA9dvIlEJc4ckbr/9diPM/cEHH7QLL7zQtt56a+/2DUHgHl5bW2sXX3yxbbjhhrbTTjvZpZde6hP1zJw5078nDoVzIA+uG7sJ8nES22OPPbyvB67kJO957733DJd1vE3Z4u3YsSOnlFwgLUrJJ+oEIVBBCMRGGvzYzj33XMMpa6+99jIMo5AGd/+tttrKunfvbltuuaV99NFHPgwevwscvQhOw++id+/ePoPXhx9+aGgUw4YNs/3228/QNpgPMoARIYsMnL3w+8DXg2xgHF9ttdXs97//vR111FHe94O6UgvEB1mVep7aC4FKQiA20sBJq0ePHh47CIE3EAlLCGweJNYhxH377bf37uLYKrCDEJfCMoUfK2QB0XAuxMDrRhttxIshg6hWSAVZxJ7gAEYgHA1w/OrQoQNvV7gQ34IL+woL0IlCoAIQiI00+ME555pAtt566/kl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>

[image6]: <data:image/png;base64,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>