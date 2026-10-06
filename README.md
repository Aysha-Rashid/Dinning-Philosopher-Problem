# Dining Philosophers

A multithreaded implementation of the **Dining Philosophers Problem** written in **C** as part of the 42 curriculum.

The project focuses on **POSIX threads, mutexes, thread synchronization, shared resources, timing, race-condition prevention, and concurrent programming**.

---

## Overview

The simulation represents philosophers sitting around a table with one fork between each philosopher.

Each philosopher repeatedly:

```
        ┌───────────┐
        │  Thinking │
        └─────┬─────┘
              │
              ▼
        ┌───────────┐
        │   Eating  │
        └─────┬─────┘
              │
              ▼
        ┌───────────┐
        │  Sleeping │
        └─────┬─────┘
              │
              └──────────► Thinking
```

To eat, a philosopher must acquire both the left and right forks.

Each fork is protected by a **mutex**, preventing multiple philosophers from accessing the same fork simultaneously.

---

## Features

### Multithreading

Each philosopher runs in its own POSIX thread:

```
Philosopher 1 ──► Thread 1
Philosopher 2 ──► Thread 2
Philosopher 3 ──► Thread 3
       ...
Philosopher N ──► Thread N
```

The simulation also creates a dedicated **monitor thread** that checks the state of every philosopher.

---

### Fork Management

Each philosopher has:

* One local `left_fork`
* A pointer to the next philosopher's `left_fork` as its `right_fork`

This creates the circular fork arrangement:

```
        P1
       /  \
     F1    F2
     /      \
   P4        P2
     \      /
     F4    F3
       \  /
        P3
```

Forks are implemented using:

```c
pthread_mutex_t
```

A philosopher must lock both forks before eating and unlock them afterward.

---

## Philosopher Lifecycle

The philosopher routine follows this cycle:

```
        ┌────────────┐
        │   THINKING │
        └─────┬──────┘
              │
              ▼
      Lock right fork
              │
              ▼
       Lock left fork
              │
              ▼
           EATING
              │
              ▼
       Update meal time
              │
              ▼
       Release both forks
              │
              ▼
          SLEEPING
              │
              ▼
          THINKING
```

For multiple philosophers, even-numbered philosophers initially sleep for a short period to stagger their execution and reduce contention when the simulation starts.

---

## Thread Synchronization

Several mutexes are used to protect shared resources.

### Fork mutexes

Prevent two philosophers from taking the same fork simultaneously.

```c
pthread_mutex_lock(philo->right_fork);
pthread_mutex_lock(&philo->left_fork);
```

### Output mutex

A dedicated mutex protects console output so that multiple threads do not print over each other.

```c
pthread_mutex_lock(&data->write);
```

This keeps the simulation output readable.

### Death-check mutex

Used when accessing the philosopher's last meal time and death-related state.

```c
pthread_mutex_lock(&data->death_check);
```

---

## Monitoring

A separate monitor thread continuously checks every philosopher.

```
              ┌─────────────────┐
              │  Monitor Thread │
              └────────┬────────┘
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
       Check Philosopher 1   Check Philosopher 2
             │                   │
             └─────────┬─────────┘
                       ▼
              Check remaining
               philosophers
                       │
                       ▼
              Has someone died?
                  /       \
                Yes        No
                │           │
                ▼           └──► Continue
          Stop simulation
```

The monitor compares the current time with each philosopher's `last_meal_time`.

If:

```text
current_time - last_meal_time >= time_to_die
```

the philosopher is considered dead and the simulation is stopped.

---

## Simulation Termination

The simulation can terminate when:

* A philosopher dies.
* Every philosopher has completed the required number of meals.

A shared `every_die` flag is used to notify all philosopher threads when the simulation should stop.

```c
data->resources[i].every_die = 1;
```

This allows the running threads to exit their routines cleanly.

---

## Special Case: One Philosopher

The single-philosopher case is handled separately.

A single philosopher can only take one fork and therefore can never acquire two forks to eat.

The simulation therefore produces:

```
0    1 has taken a fork
time 1 died
```

after `time_to_die` has elapsed.

---

## Time Management

The project uses:

```c
gettimeofday()
```

to obtain the current time in milliseconds.

