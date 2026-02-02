# GPU_MIG_TEST
NVIDIA GPU MIG(Multi-Instance GPU) 기능 테스트

### (1) 테스트 환경
```text
GPU = NVIDIA RTX PRO 6000 Blackwell Max-Q Workstation Edition
VRAM = 96GB
nvidia-smi driver version = 580.105.08
공식홈페이지 상 MIG 기능 탑재 명시됨. 
( https://www.nvidia.com/ko-kr/products/workstations/professional-desktop-gpus/rtx-pro-6000-max-q/ )
```

### (2) GPU설치 서버 내 등록된 CUDA 환경.
```text
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA RTX PRO 6000 Blac...    On  |   00000000:17:00.0 Off |                  Off |
| 30%   34C    P8             15W /  300W |     120MiB /  97887MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            1648      G   /usr/lib/xorg/Xorg                       42MiB |
|    0   N/A  N/A            1849      G   /usr/bin/gnome-shell                     23MiB |
+-----------------------------------------------------------------------------------------+
```

### (3) MIG.M 항목 확인.
N/A 라고 되어있음.
```bash
$ nvidia-smi --query-gpu=driver_version,name,mig.mode.current
580.105.08, NVIDIA RTX PRO 6000 Blackwell Max-Q Workstation Edition, [N/A]
```

### (4) Display 모드가 아닌 Compute 모드인지 확인. (MIG 모드는 Display 모드가 아닌 Compute 모드에서 가능하다고 함.)
* 현재 어떤 Compute 모드인지 확인. (= Default)
```bash
$ nvidia-smi --query-gpu=compute_mode
compute_mode
Default
```
```text
"0: Default" means multiple contexts are allowed per device.
"1: Exclusive_Thread", deprecated, use Exclusive_Process instead
"2: Prohibited" means no contexts are allowed per device (no compute apps).
"3: Exclusive_Process" means only one context is allowed per device, usable from multiple threads at a time.

```

* 현재 gpu가 모니터와 연결되어있는지 확인. (= NO)
```bash
$ nvidia-smi --query-gpu=display_attached
display_attached
No
```

* 현재 GPU가 화면을 그리기 위해 메모리 공간을 할당하고 작동 중인가? (= NO)
```bash
$ nvidia-smi --query-gpu=display_active
display_active
Disabled
```

* 현재 지속성 모드 중인가? (=YES)

지속성 모드란?
보통 리눅스 드라이버는 GPU를 사용하는 프로세스(프로그램)가 없으면 리소스를 아끼기 위해 드라이버를 메모리에서 내리고 GPU를 절전 상태로 보냅니다. 하지만 다시 프로그램을 실행할 때마다 드라이버를 새로 로드하느라 약 1~2초 정도의 **지연 시간(Latency)**이 발생

```bash
$ nvidia-smi --query-gpu=persistence_mode
persistence_mode
Enabled
```

* 어떤 주소 지정 모드로 되어있는가? (= HMM)
```bash
$ nvidia-smi --query-gpu=addressing_mode
addressing_mode
HMM
```
```text
GPU가 컴퓨터의 메인 메모리(RAM)에 있는 데이터에 접근하는 방식입니다.

HMM (Heterogeneous Memory Management): 소프트웨어 방식. CPU의 페이지 테이블(메모리 지도)을 GPU가 거울처럼 복사해서 가져옵니다.

ATS (Address Translation Services): 하드웨어 방식. CPU와 GPU가 하나의 메모리 지도를 같이 씁니다. 중간 복사 과정이 없어서 더 효율적이고 빠릅니다.

None: 위 두 기능이 꺼진 상태로, 일반적인 비디오 메모리 할당 방식을 사용합니다.

왜 쓰나요? malloc 같은 일반적인 명령어로 할당한 CPU 메모리를 GPU가 자기 것처럼 편하게 쓰게 하여, 복잡한 데이터 전송 코드(cudaMemcpy 등)를 줄이기 위해 사용합니다.
```

