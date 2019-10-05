### OS의 효율적인 동작을 보조해주는 하드웨어

#### MMU (Memory Management Unit)
CPU가 메모리에 접근하는 것을 관리하는 하드웨어 장치
- CPU는 메모리에 논리 주소를 요구하고 중간에 MMU가 실제 메모리에 적재된 물리주소로 치환하여 메모리에 질의한다. 코드가 컴파일되면 실행되어야할 명령어들의 주소(논리주소)가 고정되어 들어가있다. 하지만 프로그램을 실행시킬때마다 같은 물리주소에 적재할 수는 없기 때문에 MMU와 같이 논리주소를 물리주소로 치환해주는 중간 장치가 있어야한다.

#### DMA (Direct Memory Access)
주변장치들(하드디스크, 그래픽 카드, 네트워크 카드, 사운드 카드 등)이 메모리에 직접 접근하여 읽거나 쓸 수 있도록 하는 기능으로서, 컴퓨터 내부의 버스가 지원하는 기능이다. 대개의 경우에 메모리의 일정 부분이 DMA에 사용될 영역으로 지정되며, DMA가 지원되면 중앙처리장치가 데이터 전송에 관여하지 않아도 되므로 컴퓨터 성능이 좋아진다.
- 프로세스는 다음과 같은 상태를 가진다
  - New: The process is being created
  - Running: Instructions are being executed
  - Waiting: The process is waiting for some event to occur **(such as and I/O completion or reception of a signal)**
  - Ready: The process is waiting to be assigned to a processor
  - Terminated: The process has finished execution
- Waiting의 상태는 어떠한 작업을 기다려야하는 상태인데 이때 기다려야하는 작업이 I/O 작업이고 CPU가 이를 계속 기다린다면 리소스 낭비가 될 것이다. 따라서 I/O작업은 DMA장치에게 맡기고 CPU는 다른 프로세스에 리소스를 할당하여 실행시키다가 DMA가 I/O작업이 끝났다고 Signal을 보냈을때 그 프로세스를 Ready상태로 바꾸어 CPU의 스케줄링에 포함되게 하면 된다.
