# Project1 : Implementing a simple scheduler

# Design

## 1. FCFS

Round Robin과 FCFS를 비교해보면, RR은 기본적으로 FCFS를 기반으로 time quantum이 추가된 것으로 볼 수 있으므로,

기존의 RR방식에서 timer interrupt를 통한 context switching이 일어나지 않게만 바꿔주면 FCFS가 된다.

- 이를 위해 처음에는 scheduler() 함수 내의 intr_on()을 off로 바꾸어야겠단 생각이 들었지만,  그렇게 되면 timer interrupt 뿐만이 아니라 모든 interrupt가 비활성화되어 사용자입력이나 디스크I/O까지 비활성화시켜버린다는 것을 깨달았다.
- 따라서 다음 방법이 적합하다고 생각되었다. 1tick이 증가할 때마다 trap.c의 usertrap()과 kerneltrap()에서 yield하여 순서를 넘겨 주게 되는데, FCFS모드와 MLFQ모드로 분기하여 **FCFS모드에서는 yield를 하지 않도록** 하는 방법이다.
    
    (여기까지는 기존의 PCB를 수정하지 않아도 됐다.)
    

## 2. MLFQ

`proc.h` 에 다음 내용을 추가한다.

- `struct mlfq` 프로세스를 담을 큐 구조체가 필요했다.
    - 여기에는
    - 5개의 큐가 모두 이 구조체를 사용하며, front, rear는 L2 우선순위큐를 제외한 일반 큐만 사용합니다.
    - size는 큐 내의 프로세스 갯수를 의미합니다.
    - 그리고 L2큐만 인덱스 1부터 시작하며, 나머지 큐들은 인덱스 0부터 시작합니다.

- 큐를 옮길 때마다 프로세스의 다음 정보들이 바뀐다.
    
    프로세스가 가져야 할 정보이므로 struct proc에 다음 변수들을 추가했다.
    
    - `priority` 프로세스마다 우선순위가 존재
    - `ticks` 프로세스가 해당 큐에서 소비한 시간 (다른 큐로 옮기면 초기화됨)
    - `lev` 프로세스가 속한 큐를 알 수 있다. (L0에 있으면 0, L1이면 1, L2면 2)

# Implementation

먼저 큐 입출력 함수는 `inqueue` `outqueue` `exitqueue` 세 가지입니다.

공통점은 모두 인자로 받은 큐를 `L2 우선순위 큐`  / `나머지 일반 큐` 의 경우로 나누어 다르게 처리합니다. L2큐의 경우 큐에서 빼거나 삽입할 때 우선순위를 정렬해 max heap으로 heapify하는 과정을 포함합니다.

이 때 outqueue와 exitqueue의 차이점은, 

outqueue는 Round Robin 방식 스케줄링을 위해 사용하는 함수로 인자로 큐포인터를 받으면 L2큐는 우선순위가 가장 높은 프로세스를, 나머지 큐에서는 맨 앞 프로세스를 반환합니다. (dequeue와 유사)

exitqueue는 인자로 받은 특정 프로세스를 찾아 큐에서 빼는 함수로, timequantum을 모두 사용하고 큐를 옮길 때, setpriority에서 priority가 바뀔 때 exitqueue로 뺐다 넣어줌으로서 heapify를 해야할 때필요하여 만들었습니다.

1. `proc.c`

스케줄러가 fcfs모드인지 mlfq모드인지는 전역변수 ismlfq의 0,1로 구분합니다.

proc.c 내에서도 ticks 변수를 사용하기 위해 extern으로 가져와줍니다.

# Results

vscode에서 WSL환경을 연동하여 진행하였습니다.

![image](https://github.com/user-attachments/assets/fb5620a5-b18e-465a-bd7b-3ed757f3a5f7)


![image](https://github.com/user-attachments/assets/c92b88eb-3f52-4521-adf7-c1210b762797)


xv6부팅 후 test를 실행하면 다음과 같이 출력됩니다.

![image](https://github.com/user-attachments/assets/14e6864c-6569-4b6c-afed-7d412e018c3b)


- [Test 1] FCFS모드에서는 각 프로세스가 순서대로 모두 실행됨을 확인할 수 있습니다.
- [Test 2] MLFQ에서 timequantum은 각각 L0,L1가 1,3인데, 각 프로세스들이 큐에 머무는 시간이 이와 비례해서 나타남을 확인할 수 있었습니다. (L2큐에서 나머지가 모두 완료되고, L0~L2의 총합은 100000)

# Trouble shooting

1. defs.h에 선언한 inqueue 함수의 인자로 받는 struct mlfq가 undefined 되었다는 오류

defs.h 상단에 구조체 선언을 해주어야 했는데 이를 빠뜨려 발생한 오류였습니다.

![image](https://github.com/user-attachments/assets/fd94be09-b5a7-4a64-ac51-fcf38108958b)


1. 디버깅 메시지 중 Found RUNNABLE process in L0: pid=3 에서 멈추었는데, 

 fcfsmode함수에서 front,rear값을 초기화해주지 않아 발생한 문제였습니다.

따라서 다음 코드를 추가하고,

```c
  L0->front = 0;
  L0->rear = -1;
  L0->size = 0; // L1,L2도 추가
  
```

`exit()` 함수에서 종료된 프로세스를 MLFQ 큐에서 제거하기 위해이해 `exitqueue()`를 호출하여 종료된 프로세스를 큐에서 제거합니다.

```c
// MLFQ 큐에서 프로세스 제거
  if (ismlfq == 1)
  {
    exitqueue(p);
  }
```

1. lock을 잘못된 위치에서 획득하거나 풀어주어 발생한 문제
    
    불필요한 곳에 acquire lock을 하면, deadlock이 발생하거나 성능이 느려질 수 있고,
    
    lock이 필요한 곳에 누락이 된다면 critical section problem이 발생할 수 있었습니다.
    
    캡쳐하지는 못했지만 적절한 곳에 lock을 획득하고 풀어줌으로써 해결하였습니다.
