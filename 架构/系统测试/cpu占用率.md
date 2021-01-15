## 代码

* cpu_load.c

```c++
#include <iostream>
#include <pthread.h>
#include <time.h>
#include <math.h>
#include <unistd.h>

using namespace std;

typedef long long int int64;
const int NUM_THREADS = 1; //CPU core nums
int INTERVAL = 100;
int cpuinfo = 15; //CPU utilization rate

// time unit is "ms"
int64 GetTickCount()
{
    timespec now;
    int64 sec, nsec;

    clock_gettime(CLOCK_MONOTONIC, &now);
    sec = now.tv_sec;
    nsec = now.tv_nsec;

    return sec * 1000 + nsec / 1000000;
}

void* CPUCost(void *args)
{
    int busyTime = INTERVAL * cpuinfo / 100;
    int idleTime = INTERVAL - busyTime;
    int64 startTime = 0;

    std::cout << "XXXX CPUCost" << std::endl;
    std::cout << "XXXX cpuinfo = " << cpuinfo << std::endl;

    /*
     * within INTERVAL ms, INTERVAL = busyTime + idleTime,
     * spend busyTime ms to let cpu busy,
     * spend idleTime ms top let cpu idle
     */
    while (true) {
        startTime = GetTickCount();
        while((GetTickCount() - startTime) <= busyTime);
        usleep(idleTime * 1000);
    }
}

int main(int argc, char **argv)
{
    pthread_t t[NUM_THREADS];
    int ret;

    std::cout << "please input cpu utilization rate" << std::endl;
    std::cin >> cpuinfo;
    for(int i = 0; i < NUM_THREADS; i++) {
        ret = pthread_create(&t[i], NULL, CPUCost, NULL);
        if(ret)
            std::cout << "XXXX create err" << std::endl;
    }

    pthread_exit(NULL);
    return 0;
}
```

* 编译

```shell
g++ cpu_load.c -lpthread -lrt -o cpu_load
```

* 执行

```shell
./cpu_load
# 输入比例
```