```text
워크스테이션용 RTX 6000 시리즈에서 MIG를 활성화하려면, 단순히 화면 출력을 안 쓰는 것을 넘어 하드웨어 설정 자체가 "데이터센터 전용(Compute-only)" 모드로 완전히 넘어가야 함.

현재 Compute_mode: Default는 그래픽과 연산을 모두 수행할 수 있는 상태를 의미하며, 이 상태에서는 MIG가 Not Supported로 뜬다고 한다.

nvidia-smi 설정만으로는 하드웨어 레벨의 모드를 바꿀 수 없음. 
NVIDIA에서 배포하는 Display Mode Selector Tool이 반드시 필요.
(https://developer.nvidia.com/display-mode-selector-tool-home 에서 다운로드 가능. nvidia 회원가입 후 가능.)
```

### (5) Display Mode Selector Tool 설치 및 실행.
다운로드 완료 후 실행 권한 부여 후 실행

```bash
$ chmod +x displaymodeselector

$ sudo ./displaymodeselector --gpumode

NVIDIA Display Mode Selector Utility (Version 1.72.0)
Copyright (C) 2015-2025, NVIDIA Corporation. All Rights Reserved.


WARNING: This operation updates the firmware on the board and could make
         the device unusable if your host system lacks the necessary support.

Are you sure you want to continue?
Press 'y' to confirm (any other key to abort):
y
Select a number:
<0> physical_display_enabled_256MB_bar1
<1> physical_display_disabled
<2> physical_display_enabled_8GB_bar1

Select a number (ESC to quit):
1

Specifed GPU Mode "physical_display_disabled"


Update GPU Mode of all adapters to "physical_display_disabled"?
Press 'y' to confirm or 'n' to choose adapters or any other key to abort:
y

Updating GPU Mode of all eligible adapters to "physical_display_disabled"


Write InfoROM object succesfully
Successfully updated GPU mode to "physical_display_disabled" ( Mode 4 ).

A reboot is required for the update to take effect.


```

```text
각 옵션 상세 설명
<0> physical_display_enabled_256MB_bar1 (현재 기본값)
의미: 그래픽 출력 포트(DP 포트)를 활성화하고, CPU가 GPU 메모리에 접근하는 통로(BAR1)를 256MB로 설정합니다.
용도: 일반적인 워크스테이션 환경에서 모니터를 연결해 화면을 볼 때 사용합니다.
상태: 이 모드에서는 MIG 기능을 사용할 수 없습니다.


<1> physical_display_disabled (MIG 활성화를 위해 선택해야 할 모드)
의미: 그래픽 출력 포트를 완전히 차단하고, BAR1 크기를 확장(64GB 또는 128GB)하여 연산 효율을 높입니다.
용도: MIG(Multi-Instance GPU) 사용, 가상 GPU(vGPU) 환경, 또는 순수 연산(Compute) 목적으로 GPU를 사용할 때 선택합니다.
주의: 이 번호를 선택하고 재부팅하면 해당 GPU에 연결된 모니터는 화면이 나오지 않습니다.


<2> physical_display_enabled_8GB_bar1
의미: 디스플레이 포트를 유지하면서 BAR1 크기만 8GB로 확장합니다.
용도: 방송 송출, 가상 프로덕션 등 화면 출력과 고성능 데이터 전송이 동시에 필요한 특수 환경에서 사용합니다.
상태: 이 모드 역시 일반적인 Graphics 모드의 변형이므로 MIG 지원과는 거리가 멉니다.
```

### (6) Reboot 이후 적용되었는지 확인.
```bash
$ nvidia-smi
Mon Feb  2 08:15:54 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA RTX PRO 6000 Blac...    On  |   00000000:17:00.0 Off |                  Off |
| 30%   31C    P8             14W /  300W |      46MiB /  97887MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            2019      G   /usr/lib/xorg/Xorg                       10MiB |
|    0   N/A  N/A            2154      G   /usr/bin/gnome-shell                     17MiB |
+-----------------------------------------------------------------------------------------+
```

N/A -> Disabled 로 변경된 것 확인. 이제 MIG 사용가능할 것 같다.

### (7) MIG 적용

```bash
$ nvidia-smi -i 0 --query-gpu=mig.mode.current --format=csv
mig.mode.current
Disabled
$ sudo nvidia-smi -i 0 -mig 1
[sudo] password for bigdata:
Enabled MIG Mode for GPU 00000000:17:00.0
All done.
$ nvidia-smi -i 0 --query-gpu=mig.mode.current --format=csv
mig.mode.current
Enabled
```

