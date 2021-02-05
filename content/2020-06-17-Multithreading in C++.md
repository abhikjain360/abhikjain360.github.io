+++
title = "Threadpooling in C++"
date = 2020-06-17

[taxonomies]
tags = ["C++", "multithreading"]
categories = ["programming"]
+++

I wanted to use a threadpool for a certain project of mine, and there were a lot of implementations of them on github, but many of them simply pasted the code with little explanation, and there weren't many tutorials (_easy_ tutorials) that I could follow right up.

I eventually managed to make a simple one for myself, and is good enough for almost all personal projects I do. In this blog, I'll help you recreate my code, explaining stuff that someone who doesn't know much about threadpooling (and multithreading in general) can still follow. I do assume some familiarity with C++ coding, and also with some standard libraries like ```iostream``` and ```vector``` and concept of classes.

This tutorial isn't meant for people looking for super-optimized multithreaded calculations. For that there are much better libraries out there.

If you are wondering how it's different from other tutorials, well it's pretty much same really 😅. I just go in depth and try do explain every bit of code I feel new C++ programmers might not know. By the end you'll have made a simple but useful threadpool header file, which you can use practically anywhere.

You can find my code for reference [here](https://github.com/abhikjain360/threadpool).

### Why multithread?

The answer is obvious. All modern (by modern I mean by all PCs and laptops since 2005) computers have more than one core, let's use them all! It's faster, and a better use of resources. There might be some cases where you can't run processes in parallel, like the new process depending upon the output of current one. But those are the odd ones. As we'll see, multithreading can outperform simple code even for small test cases. There are many instances where algorithms by design are meant to be run in parallel (like matrix multiplication where each element of resultant matrix can be calculated independently of other, allowing us to compute then in parallel).

### Why use a threadpool?

Though your CPU can run a lot more _virtual_ threads than number of cores it has, that doesn't mean all those threads run in parallel. Each new thread waits for previous threads to finish. If there are no _physical_ threads free, it'll have to wait. Also assigning these _virtual_ threads is a rather expensive task. It often happens that if we assign threads carelessly without bothering about how much threads are we actually creating, any benefit multithreading could provide would become insignificant compared to sheer cost of assigning so many threads.

Threadpooling is creating a fixed number of threads and then re-using them again & again. As we'll see, this outperforms assigning random threads even in small test cases (which I mentioned can outperform normal code).

## Introduction

### Race Condition

Let's imagine 2 threads running is parallel, and both have access to a single variable *i*, and modify it by incrementing it by 1, without caring whether the other thread is also using *i* during the incrementing. Thus both have access to it without any bounds or checks. This is a really bad situation. Guess why?

Assume *i* has initial value 5. Let's imagine thread 1 accessed *i* and made a copy of it. At the same time thread 2 also made a copy of *i*. Then both threads incremented *i* by one making its copy in each thread equal to 6, and then putting the value back in *i*. Now the value of *i* is 6, but what we wanted to do was to each thread to increment value once, thus resulting in *i* being seven. But because threads did the increment without being aware of other threads, *i* only got incremented once. This situtation is called *race condition*, and is often not desirable.

### Mutex Locks and Atomic types

Mutex is a way to ensure that some data/resource that is being shared between threads can be used by only one thread at a time. Thus, it helps with the race condition. Basically, when a mutex lock is put on a certain part of code, that part of code can be executed by only one thread at once. By putting locks around the code where the variables shared by multiple threads are accessed/changed, we can prevent race condition.

Mutex locks can used in cpp using the ```#include <mutex>```. There are various mutex locks available in the library, and each one depends upon the use case. After initializing a mutex variable, one can different locks on it.

One of the most basic ways is to directly use the ```lock```  and ```unlock``` function of the mutex. ```lock_guard``` is also an option which unlock as soon as it goes out of scope, but it has nearly deprecated and ```scoped_lock``` is used.


```cpp
#include <mutex>

int main() {
	std::mutex mu;

	// locking using mutex's method
	mu.lock();
		// code
	mu.unlock();

	// locking using lock_guard
	{
		std::lock_guard<std::mutex> lockGuard(mu);
		// code

	} //goes out of scope here

	// locking using scoped_lock (Since C++17)
	{
		std::scoped_lock<std::mutex> scopeLock(mu);
		// code
	} //goes out of scope here

	return 0;
}
```


For our pupose, we'll use ```unique_lock```, which is very similiar to ```lock_guard``` as it is required by conditional variable in wait call (will be discussed in more detail later).

Note thread although using locks ensures thread safety, they are huge overhead. Sometimes, running on multiple threads might be slower than running on single thread if we are locking and unlocking too much. Thus, try to use locks as less often as you can, and don't do heavy things when lock is on, or else your program might loose a lot of speed while consuming much more resources.

Another way to ensure thread safety is use *atomic types*, available since C++17.  We can wrap the shared variable with an atomic type, and it'll automatically make sure that one thread accesses/modifies it at time, allowing us to write _lock-free_ code. Using atomic types, is arguably more easy and faster than using locks. But using atomic types come with it's own set of complications (like how it stores the entire variable in local cache), and we don't want to get into it now.

### Condition variables

Conditional variables allow us to control the execution and timing of threads based on certain conditions. This allows for a greater control on order of execution of commands, which would otherwise be out of control, again possibly leading to race conditions. Conditional variables allow us to send _notifications_ to threads, using the ```notify_one``` and ```notify_all``` functions.

For example, we have 2 variable ```i``` and ```j```, and we want that ```i``` gets incremented only after ```j``` is greater than 5. We can create a mutex, and put it under a lock. This lock will be under a condtional variable and wait till condition is met.

```cpp
#include <thread>
#include <mutex>
#include <conditional_variable>

int j = 0, i = 0;
std::mutex mu;
std::conditional_variable cond;

void waitForCond() {
	std::unique_lock<std::mutex> lock(mu);
	cond.wait(lock);

	i += 1;
}

void increment_j() {
	for (int k  = 0; k < 10; ++k) {
		j += 1;
		if (j > 5)
			cond.notify_one();
	}
}

int main() {
	std::thread worker1(waitForCond);
	std::thread worker2(increment_j);

	return 0;
}

```

We will use condition variables in threadpool to keep the created threads on hold until a new task has been alloted, after which we can take a task of the the task list and run it.

## Let's start!

First we'll create a project root directory, which I name threadpool (rather unimaginative, I know). In this we create a ```threadpool.h``` header file. First some headers.

```cpp
#pragma once

#include <thread>
#include <vector>
#include <queue>
```

The ```#pragma once``` at the top is to avoid including library again in projects having more than a few files. ```thread``` library allows us to access the CPU threads. We will use a vector to store our execution threads, and a queue to store our tasks as functions.

Let's create a class named threadpool. We define an alias for function types, and call it ```Task```. In order to define ```Task```, we need to include ```functional``` library as well.

```cpp
#pragma once

#include <thread>
#include <vector>
#include <queue>
#include <functional>

class threadpool {
  public:
	using Task = std::function<void()>;

  private:
	std::vector<std::thread> threads;
	std::queue<Task> tasks;
};
```

Creating a execution thread is rather simple. We can create one by writing

```cpp
std::thread(function_pointer, args...)
```

where ```function_pointer``` is the pointer to the function you want to give to threadpool, and corresponding arguments. A point to keep in mind is that when creating a thread, it'll _move_ the non-primitive data types, meaning you no longer would be able to access them. So if there is an variable/object you want to use again, you have to pass a reference to that object using ```std::ref()```.

Whenever we run a C++ application, we atleast use 1 thread, executing the code in the ```main``` function, one line at a time. This is called _main threads_. When we run a C++ application which uses multiple threads, there will still be 1 main thread, which starts when the application starts, and stops when the application stops. Now this main thread runs independent of other threads that we create during the execution of the application, called _child threads_. But as main function runs on independent thread, it won't wait for other threads to finish, and terminate the application, unless spefically asked to wait for other threads to finish.

We can ask main thread to wait for other threads by using ```join()```. This can be done by saving the construction of ```std::thread``` in a variable, and then joining it.

```cpp
std::thread worker(function_pointer, args...);
worker.join();
```

So in our constructor of threadpool, we need to create a bunch of threads, push them in the ```std::vector<std::thread> threads```, and in our deconstructor we need to join all these vectors. We'll also create a function to add Tasks at the back queue ```tasks``` and pop out tasks from front, giving it to each thread.

One question remains, that when we create threads to push back in ```threads``` vector, what function and arguments are we going to pass to constructor of the ```std::thread```?

We can not simply pass the task in front of queue ```tasks```, for it'll create a thread for each task. That is no different than how one will use thread without a threadpool. Instead, we'll pass a function, which on finding the queue of tasks non-empty, executes the task at front of the queue, and then pops it off so that it doesn't get executed more than once by different threads, and keeps on running until the threadpool object is not destroyed. That's it. This is the most important part of threadpooling.

Another thing to keep in mind is to avoid a problem which comes with using multiple threads at once, the problem of data races. But we will for now pretend that such a problem doesn't exist, and simply focus on making our threadpool class. Note that threadpool won't be usable until be actually use mutex and remove possibility of data races.

Also from now on instead of writing entire program when we update our code, I'll just put the updated part with enough context to let you know where to put the code, to make things easy for both of us.

### The thread_manager function

Before we write the constructor and deconstructor for the threadpool class, we'll write the function to pass onto the threads we create. This function will execute functions from the tasks queue, as mentioned before. We also want to be able to stop thread when we destroy the threadpool object, we'll create a new variable ```bool stopvar``` which will keep track of when to stop the thread. We will set it to ```false``` when creating threadpool.

```cpp
class threadpool {
  private:
	void thread_manager();

	std::vector<std::thread> threads;
	std::queue<Task> tasks;
	bool stopvar;
};

void threadpool::thread_manager() {
	while (true) {
		Task task;
		{
			if (stopvar && tasks.empty()) break;

			task = std::move(tasks.front());
			tasks.pop();
		}
		task();
	}
}

```
