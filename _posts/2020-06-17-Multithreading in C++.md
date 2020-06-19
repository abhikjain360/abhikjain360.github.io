---
layout: post
title: "Threadpooling in C++"
author: "Abhik Jain"
---

I wanted to use a threadpool for a certain project of mine, and there were a lot of implementations of them on github, but most of them were not simple, and there weren't many tutorials (_easy_ tutorials) that I could follow right up.

I eventually managed to make a simple one for myself, and is good enough for almost all personal projects I do. In this blog, I'll help you recreate my code, explaining stuff that someone who doesn't know much about threadpooling (and multithreading in general) can still follow. I do assume some familiarity with C++ coding, and also with some standard libraries like ```iostream``` and ```vector``` and concept of classes.

This tutorial isn't meant for people looking for super-optimized multithreaded calculations. For that there are much better libraries available, like one [here](https://github.com/oneapi-src/oneTBB).

If you are wondering how it's different from other tutorials, well it's pretty much same really 😅.

I just go in depth and try do explain every bit of code I feel new C++ programmers might not know. By the end you'll have made a simple but useful threadpool header file, which you can use practically anywhere. Your imagination is the limit!

You can find my code for reference [here](https://github.com/abhikjain360/threadpool).

### Why multithread?

The answer is obvious. All modern (by modern I mean by all PCs and laptops since 2005) computers have more than one core, let's use them all! It's faster, and a better use of resources. There might be some cases where you can't run processes in parallel, like the new process depending upon the output of current one. But those are the odd ones. As we'll see, multithreading can outperform simple code even for small test cases.

### Why use a threadpool?

Though your CPU can run a lot more _virtual_ threads than number of cores it has, that doesn't mean all those threads run in parallel. Each new thread waits for previous threads to finish. If there are no _physical_ threads free, it'll have to wait. Also assigning these _virtual_ threads is a rather expensive task. It often happens that if we assign threads carelessly without bothering about how much threads are we actually creating, any benefit multithreading could provide would become insignificant compared to sheer cost of assigning so many threads.

Threadpooling is creating a fixed number of threads and then re-using them again & again. As we'll see, this outperforms assigning random threads even in small test cases (which I mentioned can outperform normal code).

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
std::thread t(function_pointer, args...);
t.join();
```

So in our constructor of threadpool, we need to create a bunch of threads, push them in the ```std::vector<std::thread> threads```, and in our deconstructor we need to join all these vectors. We'll also create a function to add Tasks at the back queue ```tasks``` and pop out tasks from front, giving it to each thread.

One question remains, that when we create threads to push back in ```threads``` vector, what function and arguments are we going to pass to constructor of the ```std::thread```?

We can not simply pass the task in front of queue ```tasks```, for it'll create a thread for each task. That is no different than how one will use thread without a threadpool. Instead, we'll pass a function, which on finding the queue of tasks non-empty, executes the task at front of the queue, and then pops it off so that it doesn't get executed more than once by different threads, and keeps on running until the threadpool object is not destroyed. That's it. This is the most important part of threadpooling.

Another thing to keep in mind is to avoid a problem which comes with using multiple threads at once, the problem of data races. But we will for now pretend that such a problem doesn't exist, and simply focus on making our threadpool class. Note that threadpool won't be usable until be actually use mutex and remove possibility of data races.

Also from now on instead of writing entire program when we update our code, I'll just put the updated part with enough context to let you know where to put the code, to make things easy for both of us.

### The ```thread_manager``` function

Before we write the constructor and deconstructor for the threadpool class, we'll write the function to pass onto the threads we create. This function will execute functions from the tasks queue, as mentioned before. We also want to be able to stop thread when we destroy the threadpool object, we'll create a new variable ```bool stopvar``` which will keep track of when to stop the thread. We will set it to ```false``` when creating threadpool.

```cpp
class threadpool {
  private:
	void thread_manager();

	std::vector<std::thread> threads;
	std::queue<Task> tasks;
	bool stopvar;
};

inline void threadpool::thread_manager() {
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