```bash
$ nvidia-smi
Mon Feb  2 08:21:00 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA RTX PRO 6000 Blac...    On  |   00000000:17:00.0 Off |                  Off |
| 30%   31C    P8             18W /  300W |      46MiB /  97887MiB |     N/A      Default |
|                                         |                        |              Enabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| MIG devices:                                                                            |
+------------------+----------------------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |              Shared Memory-Usage |        Vol|        Shared         |
|      ID  ID  Dev |                Shared BAR1-Usage | SM     Unc| CE ENC  DEC  OFA  JPG |
|                  |                                  |        ECC|                       |
|==================+==================================+===========+=======================|
|  No MIG devices found                                                                   |
+-----------------------------------------------------------------------------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            2019      G   /usr/lib/xorg/Xorg                       10MiB |
|    0   N/A  N/A            2154      G   /usr/bin/gnome-shell                     17MiB |
+-----------------------------------------------------------------------------------------+
```

사용 가능한 인스턴스 리스트 확인
```bash
$ nvidia-smi mig -lgip
+-------------------------------------------------------------------------------+
| GPU instance profiles:                                                        |
| GPU   Name               ID    Instances   Memory     P2P    SM    DEC   ENC  |
|                                Free/Total   GiB              CE    JPEG  OFA  |
|===============================================================================|
|   0  MIG 1g.24gb         14     4/4        23.62      No     46     1     1   |
|                                                               1     1     0   |
+-------------------------------------------------------------------------------+
|   0  MIG 1g.24gb+me      21     1/1        23.62      No     46     1     1   |
|                                                               1     1     1   |
+-------------------------------------------------------------------------------+
|   0  MIG 1g.24gb+gfx     47     4/4        23.62      No     46     1     1   |
|                                                               1     1     0   |
+-------------------------------------------------------------------------------+
|   0  MIG 1g.24gb+me.all  65     1/1        23.62      No     46     4     4   |
|                                                               1     4     1   |
+-------------------------------------------------------------------------------+
|   0  MIG 1g.24gb-me      67     4/4        23.62      No     46     0     0   |
|                                                               1     0     0   |
+-------------------------------------------------------------------------------+
|   0  MIG 2g.48gb          5     2/2        47.38      No     94     2     2   |
|                                                               2     2     0   |
+-------------------------------------------------------------------------------+
|   0  MIG 2g.48gb+gfx     35     2/2        47.38      No     94     2     2   |
|                                                               2     2     0   |
+-------------------------------------------------------------------------------+
|   0  MIG 2g.48gb+me.all  64     1/1        47.38      No     94     4     4   |
|                                                               2     4     1   |
+-------------------------------------------------------------------------------+
|   0  MIG 2g.48gb-me      66     2/2        47.38      No     94     0     0   |
|                                                               2     0     0   |
+-------------------------------------------------------------------------------+
|   0  MIG 4g.96gb          0     1/1        95.00      No     188    4     4   |
|                                                               4     4     1   |
+-------------------------------------------------------------------------------+
|   0  MIG 4g.96gb+gfx     32     1/1        95.00      No     188    4     4   |
|                                                               4     4     1   |
+-------------------------------------------------------------------------------+
```
* 용어 의미
```text
MIG [n]g.[m]gb: 인스턴스의 크기.

n (Compute Slice): GPU 연산 유닛(SM)을 몇 조각으로 나눴는지 나타냅니다.

m (Memory Size): 할당된 VRAM 용량입니다.

ID: 프로필 고유 번호입니다. 인스턴스를 생성할 때(-cgi [ID]) 사용합니다.

Instances Free/Total: 생성 가능한 최대 개수와 현재 남은 개수입니다.

SM: 연산 유닛(Streaming Multiprocessor) 개수입니다. (숫자가 클수록 연산이 빠름)

DEC / ENC: 비디오 디코더와 인코더 엔진 개수입니다.

JPEG / OFA: JPEG 변환 엔진과 Optical Flow Accelerator(움직임 추적 엔진) 개수입니다.
```

* 접미사/접두사 의미
```text
+me (Media Extensions): 비디오 인코딩/디코딩, JPEG 엔진 등 미디어 가속기를 포함한 프로필입니다. 영상 처리 작업에 유리합니다.

+gfx (Graphics): 그래픽 렌더링 관련 기능을 포함합니다.

-me: 미디어 가속기를 제외한 프로필입니다. 오직 연산(CUDA)에만 집중할 때 사용하며, 남는 미디어 엔진을 다른 인스턴스에 몰아줄 수 있습니다.

+me.all: 해당 GPU가 가진 모든 미디어 가속 성능을 이 하나의 인스턴스에 몰아넣겠다는 뜻입니다.
```

