# Priority_Scheduler_cpp — README.md

> `psched`: a priority-based task scheduler for modern C++

![status-badge](https://img.shields.io/badge/status-active-brightgreen) ![lang-badge](https://img.shields.io/badge/C%2B%2B-17-blue) ![cmake-badge](https://img.shields.io/badge/build-CMake-informational)

`psched` provides an N-queue, M-thread scheduler with **aging** to prevent starvation. Tasks are placed on a queue based on priority; a thread pool executes from highest priority first, while aging periodically bumps long-waiting tasks.

## Features

* Multiple concurrent queues, one per priority level
* Thread pool worker execution
* Aging policy to reduce starvation
* Rich configuration via template parameters
* Collect per-task stats (waiting, burst, turnaround time)

## Getting Started

### Build (CMake)

```bash
git clone https://github.com/Kislay0/Priority_Scheduler_cpp.git
cd Priority_Scheduler_cpp
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

### Install (optional)

Generates `psched.pc` for pkg-config:

```bash
cmake --install build
# Then from another project:
g++ app.cpp $(pkg-config --cflags --libs psched)
```

### Single-header Use

Include from `single_include/` if you prefer header-only consumption:

```cpp
#include <psched/priority_scheduler.h>
```

## Example

Define a scheduler with 3 threads, 3 queues (each maintains size 100, discarding oldest), and an aging policy that marks tasks as starving after 250ms and increases their priority by 1:

```cpp
#include <iostream>
#include <psched/priority_scheduler.h>
using namespace psched;

int main(){
  PriorityScheduler<
    threads<3>,
    queues<3, maintain_size<100, discard::oldest_task>>,
    aging_policy<task_starvation_after<std::chrono::milliseconds, 250>,
                 increment_priority_by<1>>
  > scheduler;

  Task a(
    []{ std::this_thread::sleep_for(std::chrono::milliseconds(130)); },
    [](const TaskStats& s){
      std::cout << "[Task a] Waiting=" << s.waiting_time() << "ms, "
                << "Burst=" << s.burst_time() << "ms, "
                << "Turnaround=" << s.turnaround_time() << "ms\n";
    }
  );

  auto timer_a = std::thread([&]{
    while(true){
      scheduler.schedule<priority<0>>(a);
      std::this_thread::sleep_for(std::chrono::milliseconds(250));
    }
  });

  // Define and schedule tasks b and c similarly at priorities 1 and 2
  // ...

  timer_a.join();
}
```

## Concepts & Types (cheat sheet)

* `threads<N>` — size of worker thread pool
* `queues<N, maintain_size<limit, discard::oldest_task>>` — number and behavior of priority queues
* `priority<I>` — compile-time priority tag (0 = lowest)
* `aging_policy<task_starvation_after<Duration, T>, increment_priority_by<K>>` — starvation detection and promotion
* `Task` — wrap a callable plus an optional post-completion callback
* `TaskStats` — inspect `waiting_time()`, `burst_time()`, `turnaround_time()`

## Samples

See `samples/` for runnable examples and `img/` for illustrative output.

## Roadmap

* Dynamic priority ranges; runtime-configurable policies
* Cooperative cancellation & timeouts
* Benchmarks and micro-bench suite

## Contributing

Issues and PRs welcome. Please accompany changes with tests or sample updates.

<!-- ## License

*Add a license file to clarify reuse.* -->