A custom sleep function was implemented using `usleep()`:

```c
ft_sleep(size_t milliseconds)
```

Instead of relying on a single long `usleep()`, the function repeatedly checks the elapsed time.

This provides more precise timing for the simulation.

---

## Program Arguments

The program accepts:

```text
./philo number_of_philosophers time_to_die time_to_eat time_to_sleep [number_of_times_each_philosopher_must_eat]
```

Example:

```bash
./philo 5 800 200 200
```

Arguments:

| Argument                                    | Description                                      |
| ------------------------------------------- | ------------------------------------------------ |
| `number_of_philosophers`                    | Number of philosophers and forks                 |
| `time_to_die`                               | Maximum time a philosopher can go without eating |
| `time_to_eat`                               | Time spent eating                                |
| `time_to_sleep`                             | Time spent sleeping                              |
| `number_of_times_each_philosopher_must_eat` | Optional meal limit                              |

The implementation also validates the arguments and limits the number of philosophers to **200**.

---

## Output

Each action is printed with a timestamp and philosopher ID.

Example:

```text
0 1 has taken a fork
0 1 has taken a fork
0 1 is eating
200 1 is sleeping
400 1 is thinking
```

The output is protected by a mutex to prevent messages from different threads from being mixed together.

---

## Architecture

The project is separated into several source files:

```text
├── main.c
│   └── Program entry and cleanup
│
├── philo.h
│   └── Structures, prototypes and libraries
│
├── simulation.c
│   ├── Philosopher routine
│   ├── Eating
│   ├── Monitoring
│   └── Thread management
│
├── time.c
│   ├── Time calculation
│   ├── Custom sleep
│   └── Single philosopher handling
│
├── utils.c
│   ├── Argument parsing
│   ├── Initialization
│   ├── Mutex setup
│   └── Simulation termination
│
└── extra.c
    ├── ft_atoi
    ├── ft_strcmp
    ├── ft_isdigit
    └── Output handling
```

---

## Main Components

### `t_philo`

Stores the state and resources of each philosopher:

```c
typedef struct s_philo
{
    int current_philo;
    int no_of_meal;
    size_t time_to_eat;
    size_t time_to_sleep;
    size_t time_to_die;
    size_t last_meal_time;
    pthread_mutex_t left_fork;
    pthread_mutex_t *right_fork;
    pthread_t thread;
    ...
} t_philo;
```

### `t_data`

Stores information shared by the simulation:

```c
typedef struct s_data
{
    int total_philo;
    int dead_flag;
    size_t time;
    t_philo *resources;
    pthread_mutex_t write;
    pthread_mutex_t death_check;
} t_data;
```

---

## Thread Architecture

The overall execution can be represented as:

```text
                    Simulation
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
   Philosopher Threads           Monitor Thread
          │                           │
     ┌────┼────┐                      │
     ▼    ▼    ▼                      ▼
    P1   P2   P3 ... PN        Check philosopher
     │    │    │                      │
     └────┴────┴──────────────────────┘
                  │
                  ▼
             Stop condition
```

Each philosopher performs its own work concurrently while the monitor observes the state of the simulation.

---

## Technologies

* **C**
* **POSIX Threads (`pthread`)**
* **Mutexes**
* **Multithreading**
* **Thread synchronization**
* **`gettimeofday()`**
* **`usleep()`**
* **Make**
* GCC/Clang

---

## Compilation

Clone the repository:

```bash
git clone <repository-url>
cd Dinning-Philosopher-Problem-main
```

Compile:

```bash
make
```

This produces:

```text
philo
```

Clean object files:

```bash
make clean
```

Remove all generated files:

```bash
make fclean
```

Rebuild:

```bash
make re
```

---

## Running the Simulation

Example with 5 philosophers:

```bash
./philo 5 800 200 200
```

With a meal limit:

```bash
./philo 5 800 200 200 5
```

The simulation runs until a philosopher dies or all philosophers complete the required number of meals.

---

## 42 Project

The **Dining Philosophers** project is part of the **42 curriculum** and provides practical experience with concurrent programming.

The main challenge was coordinating multiple threads accessing shared resources while maintaining correct timing, synchronization, and simulation state.