GPU 사용 중인 프로세스 삭제 후 인스턴스 생성. (점유중인 프로세스가 있으면 생성 불가.).
```bash
$ sudo nvidia-smi mig -cgi 14 -C
Unable to create a GPU instance on GPU  0 using profile 14: In use by another client
Failed to create GPU instances: In use by another client
```

현재 Xorg, gnome-shell 이 점유 하고 있음.
```bash
$ nvidia-smi
Mon Feb  2 08:54:22 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA RTX PRO 6000 Blac...    On  |   00000000:17:00.0 Off |                  Off |
| 30%   31C    P8             14W /  300W |      46MiB /  97887MiB |     N/A      Default |
|                                         |                        |              Enabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| MIG devices:                                                                            |
+------------------+----------------------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |              Shared Memory-Usage |        Vol|        Shared         |
|      ID  ID  Dev |                Shared BAR1-Usage | SM     Unc| CE ENC  DEC  OFA  JPG |
|                  |                                  |        ECC|                       |
|==================+==================================+===========+=======================|
|  No MIG devices found                                                                   |
+-----------------------------------------------------------------------------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            2019      G   /usr/lib/xorg/Xorg                       10MiB |<- (이부분)
|    0   N/A  N/A            2154      G   /usr/bin/gnome-shell                     17MiB |<- (이부분)
+-----------------------------------------------------------------------------------------+

$ sudo systemctl stop gdm  (점유 중인 프로세스 삭제)

$ nvidia-smi (삭제 확인)
Mon Feb  2 08:55:38 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA RTX PRO 6000 Blac...    On  |   00000000:17:00.0 Off |                  Off |
| 30%   31C    P8             17W /  300W |       0MiB /  97887MiB |     N/A      Default |
|                                         |                        |              Enabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| MIG devices:                                                                            |
+------------------+----------------------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |              Shared Memory-Usage |        Vol|        Shared         |
|      ID  ID  Dev |                Shared BAR1-Usage | SM     Unc| CE ENC  DEC  OFA  JPG |
|                  |                                  |        ECC|                       |
|==================+==================================+===========+=======================|
|  No MIG devices found                                                                   |
+-----------------------------------------------------------------------------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |<- (삭제확인)
+-----------------------------------------------------------------------------------------+

```

```bash
$ sudo nvidia-smi mig -cgi 14 -C
Successfully created GPU instance ID  3 on GPU  0 using profile MIG 1g.24gb (ID 14)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  3 using profile MIG 1g.24gb (ID  0)
```
ID = 14 인 인스턴스 생성.
```bash
$ nvidia-smi -L
GPU 0: NVIDIA RTX PRO 6000 Blackwell Max-Q Workstation Edition (UUID: GPU-43cc581f-8084-8750-9c33-32a7cbcc8a7c)
  MIG 1g.24gb     Device  0: (UUID: MIG-a03442ea-7e39-5c87-a571-2c92a740e2b4) <- (이 부분 확인)
```

### (8) 생성된 MIG 사용.
```bash
$ docker run --rm -it \
> --gpus '"device=MIG-a03442ea-7e39-5c87-a571-2c92a740e2b4"' \
> langchain/langchain:latest nvidia-smi
Mon Feb  2 00:04:39 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA RTX PRO 6000 Blac...    On  |   00000000:17:00.0 Off |                  Off |
| 30%   30C    P8             14W /  300W |                  N/A   |     N/A      Default |
|                                         |                        |              Enabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| MIG devices:                                                                            |
+------------------+----------------------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |              Shared Memory-Usage |        Vol|        Shared         |
|      ID  ID  Dev |                Shared BAR1-Usage | SM     Unc| CE ENC  DEC  OFA  JPG |
|                  |                                  |        ECC|                       |
|==================+==================================+===========+=======================|
|  0    3   0   0  |              64MiB / 24192MiB    | 46    N/A |  1   1    1    0    1 |
|                  |               0MiB /  8327MiB    |           |                       |
+------------------+----------------------------------+-----------+-----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

정확히 생성한 24gb 크기의 인스턴스인 것을 확인.