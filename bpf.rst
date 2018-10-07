.. only:: not (epub or latex or html)

    경고 : 출시되지 않은 Cilium 설명서를보고 있습니다.
    공식 릴리스 버전을 사용하십시오.
    http://docs.cilium.io

.. _bpf_guide:

**********************
BPF 와 XDP 참조 가이드
**********************

.. note:: 이 문서 과정은 기술적인 면에서 BPF와 XDP를 이해하고자 하는 개발자와
          사용자를 대상으로 만들어졌습니다. 현재의 참조안내서를 읽는 동안
          Cilium에 대한 이해하는데 있어서 도움이 될 수 있지만, Cilium을 사용
          하는 것에 있어서 필수사항이 아닙니다. 더 높은 수준의 소개에 대해서는
          :ref:`gs_guide` 와 :ref:`arch_guide` 참조하십시오.

BPF는 Linux 커널에서 매우 유연하고 효율적인 가상 시스템과 같은 구조 이며,
다양한 방식으로 바이트 코드를 안전하게 실행할 수 있습니다. 또한, 많은 리눅스
커널 서브 시스템에서 사용되며 네트워킹, 추적 및 보안 (예 : 보호된 영역 내에서
프로그램을 동작시키는 것을 뜻하며,sandboxing 라고 합니다)이 가장 두드러집니다.

BPF는 1992 년부터 존재 하였지만, 현재의 문서는 커널 3.18에 처음으로 등장한
버클리 패킷 필터 (eBPF) 버전을 다루고 있으며, 고전적인 BPF (cBPF)라고 불리는
의미에 대해서는 요즘은 거의 사용 되지 않습니다. cBPF는 많은 사람들에게 tcpdump가
사용하는 패킷 필터 언어로 친숙히 알려져 있습니다. 최신 Linux 커널은 eBPF만 실행
하고 로드 된 cBPF 바이트 코드는 프로그램 실행 전에 커널에서 eBPF 표현으로 편리
하게 변환됩니다. 이 문서는 일반적으로 eBPF와 cBPF 간의 명시적인 차이점이 따로
언급되지 않는 상태에서는 통상적으로 BPF라는 용어를 사용합니다.

Berkeley Packet Filter라는 이름이 특정 목적을 필터링하는 패킷을 암시 하긴 하지만,
명령어 세트는 요즘에는 앞서 설명한 의미와는 별도로 BPF에 대한 많은 사용 사례가
있으며 일반적이며 유연합니다. BPF를 사용하는 프로젝트 목록은 추가 자료
:ref:`bpf_users` 를 참조하십시오.

Cilium은 BPF를 데이터 경로에 많이 사용하며, 자세한 내용은 개념 :ref:`arch_guide`
을 참조하십시오. 이 장의 목표는 BPF에 대한 이해와 tc(트래픽 제어) 및 XDP
(고속 데이터 경로)로 BPF 프로그램을 로드하는 것을 포함한 네트워크 관련 용도 와
Cilium에서 BPF 템플릿을 개발하는데 도움이되는 BPF 참조 가이드를 제공하는 것
입니다.

BPF 구조
========

BPF는 명령 세트만 제공함에 있어서 다른 정의하지 않지만, 효율적인 키/값 저장소
역할을 하는 map이 있으며, 커널 기능과 상호 작용하고 활용하는 도와주는 함수인
helper 함수, 다른 BPF 프로그램을 호출하는 tail call, 보안 강화 기본 요소,
객체(map, 프로그램) 고정용 가상 파일 시스템, BPF를 네트워크 카드와 같은 오프로드
허용하는 기반을 언급하고 있습니다.

LLVM은 BPF 백엔드를 제공하므로 clang과 같은 도구를 사용하여 C를 BPF 객체 파일로
컴파일한 다음에 커널에 로드 할 수 있습니다. BPF는 리눅스 커널에 많은 연관 관계가
있으며, 원래의 커널 성능을 저해시키지 않으면서 완전한 프로그래밍이 가능합니다.

마지막으로, 그렇지만 앞에서 설명한 것과 마찬가지로 중요한 것은 BPF를 사용하는
커널 서브 시스템은 BPF의 기반의 일부라는 것입니다. 이 문서에서 다루는 두 가지
주요 하위 시스템은 BPF 프로그램을 연결할 수 있는 tc 및 XDP 입니다.
XDP BPF 프로그램은 초기 네트워킹 드라이버 단계에 연결되어 패킷 수신시 BPF 프로그
램 실행을 유발하게 합니다. 정의에 따르면 패킷이 소프트웨어보다 더 빠른 시점에서
처리 될 수 없으므로 최상의 패킷 처리 성능을 얻을 수 있습니다. 그러나 이러한 처리
는 네트워킹 스택의 초기에 발생하기 때문에 스택은 패킷에서 아직 메타 데이터를
추출하지 못했습니다. 반면에 tc BPF 프로그램은 커널 스택에서 나중에 실행 되므로
더 많은 메타 데이터와 코어 커널 기능에 접근 할 수 있습니다. tc 및 XDP 프로그램
외 에도 추적 (kprobes, uprobes, tracepoints 등)와 같은 BPF를 사용하는 다양한
다른 커널 하위 시스템이 있습니다.

다음 하위 절에서는 앞에서 열거한 BPF 아키텍처의 개별적인 측면에 대해 자세히
설명합니다.

명령어 집합
-----------

BPF는 범용 RISC 명령어 집합이며 compiler 하위 단계에서 (예를 들어 LLVM)를 통해
BPF 명령어로 컴파일 가능한 C언어로 프로그램을 작성하기 위해 설계 되었으며,
이후 커널은 나중에 커널 내부의 최적의 실행 성능을 위해 커널 내 JIT 컴파일러를
통해 고유의 연산자 코드(opcode)로 매핑 할 수 있습니다.

이러한 명령어들을 커널에 적용 하게되면 다음과 같은 장점이 있습니다:

* 커널/사용자 영역 간 경계를 넘지 않아도 커널 프로그래밍 할 수 있습니다.
  예를 들어, Cilium의 경우에서는 네트워킹과 관련된 BPF 프로그램은 유연한 컨테
  이너 정책, 로드 밸런싱 구현이 가능 하며, 다른 의미로써 패킷을 사용자 영역으로
  이동시키고 커널로 다시 보낼 필요가 없다는 것을 의미합니다. 또한, BPF 프로그램
  과 커널/사용자 영역 사이의 상태는 필요할 때마다 맵을 통해 공유 할 수 있습니다.

* 프로그램 가능한 데이터 경로의 유연성을 감안할 때, 프로그램은 사용 사례에는
  요구하지 않는 기능에 대해 컴파일 제외 하여 성능을 크게 향상시킬 수 있습니다.
  예를 들어, 컨테이너에서 IPv4가 필요하지 않은 경우, BPF 프로그램은 fast-path에
  서의 자원들을 절약 하기 위해 IPv6 만 처리하도록 구축 될 수 있습니다.

* 네트워킹 (예 : tc 및 XDP)의 경우 BPF 프로그램은 커널, 시스템 서비스 또는
  컨테이너를 다시 시작하지 않고 트래픽 중단 없이 최소한으로 업데이트 할 수 있습
  니다. 또한 BPF 맵을 통해 모든 프로그램 상태를 업데이트 할 수 있습니다.

* BPF는 사용자 영역에 대해 안정적인 ABI를 제공 하며 써드파티 커널 모듈을 따로
  필요로 하지 않습니다. BPF는 모든 곳에서 포함되는 Linux 커널의 핵심 부분이며
  기존 BPF 프로그램이 최신 커널버전으로 계속 실행 될수 있도록 보장합니다.
  앞서 설명한 보장은 사용자 공간 응용 프로그램과 관련하여 커널이 system 콜을
  제공하는 것과 동일한 보장입니다. 또한 BPF 프로그램은 다른 아키텍처에서
  이식 가능합니다.

* BPF 프로그램은 커널과 함께 작동하며 커널이 제공하는 안전 보장 뿐만 아니라
  기존 커널 기반(예 : 드라이버, net device, 터널, 프로토콜 스택, 소켓) 및 도구
  (예 : iproute2)를 사용합니다. 커널 모듈과는 달리 BPF 프로그램으로 커널을
  충돌이 발생 할수 없도록 하며, 항상 종료 되도록 보장하기 위해 커널 내 verifier
  를 통해 검증 됩니다.예를 들어, XDP 프로그램은 기존의  커널 내부 드라이버를
  재사용하며, XDP 프로그램을 다른 모델처럼 사용자 또는 전체 드라이버를 사용자
  공간에 공개하지 않고 xdp 패킷 프레임을 포함하는 커널에서 제공된 DMA 버퍼에서
  동작합니다. 또한 XDP 프로그램은 기존 프로토콜 스택을 bypass 하는 대신에
  다시 사용합니다. BPF는 특정 사용 사례를 해결하기 위한 프로그램을 만들기 위해
  커널 기능에 대한 일반적인 "glue code" 로 고려되어 질수 있습니다.

커널 내부의 BPF 프로그램 실행은 항상 이벤트 동작 입니다. 예를 들어, 들어오는
경로에 BPF 프로그램이 연결된 네트워킹 장치는 패킷이  수신되면 프로그램의 실행
을 트리거합니다. BPF 프로그램이 연결된 kprobes가 위치한 커널 주소는 해당 주소
의 코드가 도착하면 트랩 되며,  실행 한 다음 측정을 위해 kprobes 콜백 함수를
호출하며 이후에 연결된 BPF 프로그램의 실행을 트리거 합니다.

BPF는 11 개의 64 비트 레지스터와 32 비트 서브 레지스터로 구성 되며, 프로그램
카운터 및 512 바이트의 큰 BPF 스택 공간이 있습니다. BPF 레지스터의 이름은
``r0`` - ``r10`` 입니다. 작동 방식는 기본적으로 64 비트이며 32 비트 하위 레지
스터는 특수 ALU(산술 논리 장치) 작업을 통해서만 액세스 할 수 있습니다. 32 비트
하위 하위 레지스터는 기록 될 때 64 비트로 zero-extend 됩니다.

레지스터 ``r10`` 은 읽기 전용이며 BPF 스택 공간에 액세스하기 위해 프레임
포인터 주소를 포함하는 유일한 레지스터입니다.  나머지 ``r0`` - ``r9`` 레지스터는
범용이며 읽기/쓰기 특성을 가지고 있습니다.

BPF 프로그램은 핵심 커널에 의해 정의 된 사전 정의 된 Helper 함수를 호출 할 수 있습니다
(모듈별로 사용하지 않습니다). BPF 콜 규칙은 다음과 같이 정의됩니다:

* ``r0`` 레지스터리는 helper 함수 콜의 반환 값이 포함됩니다.
* ``r1`` - ``r5`` 레지스터리는 BPF 프로그램에서 커널 helper 기능에 대한 인수를 가지게 됩니다.
* ``r6`` - ``r9`` 레지스터리는 helper 함수 콜시 callee가 저장하는 레지스터 입니다.

BPF 호출 규약은 ``x86_64``, ``arm64`` 및 기타 ABI 를 직접 매핑 할만큼 충분히 일반적이며,
따라서 모든 BPF 레지스터들은  HW CPU 레지스터들과 1:1 매핑 되며, JIT는 호출 명령어를
단지 전달하며, 함수 인자들을 배치 하기 위한 추가적인 행동들은 없습니다.이 호출 규약은
성능 저하 없이 일반적인 호출 상황을 포함하도록 모델링 되었습니다. 6 개 이상의 인자가
있는 호출은 현재 지원되지 않습니다. 커널에서 BPF (``BPF_call_0()`` 함수에서 ``BPF_call_5()``
함수)의 Helper 함수는 특별히 규칙을 염두에 두고 고안되었습니다.

레지스터 ``r0`` 은 BPF 프로그램의 종료 값을 포함하는 레지스터 이기도합니다.
종료 값의 의미는 프로그램 유형에 따라 정의됩니다. 커널에게 실행을 다시 전달 할때,
종료 값은 32비트 값으로 전달됩니다.

레지스터 ``r1`` - ``r5`` 는 스크래치 레지스터 이며, BPF 프로그램은 BPF 스택에 이들을
spilling 하거나 이러한 인수가 여러 helper 함수 콜에 걸쳐 재사용 될 경우  callee 저장
레지스터로 이동 중 하나가 필요하게 됩니다. spilling은 레지스터의 변수가 BPF 스택으로
이동 하는 것을 뜻합니다. 변수를 BPF 스택에서 레지스터로 이동하는 반대 작업을 채우기
라고합니다. BPF 스택에서 레지스트로 변구를 이동하는 역동작을 가리켜  filling 이라고
합니다. 제한된 레지스터 수로 인해 spilling/filling 이 동작이 있는 이유 입니다.

BPF 프로그램 실행을 시작하면 레지스터 ``r1`` 은 처음에 프로그램의 컨텍스트를 포함합니다.
컨텍스트는 프로그램의 입력 인수 뜻하며, (일반적인 C 프로그램의 경우 ``argc/argv`` 쌍과
유사하다고 이해하시면 됩니다). BPF는 단일 컨텍스트에서 작업하도록 제한됩니다.
컨텍스트는 프로그램의 유형에 따라 달리지며, 예를 들어, 네트워킹 프로그램은
네트워크 패킷 (``skb``)의  kernel representation을 입력 인수로 가질 수 있습니다.

BPF의 일반적인 연산은 포인터 연산을 수행하기 위해 64 비트 아키텍처의 자연적인 모델을
따르는 64 비트이며,  포인터를 전달 할 뿐만 아니라 64 비트 값을 helper 함수에 전달하며,
그리고 64bit atomic 동작을 허용합니다.

프로그램 당 최대 명령어은 4096 BPF 명령어 으로 제한이 되며 이는 설계적으로 모든
BPF 프로그램은 빨리 종료  되는것을 의미 합니다. 명령어 세트가 순방향 및 역방향 점프를
포함하지만 커널 내 BPF verifier는 루프를 금지 하므로 종료가 항상 보장됩니다. BPF 프로
그램은 커널 내부에서 실행되기 때문에 verifier의 job는 시스템의 안정성에 영향을 주지
않고 실행이 안전하다는 것을 확인하는 것입니다. 즉, 명령어 세트 관점에서 루프를 구현할
수 있지만 verifier는 이를 제한 합니다. 그러나 한 BPF 프로그램이 다른 BPF 프로그램으로
이동할 수 있도록 하는 tail 호출 개념도 있습니다. 이것 또한, 32개의 상위 nesting 제한이
있으며, 일반적으로 프로그램의 논리 부분을 분리 하는데 사용되며, 예를 들어 다음 단계로
진입을 뜻합니다.

명령어 형식은 두 개의 피연산자 명령어들로 모델링 되며 주입 단계에서 BPF 명령어를 기본
명령어에 매핑하는 데 도움이 됩니다. 명령어 세트는 고정 크기이며 모든 명령어가 64 비트
인코딩을 가지는것을 의미합니다. 현재 87 개의 명령어가 구현 되었으며 인코딩을 통해
필요할 때 further 명령어을 사용하여 세트를 확장 할 수 있습니다. big-endian 머신에서
단일 64 비트 명령어의 명령어 인코딩은 최상위 비트(MSB)에서 최하위 비트(LSB)까지의
비트 시퀀스로 구성이 되며, ``op:8``, ``dst_reg:4``, ``src_reg:4``, ``off:16``,
``imm:32``, ``off``  그리고 ``imm`` 은 부호 타입 입니다. 인코딩은 커널 헤더의 일부이며
``linux/bpf_common.h`` 를 포함하는 ``linux/bpf.h`` 헤더 파일에 정의되어 있습니다.

``op`` 는 수행 할 실제 작업을 정의합니다. ``op`` 에 대한 대부분의 인코딩은 cBPF에서
다시 사용 되었습니다. 동작은 레지스터 또는 즉각적인 피연산자를 기반으로 할 수 있습니다.
``op`` 자체의 인코딩은 사용할 방식(레지스터 기반 동작을 표현하기 위한 ``BPF_X`` 그리고
즉각적인 기반 각각 동작을 위한 ``BPF_K``)에 대한 정보를 제공합니다. 후자의 경우 대상
피연산자는 항상 레지스터 입니다. ``dst_reg`` 및 ``src_reg`` 는 모두 동작을 위해 사용될
레지스터 피연산자 (예 : ``r0`` - ``r9``)에 대한 추가 정보를 제공합니다.
``off`` 는 예를 들어 BPF에 이용 가능한 스택 또는 다른 버퍼 (예를 들어, 맵 값, 패킷 데
이터 등)를 어드레싱하거나, jump 명령어에서 타겟을 점프 와 같은 상대 오프셋을 제공 하기
위해 일부 명령어 에서 사용됩니다. ``imm`` 은 상수/즉각적인 값을 포함합니다.

사용 가능한 ``op`` 명령어는 다양한 명령어 클래스로 분류 할 수 있습니다. 이러한 클래스는
``op`` 필드에도 인코딩 됩니다. 연산 필드는 최하위 비트(lsb) 부터 최상위 비트(msb)까지
이며, ``code:4 bits`` , ``source:1 bits`` 그리고 ``instruction class:3 bits`` 으로 나뉩
니다. ``instruction class`` 필드는 일반적인 명령어 클래스이며, ``operation code`` 는 해당
클래스의 특정 연산 코드를 나타내며, ``source`` 필드는 소스 피연산자가 레지스터인지
즉각적인 값 인지를 알려 줍니다. 가능한 명령 클래스는 다음과 같습니다:

* ``BPF_LD``, ``BPF_LDX`` : 두개 모두 load 동작을 위한 클래스 입니다. ``BPF_LD`` 는
  ``imm 필드의 32 bits`` 와 패킷 데이터의 byte / half-word / word 로드 하기 의한
  두개의 명령어를 포괄하는 특수 명령어 이며, double word를 로드 하는데 사용합니다.
  이후 에는 주로 최적화 된 JIT 코드를 가지고 있기 때문에 cBPF를 BPF 변환으로 유지하기
  위해 주로 cBPF에서 가져와서 적용 되었습니다. 현재의 BPF의 경우에는 이러한 패킷 로드
  명령어는 관련성이 낮습니다. ``BPF_LDX`` 클래스는 메모리에서
  byte / half-word / word / double-word 로드에 대한 명령어를 저장합니다.
  이 컨텍스트의 메모리는 일반적이며 스택 메모리, 맵 값 데이터, 패킷 데이터 등
  일 수 있습니다.

* ``BPF_ST``, ``BPF_STX``: 두개 모두 저장 연산을 위한 클래스 입니다. ``BPF_LDX``
  와 비슷하게 ``BPF_STX`` 는 마찬가지로 저장하는 역활을 하며 레지스터의 데이터를
  메모리에 저장하는 데 사용되며 다시 스택 메모리, 맵 값, 패킷 데이터 등이 될 수
  있습니다. BPF_STX는 또한 예를 들어, 카운터에 사용할 수있는 단어 및 double-word 기반
  atomic 추가 연산을 수행하기위한 특수 명령어를 가지게 됩니다. ``BPF_ST`` 클래스는
  소스 피연산자가 즉각적인 값이라는 것을 메모리에 저장하기 위한 명령을
  제공함으로써 ``BPF_STX`` 와 유사합니다.

* ``BPF_ALU``, ``BPF_ALU64``: 두개의 모두 ALU 동작을 포함한 클래스 입니다. 일반적으로,
  ``BPF_ALU`` 동작은 32bit 방식 이며, ``BPF_ALU64`` 는 64bit 방식 입니다. 두 ALU 클래스
  는 모두 레지스터 기반의 원본 피연산자와 즉각적인 기반의 피연산자로 기본 연산을
  수행합니다. add(``+``), sub(``-``), and(``&``), or(``|``), left shift(``<<``),
  right shift (``>>``), xor(``^``), mul (``*``), div (``/``), mod (``%``), neg ``(~``)
  연산을 두개의 class가 지원합니다. 또한 mov(``<X> := <Y>``)는 두 피연산자 방식에서 두 클래스의
  특수한 ALU 연산으로 추가되었습니다. ``BPF_ALU64`` 에는 signed right shift도 포함됩니다.
  ``BPF_ALU`` 는 주어진 source 레지스터에 대한 half-word / word / double-word 에 대한
  엔디안 변환 명령을 추가로 포함합니다.

* ``BPF_JMP``: 점프 동작 전용 클래스 입니다. 점프는 무조건 그리고 조건부 일 수 있습니다.
  무조건 점프는 단순히 프로그램 카운터를 앞으로 이동시켜 현재 명령과 관련하여 실행될
  다음 명령이 ``off + 1`` 이 되도록 하며, 여기서 ``off`` 는 명령에 인코딩 된 상수 오프셋
  입니다. ``off`` 가 signed 때문에 점프는 루프를 생성하지 않고 프로그램 범위 내에있는
  한 다시 실행할 수 있습니다. 조건부 점프는 레지스터 기반 및 즉각적인 기반 소스 피연산자
  모두에서 작동합니다. 점프 작업의 조건이 ``참`` 일 경우, ``off + 1`` 로 상대 점프가 수행되고,
  그렇지 않으면 다음 명령 (0 + 1)이 수행됩니다. 이 fall-through jump 로직은 cBPF와 비교하여
  다르므로 보다 자연스럽게 CPU 분기 예측 로직에 적합하므로 더 나은 분기 예측이 가능하게 됩니다.
  사용 가능한 조건은 jeq (``==``),  jne (``!=``), jgt(``>``), jge (``>=``), jsgt (signed ``>``),
  jsge (signed ``>=``), jlt (``<``), jle (``<=``), jslt (signed ``<``), jsle (signed ``<=``) 그리고
  jset (만약 ``DST & SRC`` 점프). 그 외에도 이 클래스에는 세 가지 특별한 점프 작업들이 있으며:
  다시말해서, BPF 프로그램을 종료하고 ``r0`` 의 현재 값을 반환 코드로 반환하는 종료 명령,
  사용 가능한 BPF helper 함수 중 하나로 함수 호출을 발행하는 콜 명령어 및 다른 BPF 프로
  그램으로 점프하는 tail 호출 명령어가 있습니다.

Linux 커널은 BPF 명령어로 어셈블된 프로그램을 실행하는 BPF 인터프리터와 함께 제공됩니다.
아직 cBPF JIT와 함께 제공되고 아직 eBPF JIT로 마이그레이션되지 않은 아키텍처를 제외하고는
cBPF 프로그램도 커널에서 투명하게 eBPF 프로그램으로 변환됩니다.

현재 ``x86_64``, ``arm64``, ``ppc64``, ``s390x``, ``mips64``, ``sparc64`` 및 ``arm`` 아키텍처
에는 커널 내 eBPF JIT 컴파일러가 제공됩니다.

프로그램을 커널에 로드하거나 BPF 맵을 작성하는 것와 같은 모든 BPF 처리는 중앙 ``bpf()``
시스템 호출을 통해 관리됩니다. 또한 맵 엔트리 (lookup / update / delete)를 관리하고,
프로그램뿐만 아니라 고정 된 맵을 BPF 파일 시스템에 고정시키는 데에도 사용됩니다.

Helper 함수
-----------

Helper 함수는 BPF 프로그램이 코어 커널에 정의된 함수 콜 참조하여 커널에서 데이터를
검색 / 푸시 할수있게 해주는 개념입니다. 사용 가능한 Helper 기능은 각 BPF 프로그램
유형 마다 다를 수 있으며, 예를 들어, 소켓에 연결된 BPF 프로그램은 TC 계층에 연결된
BPF 프로그램과 비교하여 Helper 하위 집합 만 호출 할 수 있습니다.
경량 터널링을 위한 캡슐화 및 캡슐해제 Helper는 tc 레이어 아래쪽에서 사용할 수 있는
기능의 예를 구성하는 반면 사용자 공간에 알림을 푸시 하기위한 event output Helper는
tc 및 XDP 프로그램에서 사용 할 수 있습니다.

각 helper 함수는 시스템 호출과 유사한 공통적으로 공유되는 함수 대표로 구현됩니다.
대표는 다음과 같이 정의됩니다:

::

    u64 fn(u64 r1, u64 r2, u64 r3, u64 r4, u64 r5)

이전 섹션에서 설명한대로 콜 규칙은 모든 BPF helper 함수에 적용됩니다.

커널은 helper 함수를 매크로 ``BPF_call_0 ()`` 에서 ``BPF_call_5 ()`` 까지 시스템 콜
과 유사하게 규격화 하였습니다. 다음 예제는 map 구현에 해당하는 콜백을 콜하여 맵 요소
를 업데이트 하는 helper 함수를 발췌하였습니다.

::

    BPF_CALL_4(bpf_map_update_elem, struct bpf_map *, map, void *, key,
               void *, value, u64, flags)
    {
        WARN_ON_ONCE(!rcu_read_lock_held());
        return map->ops->map_update_elem(map, key, value, flags);
    }

    const struct bpf_func_proto bpf_map_update_elem_proto = {
        .func           = bpf_map_update_elem,
        .gpl_only       = false,
        .ret_type       = RET_INTEGER,
        .arg1_type      = ARG_CONST_MAP_PTR,
        .arg2_type      = ARG_PTR_TO_MAP_KEY,
        .arg3_type      = ARG_PTR_TO_MAP_VALUE,
        .arg4_type      = ARG_ANYTHING,
    };

이러한 접근에는 여러 가지 장점이 있습니다: 기존의 cBPF는 보조 helper 함수를
호출 하기 위해 무리한 패킷 오프셋에서 데이터를 가져 오기 위해 로드 명령어를
오버로드 했으며, 각 cBPF JIT는 이러한 cBPF 확장에 대한 지원을 구현해야했습
니다.  하지만 eBPF의 경우, 새로 추가된 각 helper함수는 JIT 가 투명하고 효율
적인 컴파일된 컴파일이 되며, 이러한 의미는 JIT 컴파일러는 레지스터 매핑과
같은 방식으로 이루어지기 때문에 단지 호출 명령을 내보내며 되며, 레지스터
매핑과 같은 방식은 BPF 레지스터 할당은 이미 기본 아키텍처의 호출 개념과
일치합니다. 이렇게 하면 새로운 helper 기능으로 core 커널을 쉽게 확장 할 수
있습니다. 하지만 모든 BPF helper 함수는 core 커널의 일부이며 커널 모듈을
통해 확장하거나 추가 할 수 없습니다.

앞서 언급한 함수 대표는 또한 verifier가 유형 검사를 수행 하도록 허용합니다.
구조체 ``bpf_func_proto`` 는 helper에 대해 알 필요가 있는 모든 필요한 정보를
verifier에게 넘겨주기 위해 사용되며, 따라서 verifier는 helper가 BPF 프로그램
의 분석 된 레지스터의 현재 내용과 일치에서 예상되는 유형을 확인할 수 있습니다.

인수 유형은 모든 종류의 값 전달에서 BPF 스택 버퍼에 대한 포인터 / 크기 쌍까지
다양합니다. 후자의 경우, verifier는 예를 들어 버퍼가 이전에 초기화되었는지
여부와 같은 추가 검사를 수행 할 수도 있습니다.

The list of available BPF helper functions is rather long and constantly growing,
for example, at the time of this writing, tc BPF programs can choose from 38
different BPF helpers. The kernel's ``struct bpf_verifier_ops`` contains a
``get_func_proto`` callback function that provides the mapping of a specific
``enum bpf_func_id`` to one of the available helpers for a given BPF program
type.

Maps
----

map은 커널 공간에 있는 효율적인 키 / 값 저장소입니다. 여러 BPF 프로그램 호출 간에
상태를 유지하기 위해 BPF 프로그램에서 액세스 할 수 있습니다. 또한 사용자 공간의
파일 설명자를 통해 액세스 할 수 있으며 다른 BPF 프로그램이나 사용자 공간 응용
프로그램과 마음대로 공유 할 수 있습니다.

서로 map을 공유하는 BPF 프로그램은 동일한 프로그램 유형이 아니어야 하며, 예를 들어,
trace 프로그램은 네트워크 프로그램 과 map을 공유 할 수 있습니다. 단일 BPF 프로그램
은 현재 최대 64 개의 다른 map에 직접 액세스 할 수 있습니다.

map 구현은 코어 커널에 의해 제공됩니다. 마음대로 데이터를 읽고 쓸 수있는 각 CPU
마다 일반적인 map 및 각 CPU 마다 아닌 일반적인 map들 있지만 helper 함수와 함께
사용 되는 일부 일반적 이지 않는 map도 있습니다.

지금 사용 가능한 일반적인 map은 ``BPF_MAP_TYPE_HASH``, ``BPF_MAP_TYPE_ARRAY``,
``BPF_MAP_TYPE_PERCPU_HASH``, ``BPF_MAP_TYPE_PERCPU_ARRAY``, ``BPF_MAP_TYPE_LRU_HASH``,
``BPF_MAP_TYPE_LRU_PERCPU_HASH`` 와 ``BPF_MAP_TYPE_LPM_TRIE`` 가 있습니다.
그들은 서로 다른 의미와 성능 특성을 지닌 다른 백엔드를 구현되었지만 조회, 업데이트 또는
삭제 작업을 수행하기 위해 BPF herlper 함수들의 동일한 공통 세트를 사용합니다.

일반적이지 않는 map은 현재 커널에서 ``BPF_MAP_TYPE_PROG_ARRAY``, ``BPF_MAP_TYPE_PERF_EVENT_ARRAY``,
``BPF_MAP_TYPE_CGROUP_ARRAY``, ``BPF_MAP_TYPE_STACK_TRACE``, ``BPF_MAP_TYPE_ARRAY_OF_MAPS``,
``BPF_MAP_TYPE_HASH_OF_MAPS`` 가 있습니다. 예를 들어, ``BPF_MAP_TYPE_PROG_ARRAY`` 는
다른 BPF 프로그램을 저장하는 배열 map이며, ``BPF_MAP_TYPE_ARRAY_OF_MAPS`` 및
``BPF_MAP_TYPE_HASH_OF_MAPS`` 는 런타임시 전체 BPF 맵을 원자적 으로 대체 할 수 있도록
다른 맵에 대한 포인터를 보유합니다. 이러한 유형의 map은 BPF 프로그램 콜을 통해 추가
(비-데이터) 상태가 유지되어야 하기 때문에 BPF helper 함수를 통해 구현 하기에는 부적합한
문제를 해결 합니다.

객체 고정
---------

BPF 맵과 프로그램은 커널 리소스 역할을 하며 커널의 익명 inode가 지원하는 파일
디스크립터를 통해서만 액세스 할 수 있으며, 장점 인것 뿐만 아니라 여러 가지
단점이 있습니다:

사용자 공간 응용 프로그램은 대부분의 파일 디스크립터 관련 API, Unix 도메인 소켓을
통과하는 파일 디스크립터 등을 투명하게  사용할 수 있지만 동시에 파일 디스크립터는
프로세스의 수명에만 제한되기 때문에 map 공유와 같은 옵션이 동작하는 것은 어려워
집니다.

따라서 iproute2 에서는 tc 또는 XDP는 커널에 프로그램을 로드하고 결국 종료되는 것
같은 특정 사용 사례에서는  여러가지 복잡한 문제가 발생합니다. 앞의 문제를 포함하여
, 또한 데이터 경로의 ingress 및 engress 위치 사이에서 맵을 공유하는 경우 와
같이 사용자 공간 측면에서도 map에 액세스 할 수 없습니다. 또한 사용자 공간 측면에서
map에 대한 액세스를 사용할 수 없습니다. 또한 타사 응용 프로그램은 BPF 프로그램
런타임 중에 map 내용에 대해 모니터링 하거나 업데이트 하려고 할수 있습니다.

이러한 한계를 극복 하기 위해서 BPF map 과 프로그램은 객체 고정 이라고 불리는 고정이
될수 있는  최소한의 커널 공간 BPF 파일 시스템 (/sys/fs/bpf)이 구현 되었습니다.
따라서 BPF 시스템 콜은 이전에 고정 된 객체를 고정(``BPF_OBJ_PIN``)하거나 검색(``BPF_OBJ_GET``)
할 수있는 두 개의 새로운 명령으로 확장 되었습니다.

예를 들어 tc와 같은 도구는 진입 및 이탈에서 map를 공유하기 위해 이러한 구조를
사용합니다. BPF 관련 파일 시스템은 싱글톤 패턴이 아니며 다중 마운트 인스턴스,
하드 및 소프트 링크 등을 지원합니다.

Tail 호출
----------

BPF와 함께 사용할 수있는 또 다른 개념을 tail 호출 이라고합니다. Tail 호출은 하나의
BPF 프로그램이 이전 프로그램으로 돌아 가지 않고 다른 프로그램을 콜 할수 있게 해주는
메커니즘으로 보여질수 있습니다. 이러한 콜은 함수 콜과 달리 최소한의 오버 헤드를 가
지며 동일한 스택 프레임을 재사용하여 긴 점프로 구현됩니다.

이러한 프로그램은 서로 관계없이 확인되므로 CPU 마다 맵을 스크래치 버퍼로 전송하거나
tc 프로그램의 경우 ``cb[]`` 영역과 같은 ``skb`` 필드를 사용해야합니다.

동일한 유형의 프로그램만 tail 콜 할 수 있으며, 또한 JIT 컴파일과 관련하여 일치해야
하므로 JIT 컴파일되거나 해석 된  프로그램 만 콜 할 수 있지만 함께 혼재되서 구동 할
수는 없습니다. tail 호출을 수행하는 데는 두 가지 구성 요소가 필요하며:  첫 번째 요
소은 사용자 공간에서 키 / 값으로 덧붙일수 있는  프로그램 배열(``BPF_MAP_TYPE_PROG_ARRAY``)
이라는 특수한 맵을 설정해야 하며, 여기서 값은 tail 호출 된 BPF 프로그램이라는
파일 디스크립터 이며, 두 번째 요소는 컨텍스트 다시 말해 프로그램 배열에 대한 참조
및 조회 키가 전달되는 ``bpf_tail_call()`` helper입니다. 그런 다음 커널은 이 helper
콜을 특수화 된 BPF 명령어로 직접 인라인 합니다. 이러한 프로그램 배열은 현재 사용자
공간 측면에서 쓰기 전용입니다.

커널은 전달 된 파일 디스크립터에서 관련 BPF 프로그램을 찾고 지정된 map 슬롯에서
프로그램 포인터를 원자적으로 대체합니다. 제공된 키에서 map 항목을 찾지 못한다면,
커널은 "fall-through" 및 ``bpf_taill_call()`` 다음에 오는 명령으로 이전 프로그램의
실행을 계속합니다. taill 호출은 강력한 유틸리티 이며, 예를 들어 파싱 네트워크 헤더는
taill 콜을 통해 구조화 될 수 있습니다. 런타임 중에는 기능을 원자적으로 추가하거나
대체 할 수 있으므로 BPF 프로그램의 실행 동작이 변경됩니다.

.. _bpf_to_bpf_calls:

BPF 호출에서 BPF 호출
---------------------

BPF helper 콜과 BPF tail 콜 외에도 BPF 핵심 구조에 추가 된 최신 기능은
BPF 호출에서 BPF 호출 입니다. 이 기능이 커널에 도입되기 전에, 일반적인
BPF C 프로그램은 다음과 같은 재사용 가능한 코드를 선언 해야 했으며,
예를 들어, 헤더에 ``always_inline`` 으로 존재하며 LLVM이 컴파일 하고
BPF 객체 모든 함수들이 인라인이 되며, 그리하여 생성된 오프젝트 파일에
여러번 복제되어 인위적으로 코드 크기가 늘어나게됩니다:

  ::

    #include <linux/bpf.h>

    #ifndef __section
    # define __section(NAME)                  \
       __attribute__((section(NAME), used))
    #endif

    #ifndef __inline
    # define __inline                         \
       inline __attribute__((always_inline))
    #endif

    static __inline int foo(void)
    {
        return XDP_DROP;
    }

    __section("prog")
    int xdp_drop(struct xdp_md *ctx)
    {
        return foo();
    }

    char __license[] __section("license") = "GPL";

이것이 필요했던 주된 이유는 BPF 프로그램 로더뿐만 아니라 verifier, 인터프리터
및 JIT에서 함수 호출 지원이 부족했기 때문입니다. Linux 커널 4.16 및 LLVM 6.0
부터이 제한이 해제 되었으며 BPF 프로그램은 더 이상 always_inline을 사용할
필요가 없습니다. 따라서 이전에 표시된 BPF 예제 코드는 다음과 같이 자연스럽게
다시 작성 될 수 있습니다:

  ::

    #include <linux/bpf.h>

    #ifndef __section
    # define __section(NAME)                  \
       __attribute__((section(NAME), used))
    #endif

    static int foo(void)
    {
        return XDP_DROP;
    }

    __section("prog")
    int xdp_drop(struct xdp_md *ctx)
    {
        return foo();
    }

    char __license[] __section("license") = "GPL";

``x86_64`` 및 ``arm64`` 와 같은 메인스트림 BPF JIT 컴파일러는 BPF 에서
BPF 호출을 지원합니다. BPF에서 BPF 호출은 생성 된 BPF 코드 크기를 많이
줄여 CPU의 명령어 캐시에 더 우호적이기 때문에 중요한 성능 최적화입니다.

BPF helper 함수에서 알려진 콜 규칙은 BPF에서 BPF 호출에도 적용되며,
이 의미는 ``r1`` ~ ``r5`` 는 콜 수신자에게 인수를 전달하기위한 것이며
결과는 r0에 리턴됩니다. ``r1`` ~ ``r5`` 는 스크래치 레지스터이며
``r6`` ~ ``r9`` 는 일반적인 콜을 통해 유지됩니다. 각각 허용 된 콜
프레임의 최대 중첩 호출 수는 ``8`` 입니다. 호출 송신자는 호출 송신자
의 포인터 (예를 들어서, 호출 송신자의 스택 프레임에 대한 포인터)를
호출 수신자에게 전달할 수 있지만 역방향으로는 동작하지 않습니다.

BPF에서 BPF 호출은 현재 BPF tail 호출과 호환되지 않으며,
후자는 현상태의 스택 설정을 그대로 재사용 할 것을 요구하기 때문에,
전자는 추가 스택 프레임을 추가하므로 tail 호출의 예상된 레이아웃을
변경이됩니다.

BPF JIT 컴파일러는 각 함수 본체에 대해 별도의 이미지를 내보내고
나중에 최종 JIT 경로에서 이미지의 함수 호출 주소를 수정합니다.
BPF 에 BPF 호출은 일반적인 BPF helper 호출로 처리 할 수 있다는
점에서 JIT에 대한 최소한의 변경이 필요하다는 것이 입증되었습니다.

JIT
---

64 비트 ``x86_64``, ``arm64``, ``ppc64``, ``s390x``, ``mips64``,
``sparc64`` 및 32 비트 ``arm`` 아키텍처는 모두 커널 내 eBPF JIT 컴파일러
와 함께 제공되며 모든 기능이 동일하며 다음을 통해 활성화 할 수 있습니다:

::

    # echo 1 > /proc/sys/net/core/bpf_jit_enable

32 비트 ``mips``, ``ppc`` 및 ``sparc`` 아키텍처에는 현재 cBPF JIT 컴파일러
가 있습니다. cBPF JIT와 BPF JIT 컴파일러가 없는 Linux 커널이 지원하는
나머지 아키텍처는 모두 커널 내 인터프리터를 통해  eBPF 프로그램을 실행할
필요가 있습니다.

커널의 소스 트리에서 eBPF JIT 지원은 ``HAVE_EBPF_JIT`` 에 대한 grep을
통해 쉽게 확인할 수 있습니다:

::

    # git grep HAVE_EBPF_JIT arch/
    arch/arm/Kconfig:       select HAVE_EBPF_JIT   if !CPU_ENDIAN_BE32
    arch/arm64/Kconfig:     select HAVE_EBPF_JIT
    arch/powerpc/Kconfig:   select HAVE_EBPF_JIT   if PPC64
    arch/mips/Kconfig:      select HAVE_EBPF_JIT   if (64BIT && !CPU_MICROMIPS)
    arch/s390/Kconfig:      select HAVE_EBPF_JIT   if PACK_STACK && HAVE_MARCH_Z196_FEATURES
    arch/sparc/Kconfig:     select HAVE_EBPF_JIT   if SPARC64
    arch/x86/Kconfig:       select HAVE_EBPF_JIT   if X86_64

JIT 컴파일러는 BPF 프로그램의 실행 속도를 인터프리터와 비교하여 명령 마다 비용을
크게 줄여주므로 속도를 크게 높입니다. 명령어는 종종 기본 아키텍처의 기본 명령어로
1:1로 매핑 될 수 있습니다. 이렇게 하면 실행 가능 이미지 크기가 줄어들며, CPU에
우호적인 더 많은 명령어 캐시가 됩니다. 특히 ``x86`` 과 같은 CISC 명령어 세트의
경우 JITs는 주어진 명령어에 대해 가능한 가장 짧은 opcode를 변환하여 프로그램
변환에 필요한 총 크기를 줄이기 위해 최적화됩니다.

Hardening
---------

BPF는 코드 잠재적 손상을 방지하기 위해 프로그램의 실행 동안 커널에서 읽기 전용으로
JIT 컴파일 된 이미지 (``struct bpf_binary_header)`` 뿐만 아니라 BPF 인터프리터 이미지
(``struct bpf_prog``)를 잠급니다. 이러한 시점에서 일어난 corruption, 예를 들어,
일부 커널 버그로 인해 general protection fault가 발생하고 따라서 corruption이 자동
으로 일어나는 것을 허용하지 않고 커널을 크래시 시킵니다. 이미지 메모리를 읽기 전용으
로 설정하는 것을 지원하는 아키텍처는 다음을 통해 결정될 수 있습니다:

이미지 메모리를 읽기 전용으로 설정하는 것을 지원하는 아키텍처는 다음을 통해
결정될 수 있습니다:

::

    $ git grep ARCH_HAS_SET_MEMORY | grep select
    arch/arm/Kconfig:    select ARCH_HAS_SET_MEMORY
    arch/arm64/Kconfig:  select ARCH_HAS_SET_MEMORY
    arch/s390/Kconfig:   select ARCH_HAS_SET_MEMORY
    arch/x86/Kconfig:    select ARCH_HAS_SET_MEMORY

``CONFIG_ARCH_HAS_SET_MEMORY`` 옵션은 설정 가능 하지 않으며, 보호 기능이 항상
내장되어 있습니다. 다른 아키텍처들은 향후 동일 할 수 있습니다.

``x86_64`` JIT 컴파일러의 경우, tail 호출을 사용하여, 간접 점프의 JiTing은
``CONFIG_RETPOLINE`` 이 설정 되었을 때, 가장 최신의 리눅스 배포판에서
retpoline 을 통해 실행됩니다.

``/proc/sys/net/core/bpf_jit_harden`` 의 값이 ``1`` 로 설정된 경우 JIT 컴파일에
대한 추가적인 hardening 단계로 권한이 없는 사용자들에게 적용됩니다. 이것은
시스템에서 동작하는 신뢰할 수없는 사용자에 대해서 (잠재적인)공격 지점을 줄임
으로써 성능을 실제적으로 약간 저하 됩니다. 이러한 프로그램 실행의 축소는 여전히
인터프리터로 전환하는 것 보다 더 나은 성능을 나타냅니다.

현재 hardening 기능을 활성화 하면 원시 opcode를 직접적인 값으로 주입하는
JIT spraying 공격을 방지하기 위해 JIT 컴파일시 BPF 프로그램의 모든 32 비트 및
64 비트 상수를 제공받지 못하게됩니다. 이러한 직접적인 값은 실행 가능한 커널
메모리에 있기 때문에 문제가 되며, 따라서 일부 커널 버그로 인해 발생할 수있는
점프는 직접적인 값의 시작으로 이동 한 다음 기본 명령어로 실행합니다.

JIT 상수 블라인드는 실제 명령어를 무작위로 지정하여 방지 하며, 이것은 직접적인
기반을 둔 소스 피연산자에서 레지스터 기반을 둔 실제 피연산자로 값의 실제 로드를
두 단계로 나누어 명령을 다시 작성하는 방식으로 변환됩니다: 1) 블라인드 된 직접 값
``rnd ^ imm`` 을 레지스터에 로드 하며, 2) 본래의 ``imm`` immediatie가 레지스터에
상주하고 실제 작업에 사용될 수 있도록 ``rnd`` 로 등록하는 xoring을 합니다.
이 예제는 로드 동작을 위해 제공 되었지만, 실제로 모든 일반 동작은 블라인드입니다.

hardening이 비활성화 된 프로그램의 JITing 예제:

::

    # echo 0 > /proc/sys/net/core/bpf_jit_harden

      ffffffffa034f5e9 + <x>:
      [...]
      39:   mov    $0xa8909090,%eax
      3e:   mov    $0xa8909090,%eax
      43:   mov    $0xa8ff3148,%eax
      48:   mov    $0xa89081b4,%eax
      4d:   mov    $0xa8900bb0,%eax
      52:   mov    $0xa810e0c1,%eax
      57:   mov    $0xa8908eb4,%eax
      5c:   mov    $0xa89020b0,%eax
      [...]

같은 프로그램이 경우 hardening이 활성화 된 권한이 없는 사용자를 BPF를 통해 로드
될 때 상수가 블라인드가됩니다:

::

    # echo 1 > /proc/sys/net/core/bpf_jit_harden

      ffffffffa034f1e5 + <x>:
      [...]
      39:   mov    $0xe1192563,%r10d
      3f:   xor    $0x4989b5f3,%r10d
      46:   mov    %r10d,%eax
      49:   mov    $0xb8296d93,%r10d
      4f:   xor    $0x10b9fd03,%r10d
      56:   mov    %r10d,%eax
      59:   mov    $0x8c381146,%r10d
      5f:   xor    $0x24c7200e,%r10d
      66:   mov    %r10d,%eax
      69:   mov    $0xeb2a830e,%r10d
      6f:   xor    $0x43ba02ba,%r10d
      76:   mov    %r10d,%eax
      79:   mov    $0xd9730af,%r10d
      7f:   xor    $0xa5073b1f,%r10d
      86:   mov    %r10d,%eax
      89:   mov    $0x9a45662b,%r10d
      8f:   xor    $0x325586ea,%r10d
      96:   mov    %r10d,%eax
      [...]

두 프로그램 모두 의미 상 동일 하며, 단지 두 번째 프로그램의 디스어셈블리에서 더 이상
원래의 직접 값이 보이지 않습니다.

동시에 hardening는 JIT 이미지 주소가 ``/proc/kallsyms`` 에 더 이상 노출되지 않도록 권한있
는 사용자에 대한 JIT kallsyms 노출에 대해 비활성화합니다.

또한 Linux 커널은 전체 BPF 인터프리터를 커널에서 제거하고 JIT 컴파일러를 영구적으로
활성화하는 ``CONFIG_BPF_JIT_ALWAYS_ON`` 옵션을 제공합니다. 이것은 Spectre v2의 상황에서
VM 기반 설정에서 사용될 때 게스트 커널이 공격이 증가 할때 호스트 커널의 BPF 인터프리터를
재사용하지 않기 위해 개발되었습니다. 컨테이너 기반 환경의 경우 ``CONFIG_BPF_JIT_ALWAYS_ON``
구성 옵션은 선택 사항이지만, JIT가 활성화되어있는 경우 인터프리터는 커널의 복잡성을 줄이기
위해 컴파일에서 제외 될 수 있습니다. 따라서 ``x86_64`` 및 ``arm64`` 와 같은 메인 스트림
아키텍처의 경우 널리 사용되는 JITs를 일반적으로 권장됩니다.

마지막으로, 커널은 ``/proc/sys/kernel/unprivileged_bpf_disabled`` sysctl 설정를 통해
권한이없는 사용자에게 ``bpf(2)`` 시스템 호출을 사용하지 못하게 하는 옵션을 제공합니다.
일회성 kill 스위치 이며, 한번 ``1`` 로 설정이되면, 새로운 커널을 부팅 할때까지
다시 ``0`` 으로 재 설정하는 옵션은 없습니다. 초기 네임스페이스에서 ``CAP_SYS_ADAMIN``
권한이 설정된 프로세서만 설정하면, 그 시점 이후 ``bpf(2)`` 시스템 호출을 사용 할수
있습니다. cilium은 이 설정에 대해서 ``1`` 로 설정합니다.

::

    # echo 1 > /proc/sys/kernel/unprivileged_bpf_disabled

Offloads
--------

BPF의 네트워킹 프로그램, 특히 tc와 XDP의 경우 NIC에서 직접 BPF 코드를 실행하기 위해
커널의 하드웨어에 대한 오프로드 인터페이스가 있습니다.

현재 Netronome의 ``nfp`` 드라이버는 BPF 명령어를 NIC에 대해 구현된 명령어 세트로 변환
하여 JIT 컴파일러를 통해 BPF를 오프로드 할 수 있도록 지원합니다. 여기에는 NIC에 대한
BPF 맵 의오프로딩이 포함되므로  offload된 BPF 프로그램은 맵 조회, 업데이트 및 삭제를
수행 할 수 있습니다.

툴체인
======

현재 사용자 공간에서의 도구, BPF 관련 자가 검사 방법 및 BPF 관련의 커널 설정 제어는
이 섹션에서 논의됩니다. BPF 관련된 도구 및 구조는 여전히 빠르게 발전하고 있으므로,
사용 가능한 모든 도구에 대한 전체 그림을 제공하지 못할 수도 있습니다.


개발환경
--------

Fedora와 Ubuntu 모두에서 BPF 개발 환경을 설정하는 단계별 가이드는 아래에 나와 있
습니다. 이것은 iproute2 빌드 및 설치는 물론 개발 커널의 빌드, 설치 및 테스트를
안내합니다.

iproute2 및 Linux 커널을 수동으로 빌드하는 단계는 일반적으로 주요 배포판이 이미
최신 커널을 기본적으로 제공하기 때문에 필요하지 않지만 리스크가 있는 가장 최신의
버전을 테스트하거나 BPF 패치에 기여하는 iproute2 및 리눅스 커널에 필요합니다.
마찬가지로, 디버깅 및 자가 검사를 위해 bpftool을 빌드하는 것은 선택 사항이지만
권장됩니다.

페도라
``````

페도라 25 이상의 버젼에 적용됩니다:

::

    $ sudo dnf install -y git gcc ncurses-devel elfutils-libelf-devel bc \
      openssl-devel libcap-devel clang llvm

.. note:: 다른 페도라 파생 버젼을 사용하는 경우, ``dnf`` 명령어가
          없는 경우 ``yum`` 을 사용해 보세요.

우분트
``````

우분투 17.04 이상의 버젼에 적용 됩니다:

::

    $ sudo apt-get install -y make gcc libssl-dev bc libelf-dev libcap-dev \
      clang gcc-multilib llvm libncurses5-dev git pkg-config libmnl bison flex

커널 컴파일
```````````

리눅스 커널을 위한 새로운 BPF 기능의 개발은 ``net-next`` git 트리 안에서 이루어지며,
최신 BPF는 ``net`` 트리에서 fix됩니다.다음 명령은 git을 통해 ``net-next`` 트리의 커널
소스를 얻습니다:

::

    $ git clone git://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git

자식 커밋 기록에 대해 관심이 없다면 ``--depth 1`` 은 자식 커밋 기록을 가장 최근
의 커밋으로 제외함으로써 훨씬 빠르게 트리를 복제합니다.

``net`` 트리에 관심이 있는 경우에는 다음 URL에서 복제 할수 있습니다:

::

    $ git clone git://git.kernel.org/pub/scm/linux/kernel/git/davem/net.git

인터넷에 리눅스 커널을 만드는 방법에 대한 수십 가지 튜토리얼이 있으며, 하나의
좋은 자료는 Kernel Newbies 웹 사이트 (https://kernelnewbies.org/KernelBuild)
이며, 위에서 언급 한 두 가지 자식 트리 중 하나를 따라 리눅스 커널을 구성 합니다.

생성 된 ``.config`` 파일에 BPF를 실행하기 위한 다음 ``CONFIG_ *`` 항목이 포함
되어 있는지 확인하십시오. 이 항목은 Cilium 구성 하는 것에 있어서 필요합니다.

::

    CONFIG_CGROUP_BPF=y
    CONFIG_BPF=y
    CONFIG_BPF_SYSCALL=y
    CONFIG_NET_SCH_INGRESS=m
    CONFIG_NET_CLS_BPF=m
    CONFIG_NET_CLS_ACT=y
    CONFIG_BPF_JIT=y
    CONFIG_LWTUNNEL_BPF=y
    CONFIG_HAVE_EBPF_JIT=y
    CONFIG_BPF_EVENTS=y
    CONFIG_TEST_BPF=m

일부 항목은 ``make menuconfig`` 를 통해 조정할 수 없습니다. 예를 들어
``CONFIG_HAVE_EBPF_JIT`` 는 지정된 아키텍처에 eBPF JIT가 있는 경우
자동으로 선택됩니다. 이 경우 ``CONFIG_HAVE_EBPF_JIT`` 는 선택 사항이지만
적극 권장됩니다. eBPF JIT 컴파일러가 없는 아키텍처는 커널 내 인터프리터
로 대체하여 BPF 명령어를 실행되며 효율적이지 않습니다.

설정확인
````````
새로 컴파일 된 커널로 부팅 한 후, BPF 기능을 테스트하기 위해 BPF 자가
테스트 그룹으로 이동합니다 (현재 작업 디렉토리가 복제 된 git 트리의
루트를 가리킵니다).

::

    $ cd tools/testing/selftests/bpf/
    $ make
    $ sudo ./test_verifier

verifier 프로그램 테스트는 수행중인 모든 현재 검사를 출력합니다.
모든 테스트를 실행 후 마지막에에 요약에 대해서 테스트 성공 및
실패 정보를 덤프합니다:

::

    Summary: 847 PASSED, 0 SKIPPED, 0 FAILED

.. note:: 커널 릴리즈 4.16 이상 버젼에서 BPF 셀프 테스트는 더 이상 인라인 될 필요가
          없는 BPF 함수 호출로 인해 LLVM 6.0 이상 버젼에 의존됩니다.자세한 정보는
          :ref:`bpf_to_bpf_calls` 에 대한 항목을 참조하거나 커널 패치 (
          https://lwn.net/Articles/741773/) 에서 커버 레터 메일을 참조하십시오.
          이 새로운 기능을 사용하지 않으면 모든 BPF 프로그램이 LLVM 6.0 이상에
          의존적이지 않습니다. 배포판에서 LLVM 6.0 이상을 제공하지 않으면 :ref:`tooling_llvm`
          섹션의 지침에 따라 컴파일 할 수 있습니다.

모든 BPF 셀프 테스트를 실행하려면 다음 명령이 필요합니다:

::

    $ sudo make run_tests

오류가 발생하면 전체 테스트 결과물과 함께 Slack을 활용 하여 문의하십시오.

iproute2 컴파일
```````````````

``net`` (fixes 전용)과 ``net-next`` (새로운 기능) 커널 트리와 비슷하게 iproute2 git
트리에는 ``master`` 와 ``net-next`` 라는 두 가지 분기가 있습니다. 마스터 분기는
``net`` 트리를 기반으로 하며 ``net-next`` 분기는 ``net-next`` 커널 트리를 기반으
로 합니다. 헤더 파일의 변경 사항을 iproute2 트리에서 동기화 할 수 있도록 하려면 현재
작업이 필요합니다.

iproute2 ``master`` 분기를 복제 하려면 다음 명령을 사용할 수 있습니다:

::

    $ git clone git://git.kernel.org/pub/scm/linux/kernel/git/shemminger/iproute2.git

마찬가지로 iproute2의 언급 된 net-next 분기에 복제 하려면 다음을 실행하십시오:

::

    $ git clone -b net-next git://git.kernel.org/pub/scm/linux/kernel/git/shemminger/iproute2.git

그런 다음 빌드 및 설치를 진행하십시오:

::

    $ cd iproute2/
    $ ./configure --prefix=/usr
    TC schedulers
     ATM    no

    libc has setns: yes
    SELinux support: yes
    ELF support: yes
    libmnl support: no
    Berkeley DB: no

    docs: latex: no
     WARNING: no docs can be built from LaTeX files
     sgml2html: no
     WARNING: no HTML docs can be built from SGML
    $ make
    [...]
    $ sudo make install

iproute2가 LLVM의 BPF 백엔드에서 ELF 파일을 처리 할 수 있도록 ``configure`` 스크립트에
``ELF support : yes`` 가 표시거 되는지 확인하십시오. libelf는 이전에 Fedora와 Ubuntu의
경우 의존성 설치 지침에 나열되었습니다.

bpftool 컴파일
``````````````

bpftool은 BPF 프로그램 및 맵의 디버깅 및 내부 검사에 필수적인 도구입니다. 커널 트리의
일부이며 ``tools/bpf/bpftool/`` 아래에 위치해 있습니다

앞에서 설명한 것처럼 ``net`` 또는 ``net-next`` 커널 트리를 복제하였는지 확인하십시오.
bpftool을 빌드하고 설치하려면 다음 단계가 필요합니다:

::

    $ cd <kernel-tree>/tools/bpf/bpftool/
    $ make
    Auto-detecting system features:
    ...                        libbfd: [ on  ]
    ...        disassembler-four-args: [ OFF ]

      CC       xlated_dumper.o
      CC       prog.o
      CC       common.o
      CC       cgroup.o
      CC       main.o
      CC       json_writer.o
      CC       cfg.o
      CC       map.o
      CC       jit_disasm.o
      CC       disasm.o
    make[1]: Entering directory '/home/foo/trees/net/tools/lib/bpf'

    Auto-detecting system features:
    ...                        libelf: [ on  ]
    ...                           bpf: [ on  ]

      CC       libbpf.o
      CC       bpf.o
      CC       nlattr.o
      LD       libbpf-in.o
      LINK     libbpf.a
    make[1]: Leaving directory '/home/foo/trees/bpf/tools/lib/bpf'
      LINK     bpftool
    $ sudo make install

.. _tooling_llvm:

LLVM
----

LLVM은 현재 BPF 백엔드를 제공하는 유일한 컴파일러 모음입니다. 현재 gcc 버젼은
BPF를 지원하지 않습니다.

BPF 백엔드는 LLVM의 3.7 릴리즈로 병합되었습니다. 주요 배포판은 LLVM을 패키징
할 때 기본적으로 BPF 백엔드를 사용 가능하게하므로 가장 최근의 배포판에서
clang 및 llvm을 설치하면 C를 BPF 오브젝트 파일로 컴파일 하기에 충분합니다.

일반적인 워크 플로우는 BPF 프로그램이 C로 작성되고 LLVM에 의해 object/ELF
파일로 컴파일되며 사용자 공간 BPF ELF 로더(예 : iproute2 또는 기타)로 구문
분석되고 커널 BPF 시스템 호출을 통해 커널에 푸시됩니다. 커널은 BPF 명령어
와 JIT를 검증하고, 프로그램을 위한 새로운 파일 디스크립터를 리턴하고, 서브
시스템 (예 : 네트워킹)에 소속 할 수 있습니다. 만약 지원된다면, 서브 시스템
은 BPF 프로그램을 하드웨어 (예컨대, NIC)로 더 오프로드 할 수 있습니다.

LLVM의 경우 BPF 대상 지원은 다음을 통해 확인할 수 있습니다:

::

    $ llc --version
    LLVM (http://llvm.org/):
    LLVM version 3.8.1
    Optimized build.
    Default target: x86_64-unknown-linux-gnu
    Host CPU: skylake

    Registered Targets:
      [...]
      bpf        - BPF (host endian)
      bpfeb      - BPF (big endian)
      bpfel      - BPF (little endian)
      [...]

기본적으로 ``bpf`` 대상은 컴파일되는 CPU의 엔디안을 사용하며, 즉, CPU의 엔디
안이 리틀 엔디안인 경우 프로그램은 리틀 엔디안 형식으로 표시되며 CPU의 엔
디안이 빅 엔디안 인 경우 프로그램 빅 엔디안 형식으로 표현됩니다. 이것은 BPF의
런타임 동작과도 일치 하며, BPF는 일반적인 형식이며, 어떤 형식의 아키텍처에도
불리하지 않도록하기 위해 실행되는 CPU의 엔디안을 사용합니다.

크로스 컴파일의 경우, 두개의 타겟 ``bpfeb`` 와 ``bpfel`` 이 도입 덕분에, BPF
프로그램은 x86에서의 리틀 엔디안에서 실행되는 노드에서 컴파일 할수 있으며,
arm에서의 빅 엔디안 형식의 노드에서 실행 할 수 있습니다. 프런트 엔드(clang)는
타겟 엔디안에서도 실행 해야 합니다.

대상으로 ``bpf`` 사용하는 것은 엔디안이 혼재되지 않는 상황에서 선호 되는 방법
입니다. 예를 들어, ``x86_64`` 에서의 컴파일은 little 엔디안이기 때문에 타겟 ``bpf``
및 ``bpfel`` 에 대해 동일한 결과물 로써, 컴파일을 트리거하는 스크립트에서는 엔디안
을 인식 할 필요가 없습니다.

최소한의 독립실행형 XDP drop 프로그램은 다음 예제 (``xdp-example.c``)와 같습니다:

::

    #include <linux/bpf.h>

    #ifndef __section
    # define __section(NAME)                  \
       __attribute__((section(NAME), used))
    #endif

    __section("prog")
    int xdp_drop(struct xdp_md *ctx)
    {
        return XDP_DROP;
    }

    char __license[] __section("license") = "GPL";

그런 다음 다음과 같이 컴파일하고 커널에 로드 할 수 있습니다:

::

    $ clang -O2 -Wall -target bpf -c xdp-example.c -o xdp-example.o
    # ip link set dev em1 xdp obj xdp-example.o

.. note:: 위와 같이 네트워크 장치에 XDP BPF 프로그램을 연결하려면 XDP를 지원
          하는 장치가있는 Linux 4.11 혹은 Linux 4.12 이 요구됩니다.

생성 된 오브젝트 파일의 경우 LLVM (> = 3.9)은 공식적인 BPF 시스템 값, 즉 ``EM_BPF``
(10진수:``247`` / 16진수:``0xf7``)를 사용합니다. 이 예제에서 프로그램은 ``x86_64``
에서 ``bpf`` 대상으로 컴파일 되었으므로 ``LSB`` (``MSB`` 와 반대)가 endian와 관련
하여 표시됩니다:

::

    $ file xdp-example.o
    xdp-example.o: ELF 64-bit LSB relocatable, *unknown arch 0xf7* version 1 (SYSV), not stripped

``readelf -a xdp-example.o`` 은 ELF 파일에 대한 추가 정보를 덤프 하며, 생성 된
섹션 헤더, 재배치 항목 및 심볼 테이블을 자가검사 할 때 유용 할 수 있습니다.

Clang 및 LLVM을 처음부터 컴파일 해야하는 경우에는 다음 명령을 사용할 수 있습니다:

::

    $ git clone http://llvm.org/git/llvm.git
    $ cd llvm/tools
    $ git clone --depth 1 http://llvm.org/git/clang.git
    $ cd ..; mkdir build; cd build
    $ cmake .. -DLLVM_TARGETS_TO_BUILD="BPF;X86" -DBUILD_SHARED_LIBS=OFF -DCMAKE_BUILD_TYPE=Release -DLLVM_BUILD_RUNTIME=OFF
    $ make -j $(getconf _NPROCESSORS_ONLN)

    $ ./bin/llc --version
    LLVM (http://llvm.org/):
    LLVM version x.y.zsvn
    Optimized build.
    Default target: x86_64-unknown-linux-gnu
    Host CPU: skylake

    Registered Targets:
      bpf    - BPF (host endian)
      bpfeb  - BPF (big endian)
      bpfel  - BPF (little endian)
      x86    - 32-bit X86: Pentium-Pro and above
      x86-64 - 64-bit X86: EM64T and AMD64

    $ export PATH=$PWD/bin:$PATH   # add to ~/.bashrc

``--version`` 정보에 ``Optimized build`` 가 포함되어 있는지 확인해야 하며,
그렇지 않으면 디버깅 방식에서 LLVM을 사용하는 프로그램의 컴파일 시간이 크게
늘어납니다 (예:10 배 이상).

디버깅을 위해 clang에서는 다음과 같이 어셈블러 출력을 생성 할 수 있습니다:

::

    $ clang -O2 -S -Wall -target bpf -c xdp-example.c -o xdp-example.S
    $ cat xdp-example.S
        .text
        .section    prog,"ax",@progbits
        .globl      xdp_drop
        .p2align    3
    xdp_drop:                             # @xdp_drop
    # BB#0:
        r0 = 1
        exit

        .section    license,"aw",@progbits
        .globl    __license               # @__license
    __license:
        .asciz    "GPL"

Starting from LLVM's release 6.0, there is also assembler parser support. You can
program using BPF assembler directly, then use llvm-mc to assemble it into an
object file. For example, you can assemble the xdp-example.S listed above back
into object file using:

::

    $ llvm-mc -triple bpf -filetype=obj -o xdp-example.o xdp-example.S


또한 최신 LLVM 버전 (>= 4.0)은 디버그 정보를 dwarf 형식으로 객체 파일에 저장
할 수 있습니다. 이것은 컴파일 ``-g`` 파라메터를 추가하여 일반적인 워크플로우를
통해 수행 할 수 있습니다.

::

    $ clang -O2 -g -Wall -target bpf -c xdp-example.c -o xdp-example.o
    $ llvm-objdump -S -no-show-raw-insn xdp-example.o

    xdp-example.o:        file format ELF64-BPF

    Disassembly of section prog:
    xdp_drop:
    ; {
        0:        r0 = 1
    ; return XDP_DROP;
        1:        exit

``llvm-objdump`` 도구는 컴파일러에서 사용 된 원본 C 코드로 어셈블러 출력에 주석을
추가 할 수 있습니다. 이 경우 간단한 예제는 C 코드를 많이 포함하지 않지만 ``0:``
및 ``1:`` 로 표시된 줄 번호는 커널의 verifier 로그에 직접적으로 이어 집니다.

즉, BPF 프로그램이 verifier에 의해 거부되는 경우 ``llvm-objdump`` 는 명령을 원래의
C코드와 연관시키는 데 도움을 줄 수 있으며 이는 분석하는데 있어서 매우 유용합니다.

::

    # ip link set dev em1 xdp obj xdp-example.o verb

    Prog section 'prog' loaded (5)!
     - Type:         6
     - Instructions: 2 (0 over limit)
     - License:      GPL

    Verifier analysis:

    0: (b7) r0 = 1
    1: (95) exit
    processed 2 insns

verifier 분석에서 볼 수 있듯이, llvm-objdump 출력 값은 커널과 동일한 BPF
어셈블러 코드를 덤프 합니다.

``-no-show-raw-insn`` 옵션을 제외하면 ``raw struct bpf_insn`` 을 어셈블리
앞에 hex(16진수)로 덤프 합니다:

::

    $ llvm-objdump -S xdp-example.o

    xdp-example.o:        file format ELF64-BPF

    Disassembly of section prog:
    xdp_drop:
    ; {
       0:       b7 00 00 00 01 00 00 00     r0 = 1
    ; return foo();
       1:       95 00 00 00 00 00 00 00     exit

LLVM IR(중간 표현) 디버깅인 경우, BPF 컴파일 프로세스를 두 단계로 나뉠 수
있으며, 나중에 llc에 전달 될수 있는 바이너리 LLVM IR(중간 표현) 중간 파일
인 ``xdp-example.bc`` 생성 합니다.

::

    $ clang -O2 -Wall -emit-llvm -c xdp-example.c -o xdp-example.bc
    $ llc xdp-example.bc -march=bpf -filetype=obj -o xdp-example.o

생성 된 LLVM IR(중간 표현)은 또한 다음을 통해 사람이 읽을 수있는 형식으로
덤프 될 수 있습니다:

::

    $ clang -O2 -Wall -emit-llvm -S -c xdp-example.c -o -

LLVM is able to attach debug information such as the description of used data
types in the program to the generated BPF object file. By default this is in
DWARF format.

A heavily simplified version used by BPF is called BTF (BPF Type Format). The
resulting DWARF can be converted into BTF and is later on loaded into the
kernel through BPF object loaders. The kernel will then verify the BTF data
for correctness and keeps track of the data types the BTF data is containing.

BPF maps can then be annotated with key and value types out of the BTF data
such that a later dump of the map exports the map data along with the related
type information. This allows for better introspection, debugging and value
pretty printing. Note that BTF data is a generic debugging data format and
as such any DWARF to BTF converted data can be loaded (e.g. kernel's vmlinux
DWARF data could be converted to BTF and loaded). Latter is in particular
useful for BPF tracing in the future.

In order to generate BTF from DWARF debugging information, elfutils (>= 0.173)
is needed. If that is not available, then adding the ``-mattr=dwarfris`` option
to the ``llc`` command is required during compilation:

::

    $ llc -march=bpf -mattr=help |& grep dwarfris
      dwarfris - Disable MCAsmInfo DwarfUsesRelocationsAcrossSections.
      [...]

The reason using ``-mattr=dwarfris`` is because the flag ``dwarfris`` (``dwarf
relocation in section``) disables DWARF cross-section relocations between DWARF
and the ELF's symbol table since libdw does not have proper BPF relocation
support, and therefore tools like ``pahole`` would otherwise not be able to
properly dump structures from the object.

elfutils (>= 0.173) implements proper BPF relocation support and therefore
the same can be achieved without the ``-mattr=dwarfris`` option. Dumping
the structures from the object file could be done from either DWARF or BTF
information. ``pahole`` uses the LLVM emitted DWARF information at this
point, however, future ``pahole`` versions could rely on BTF if available.

For converting DWARF into BTF, a recent pahole version (>= 1.12) is required.
A recent pahole version can also be obtained from its official git repository
if not available from one of the distribution packages:

::

    $ git clone https://git.kernel.org/pub/scm/devel/pahole/pahole.git

``pahole`` comes with the option ``-J`` to convert DWARF into BTF from an
object file. ``pahole`` can be probed for BTF support as follows (note that
the ``llvm-objcopy`` tool is required for ``pahole`` as well, so check its
presence, too):

::

    $ pahole --help | grep BTF
    -J, --btf_encode           Encode as BTF

Generating debugging information also requires the front end to generate
source level debug information by passing ``-g`` to the ``clang`` command
line. Note that ``-g`` is needed independently of whether ``llc``'s
``dwarfris`` option is used. Full example for generating the object file:

::

    $ clang -O2 -g -Wall -target bpf -emit-llvm -c xdp-example.c -o xdp-example.bc
    $ llc xdp-example.bc -march=bpf -mattr=dwarfris -filetype=obj -o xdp-example.o

Alternatively, by using clang only to build a BPF program with debugging
information (again, the dwarfris flag can be omitted when having proper
elfutils version):

::

    $ clang -target bpf -O2 -g -c -Xclang -target-feature -Xclang +dwarfris -c xdp-example.c -o xdp-example.o

After successful compilation ``pahole`` can be used to properly dump structures
of the BPF program based on the DWARF information:

::

    $ pahole xdp-example.o
    struct xdp_md {
            __u32                      data;                 /*     0     4 */
            __u32                      data_end;             /*     4     4 */
            __u32                      data_meta;            /*     8     4 */

            /* size: 12, cachelines: 1, members: 3 */
            /* last cacheline: 12 bytes */
    };

Through the option ``-J`` ``pahole`` can eventually generate the BTF from
DWARF. In the object file DWARF data will still be retained alongside the
newly added BTF data. Full ``clang`` and ``pahole`` example combined:

::

    $ clang -target bpf -O2 -Wall -g -c -Xclang -target-feature -Xclang +dwarfris -c xdp-example.c -o xdp-example.o
    $ pahole -J xdp-example.o

The presence of a ``.BTF`` section can be seen through ``readelf`` tool:

::

    $ readelf -a xdp-example.o
    [...]
      [18] .BTF              PROGBITS         0000000000000000  00000671
    [...]

BPF loaders such as iproute2 will detect and load the BTF section, so that
BPF maps can be annotated with type information.

LVM은 기본적으로 코드를 생성하기 위해 BPF 기본 명령어 세트를 사용하여 생성
된 오브젝트 파일이 long-term stable kernel(예 : 4.9 이상)과 같은 이전의
커널로 로드 될 수 있는지 확인합니다.

그러나 LLVM에는 BPF 명령어 세트의 다른 버전을 선택하기 위해 BPF 백엔드에
대한 ``-mcpu`` 선택 옵션이 있어서, 보다 효율적이고 작은 코드를 생성하기 위해
BPF 기본 명령어 세트의 맨 위에 있는 명령어 세트 확장을 사용합니다.

사용 가능한 -mcpu 옵션은 다음을 통해 쿼리 할 수 있습니다:

::

    $ llc -march bpf -mcpu=help
    Available CPUs for this target:

      generic - Select the generic processor.
      probe   - Select the probe processor.
      v1      - Select the v1 processor.
      v2      - Select the v2 processor.
    [...]

``geneirc`` 프로세서는 기본 프로세서이며 BPF의 기본 명령어 세트 v1 입니다.
옵션 ``v1`` 과 ``v2`` 는 일반적으로 BPF 프로그램이 크로스 컴파일 되는 환경에서
유용하며, 프로그램이 로드 되는 대상 호스트와 컴파일 된 대상 호스트가 다릅니다
(따라서 사용 가능한 BPF 커널 기능도 다를 수 있습니다.).

Cilium에서 내부적으로 사용하는 ``-mcpu`` 권장 옵션은 ``-mcpu=probe`` 입니다!
여기서 LLVM BPF 백엔드는 BPF 명령어 세트 확장의 가용성을 커널에 쿼리하며,
커널에서 사용 가능한 것으로 확인되면 LLVM은 필요할 때마다 BPF 프로그램을
컴파일 하기 위해 옵션을 사용합니다.

llc의 ``-mcpu = probe`` 를 사용한 전체 명령 행 예제:

::

    $ clang -O2 -Wall -target bpf -emit-llvm -c xdp-example.c -o xdp-example.bc
    $ llc xdp-example.bc -march=bpf -mcpu=probe -filetype=obj -o xdp-example.o

일반적으로 LLVM IR 생성은 아키텍처 독립적입니다.
그러나 ``clang -target bpf`` 를 ``-target bpf`` 를 제외하고 사용하면
몇 가지 차이점이 있으며, 기본 아키텍처에 따라 ``x86_64``, ``arm64``
또는 기타 등의 clang의 기본 대상을 사용할 수 있습니다.

커널의 ``Documentation/bpf/bpf_devel_QA.txt`` 에서 인용:

* BPF 프로그램은 파일 범위 인라인 어셈블리 코드가 있는 헤더 파일을 재귀적으로
  포함 할 수 있습니다. 기본 타켓은 잘 처리 할 수 있지만, bpf 백엔드 어셈블러
  가 이러한 어셈블리 코드를 이해하지 못하면 대부분의 경우에 bpf 대상이 실패
  할 수 있습니다.

* -g 옵션 없이 컴파일하면 예를 들어 ``.eh_frame`` 및 ``.rela.eh_frame`` 과 같은
  추가 elf 섹션이 기본 대상을 가지는 오프젝트 파일에 존재 할 수 있으며, bpf 대상
  과는 다릅니다.

* 기본 타겟은 C언어 switch 문을 switch table lookup 및 Jump 동작으로 변환 할 수
  있습니다. switch table이 전역 읽기 전용 섹션에 있으므로 bpf 프로그램이 로드
  되지 않습니다. bpf 대상은 switch table 최적화를 지원하지 않습니다. clang 옵션
  ``-fno-jump-tables`` 를 사용하여 switch table 생성을 비 활성화 할 수 있습니다.

* clang ``-target bpf`` 의 경우, 포인터 또는 longi / unsigned long 타입은 기본
  clang 바이너리 또는 기본 타겟(또는 커널)이 32 비트인지 여부에 관계없이 항상
  64 비트의 크기를 가집니다. 그러나 네이티브 clang 타겟이 사용될 때 기본 아키
  텍처의 규칙에 따라 이러한 유형을 컴파일 하며, 32 비트 아키텍처의 경우 포인터
  또는 long/unsigned long 타입을 의미 하며, BPF 컨텍스트 구조는 32 비트의 너비
  를 가지며, BPF LLVM 백엔드는 여전히 64 비트로 작동합니다.

native 대상은 CPU 레지스터 또는 CPU의 레지스터 너비가 중요한 다른 커널 구조를
매핑하는 ``struct pt_regs`` 이동 하는 경우 추적에 주로 필요합니다. 네트워킹
과 같은 다른 모든 경우에는 ``clang -target bpf`` 를 사용하는 것이 좋습니다.

Also, LLVM started to support 32-bit subregisters and BPF ALU32 instructions since
LLVM's release 7.0. A new code generation attribute ``alu32`` is added. When it is
enabled, LLVM will try to use 32-bit subregisters whenever possible, typically
when there are operations on 32-bit types. The associated ALU instructions with
32-bit subregisters will become ALU32 instructions. For example, for the
following sample code:

::

    $ cat 32-bit-example.c
        void cal(unsigned int *a, unsigned int *b, unsigned int *c)
        {
          unsigned int sum = *a + *b;
          *c = sum;
        }

At default code generation, the assembler will looks like:

::

    $ clang -target bpf -emit-llvm -S 32-bit-example.c
    $ llc -march=bpf 32-bit-example.ll
    $ cat 32-bit-example.s
        cal:
          r1 = *(u32 *)(r1 + 0)
          r2 = *(u32 *)(r2 + 0)
          r2 += r1
          *(u32 *)(r3 + 0) = r2
          exit

64-bit registers are used, hence the addition means 64-bit addition. Now, if you
enable the new 32-bit subregisters support by specifying ``-mattr=+alu32``, then
the assembler will looks like:

::

    $ llc -march=bpf -mattr=+alu32 32-bit-example.ll
    $ cat 32-bit-example.s
        cal:
          w1 = *(u32 *)(r1 + 0)
          w2 = *(u32 *)(r2 + 0)
          w2 += w1
          *(u32 *)(r3 + 0) = w2
          exit

``w`` register, meaning 32-bit subregister, will be used instead of 64-bit ``r``
register.

Enable 32-bit subregisters might help reducing type extension instruction
sequences. It could also help kernel eBPF JIT compiler for 32-bit architectures
for which registers pairs are used to model the 64-bit eBPF registers and extra
instructions are needed for manipulating the high 32-bit. Given read from 32-bit
subregister is guaranteed to read from low 32-bit only even though write still
needs to clear the high 32-bit, if the JIT compiler has known the definition of
one register only has subregister reads, then instructions for setting the high
32-bit of the destination could be eliminated.

Note inline assembly for BPF is currently unsupported.

BPF를 위한 C 프로그램을 작성 할때, C를 사용하여 일반적인 어플리케이션 개발과 비교
되며, 알고 있어야 할 몇 가지 저지르기 쉬운 실수가 있습니다.다음 항목에서는 BPF
모델의 몇 가지 차이점에 대해 설명합니다:

1. **모든 것이 인라인 될 필요가 있으며, 함수 콜 (구 LLVM 버전에서)이나 공유
   라이브러리 호출이 없습니다.**

  공유 라이브러리 등은 BPF와 함께 사용할 수 없습니다. 그러나 BPF 프로그램에서
  사용되는 공통 라이브러리 코드는 헤더 파일에 배치되고 주 프로그램에 포함될 수
  있습니다. 예를 들어, Cilium은 이것을 많이 사용합니다 (``bpf/lib/`` 참조).
  그러나 이것은 여전히 헤더 파일을 포함 할 수 있으며, 예를 들어 커널이나
  다른 라이브러리에서 가져 와서 정적 인라인 함수 나 매크로 / 정의를 재사용
  할 수 있습니다.

  BPF에서 BPF 함수 호출이 지원되는 최신 커널 (4.16+)과 LLVM (6.0+)이 사용하지
  않는 경우에 LLVM은 전체 코드를 컴파일하고 주어진 프로그램 섹션에 대한 BPF
  명령어의 flat sequence로 인라인 해야합니다. 이 경우 아래에 표시된 것처럼
  모든 라이브러리 함수에 대해 ``__inline`` 과 같은 주석을 사용하는 것이 가장
  좋습니다. ``always_inline`` 을 사용하는 것이 추천하며, 컴파일러는 여전히
  ``inline`` 으로 주석 처리 된 큰 함수의 uninline 할 수 있기 때문입니다.

  후자가 발생하면 LLVM은 ELF 파일로 재배치 엔트리를 생성하며, 이 엔트리는
  iproute2와 같은 BPF ELF 로더가 해석 할 수 없으며, 따라서 BPF 맵만이 로더가
  처리 할 수있는 유효한 재배치 엔트리이기 때문에 오류가 발생합니다.

   ::

    #include <linux/bpf.h>

    #ifndef __section
    # define __section(NAME)                  \
       __attribute__((section(NAME), used))
    #endif

    #ifndef __inline
    # define __inline                         \
       inline __attribute__((always_inline))
    #endif

    static __inline int foo(void)
    {
        return XDP_DROP;
    }

    __section("prog")
    int xdp_drop(struct xdp_md *ctx)
    {
        return foo();
    }

    char __license[] __section("license") = "GPL";

2. **여러 프로그램은 서로 다른 섹션의 단일 C 파일 내에 상주 할 수 있습니다.**

   BPF 용 C 프로그램은 섹션 주석을 많이 사용합니다. C 파일은 일반적으로 3 개
   이상의 섹션으로 구성됩니다.BPF ELF 로더는 이 이름들을 사용하여 bpf 시스템
   콜을 통해 프로 그램 과 맵을 로드하기 위해 관련 정보를 추출하고 준비합니다.
   예를 들어, iproute2는 ``map`` 과 ``'license`` 를 기본 섹션 이름으로 사용하
   여 맵 작성에 필요한 메타 데이터와 BPF 프로그램에 대한 라이센스를 각각 찾습
   니다.프로그램 생성시 마지막에 커널에 푸시가 되며, 프로그램이 GPL호환 라이센
   스를 보유한 경우에만 GPL로 노출되는 일부 Helper 기능 들이 활성화 되며,예를
   들어 ``bpf_ktime_get_ns()``, ``bfp_probe_read()`` 및 기타가 해당이 됩니다.

   C programs for BPF make heavy use of section annotations. A C file is
   typically structured into 3 or more sections. BPF ELF loaders use these
   names to extract and prepare the relevant information in order to load
   the programs and maps through the bpf system call. For example, iproute2
   uses ``maps`` and ``license`` as default section name to find metadata
   needed for map creation and the license for the BPF program, respectively.
   On program creation time the latter is pushed into the kernel as well,
   and enables some of the helper functions which are exposed as GPL only
   in case the program also holds a GPL compatible license, for example
   ``bpf_ktime_get_ns()``, ``bpf_probe_read()`` and others.

   나머지 섹션 이름은 BPF 프로그램 코드이며, 예를 들어 아래 코드는 두 개의
   프로그램 섹션 인 ``ingress`` 와 ``egress`` 를 포함하도록 수정되었습니다.
   toy 예제 코드는 둘 다 map 와 ``account_data ()`` 함수와 같은 일반적인
   정적 인라인  helper를 공유 할 수 있음을 보여 줍니다.

   ``xdp-example.c`` 예제는 tc로 로드되고 netdevice의 ingress 및 egress
   hook에 연결될 수 있는 ``tc-example.c`` 예제로 수정되었습니다. 전송
   된 바이트를 두 개의 map 슬롯을 가지고 있으며, 하나는 ingress hook에
   있는 트래픽 용이고 다른 하나는 egress hook에 있는 ``acc_map`` 이라는
   맵에 기록 됩니다.

   ::

    #include <linux/bpf.h>
    #include <linux/pkt_cls.h>
    #include <stdint.h>
    #include <iproute2/bpf_elf.h>

    #ifndef __section
    # define __section(NAME)                  \
       __attribute__((section(NAME), used))
    #endif

    #ifndef __inline
    # define __inline                         \
       inline __attribute__((always_inline))
    #endif

    #ifndef lock_xadd
    # define lock_xadd(ptr, val)              \
       ((void)__sync_fetch_and_add(ptr, val))
    #endif

    #ifndef BPF_FUNC
    # define BPF_FUNC(NAME, ...)              \
       (*NAME)(__VA_ARGS__) = (void *)BPF_FUNC_##NAME
    #endif

    static void *BPF_FUNC(map_lookup_elem, void *map, const void *key);

    struct bpf_elf_map acc_map __section("maps") = {
        .type           = BPF_MAP_TYPE_ARRAY,
        .size_key       = sizeof(uint32_t),
        .size_value     = sizeof(uint32_t),
        .pinning        = PIN_GLOBAL_NS,
        .max_elem       = 2,
    };

    static __inline int account_data(struct __sk_buff *skb, uint32_t dir)
    {
        uint32_t *bytes;

        bytes = map_lookup_elem(&acc_map, &dir);
        if (bytes)
                lock_xadd(bytes, skb->len);

        return TC_ACT_OK;
    }

    __section("ingress")
    int tc_ingress(struct __sk_buff *skb)
    {
        return account_data(skb, 0);
    }

    __section("egress")
    int tc_egress(struct __sk_buff *skb)
    {
        return account_data(skb, 1);
    }

    char __license[] __section("license") = "GPL";

  이 예제는 또한 프로그램을 개발할 때 알아두면 유용한 몇 가지 다른 것들을
  보여줍니다. 이 코드에는 커널 헤더, 표준 C 헤더 및 ``struct bpf_elf_map``
  의 정의가 포함 된 iproute2 특정 헤더가 포함됩니다. iproute2는 일반적인
  BPF ELF 로더를 가지고 있으며 그리고 그러한 ``struct bpf_elf_map`` 의
  정의는 XDP 및 tc 형태 프로그램에 대해 매우 동일합니다.

  ``struct bpf_elf_map`` 항목은 프로그램에서 map을 정의하며 두 BPF 프로그
  램에서 사용되는 map을 생성하는 데 필요한 모든 관련 정보 (예를 들어
  키 / 값 크기 등)를 포함합니다. 구조체는 로더가 찾을 수 있도록 ``map``
  섹션 에 배치해야 합니다. 다른 변수 이름을 가진이 타입의 map 선언이 여러
  개있을 수 있지만, 모두 ``__section( "maps")`` 으로 주석을 추가해야 합니다.

  ``struct bpf_elf_map`` 은 iproute2에만 적용됩니다. 다른 BPF ELF 로더는
  다른 형식을 가질 수 있으며, 예를 들어, ``perf`` 에 의해 주로 사용되는 커널
  소스트리의 libbpf는 다른 사양을 갖습니다. iproute2는 ``struct bpf_elf_map``
  에 대한 하위 호환성을 보장합니다. Cilium은 iproute2 모델을 따릅니다.

  이 예제는 또한 BPF helper 함수가 C 코드로 매핑되고 사용되는 방법을
  보여줍니다.여기에서 ``map_lookup_elem()`` 함수는 ``uapi/linux/bpf.h``
  에서 helper로 표시되는 ``BPF_FUNC_map_lookup_elem`` 열거 형 값에 매핑
  하여 정의합니다.프로그램이 나중에 커널에 로드 될 때, verifier는 전달
  된 인수가 예상되는 유형인지 확인하고 helper 호출을 실제 함수 호출로
  재 지정합니다. 또한 ``map_lookup_elem()`` 함수는 ``map`` 를 BPF helper
  함수에 전달하는 방법을 보여줍니다. 여기서 ``maps`` 섹션의 ``&acc_map``
  은 ``map_lookup_elem()`` 함수의 첫 번째 인수로 전달됩니다.

  정의 된 배열 map이 전역 이므로, 어카운팅은 ``lock_xadd()`` 로 정의되는
  atomic operation을 사용해야합니다. LLVM은 ``__sync_fetch_and_add()`` 를
  내장 함수로 매핑하여 word 크기에 대해 BPF atomic add 명령어 즉
  ``BPF_STX | BPF_XADD | BPF_W`` 를 매핑합니다.

  마지막으로, ``struct bpf_elf_map`` 는 map이 ``PIN_GLOBAL_NS`` 로 고정
  되도록 나타냅니다. 즉, tc는 map을 BPF pseudo file system 에 노드 로
  고정 합니다. 기본적으로 주어진 예제에서는 ``/sys/fs/bpf/tc/globalals/acc_map``
  에 고정됩니다. ``PIN_GLOBAL_NS`` 로 인해 map은 ``/sys/fs/bpf/tc/globals/``
  에 위치 하게 됩니다. ``global`` 은 개체 파일에 걸쳐있는 전역 네임 스페이스
  역할을합니다. 예제에서 ``PIN_OBJECT_NS`` 를 사용하면 tc는 오브젝트 파일에
  대한 로컬 디렉토리를 작성합니다. 예를 들어, BPF 코드가있는 다른 C 파일은
  위의 ``PIN_GLOBAL_NS`` 고정과 동일한 ``acc_map`` 정의를 가질 수 있습니다.
  이 경우 map은 다양한 오브젝트 파일에서 비롯된 BPF 프로그램 간에 공유됩니다.
  ``PIN_NONE`` 은 map가 BPF 파일 시스템에 노드로 배치 되지 않으며, 결과적으로
  tc가 종료 된 후 사용자 공간에서 액세스 할 수 없음을 의미합니다.또한 tc가 각
  프로그램에 대해 두 개의 개별 맵 인스턴스를 생성 한다는 것은 해당 이름으로
  이전에 고정 된 맵을 검색 할 수 없기 때문입니다. 언급 된 경로의 ``acc_map``
  부분은 소스코드에 지정된 map의 이름입니다.

  따라서, ``ingress`` 프로그램을 로딩 할 때, tc는 BPF 파일 시스템에 그러한
  맵이 존재하지 않으며 새로운 맵을 생성한다는 것을 알게 될 것입니다.성공하
  면 map가 고정되어 ``egress`` 프로그램이 tc를 통해 로드 될 때, 해당 map이
  BPF 파일 시스템에 이미 있음 을 알게되고 ``egress`` 프로그램에 재사용 하게
  됩니다. 또한 로더는 맵의 속성 (키 / 값 크기 등)이 일치하는 동일한 이름의
  map이 있는 경우에도 확인합니다.

  tc가 같은 map을 검색 할 수있는 것처럼, 타사 응용 프로그램은 bpf 시스템 호출
  에서 ``BPF_OBJ_GET`` 명령을 사용하여 동일한 맵 인스턴스를 가리키는 새 파일
  디스크립터를 생성 할 수 있으며,이 디스크립터는 맵 요소를 검색 / 갱신 / 삭제
  하는데 사용 될수 있습니다.

  다음과 같이 코드를 컴파일하고 iproute2를 통해 로드 할 수 있습니다:

  ::

    $ clang -O2 -Wall -target bpf -c tc-example.c -o tc-example.o

    # tc qdisc add dev em1 clsact
    # tc filter add dev em1 ingress bpf da obj tc-example.o sec ingress
    # tc filter add dev em1 egress bpf da obj tc-example.o sec egress

    # tc filter show dev em1 ingress
    filter protocol all pref 49152 bpf
    filter protocol all pref 49152 bpf handle 0x1 tc-example.o:[ingress] direct-action id 1 tag c5f7825e5dac396f

    # tc filter show dev em1 egress
    filter protocol all pref 49152 bpf
    filter protocol all pref 49152 bpf handle 0x1 tc-example.o:[egress] direct-action id 2 tag b2fd5adc0f262714

    # mount | grep bpf
    sysfs on /sys/fs/bpf type sysfs (rw,nosuid,nodev,noexec,relatime,seclabel)
    bpf on /sys/fs/bpf type bpf (rw,relatime,mode=0700)

    # tree /sys/fs/bpf/
    /sys/fs/bpf/
    +-- ip -> /sys/fs/bpf/tc/
    +-- tc
    |   +-- globals
    |       +-- acc_map
    +-- xdp -> /sys/fs/bpf/tc/

    4 directories, 1 file

  패킷이 ``em1`` 장치를 통과 하자마자 BPF map의 카운터가 증가합니다.

3. **There are no global variables allowed.**

  For the reasons already mentioned in point 1, BPF cannot have global variables
  as often used in normal C programs.

  However, there is a work-around in that the program can simply use a BPF map
  of type ``BPF_MAP_TYPE_PERCPU_ARRAY`` with just a single slot of arbitrary
  value size. This works, because during execution, BPF programs are guaranteed
  to never get preempted by the kernel and therefore can use the single map entry
  as a scratch buffer for temporary data, for example, to extend beyond the stack
  limitation. This also functions across tail calls, since it has the same
  guarantees with regards to preemption.

  Otherwise, for holding state across multiple BPF program runs, normal BPF
  maps can be used.

4. **There are no const strings or arrays allowed.**

  Defining ``const`` strings or other arrays in the BPF C program does not work
  for the same reasons as pointed out in sections 1 and 3, which is, that relocation
  entries will be generated in the ELF file which will be rejected by loaders due
  to not being part of the ABI towards loaders (loaders also cannot fix up such
  entries as it would require large rewrites of the already compiled BPF sequence).

  In the future, LLVM might detect these occurrences and early throw an error
  to the user.

  Helper functions such as ``trace_printk()`` can be worked around as follows:

  ::

    static void BPF_FUNC(trace_printk, const char *fmt, int fmt_size, ...);

    #ifndef printk
    # define printk(fmt, ...)                                      \
        ({                                                         \
            char ____fmt[] = fmt;                                  \
            trace_printk(____fmt, sizeof(____fmt), ##__VA_ARGS__); \
        })
    #endif

  The program can then use the macro naturally like ``printk("skb len:%u\n", skb->len);``.
  The output will then be written to the trace pipe. ``tc exec bpf dbg`` can be
  used to retrieve the messages from there.

  The use of the ``trace_printk()`` helper function has a couple of disadvantages
  and thus is not recommended for production usage. Constant strings like the
  ``"skb len:%u\n"`` need to be loaded into the BPF stack each time the helper
  function is called, but also BPF helper functions are limited to a maximum
  of 5 arguments. This leaves room for only 3 additional variables which can be
  passed for dumping.

  Therefore, despite being helpful for quick debugging, it is recommended (for networking
  programs) to use the ``skb_event_output()`` or the ``xdp_event_output()`` helper,
  respectively. They allow for passing custom structs from the BPF program to
  the perf event ring buffer along with an optional packet sample. For example,
  Cilium's monitor makes use of these helpers in order to implement a debugging
  framework, notifications for network policy violations, etc. These helpers pass
  the data through a lockless memory mapped per-CPU ``perf`` ring buffer, and
  is thus significantly faster than ``trace_printk()``.

5. **Use of LLVM built-in functions for memset()/memcpy()/memmove()/memcmp().**

  Since BPF programs cannot perform any function calls other than those to BPF
  helpers, common library code needs to be implemented as inline functions. In
  addition, also LLVM provides some built-ins that the programs can use for
  constant sizes (here: ``n``) which will then always get inlined:

  ::

    #ifndef memset
    # define memset(dest, chr, n)   __builtin_memset((dest), (chr), (n))
    #endif

    #ifndef memcpy
    # define memcpy(dest, src, n)   __builtin_memcpy((dest), (src), (n))
    #endif

    #ifndef memmove
    # define memmove(dest, src, n)  __builtin_memmove((dest), (src), (n))
    #endif

  The ``memcmp()`` built-in had some corner cases where inlining did not take place
  due to an LLVM issue in the back end, and is therefore not recommended to be
  used until the issue is fixed.

6. **There are no loops available (yet).**

  The BPF verifier in the kernel checks that a BPF program does not contain
  loops by performing a depth first search of all possible program paths besides
  other control flow graph validations. The purpose is to make sure that the
  program is always guaranteed to terminate.

  A very limited form of looping is available for constant upper loop bounds
  by using ``#pragma unroll`` directive. Example code that is compiled to BPF:

  ::

    #pragma unroll
        for (i = 0; i < IPV6_MAX_HEADERS; i++) {
            switch (nh) {
            case NEXTHDR_NONE:
                return DROP_INVALID_EXTHDR;
            case NEXTHDR_FRAGMENT:
                return DROP_FRAG_NOSUPPORT;
            case NEXTHDR_HOP:
            case NEXTHDR_ROUTING:
            case NEXTHDR_AUTH:
            case NEXTHDR_DEST:
                if (skb_load_bytes(skb, l3_off + len, &opthdr, sizeof(opthdr)) < 0)
                    return DROP_INVALID;

                nh = opthdr.nexthdr;
                if (nh == NEXTHDR_AUTH)
                    len += ipv6_authlen(&opthdr);
                else
                    len += ipv6_optlen(&opthdr);
                break;
            default:
                *nexthdr = nh;
                return len;
            }
        }

  Another possibility is to use tail calls by calling into the same program
  again and using a ``BPF_MAP_TYPE_PERCPU_ARRAY`` map for having a local
  scratch space. While being dynamic, this form of looping however is limited
  to a maximum of 32 iterations.

  In the future, BPF may have some native, but limited form of implementing loops.

7. **Partitioning programs with tail calls.**

  Tail calls provide the flexibility to atomically alter program behavior during
  runtime by jumping from one BPF program into another. In order to select the
  next program, tail calls make use of program array maps (``BPF_MAP_TYPE_PROG_ARRAY``),
  and pass the map as well as the index to the next program to jump to. There is no
  return to the old program after the jump has been performed, and in case there was
  no program present at the given map index, then execution continues on the original
  program.

  For example, this can be used to implement various stages of a parser, where
  such stages could be updated with new parsing features during runtime.

  Another use case are event notifications, for example, Cilium can opt in packet
  drop notifications during runtime, where the ``skb_event_output()`` call is
  located inside the tail called program. Thus, during normal operations, the
  fall-through path will always be executed unless a program is added to the
  related map index, where the program then prepares the metadata and triggers
  the event notification to a user space daemon.

  Program array maps are quite flexible, enabling also individual actions to
  be implemented for programs located in each map index. For example, the root
  program attached to XDP or tc could perform an initial tail call to index 0
  of the program array map, performing traffic sampling, then jumping to index 1
  of the program array map, where firewalling policy is applied and the packet
  either dropped or further processed in index 2 of the program array map, where
  it is mangled and sent out of an interface again. Jumps in the program array
  map can, of course, be arbitrary. The kernel will eventually execute the
  fall-through path when the maximum tail call limit has been reached.

  Minimal example extract of using tail calls:

  ::

    [...]

    #ifndef __stringify
    # define __stringify(X)   #X
    #endif

    #ifndef __section
    # define __section(NAME)                  \
       __attribute__((section(NAME), used))
    #endif

    #ifndef __section_tail
    # define __section_tail(ID, KEY)          \
       __section(__stringify(ID) "/" __stringify(KEY))
    #endif

    #ifndef BPF_FUNC
    # define BPF_FUNC(NAME, ...)              \
       (*NAME)(__VA_ARGS__) = (void *)BPF_FUNC_##NAME
    #endif

    #define BPF_JMP_MAP_ID   1

    static void BPF_FUNC(tail_call, struct __sk_buff *skb, void *map,
                         uint32_t index);

    struct bpf_elf_map jmp_map __section("maps") = {
        .type           = BPF_MAP_TYPE_PROG_ARRAY,
        .id             = BPF_JMP_MAP_ID,
        .size_key       = sizeof(uint32_t),
        .size_value     = sizeof(uint32_t),
        .pinning        = PIN_GLOBAL_NS,
        .max_elem       = 1,
    };

    __section_tail(JMP_MAP_ID, 0)
    int looper(struct __sk_buff *skb)
    {
        printk("skb cb: %u\n", skb->cb[0]++);
        tail_call(skb, &jmp_map, 0);
        return TC_ACT_OK;
    }

    __section("prog")
    int entry(struct __sk_buff *skb)
    {
        skb->cb[0] = 0;
        tail_call(skb, &jmp_map, 0);
        return TC_ACT_OK;
    }

    char __license[] __section("license") = "GPL";

  When loading this toy program, tc will create the program array and pin it
  to the BPF file system in the global namespace under ``jmp_map``. Also, the
  BPF ELF loader in iproute2 will also recognize sections that are marked as
  ``__section_tail()``. The provided ``id`` in ``struct bpf_elf_map`` will be
  matched against the id marker in the ``__section_tail()``, that is, ``JMP_MAP_ID``,
  and the program therefore loaded at the user specified program array map index,
  which is ``0`` in this example. As a result, all provided tail call sections
  will be populated by the iproute2 loader to the corresponding maps. This mechanism
  is not specific to tc, but can be applied with any other BPF program type
  that iproute2 supports (such as XDP, lwt).

  The generated elf contains section headers describing the map id and the
  entry within that map:

  ::

    $ llvm-objdump -S --no-show-raw-insn prog_array.o | less
    prog_array.o:   file format ELF64-BPF

    Disassembly of section 1/0:
    looper:
           0:       r6 = r1
           1:       r2 = *(u32 *)(r6 + 48)
           2:       r1 = r2
           3:       r1 += 1
           4:       *(u32 *)(r6 + 48) = r1
           5:       r1 = 0 ll
           7:       call -1
           8:       r1 = r6
           9:       r2 = 0 ll
          11:       r3 = 0
          12:       call 12
          13:       r0 = 0
          14:       exit
    Disassembly of section prog:
    entry:
           0:       r2 = 0
           1:       *(u32 *)(r1 + 48) = r2
           2:       r2 = 0 ll
           4:       r3 = 0
           5:       call 12
           6:       r0 = 0
           7:       exi

  In this case, the ``section 1/0`` indicates that the ``looper()`` function
  resides in the map id ``1`` at position ``0``.

  The pinned map can be retrieved by a user space applications (e.g. Cilium daemon),
  but also by tc itself in order to update the map with new programs. Updates
  happen atomically, the initial entry programs that are triggered first from the
  various subsystems are also updated atomically.

  Example for tc to perform tail call map updates:

  ::

    # tc exec bpf graft m:globals/jmp_map key 0 obj new.o sec foo

  In case iproute2 would update the pinned program array, the ``graft`` command
  can be used. By pointing it to ``globals/jmp_map``, tc will update the
  map at index / key ``0`` with a new program residing in the object file ``new.o``
  under section ``foo``.

8. **Limited stack space of maximum 512 bytes.**

  Stack space in BPF programs is limited to only 512 bytes, which needs to be
  taken into careful consideration when implementing BPF programs in C. However,
  as mentioned earlier in point 3, a ``BPF_MAP_TYPE_PERCPU_ARRAY`` map with a
  single entry can be used in order to enlarge scratch buffer space.

9. **Use of BPF inline assembly possible.**

  LLVM also allows to use inline assembly for BPF for the rare cases where it
  might be needed. The following (nonsense) toy example shows a 64 bit atomic
  add. Due to lack of documentation, LLVM source code in ``lib/Target/BPF/BPFInstrInfo.td``
  as well as ``test/CodeGen/BPF/`` might be helpful for providing some additional
  examples. Test code:

  ::

    #include <linux/bpf.h>

    #ifndef __section
    # define __section(NAME)                  \
       __attribute__((section(NAME), used))
    #endif

    __section("prog")
    int xdp_test(struct xdp_md *ctx)
    {
        __u64 a = 2, b = 3, *c = &a;
        /* just a toy xadd example to show the syntax */
        asm volatile("lock *(u64 *)(%0+0) += %1" : "=r"(c) : "r"(b), "0"(c));
        return a;
    }

    char __license[] __section("license") = "GPL";

  The above program is compiled into the following sequence of BPF
  instructions:

  ::

    Verifier analysis:

    0: (b7) r1 = 2
    1: (7b) *(u64 *)(r10 -8) = r1
    2: (b7) r1 = 3
    3: (bf) r2 = r10
    4: (07) r2 += -8
    5: (db) lock *(u64 *)(r2 +0) += r1
    6: (79) r0 = *(u64 *)(r10 -8)
    7: (95) exit
    processed 8 insns (limit 131072), stack depth 8

iproute2
--------

There are various front ends for loading BPF programs into the kernel such as bcc,
perf, iproute2 and others. The Linux kernel source tree also provides a user space
library under ``tools/lib/bpf/``, which is mainly used and driven by perf for
loading BPF tracing programs into the kernel. However, the library itself is
generic and not limited to perf only. bcc is a toolkit providing many useful
BPF programs mainly for tracing that are loaded ad-hoc through a Python interface
embedding the BPF C code. Syntax and semantics for implementing BPF programs
slightly differ among front ends in general, though. Additionally, there are also
BPF samples in the kernel source tree (``samples/bpf/``) which parse the generated
object files and load the code directly through the system call interface.

This and previous sections mainly focus on the iproute2 suite's BPF front end for
loading networking programs of XDP, tc or lwt type, since Cilium's programs are
implemented against this BPF loader. In future, Cilium will be equipped with a
native BPF loader, but programs will still be compatible to be loaded through
iproute2 suite in order to facilitate development and debugging.

All BPF program types supported by iproute2 share the same BPF loader logic
due to having a common loader back end implemented as a library (``lib/bpf.c``
in iproute2 source tree).

The previous section on LLVM also covered some iproute2 parts related to writing
BPF C programs, and later sections in this document are related to tc and XDP
specific aspects when writing programs. Therefore, this section will rather focus
on usage examples for loading object files with iproute2 as well as some of the
generic mechanics of the loader. It does not try to provide a complete coverage
of all details, but enough for getting started.

**1. Loading of XDP BPF object files.**

  Given a BPF object file ``prog.o`` has been compiled for XDP, it can be loaded
  through ``ip`` to a XDP-supported netdevice called ``em1`` with the following
  command:

  ::

    # ip link set dev em1 xdp obj prog.o

  The above command assumes that the program code resides in the default section
  which is called ``prog`` in XDP case. Should this not be the case, and the
  section is named differently, for example, ``foobar``, then the program needs
  to be loaded as:

  ::

    # ip link set dev em1 xdp obj prog.o sec foobar

  Note that it is also possible to load the program out of the ``.text`` section.
  Changing the minimal, stand-alone XDP drop program by removing the ``__section()``
  annotation from the ``xdp_drop`` entry point would look like the following:

  ::

    #include <linux/bpf.h>

    #ifndef __section
    # define __section(NAME)                  \
       __attribute__((section(NAME), used))
    #endif

    int xdp_drop(struct xdp_md *ctx)
    {
        return XDP_DROP;
    }

    char __license[] __section("license") = "GPL";

  And can be loaded as follows:

  ::

    # ip link set dev em1 xdp obj prog.o sec .text

  By default, ``ip`` will throw an error in case a XDP program is already attached
  to the networking interface, to prevent it from being overridden by accident. In
  order to replace the currently running XDP program with a new one, the ``-force``
  option must be used:

  ::

    # ip -force link set dev em1 xdp obj prog.o

  Most XDP-enabled drivers today support an atomic replacement of the existing
  program with a new one without traffic interruption. There is always only a
  single program attached to an XDP-enabled driver due to performance reasons,
  hence a chain of programs is not supported. However, as described in the
  previous section, partitioning of programs can be performed through tail
  calls to achieve a similar use case when necessary.

  The ``ip link`` command will display an ``xdp`` flag if the interface has an XDP
  program attached. ``ip link | grep xdp`` can thus be used to find all interfaces
  that have XDP running. Further introspection facilities are provided through
  the detailed view with ``ip -d link`` and ``bpftool`` can be used to retrieve
  information about the attached program based on the BPF program ID shown in
  the ``ip link`` dump.

  In order to remove the existing XDP program from the interface, the following
  command must be issued:

  ::

    # ip link set dev em1 xdp off

  In the case of switching a driver's operation mode from non-XDP to native XDP
  and vice versa, typically the driver needs to reconfigure its receive (and
  transmit) rings in order to ensure received packet are set up linearly
  within a single page for BPF to read and write into. However, once completed,
  then most drivers only need to perform an atomic replacement of the program
  itself when a BPF program is requested to be swapped.

  In total, XDP supports three operation modes which iproute2 implements as well:
  ``xdpdrv``, ``xdpoffload`` and ``xdpgeneric``.

  ``xdpdrv`` stands for native XDP, meaning the BPF program is run directly in
  the driver's receive path at the earliest possible point in software. This is
  the normal / conventional XDP mode and requires driver's to implement XDP
  support, which all major 10G/40G/+ networking drivers in the upstream Linux
  kernel already provide.

  ``xdpgeneric`` stands for generic XDP and is intended as an experimental test
  bed for drivers which do not yet support native XDP. Given the generic XDP hook
  in the ingress path comes at a much later point in time when the packet already
  enters the stack's main receive path as a ``skb``, the performance is significantly
  less than with processing in ``xdpdrv`` mode. ``xdpgeneric`` therefore is for
  the most part only interesting for experimenting, less for production environments.

  Last but not least, the ``xdpoffload`` mode is implemented by SmartNICs such
  as those supported by Netronome's nfp driver and allow for offloading the entire
  BPF/XDP program into hardware, thus the program is run on each packet reception
  directly on the card. This provides even higher performance than running in
  native XDP although not all BPF map types or BPF helper functions are available
  for use compared to native XDP. The BPF verifier will reject the program in
  such case and report to the user what is unsupported. Other than staying in
  the realm of supported BPF features and helper functions, no special precautions
  have to be taken when writing BPF C programs.

  When a command like ``ip link set dev em1 xdp obj [...]`` is used, then the
  kernel will attempt to load the program first as native XDP, and in case the
  driver does not support native XDP, it will automatically fall back to generic
  XDP. Thus, for example, using explicitly ``xdpdrv`` instead of ``xdp``, the
  kernel will only attempt to load the program as native XDP and fail in case
  the driver does not support it, which provides a guarantee that generic XDP
  is avoided altogether.

  Example for enforcing a BPF/XDP program to be loaded in native XDP mode,
  dumping the link details and unloading the program again:

  ::

     # ip -force link set dev em1 xdpdrv obj prog.o
     # ip link show
     [...]
     6: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp qdisc mq state UP mode DORMANT group default qlen 1000
         link/ether be:08:4d:b6:85:65 brd ff:ff:ff:ff:ff:ff
         prog/xdp id 1 tag 57cd311f2e27366b
     [...]
     # ip link set dev em1 xdpdrv off

  Same example now for forcing generic XDP, even if the driver would support
  native XDP, and additionally dumping the BPF instructions of the attached
  dummy program through bpftool:

  ::

    # ip -force link set dev em1 xdpgeneric obj prog.o
    # ip link show
    [...]
    6: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpgeneric qdisc mq state UP mode DORMANT group default qlen 1000
        link/ether be:08:4d:b6:85:65 brd ff:ff:ff:ff:ff:ff
        prog/xdp id 4 tag 57cd311f2e27366b                <-- BPF program ID 4
    [...]
    # bpftool prog dump xlated id 4                       <-- Dump of instructions running on em1
    0: (b7) r0 = 1
    1: (95) exit
    # ip link set dev em1 xdpgeneric off

  And last but not least offloaded XDP, where we additionally dump program
  information via bpftool for retrieving general metadata:

  ::

     # ip -force link set dev em1 xdpoffload obj prog.o
     # ip link show
     [...]
     6: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpoffload qdisc mq state UP mode DORMANT group default qlen 1000
         link/ether be:08:4d:b6:85:65 brd ff:ff:ff:ff:ff:ff
         prog/xdp id 8 tag 57cd311f2e27366b
     [...]
     # bpftool prog show id 8
     8: xdp  tag 57cd311f2e27366b dev em1                  <-- Also indicates a BPF program offloaded to em1
         loaded_at Apr 11/20:38  uid 0
         xlated 16B  not jited  memlock 4096B
     # ip link set dev em1 xdpoffload off

  Note that it is not possible to use ``xdpdrv`` and ``xdpgeneric`` or other
  modes at the same time, meaning only one of the XDP operation modes must be
  picked.

  A switch between different XDP modes e.g. from generic to native or vice
  versa is not atomically possible. Only switching programs within a specific
  operation mode is:

  ::

     # ip -force link set dev em1 xdpgeneric obj prog.o
     # ip -force link set dev em1 xdpoffload obj prog.o
     RTNETLINK answers: File exists
     # ip -force link set dev em1 xdpdrv obj prog.o
     RTNETLINK answers: File exists
     # ip -force link set dev em1 xdpgeneric obj prog.o    <-- Succeeds due to xdpgeneric
     #

  Switching between modes requires to first leave the current operation mode
  in order to then enter the new one:

  ::

     # ip -force link set dev em1 xdpgeneric obj prog.o
     # ip -force link set dev em1 xdpgeneric off
     # ip -force link set dev em1 xdpoffload obj prog.o
     # ip l
     [...]
     6: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpoffload qdisc mq state UP mode DORMANT group default qlen 1000
         link/ether be:08:4d:b6:85:65 brd ff:ff:ff:ff:ff:ff
         prog/xdp id 17 tag 57cd311f2e27366b
     [...]
     # ip -force link set dev em1 xdpoffload off

**2. Loading of tc BPF object files.**

  Given a BPF object file ``prog.o`` has been compiled for tc, it can be loaded
  through the tc command to a netdevice. Unlike XDP, there is no driver dependency
  for supporting attaching BPF programs to the device. Here, the netdevice is called
  ``em1``, and with the following command the program can be attached to the networking
  ``ingress`` path of ``em1``:

  ::

    # tc qdisc add dev em1 clsact
    # tc filter add dev em1 ingress bpf da obj prog.o

  The first step is to set up a ``clsact`` qdisc (Linux queueing discipline). ``clsact``
  is a dummy qdisc similar to the ``ingress`` qdisc, which can only hold classifier
  and actions, but does not perform actual queueing. It is needed in order to attach
  the ``bpf`` classifier. The ``clsact`` qdisc provides two special hooks called
  ``ingress`` and ``egress``, where the classifier can be attached to. Both ``ingress``
  and ``egress`` hooks are located in central receive and transmit locations in the
  networking data path, where every packet on the device passes through. The ``ingress``
  hook is called from ``__netif_receive_skb_core() -> sch_handle_ingress()`` in the
  kernel and the ``egress`` hook from ``__dev_queue_xmit() -> sch_handle_egress()``.

  The equivalent for attaching the program to the ``egress`` hook looks as follows:

  ::

    # tc filter add dev em1 egress bpf da obj prog.o

  The ``clsact`` qdisc is processed lockless from ``ingress`` and ``egress``
  direction and can also be attached to virtual, queue-less devices such as
  ``veth`` devices connecting containers.

  Next to the hook, the ``tc filter`` command selects ``bpf`` to be used in ``da``
  (direct-action) mode. ``da`` mode is recommended and should always be specified.
  It basically means that the ``bpf`` classifier does not need to call into external
  tc action modules, which are not necessary for ``bpf`` anyway, since all packet
  mangling, forwarding or other kind of actions can already be performed inside
  the single BPF program which is to be attached, and is therefore significantly
  faster.

  At this point, the program has been attached and is executed once packets traverse
  the device. Like in XDP, should the default section name not be used, then it
  can be specified during load, for example, in case of section ``foobar``:

  ::

    # tc filter add dev em1 egress bpf da obj prog.o sec foobar

  iproute2's BPF loader allows for using the same command line syntax across
  program types, hence the ``obj prog.o sec foobar`` is the same syntax as with
  XDP mentioned earlier.

  The attached programs can be listed through the following commands:

  ::

    # tc filter show dev em1 ingress
    filter protocol all pref 49152 bpf
    filter protocol all pref 49152 bpf handle 0x1 prog.o:[ingress] direct-action id 1 tag c5f7825e5dac396f

    # tc filter show dev em1 egress
    filter protocol all pref 49152 bpf
    filter protocol all pref 49152 bpf handle 0x1 prog.o:[egress] direct-action id 2 tag b2fd5adc0f262714

  The output of ``prog.o:[ingress]`` tells that program section ``ingress`` was
  loaded from the file ``prog.o``, and ``bpf`` operates in ``direct-action`` mode.
  The program ``id`` and ``tag`` is appended for each case, where the latter denotes
  a hash over the instruction stream which can be correlated with the object file
  or ``perf`` reports with stack traces, etc. Last but not least, the ``id``
  represents the system-wide unique BPF program identifier that can be used along
  with ``bpftool`` to further inspect or dump the attached BPF program.

  tc can attach more than just a single BPF program, it provides various other
  classifiers which can be chained together. However, attaching a single BPF program
  is fully sufficient since all packet operations can be contained in the program
  itself thanks to ``da`` (``direct-action``) mode, meaning the BPF program itself
  will already return the tc action verdict such as ``TC_ACT_OK``, ``TC_ACT_SHOT``
  and others. For optimal performance and flexibility, this is the recommended usage.

  In the above ``show`` command, tc also displays ``pref 49152`` and
  ``handle 0x1`` next to the BPF related output. Both are auto-generated in
  case they are not explicitly provided through the command line. ``pref``
  denotes a priority number, which means that in case multiple classifiers are
  attached, they will be executed based on ascending priority, and ``handle``
  represents an identifier in case multiple instances of the same classifier have
  been loaded under the same ``pref``. Since in case of BPF, a single program is
  fully sufficient, ``pref`` and ``handle`` can typically be ignored.

  Only in the case where it is planned to atomically replace the attached BPF
  programs, it would be recommended to explicitly specify ``pref`` and ``handle``
  a priori on initial load, so that they do not have to be queried at a later
  point in time for the ``replace`` operation. Thus, creation becomes:

  ::

    # tc filter add dev em1 ingress pref 1 handle 1 bpf da obj prog.o sec foobar

    # tc filter show dev em1 ingress
    filter protocol all pref 1 bpf
    filter protocol all pref 1 bpf handle 0x1 prog.o:[foobar] direct-action id 1 tag c5f7825e5dac396f

  And for the atomic replacement, the following can be issued for updating the
  existing program at ``ingress`` hook with the new BPF program from the file
  ``prog.o`` in section ``foobar``:

  ::

    # tc filter replace dev em1 ingress pref 1 handle 1 bpf da obj prog.o sec foobar

  Last but not least, in order to remove all attached programs from the ``ingress``
  respectively ``egress`` hook, the following can be used:

  ::

    # tc filter del dev em1 ingress
    # tc filter del dev em1 egress

  For removing the entire ``clsact`` qdisc from the netdevice, which implicitly also
  removes all attached programs from the ``ingress`` and ``egress`` hooks, the
  below command is provided:

  ::

    # tc qdisc del dev em1 clsact

  tc BPF programs can also be offloaded if the NIC and driver has support for it
  similarly as with XDP BPF programs. Netronome's nfp supported NICs offer both
  types of BPF offload.

  ::

    # tc qdisc add dev em1 clsact
    # tc filter replace dev em1 ingress pref 1 handle 1 bpf skip_sw da obj prog.o
    Error: TC offload is disabled on net device.
    We have an error talking to the kernel

  If the above error is shown, then tc hardware offload first needs to be enabled
  for the device through ethtool's ``hw-tc-offload`` setting:

  ::

    # ethtool -K em1 hw-tc-offload on
    # tc qdisc add dev em1 clsact
    # tc filter replace dev em1 ingress pref 1 handle 1 bpf skip_sw da obj prog.o
    # tc filter show dev em1 ingress
    filter protocol all pref 1 bpf
    filter protocol all pref 1 bpf handle 0x1 prog.o:[classifier] direct-action skip_sw in_hw id 19 tag 57cd311f2e27366b

  The ``in_hw`` flag confirms that the program has been offloaded to the NIC.

  Note that BPF offloads for both tc and XDP cannot be loaded at the same time,
  either the tc or XDP offload option must be selected.

**3. Testing BPF offload interface via netdevsim driver.**

  The netdevsim driver which is part of the Linux kernel provides a dummy driver
  which implements offload interfaces for XDP BPF and tc BPF programs and
  facilitates testing kernel changes or low-level user space programs
  implementing a control plane directly against the kernel's UAPI.

  A netdevsim device can be created as follows:

  ::

    # ip link add dev sim0 type netdevsim
    # ip link set dev sim0 up
    # ethtool -K sim0 hw-tc-offload on
    # ip l
    [...]
    7: sim0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
        link/ether a2:24:4c:1c:c2:b3 brd ff:ff:ff:ff:ff:ff

  After that step, XDP BPF or tc BPF programs can be test loaded as shown
  in the various examples earlier:

  ::

    # ip -force link set dev sim0 xdpoffload obj prog.o
    # ip l
    [...]
    7: sim0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 xdpoffload qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
        link/ether a2:24:4c:1c:c2:b3 brd ff:ff:ff:ff:ff:ff
        prog/xdp id 20 tag 57cd311f2e27366b

These two workflows are the basic operations to load XDP BPF respectively tc BPF
programs with iproute2.

There are other various advanced options for the BPF loader that apply both to XDP
and tc, some of them are listed here. In the examples only XDP is presented for
simplicity.

**1. Verbose log output even on success.**

  The option ``verb`` can be appended for loading programs in order to dump the
  verifier log, even if no error occurred:

  ::

    # ip link set dev em1 xdp obj xdp-example.o verb

    Prog section 'prog' loaded (5)!
     - Type:         6
     - Instructions: 2 (0 over limit)
     - License:      GPL

    Verifier analysis:

    0: (b7) r0 = 1
    1: (95) exit
    processed 2 insns

**2. Load program that is already pinned in BPF file system.**

  Instead of loading a program from an object file, iproute2 can also retrieve
  the program from the BPF file system in case some external entity pinned it
  there and attach it to the device:

  ::

  # ip link set dev em1 xdp pinned /sys/fs/bpf/prog

  iproute2 can also use the short form that is relative to the detected mount
  point of the BPF file system:

  ::

  # ip link set dev em1 xdp pinned m:prog

When loading BPF programs, iproute2 will automatically detect the mounted
file system instance in order to perform pinning of nodes. In case no mounted
BPF file system instance was found, then tc will automatically mount it
to the default location under ``/sys/fs/bpf/``.

In case an instance has already been found, then it will be used and no additional
mount will be performed:

  ::

    # mkdir /var/run/bpf
    # mount --bind /var/run/bpf /var/run/bpf
    # mount -t bpf bpf /var/run/bpf
    # tc filter add dev em1 ingress bpf da obj tc-example.o sec prog
    # tree /var/run/bpf
    /var/run/bpf
    +-- ip -> /run/bpf/tc/
    +-- tc
    |   +-- globals
    |       +-- jmp_map
    +-- xdp -> /run/bpf/tc/

    4 directories, 1 file

By default tc will create an initial directory structure as shown above,
where all subsystem users will point to the same location through symbolic
links for the ``globals`` namespace, so that pinned BPF maps can be reused
among various BPF program types in iproute2. In case the file system instance
has already been mounted and an existing structure already exists, then tc will
not override it. This could be the case for separating ``lwt``, ``tc`` and
``xdp`` maps in order to not share ``globals`` among all.

As briefly covered in the previous LLVM section, iproute2 will install a
header file upon installation which can be included through the standard
include path by BPF programs:

  ::

    #include <iproute2/bpf_elf.h>

The purpose of this header file is to provide an API for maps and default section
names used by programs. It's a stable contract between iproute2 and BPF programs.

The map definition for iproute2 is ``struct bpf_elf_map``. Its members have
been covered earlier in the LLVM section of this document.

When parsing the BPF object file, the iproute2 loader will walk through
all ELF sections. It initially fetches ancillary sections like ``maps`` and
``license``. For ``maps``, the ``struct bpf_elf_map`` array will be checked
for validity and whenever needed, compatibility workarounds are performed.
Subsequently all maps are created with the user provided information, either
retrieved as a pinned object, or newly created and then pinned into the BPF
file system. Next the loader will handle all program sections that contain
ELF relocation entries for maps, meaning that BPF instructions loading
map file descriptors into registers are rewritten so that the corresponding
map file descriptors are encoded into the instructions immediate value, in
order for the kernel to be able to convert them later on into map kernel
pointers. After that all the programs themselves are created through the BPF
system call, and tail called maps, if present, updated with the program's file
descriptors.

bpftool
-------

bpftool is the main introspection and debugging tool around BPF and developed
and shipped along with the Linux kernel tree under ``tools/bpf/bpftool/``.

The tool can dump all BPF programs and maps that are currently loaded in
the system, or list and correlate all BPF maps used by a specific program.
Furthermore, it allows to dump the entire map's key / value pairs, or
lookup, update, delete individual ones as well as retrieve a key's neighbor
key in the map. Such operations can be performed based on BPF program or
map IDs or by specifying the location of a BPF file system pinned program
or map. The tool additionally also offers an option to pin maps or programs
into the BPF file system.

For a quick overview of all BPF programs currently loaded on the host
invoke the following command:

  ::

     # bpftool prog
     398: sched_cls  tag 56207908be8ad877
        loaded_at Apr 09/16:24  uid 0
        xlated 8800B  jited 6184B  memlock 12288B  map_ids 18,5,17,14
     399: sched_cls  tag abc95fb4835a6ec9
        loaded_at Apr 09/16:24  uid 0
        xlated 344B  jited 223B  memlock 4096B  map_ids 18
     400: sched_cls  tag afd2e542b30ff3ec
        loaded_at Apr 09/16:24  uid 0
        xlated 1720B  jited 1001B  memlock 4096B  map_ids 17
     401: sched_cls  tag 2dbbd74ee5d51cc8
        loaded_at Apr 09/16:24  uid 0
        xlated 3728B  jited 2099B  memlock 4096B  map_ids 17
     [...]

Similarly, to get an overview of all active maps:

  ::

    # bpftool map
    5: hash  flags 0x0
        key 20B  value 112B  max_entries 65535  memlock 13111296B
    6: hash  flags 0x0
        key 20B  value 20B  max_entries 65536  memlock 7344128B
    7: hash  flags 0x0
        key 10B  value 16B  max_entries 8192  memlock 790528B
    8: hash  flags 0x0
        key 22B  value 28B  max_entries 8192  memlock 987136B
    9: hash  flags 0x0
        key 20B  value 8B  max_entries 512000  memlock 49352704B
    [...]

Note that for each command, bpftool also supports json based output by
appending ``--json`` at the end of the command line. An additional
``--pretty`` improves the output to be more human readable.

  ::

     # bpftool prog --json --pretty

For dumping the post-verifier BPF instruction image of a specific BPF
program, one starting point could be to inspect a specific program, e.g.
attached to the tc ingress hook:

  ::

     # tc filter show dev cilium_host egress
     filter protocol all pref 1 bpf chain 0
     filter protocol all pref 1 bpf chain 0 handle 0x1 bpf_host.o:[from-netdev] \
                         direct-action not_in_hw id 406 tag e0362f5bd9163a0a jited

The program from the object file ``bpf_host.o``, section ``from-netdev`` has
a BPF program ID of ``406`` as denoted in ``id 406``. Based on this information
bpftool can provide some high-level metadata specific to the program:

  ::

     # bpftool prog show id 406
     406: sched_cls  tag e0362f5bd9163a0a
          loaded_at Apr 09/16:24  uid 0
          xlated 11144B  jited 7721B  memlock 12288B  map_ids 18,20,8,5,6,14

The program of ID 406 is of type ``sched_cls`` (``BPF_PROG_TYPE_SCHED_CLS``),
has a ``tag`` of ``e0362f5bd9163a0a`` (sha sum over the instruction sequence),
it was loaded by root ``uid 0`` on ``Apr 09/16:24``. The BPF instruction
sequence is ``11,144 bytes`` long and the JITed image ``7,721 bytes``. The
program itself (excluding maps) consumes ``12,288 bytes`` that are accounted /
charged against user ``uid 0``. And the BPF program uses the BPF maps with
IDs ``18``, ``20``, ``8``, ``5``, ``6`` and ``14``. The latter IDs can further
be used to get information or dump the map themselves.

Additionally, bpftool can issue a dump request of the BPF instructions the
program runs:

  ::

     # bpftool prog dump xlated id 406
      0: (b7) r7 = 0
      1: (63) *(u32 *)(r1 +60) = r7
      2: (63) *(u32 *)(r1 +56) = r7
      3: (63) *(u32 *)(r1 +52) = r7
     [...]
     47: (bf) r4 = r10
     48: (07) r4 += -40
     49: (79) r6 = *(u64 *)(r10 -104)
     50: (bf) r1 = r6
     51: (18) r2 = map[id:18]                    <-- BPF map id 18
     53: (b7) r5 = 32
     54: (85) call bpf_skb_event_output#5656112  <-- BPF helper call
     55: (69) r1 = *(u16 *)(r6 +192)
     [...]

bpftool correlates BPF map IDs into the instruction stream as shown above
as well as calls to BPF helpers or other BPF programs.

The instruction dump reuses the same 'pretty-printer' as the kernel's BPF
verifier. Since the program was JITed and therefore the actual JIT image
that was generated out of above ``xlated`` instructions is executed, it
can be dumped as well through bpftool:

  ::

     # bpftool prog dump jited id 406
      0:        push   %rbp
      1:        mov    %rsp,%rbp
      4:        sub    $0x228,%rsp
      b:        sub    $0x28,%rbp
      f:        mov    %rbx,0x0(%rbp)
     13:        mov    %r13,0x8(%rbp)
     17:        mov    %r14,0x10(%rbp)
     1b:        mov    %r15,0x18(%rbp)
     1f:        xor    %eax,%eax
     21:        mov    %rax,0x20(%rbp)
     25:        mov    0x80(%rdi),%r9d
     [...]

Mainly for BPF JIT developers, the option also exists to interleave the
disassembly with the actual native opcodes:

  ::

     # bpftool prog dump jited id 406 opcodes
      0:        push   %rbp
                55
      1:        mov    %rsp,%rbp
                48 89 e5
      4:        sub    $0x228,%rsp
                48 81 ec 28 02 00 00
      b:        sub    $0x28,%rbp
                48 83 ed 28
      f:        mov    %rbx,0x0(%rbp)
                48 89 5d 00
     13:        mov    %r13,0x8(%rbp)
                4c 89 6d 08
     17:        mov    %r14,0x10(%rbp)
                4c 89 75 10
     1b:        mov    %r15,0x18(%rbp)
                4c 89 7d 18
     [...]

The same interleaving can be done for the normal BPF instructions which
can sometimes be useful for debugging in the kernel:

  ::

     # bpftool prog dump xlated id 406 opcodes
      0: (b7) r7 = 0
         b7 07 00 00 00 00 00 00
      1: (63) *(u32 *)(r1 +60) = r7
         63 71 3c 00 00 00 00 00
      2: (63) *(u32 *)(r1 +56) = r7
         63 71 38 00 00 00 00 00
      3: (63) *(u32 *)(r1 +52) = r7
         63 71 34 00 00 00 00 00
      4: (63) *(u32 *)(r1 +48) = r7
         63 71 30 00 00 00 00 00
      5: (63) *(u32 *)(r1 +64) = r7
         63 71 40 00 00 00 00 00
      [...]

The basic blocks of a program can also be visualized with the help of
``graphviz``. For this purpose bpftool has a ``visual`` dump mode that
generates a dot file instead of the plain BPF ``xlated`` instruction
dump that can later be converted to a png file:

  ::

     # bpftool prog dump xlated id 406 visual &> output.dot
     $ dot -Tpng output.dot -o output.png

Another option would be to pass the dot file to dotty as a viewer, that
is ``dotty output.dot``, where the result for the ``bpf_host.o`` program
looks as follows (small extract):

.. image:: images/bpf_dot.png
    :align: center

Note that the ``xlated`` instruction dump provides the post-verifier BPF
instruction image which means that it dumps the instructions as if they
were to be run through the BPF interpreter. In the kernel, the verifier
performs various rewrites of the original instructions provided by the
BPF loader.

One example of rewrites is the inlining of helper functions in order to
improve runtime performance, here in the case of a map lookup for hash
tables:

  ::

     # bpftool prog dump xlated id 3
      0: (b7) r1 = 2
      1: (63) *(u32 *)(r10 -4) = r1
      2: (bf) r2 = r10
      3: (07) r2 += -4
      4: (18) r1 = map[id:2]                      <-- BPF map id 2
      6: (85) call __htab_map_lookup_elem#77408   <-+ BPF helper inlined rewrite
      7: (15) if r0 == 0x0 goto pc+2                |
      8: (07) r0 += 56                              |
      9: (79) r0 = *(u64 *)(r0 +0)                <-+
     10: (15) if r0 == 0x0 goto pc+24
     11: (bf) r2 = r10
     12: (07) r2 += -4
     [...]

bpftool correlates calls to helper functions or BPF to BPF calls through
kallsyms. Therefore, make sure that JITed BPF programs are exposed to
kallsyms (``bpf_jit_kallsyms``) and that kallsyms addresses are not
obfuscated (calls are otherwise shown as ``call bpf_unspec#0``):

  ::

     # echo 0 > /proc/sys/kernel/kptr_restrict
     # echo 1 > /proc/sys/net/core/bpf_jit_kallsyms

BPF to BPF calls are correlated as well for both, interpreter as well
as JIT case. In the latter, the tag of the subprogram is shown as
call target. In each case, the ``pc+2`` is the pc-relative offset of
the call target, which denotes the subprogram.

  ::

     # bpftool prog dump xlated id 1
     0: (85) call pc+2#__bpf_prog_run_args32
     1: (b7) r0 = 1
     2: (95) exit
     3: (b7) r0 = 2
     4: (95) exit

JITed variant of the dump:

  ::

     # bpftool prog dump xlated id 1
     0: (85) call pc+2#bpf_prog_3b185187f1855c4c_F
     1: (b7) r0 = 1
     2: (95) exit
     3: (b7) r0 = 2
     4: (95) exit

In the case of tail calls, the kernel maps them into a single instruction
internally, bpftool will still correlate them as a helper call for ease
of debugging:

  ::

     # bpftool prog dump xlated id 2
     [...]
     10: (b7) r2 = 8
     11: (85) call bpf_trace_printk#-41312
     12: (bf) r1 = r6
     13: (18) r2 = map[id:1]
     15: (b7) r3 = 0
     16: (85) call bpf_tail_call#12
     17: (b7) r1 = 42
     18: (6b) *(u16 *)(r6 +46) = r1
     19: (b7) r0 = 0
     20: (95) exit

     # bpftool map show id 1
     1: prog_array  flags 0x0
           key 4B  value 4B  max_entries 1  memlock 4096B

Dumping an entire map is possible through the ``map dump`` subcommand
which iterates through all present map elements and dumps the key /
value pairs.

If no BTF (BPF Type Format) data is available for a given map, then
the key / value pairs are dumped as hex:

  ::

     # bpftool map dump id 5
     key:
     f0 0d 00 00 00 00 00 00  0a 66 00 00 00 00 8a d6
     02 00 00 00
     value:
     00 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00
     00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
     00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
     00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
     00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
     00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
     00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
     key:
     0a 66 1c ee 00 00 00 00  00 00 00 00 00 00 00 00
     01 00 00 00
     value:
     00 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00
     00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
     [...]
     Found 6 elements

However, with BTF, the map also holds debugging information about
the key and value structures. For example, BTF in combination with
BPF maps and the BPF_ANNOTATE_KV_PAIR() macro from iproute2 will
result in the following dump (``test_xdp_noinline.o`` from kernel
selftests):

  ::

     # cat tools/testing/selftests/bpf/test_xdp_noinline.c
       [...]
        struct ctl_value {
              union {
                      __u64 value;
                      __u32 ifindex;
                      __u8 mac[6];
              };
        };

        struct bpf_map_def __attribute__ ((section("maps"), used)) ctl_array = {
               .type		= BPF_MAP_TYPE_ARRAY,
               .key_size	= sizeof(__u32),
               .value_size	= sizeof(struct ctl_value),
               .max_entries	= 16,
               .map_flags	= 0,
        };
        BPF_ANNOTATE_KV_PAIR(ctl_array, __u32, struct ctl_value);

        [...]

The BPF_ANNOTATE_KV_PAIR() macro forces a map-specific ELF section
containing an empty key and value, this enables the iproute2 BPF loader
to correlate BTF data with that section and thus allows to choose the
corresponding types out of the BTF for loading the map.

Compiling through LLVM and generating BTF through debugging information
by ``pahole``:

  ::

     # clang [...] -O2 -target bpf -g -emit-llvm -c test_xdp_noinline.c -o - |
       llc -march=bpf -mcpu=probe -mattr=dwarfris -filetype=obj -o test_xdp_noinline.o
     # pahole -J test_xdp_noinline.o

Now loading into kernel and dumping the map via bpftool:

  ::

     # ip -force link set dev lo xdp obj test_xdp_noinline.o sec xdp-test
     # ip a
     1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 xdpgeneric/id:227 qdisc noqueue state UNKNOWN group default qlen 1000
         link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
         inet 127.0.0.1/8 scope host lo
            valid_lft forever preferred_lft forever
         inet6 ::1/128 scope host
            valid_lft forever preferred_lft forever
     [...]
     # bpftool prog show id 227
     227: xdp  tag a85e060c275c5616  gpl
         loaded_at 2018-07-17T14:41:29+0000  uid 0
         xlated 8152B  not jited  memlock 12288B  map_ids 381,385,386,382,384,383
     # bpftool map dump id 386
      [{
           "key": 0,
           "value": {
               "": {
                   "value": 0,
                   "ifindex": 0,
                   "mac": []
               }
           }
       },{
           "key": 1,
           "value": {
               "": {
                   "value": 0,
                   "ifindex": 0,
                   "mac": []
               }
           }
       },{
     [...]

Lookup, update, delete, and 'get next key' operations on the map for specific
keys can be performed through bpftool as well.

BPF sysctls
-----------

The Linux kernel provides few sysctls that are BPF related and covered in this section.

* ``/proc/sys/net/core/bpf_jit_enable``: Enables or disables the BPF JIT compiler.

  +-------+-------------------------------------------------------------------+
  | Value | Description                                                       |
  +-------+-------------------------------------------------------------------+
  | 0     | Disable the JIT and use only interpreter (kernel's default value) |
  +-------+-------------------------------------------------------------------+
  | 1     | Enable the JIT compiler                                           |
  +-------+-------------------------------------------------------------------+
  | 2     | Enable the JIT and emit debugging traces to the kernel log        |
  +-------+-------------------------------------------------------------------+

  As described in subsequent sections, ``bpf_jit_disasm`` tool can be used to
  process debugging traces when the JIT compiler is set to debugging mode (option ``2``).

* ``/proc/sys/net/core/bpf_jit_harden``: Enables or disables BPF JIT hardening.
  Note that enabling hardening trades off performance, but can mitigate JIT spraying
  by blinding out the BPF program's immediate values. For programs processed through
  the interpreter, blinding of immediate values is not needed / performed.

  +-------+-------------------------------------------------------------------+
  | Value | Description                                                       |
  +-------+-------------------------------------------------------------------+
  | 0     | Disable JIT hardening (kernel's default value)                    |
  +-------+-------------------------------------------------------------------+
  | 1     | Enable JIT hardening for unprivileged users only                  |
  +-------+-------------------------------------------------------------------+
  | 2     | Enable JIT hardening for all users                                |
  +-------+-------------------------------------------------------------------+

* ``/proc/sys/net/core/bpf_jit_kallsyms``: Enables or disables export of JITed
  programs as kernel symbols to ``/proc/kallsyms`` so that they can be used together
  with ``perf`` tooling as well as making these addresses aware to the kernel for
  stack unwinding, for example, used in dumping stack traces. The symbol names
  contain the BPF program tag (``bpf_prog_<tag>``). If ``bpf_jit_harden`` is enabled,
  then this feature is disabled.

  +-------+-------------------------------------------------------------------+
  | Value | Description                                                       |
  +-------+-------------------------------------------------------------------+
  | 0     | Disable JIT kallsyms export (kernel's default value)              |
  +-------+-------------------------------------------------------------------+
  | 1     | Enable JIT kallsyms export for privileged users only              |
  +-------+-------------------------------------------------------------------+

* ``/proc/sys/kernel/unprivileged_bpf_disabled``: Enables or disable unprivileged
  use of the ``bpf(2)`` system call. The Linux kernel has unprivileged use of
  ``bpf(2)`` enabled by default, but once the switch is flipped, unprivileged use
  will be permanently disabled until the next reboot. This sysctl knob is a one-time
  switch, meaning if once set, then neither an application nor an admin can reset
  the value anymore. This knob does not affect any cBPF programs such as seccomp
  or traditional socket filters that do not use the ``bpf(2)`` system call for
  loading the program into the kernel.

  +-------+-------------------------------------------------------------------+
  | Value | Description                                                       |
  +-------+-------------------------------------------------------------------+
  | 0     | Unprivileged use of bpf syscall enabled (kernel's default value)  |
  +-------+-------------------------------------------------------------------+
  | 1     | Unprivileged use of bpf syscall disabled                          |
  +-------+-------------------------------------------------------------------+

Kernel Testing
--------------

The Linux kernel ships a BPF selftest suite, which can be found in the kernel
source tree under ``tools/testing/selftests/bpf/``.

::

    $ cd tools/testing/selftests/bpf/
    $ make
    # make run_tests

The test suite contains test cases against the BPF verifier, program tags,
various tests against the BPF map interface and map types. It contains various
runtime tests from C code for checking LLVM back end, and eBPF as well as cBPF
asm code that is run in the kernel for testing the interpreter and JITs.

JIT Debugging
-------------

For JIT developers performing audits or writing extensions, each compile run
can output the generated JIT image into the kernel log through:

::

    # echo 2 > /proc/sys/net/core/bpf_jit_enable

Whenever a new BPF program is loaded, the JIT compiler will dump the output,
which can then be inspected with ``dmesg``, for example:

::

    [ 3389.935842] flen=6 proglen=70 pass=3 image=ffffffffa0069c8f from=tcpdump pid=20583
    [ 3389.935847] JIT code: 00000000: 55 48 89 e5 48 83 ec 60 48 89 5d f8 44 8b 4f 68
    [ 3389.935849] JIT code: 00000010: 44 2b 4f 6c 4c 8b 87 d8 00 00 00 be 0c 00 00 00
    [ 3389.935850] JIT code: 00000020: e8 1d 94 ff e0 3d 00 08 00 00 75 16 be 17 00 00
    [ 3389.935851] JIT code: 00000030: 00 e8 28 94 ff e0 83 f8 01 75 07 b8 ff ff 00 00
    [ 3389.935852] JIT code: 00000040: eb 02 31 c0 c9 c3

``flen`` is the length of the BPF program (here, 6 BPF instructions), and ``proglen``
tells the number of bytes generated by the JIT for the opcode image (here, 70 bytes
in size). ``pass`` means that the image was generated in 3 compiler passes, for
example, ``x86_64`` can have various optimization passes to further reduce the image
size when possible. ``image`` contains the address of the generated JIT image, ``from``
and ``pid`` the user space application name and PID respectively, which triggered the
compilation process. The dump output for eBPF and cBPF JITs is the same format.

In the kernel tree under ``tools/bpf/``, there is a tool called ``bpf_jit_disasm``. It
reads out the latest dump and prints the disassembly for further inspection:

::

    # ./bpf_jit_disasm
    70 bytes emitted from JIT compiler (pass:3, flen:6)
    ffffffffa0069c8f + <x>:
       0:       push   %rbp
       1:       mov    %rsp,%rbp
       4:       sub    $0x60,%rsp
       8:       mov    %rbx,-0x8(%rbp)
       c:       mov    0x68(%rdi),%r9d
      10:       sub    0x6c(%rdi),%r9d
      14:       mov    0xd8(%rdi),%r8
      1b:       mov    $0xc,%esi
      20:       callq  0xffffffffe0ff9442
      25:       cmp    $0x800,%eax
      2a:       jne    0x0000000000000042
      2c:       mov    $0x17,%esi
      31:       callq  0xffffffffe0ff945e
      36:       cmp    $0x1,%eax
      39:       jne    0x0000000000000042
      3b:       mov    $0xffff,%eax
      40:       jmp    0x0000000000000044
      42:       xor    %eax,%eax
      44:       leaveq
      45:       retq

Alternatively, the tool can also dump related opcodes along with the disassembly.

::

    # ./bpf_jit_disasm -o
    70 bytes emitted from JIT compiler (pass:3, flen:6)
    ffffffffa0069c8f + <x>:
       0:       push   %rbp
        55
       1:       mov    %rsp,%rbp
        48 89 e5
       4:       sub    $0x60,%rsp
        48 83 ec 60
       8:       mov    %rbx,-0x8(%rbp)
        48 89 5d f8
       c:       mov    0x68(%rdi),%r9d
        44 8b 4f 68
      10:       sub    0x6c(%rdi),%r9d
        44 2b 4f 6c
      14:       mov    0xd8(%rdi),%r8
        4c 8b 87 d8 00 00 00
      1b:       mov    $0xc,%esi
        be 0c 00 00 00
      20:       callq  0xffffffffe0ff9442
        e8 1d 94 ff e0
      25:       cmp    $0x800,%eax
        3d 00 08 00 00
      2a:       jne    0x0000000000000042
        75 16
      2c:       mov    $0x17,%esi
        be 17 00 00 00
      31:       callq  0xffffffffe0ff945e
        e8 28 94 ff e0
      36:       cmp    $0x1,%eax
        83 f8 01
      39:       jne    0x0000000000000042
        75 07
      3b:       mov    $0xffff,%eax
        b8 ff ff 00 00
      40:       jmp    0x0000000000000044
        eb 02
      42:       xor    %eax,%eax
        31 c0
      44:       leaveq
        c9
      45:       retq
        c3

More recently, ``bpftool`` adapted the same feature of dumping the BPF JIT
image based on a given BPF program ID already loaded in the system (see
bpftool section).

For performance analysis of JITed BPF programs, ``perf`` can be used as
usual. As a prerequisite, JITed programs need to be exported through kallsyms
infrastructure.

::

    # echo 1 > /proc/sys/net/core/bpf_jit_enable
    # echo 1 > /proc/sys/net/core/bpf_jit_kallsyms

Enabling or disabling ``bpf_jit_kallsyms`` does not require a reload of the
related BPF programs. Next, a small workflow example is provided for profiling
BPF programs. A crafted tc BPF program is used for demonstration purposes,
where perf records a failed allocation inside ``bpf_clone_redirect()`` helper.
Due to the use of direct write, ``bpf_try_make_head_writable()`` failed, which
would then release the cloned ``skb`` again and return with an error message.
``perf`` thus records all ``kfree_skb`` events.

::

    # tc qdisc add dev em1 clsact
    # tc filter add dev em1 ingress bpf da obj prog.o sec main
    # tc filter show dev em1 ingress
    filter protocol all pref 49152 bpf
    filter protocol all pref 49152 bpf handle 0x1 prog.o:[main] direct-action id 1 tag 8227addf251b7543

    # cat /proc/kallsyms
    [...]
    ffffffffc00349e0 t fjes_hw_init_command_registers    [fjes]
    ffffffffc003e2e0 d __tracepoint_fjes_hw_stop_debug_err    [fjes]
    ffffffffc0036190 t fjes_hw_epbuf_tx_pkt_send    [fjes]
    ffffffffc004b000 t bpf_prog_8227addf251b7543

    # perf record -a -g -e skb:kfree_skb sleep 60
    # perf script --kallsyms=/proc/kallsyms
    [...]
    ksoftirqd/0     6 [000]  1004.578402:    skb:kfree_skb: skbaddr=0xffff9d4161f20a00 protocol=2048 location=0xffffffffc004b52c
       7fffb8745961 bpf_clone_redirect (/lib/modules/4.10.0+/build/vmlinux)
       7fffc004e52c bpf_prog_8227addf251b7543 (/lib/modules/4.10.0+/build/vmlinux)
       7fffc05b6283 cls_bpf_classify (/lib/modules/4.10.0+/build/vmlinux)
       7fffb875957a tc_classify (/lib/modules/4.10.0+/build/vmlinux)
       7fffb8729840 __netif_receive_skb_core (/lib/modules/4.10.0+/build/vmlinux)
       7fffb8729e38 __netif_receive_skb (/lib/modules/4.10.0+/build/vmlinux)
       7fffb872ae05 process_backlog (/lib/modules/4.10.0+/build/vmlinux)
       7fffb872a43e net_rx_action (/lib/modules/4.10.0+/build/vmlinux)
       7fffb886176c __do_softirq (/lib/modules/4.10.0+/build/vmlinux)
       7fffb80ac5b9 run_ksoftirqd (/lib/modules/4.10.0+/build/vmlinux)
       7fffb80ca7fa smpboot_thread_fn (/lib/modules/4.10.0+/build/vmlinux)
       7fffb80c6831 kthread (/lib/modules/4.10.0+/build/vmlinux)
       7fffb885e09c ret_from_fork (/lib/modules/4.10.0+/build/vmlinux)

The stack trace recorded by ``perf`` will then show the ``bpf_prog_8227addf251b7543()``
symbol as part of the call trace, meaning that the BPF program with the
tag ``8227addf251b7543`` was related to the ``kfree_skb`` event, and
such program was attached to netdevice ``em1`` on the ingress hook as
shown by tc.

Introspection
-------------

The Linux kernel provides various tracepoints around BPF and XDP which
can be used for additional introspection, for example, to trace interactions
of user space programs with the bpf system call.

Tracepoints for BPF:

::

    # perf list | grep bpf:
    bpf:bpf_map_create                                 [Tracepoint event]
    bpf:bpf_map_delete_elem                            [Tracepoint event]
    bpf:bpf_map_lookup_elem                            [Tracepoint event]
    bpf:bpf_map_next_key                               [Tracepoint event]
    bpf:bpf_map_update_elem                            [Tracepoint event]
    bpf:bpf_obj_get_map                                [Tracepoint event]
    bpf:bpf_obj_get_prog                               [Tracepoint event]
    bpf:bpf_obj_pin_map                                [Tracepoint event]
    bpf:bpf_obj_pin_prog                               [Tracepoint event]
    bpf:bpf_prog_get_type                              [Tracepoint event]
    bpf:bpf_prog_load                                  [Tracepoint event]
    bpf:bpf_prog_put_rcu                               [Tracepoint event]

Example usage with ``perf`` (alternatively to ``sleep`` example used here,
a specific application like ``tc`` could be used here instead, of course):

::

    # perf record -a -e bpf:* sleep 10
    # perf script
    sock_example  6197 [005]   283.980322:      bpf:bpf_map_create: map type=ARRAY ufd=4 key=4 val=8 max=256 flags=0
    sock_example  6197 [005]   283.980721:       bpf:bpf_prog_load: prog=a5ea8fa30ea6849c type=SOCKET_FILTER ufd=5
    sock_example  6197 [005]   283.988423:   bpf:bpf_prog_get_type: prog=a5ea8fa30ea6849c type=SOCKET_FILTER
    sock_example  6197 [005]   283.988443: bpf:bpf_map_lookup_elem: map type=ARRAY ufd=4 key=[06 00 00 00] val=[00 00 00 00 00 00 00 00]
    [...]
    sock_example  6197 [005]   288.990868: bpf:bpf_map_lookup_elem: map type=ARRAY ufd=4 key=[01 00 00 00] val=[14 00 00 00 00 00 00 00]
         swapper     0 [005]   289.338243:    bpf:bpf_prog_put_rcu: prog=a5ea8fa30ea6849c type=SOCKET_FILTER

For the BPF programs, their individual program tag is displayed.

For debugging, XDP also has a tracepoint that is triggered when exceptions are raised:

::

    # perf list | grep xdp:
    xdp:xdp_exception                                  [Tracepoint event]

Exceptions are triggered in the following scenarios:

* The BPF program returned an invalid / unknown XDP action code.
* The BPF program returned with ``XDP_ABORTED`` indicating a non-graceful exit.
* The BPF program returned with ``XDP_TX``, but there was an error on transmit,
  for example, due to the port not being up, due to the transmit ring being full,
  due to allocation failures, etc.

Both tracepoint classes can also be inspected with a BPF program itself
attached to one or more tracepoints, collecting further information
in a map or punting such events to a user space collector through the
``bpf_perf_event_output()`` helper, for example.

Miscellaneous
-------------

BPF programs and maps are memory accounted against ``RLIMIT_MEMLOCK`` similar
to ``perf``. The currently available size in unit of system pages which may be
locked into memory can be inspected through ``ulimit -l``. The setrlimit system
call man page provides further details.

The default limit is usually insufficient to load more complex programs or
larger BPF maps, so that the BPF system call will return with ``errno``
of ``EPERM``. In such situations a workaround with ``ulimit -l unlimited`` or
with a sufficiently large limit could be performed. The ``RLIMIT_MEMLOCK`` is
mainly enforcing limits for unprivileged users. Depending on the setup,
setting a higher limit for privileged users is often acceptable.

Program Types
=============

At the time of this writing, there are eighteen different BPF program types
available, two of the main types for networking are further explained in below
subsections, namely XDP BPF programs as well as tc BPF programs. Extensive
usage examples for the two program types for LLVM, iproute2 or other tools
are spread throughout the toolchain section and not covered here. Instead,
this section focuses on their architecture, concepts and use cases.

XDP
---

XDP stands for eXpress Data Path and provides a framework for BPF that enables
high-performance programmable packet processing in the Linux kernel. It runs
the BPF program at the earliest possible point in software, namely at the moment
the network driver receives the packet.

At this point in the fast-path the driver just picked up the packet from its
receive rings, without having done any expensive operations such as allocating
an ``skb`` for pushing the packet further up the networking stack, without
having pushed the packet into the GRO engine, etc. Thus, the XDP BPF program
is executed at the earliest point when it becomes available to the CPU for
processing.

XDP works in concert with the Linux kernel and its infrastructure, meaning
the kernel is not bypassed as in various networking frameworks that operate
in user space only. Keeping the packet in kernel space has several major
advantages:

* XDP is able to reuse all the upstream developed kernel networking drivers,
  user space tooling, or even other available in-kernel infrastructure such
  as routing tables, sockets, etc in BPF helper calls itself.
* Residing in kernel space, XDP has the same security model as the rest of
  the kernel for accessing hardware.
* There is no need for crossing kernel / user space boundaries since the
  processed packet already resides in the kernel and can therefore flexibly
  forward packets into other in-kernel entities like namespaces used by
  containers or the kernel's networking stack itself. This is particularly
  relevant in times of Meltdown and Spectre.
* Punting packets from XDP to the kernel's robust, widely used and efficient
  TCP/IP stack is trivially possible, allows for full reuse and does not
  require maintaining a separate TCP/IP stack as with user space frameworks.
* The use of BPF allows for full programmability, keeping a stable ABI with
  the same 'never-break-user-space' guarantees as with the kernel's system
  call ABI and compared to modules it also provides safety measures thanks to
  the BPF verifier that ensures the stability of the kernel's operation.
* XDP trivially allows for atomically swapping programs during runtime without
  any network traffic interruption or even kernel / system reboot.
* XDP allows for flexible structuring of workloads integrated into
  the kernel. For example, it can operate in "busy polling" or "interrupt
  driven" mode. Explicitly dedicating CPUs to XDP is not required. There
  are no special hardware requirements and it does not rely on hugepages.
* XDP does not require any third party kernel modules or licensing. It is
  a long-term architectural solution, a core part of the Linux kernel, and
  developed by the kernel community.
* XDP is already enabled and shipped everywhere with major distributions
  running a kernel equivalent to 4.8 or higher and supports most major 10G
  or higher networking drivers.

As a framework for running BPF in the driver, XDP additionally ensures that
packets are laid out linearly and fit into a single DMA'ed page which is
readable and writable by the BPF program. XDP also ensures that additional
headroom of 256 bytes is available to the program for implementing custom
encapsulation headers with the help of the ``bpf_xdp_adjust_head()`` BPF helper
or adding custom metadata in front of the packet through ``bpf_xdp_adjust_meta()``.

The framework contains XDP action codes further described in the section
below which a BPF program can return in order to instruct the driver how
to proceed with the packet, and it enables the possibility to atomically
replace BPF programs running at the XDP layer. XDP is tailored for
high-performance by design. BPF allows to access the packet data through
'direct packet access' which means that the program holds data pointers
directly in registers, loads the content into registers, respectively
writes from there into the packet.

The packet representation in XDP that is passed to the BPF program as
the BPF context looks as follows:

::

    struct xdp_buff {
        void *data;
        void *data_end;
        void *data_meta;
        void *data_hard_start;
        struct xdp_rxq_info *rxq;
    };

``data`` points to the start of the packet data in the page, and as the
name suggests, ``data_end`` points to the end of the packet data. Since XDP
allows for a headroom, ``data_hard_start`` points to the maximum possible
headroom start in the page, meaning, when the packet should be encapsulated,
then ``data`` is moved closer towards ``data_hard_start`` via ``bpf_xdp_adjust_head()``.
The same BPF helper function also allows for decapsulation in which case
``data`` is moved further away from ``data_hard_start``.

``data_meta`` initially points to the same location as ``data`` but
``bpf_xdp_adjust_meta()`` is able to move the pointer towards ``data_hard_start``
as well in order to provide room for custom metadata which is invisible to
the normal kernel networking stack but can be read by tc BPF programs since
it is transferred from XDP to the ``skb``. Vice versa, it can remove or reduce
the size of the custom metadata through the same BPF helper function by
moving ``data_meta`` away from ``data_hard_start`` again. ``data_meta`` can
also be used solely for passing state between tail calls similarly to the
``skb->cb[]`` control block case that is accessible in tc BPF programs.

This gives the following relation respectively invariant for the ``struct xdp_buff``
packet pointers: ``data_hard_start`` <= ``data_meta`` <= ``data`` < ``data_end``.

The ``rxq`` field points to some additional per receive queue metadata which
is populated at ring setup time (not at XDP runtime):

::

    struct xdp_rxq_info {
        struct net_device *dev;
        u32 queue_index;
        u32 reg_state;
    } ____cacheline_aligned;

The BPF program can retrieve ``queue_index`` as well as additional data
from the netdevice itself such as ``ifindex``, etc.

**BPF program return codes**

After running the XDP BPF program, a verdict is returned from the program in
order to tell the driver how to process the packet next. In the ``linux/bpf.h``
system header file all available return verdicts are enumerated:

::

    enum xdp_action {
        XDP_ABORTED = 0,
        XDP_DROP,
        XDP_PASS,
        XDP_TX,
        XDP_REDIRECT,
    };

``XDP_DROP`` as the name suggests will drop the packet right at the driver
level without wasting any further resources. This is in particular useful
for BPF programs implementing DDoS mitigation mechanisms or firewalling in
general. The ``XDP_PASS`` return code means that the packet is allowed to
be passed up to the kernel's networking stack. Meaning, the current CPU
that was processing this packet now allocates a ``skb``, populates it, and
passes it onwards into the GRO engine. This would be equivalent to the
default packet handling behavior without XDP. With ``XDP_TX`` the BPF program
has an efficient option to transmit the network packet out of the same NIC it
just arrived on again. This is typically useful when few nodes are implementing,
for example, firewalling with subsequent load balancing in a cluster and
thus act as a hairpinned load balancer pushing the incoming packets back
into the switch after rewriting them in XDP BPF. ``XDP_REDIRECT`` is similar
to ``XDP_TX`` in that it is able to transmit the XDP packet, but through
another NIC. Another option for the ``XDP_REDIRECT`` case is to redirect
into a BPF cpumap, meaning, the CPUs serving XDP on the NIC's receive queues
can continue to do so and push the packet for processing the upper kernel
stack to a remote CPU. This is similar to ``XDP_PASS``, but with the ability
that the XDP BPF program can keep serving the incoming high load as opposed
to temporarily spend work on the current packet for pushing into upper
layers. Last but not least, ``XDP_ABORTED`` which serves denoting an exception
like state from the program and has the same behavior as ``XDP_DROP`` only
that ``XDP_ABORTED`` passes the ``trace_xdp_exception`` tracepoint which
can be additionally monitored to detect misbehavior.

**Use cases for XDP**

Some of the main use cases for XDP are presented in this subsection. The
list is non-exhaustive and given the programmability and efficiency XDP
and BPF enables, it can easily be adapted to solve very specific use
cases.

* **DDoS mitigation, firewalling**

  One of the basic XDP BPF features is to tell the driver to drop a packet
  with ``XDP_DROP`` at this early stage which allows for any kind of efficient
  network policy enforcement with having an extremely low per-packet cost.
  This is ideal in situations when needing to cope with any sort of DDoS
  attacks, but also more general allows to implement any sort of firewalling
  policies with close to no overhead in BPF e.g. in either case as stand alone
  appliance (e.g. scrubbing 'clean' traffic through ``XDP_TX``) or widely
  deployed on nodes protecting end hosts themselves (via ``XDP_PASS`` or
  cpumap ``XDP_REDIRECT`` for good traffic). Offloaded XDP takes this even
  one step further by moving the already small per-packet cost entirely
  into the NIC with processing at line-rate.

..

* **Forwarding and load-balancing**

  Another major use case of XDP is packet forwarding and load-balancing
  through either ``XDP_TX`` or ``XDP_REDIRECT`` actions. The packet can
  be arbitrarily mangled by the BPF program running in the XDP layer,
  even BPF helper functions are available for increasing or decreasing
  the packet's headroom in order to arbitrarily encapsulate respectively
  decapsulate the packet before sending it out again. With ``XDP_TX``
  hairpinned load-balancers can be implemented that push the packet out
  of the same networking device it originally arrived on, or with the
  ``XDP_REDIRECT`` action it can be forwarded to another NIC for
  transmission. The latter return code can also be used in combination
  with BPF's cpumap to load-balance packets for passing up the local
  stack, but on remote, non-XDP processing CPUs.

..

* **Pre-stack filtering / processing**

  Besides policy enforcement, XDP can also be used for hardening the
  kernel's networking stack with the help of ``XDP_DROP`` case, meaning,
  it can drop irrelevant packets for a local node right at the earliest
  possible point before the networking stack sees them e.g. given we
  know that a node only serves TCP traffic, any UDP, SCTP or other L4
  traffic can be dropped right away. This has the advantage that packets
  do not need to traverse various entities like GRO engine, the kernel's
  flow dissector and others before it can be determined to drop them and
  thus this allows for reducing the kernel's attack surface. Thanks to
  XDP's early processing stage, this effectively 'pretends' to the kernel's
  networking stack that these packets have never been seen by the networking
  device. Additionally, if a potential bug in the stack's receive path
  got uncovered and would cause a 'ping of death' like scenario, XDP can be
  utilized to drop such packets right away without having to reboot the
  kernel or restart any services. Due to the ability to atomically swap
  such programs to enforce a drop of bad packets, no network traffic is
  even interrupted on a host.

  Another use case for pre-stack processing is that given the kernel has not
  yet allocated an ``skb`` for the packet, the BPF program is free to modify
  the packet and, again, have it 'pretend' to the stack that it was received
  by the networking device this way. This allows for cases such as having
  custom packet mangling and encapsulation protocols where the packet can be
  decapsulated prior to entering GRO aggregation in which GRO otherwise would
  not be able to perform any sort of aggregation due to not being aware of
  the custom protocol. XDP also allows to push metadata (non-packet data) in
  front of the packet. This is 'invisible' to the normal kernel stack, can
  be GRO aggregated (for matching metadata) and later on processed in
  coordination with a tc ingress BPF program where it has the context of
  a ``skb`` available for e.g. setting various skb fields.

..

* **Flow sampling, monitoring**

  XDP can also be used for cases such as packet monitoring, sampling or any
  other network analytics, for example, as part of an intermediate node in
  the path or on end hosts in combination also with prior mentioned use cases.
  For complex packet analysis, XDP provides a facility to efficiently push
  network packets (truncated or with full payload) and custom metadata into
  a fast lockless per CPU memory mapped ring buffer provided from the Linux
  perf infrastructure to an user space application. This also allows for
  cases where only a flow's initial data can be analyzed and once determined
  as good traffic having the monitoring bypassed. Thanks to the flexibility
  brought by BPF, this allows for implementing any sort of custom monitoring
  or sampling.

..

One example of XDP BPF production usage is Facebook's SHIV and Droplet
infrastructure which implement their L4 load-balancing and DDoS countermeasures.
Migrating their production infrastructure away from netfilter's IPVS
(IP Virtual Server) over to XDP BPF allowed for a 10x speedup compared
to their previous IPVS setup. This was first presented at the netdev 2.1
conference:

* Slides: https://www.netdevconf.org/2.1/slides/apr6/zhou-netdev-xdp-2017.pdf
* Video: https://youtu.be/YEU2ClcGqts

Another example is the integration of XDP into Cloudflare's DDoS mitigation
pipeline, which originally was using cBPF instead of eBPF for attack signature
matching through iptables' ``xt_bpf`` module. Due to use of iptables this
caused severe performance problems under attack where a user space bypass
solution was deemed necessary but came with drawbacks as well such as needing
to busy poll the NIC and expensive packet re-injection into the kernel's stack.
The migration over to eBPF and XDP combined best of both worlds by having
high-performance programmable packet processing directly inside the kernel:

* Slides: https://www.netdevconf.org/2.1/slides/apr6/bertin_Netdev-XDP.pdf
* Video: https://youtu.be/7OuOukmuivg

**XDP operation modes**

XDP has three operation modes where 'native' XDP is the default mode. When
talked about XDP this mode is typically implied.

* **Native XDP**

  This is the default mode where the XDP BPF program is run directly out
  of the networking driver's early receive path. Most widespread used NICs
  for 10G and higher support native XDP already.

..

* **Offloaded XDP**

  In the offloaded XDP mode the XDP BPF program is directly offloaded into
  the NIC instead of being executed on the host CPU. Thus, the already
  extremely low per-packet cost is pushed off the host CPU entirely and
  executed on the NIC, providing even higher performance than running in
  native XDP. This offload is typically implemented by SmartNICs
  containing multi-threaded, multicore flow processors where a in-kernel
  JIT compiler translates BPF into native instructions for the latter.
  Drivers supporting offloaded XDP usually also support native XDP for
  cases where some BPF helpers may not yet or only be available for the
  native mode.

..

* **Generic XDP**

  For drivers not implementing native or offloaded XDP yet, the kernel
  provides an option for generic XDP which does not require any driver
  changes since run at a much later point out of the networking stack.
  This setting is primarily targeted at developers who want to write and
  test programs against the kernel's XDP API, and will not operate at the
  performance rate of the native or offloaded modes. For XDP usage in a
  production environment either the native or offloaded mode is better
  suited and the recommended way to run XDP.

..

**Driver support**

Since BPF and XDP is evolving quickly in terms of feature and driver support,
the following lists native and offloaded XDP drivers as of kernel 4.17.

**Drivers supporting native XDP**

* **Broadcom**

  * bnxt

..

* **Cavium**

  * thunderx

..

* **Intel**

  * ixgbe
  * ixgbevf
  * i40e

..

* **Mellanox**

  * mlx4
  * mlx5

..

* **Netronome**

  * nfp

..

* **Others**

  * tun
  * virtio_net

..

* **Qlogic**

  * qede

..

* **Solarflare**

  * sfc [1]_

**Drivers supporting offloaded XDP**

* **Netronome**

  * nfp [2]_

Note that examples for writing and loading XDP programs are included in
the toolchain section under the respective tools.

.. [1] XDP for sfc available via out of tree driver as of kernel 4.17, but
   will be upstreamed soon.
.. [2] Some BPF helper functions such as retrieving the current CPU number
   will not be available in an offloaded setting.

tc (traffic control)
--------------------

Aside from other program types such as XDP, BPF can also be used out of the
kernel's tc (traffic control) layer in the networking data path. On a high-level
there are three major differences when comparing XDP BPF programs to tc BPF
ones:

* The BPF input context is a ``sk_buff`` not a ``xdp_buff``. When the kernel's
  networking stack receives a packet, after the XDP layer, it allocates a buffer
  and parses the packet to store metadata about the packet. This representation
  is known as the ``sk_buff``. This structure is then exposed in the BPF input
  context so that BPF programs from the tc ingress layer can use the metadata that
  the stack extracts from the packet. This can be useful, but comes with an
  associated cost of the stack performing this allocation and metadata extraction,
  and handling the packet until it hits the tc hook. By definition, the ``xdp_buff``
  doesn't have access to this metadata because the XDP hook is called before
  this work is done. This is a significant contributor to the performance
  difference between the XDP and tc hooks.

  Therefore, BPF programs attached to the tc BPF hook can, for instance, read or
  write the skb's ``mark``, ``pkt_type``, ``protocol``, ``priority``,
  ``queue_mapping``, ``napi_id``, ``cb[]`` array, ``hash``, ``tc_classid`` or
  ``tc_index``, vlan metadata, the XDP transferred custom metadata and various
  other information. All members of the ``struct __sk_buff`` BPF context used
  in tc BPF are defined in the ``linux/bpf.h`` system header.

  Generally, the ``sk_buff`` is of a completely different nature than
  ``xdp_buff`` where both come with advantages and disadvantages. For example,
  the ``sk_buff`` case has the advantage that it is rather straight forward to
  mangle its associated metadata, however, it also contains a lot of protocol
  specific information (e.g. GSO related state) which makes it difficult to
  simply switch protocols by solely rewriting the packet data. This is due to
  the stack processing the packet based on the metadata rather than having the
  cost of accessing the packet contents each time. Thus, additional conversion
  is required from BPF helper functions taking care that ``sk_buff`` internals
  are properly converted as well. The ``xdp_buff`` case however does not
  face such issues since it comes at such an early stage where the kernel
  has not even allocated an ``sk_buff`` yet, thus packet rewrites of any
  kind can be realized trivially. However, the ``xdp_buff`` case has the
  disadvantage that ``sk_buff`` metadata is not available for mangling
  at this stage. The latter is overcome by passing custom metadata from
  XDP BPF to tc BPF, though. In this way, the limitations of each program
  type can be overcome by operating complementary programs of both types
  as the use case requires.

..

* Compared to XDP, tc BPF programs can be triggered out of ingress and also
  egress points in the networking data path as opposed to ingress only in
  the case of XDP.

  The two hook points ``sch_handle_ingress()`` and ``sch_handle_egress()`` in
  the kernel are triggered out of ``__netif_receive_skb_core()`` and
  ``__dev_queue_xmit()``, respectively. The latter two are the main receive
  and transmit functions in the data path that, setting XDP aside, are triggered
  for every network packet going in or coming out of the node allowing for
  full visibility for tc BPF programs at these hook points.

..

* The tc BPF programs do not require any driver changes since they are run
  at hook points in generic layers in the networking stack. Therefore, they
  can be attached to any type of networking device.

  While this provides flexibility, it also trades off performance compared
  to running at the native XDP layer. However, tc BPF programs still come
  at the earliest point in the generic kernel's networking data path after
  GRO has been run but **before** any protocol processing, traditional iptables
  firewalling such as iptables PREROUTING or nftables ingress hooks or other
  packet processing takes place. Likewise on egress, tc BPF programs execute
  at the latest point before handing the packet to the driver itself for
  transmission, meaning **after** traditional iptables firewalling hooks like
  iptables POSTROUTING, but still before handing the packet to the kernel's
  GSO engine.

  One exception which does require driver changes however are offloaded tc
  BPF programs, typically provided by SmartNICs in a similar way as offloaded
  XDP just with differing set of features due to the differences in the BPF
  input context, helper functions and verdict codes.

..

BPF programs run in the tc layer are run from the ``cls_bpf`` classifier.
While the tc terminology describes the BPF attachment point as a "classifier",
this is a bit misleading since it under-represents what ``cls_bpf`` is
capable of. That is to say, a fully programmable packet processor being able
not only to read the ``skb`` metadata and packet data, but to also arbitrarily
mangle both, and terminate the tc processing with an action verdict. ``cls_bpf``
can thus be regarded as a self-contained entity that manages and executes tc
BPF programs.

``cls_bpf`` can hold one or more tc BPF programs. In the case where Cilium
deploys ``cls_bpf`` programs, it attaches only a single program for a given hook
in ``direct-action`` mode. Typically, in the traditional tc scheme, there is a
split between classifier and action modules, where the classifier has one
or more actions attached to it that are triggered once the classifier has a
match. In the modern world for using tc in the software data path this model
does not scale well for complex packet processing. Given tc BPF programs
attached to ``cls_bpf`` are fully self-contained, they effectively fuse the
parsing and action process together into a single unit. Thanks to ``cls_bpf``'s
``direct-action`` mode, it will just return the tc action verdict and
terminate the processing pipeline immediately. This allows for implementing
scalable programmable packet processing in the networking data path by avoiding
linear iteration of actions. ``cls_bpf`` is the only such "classifier" module
in the tc layer capable of such a fast-path.

Like XDP BPF programs, tc BPF programs can be atomically updated at runtime
via ``cls_bpf`` without interrupting any network traffic or having to restart
services.

Both the tc ingress and the egress hook where ``cls_bpf`` itself can be
attached to is managed by a pseudo qdisc called ``sch_clsact``. This is a
drop-in replacement and proper superset of the ingress qdisc since it
is able to manage both, ingress and egress tc hooks. For tc's egress hook
in ``__dev_queue_xmit()`` it is important to stress that it is not executed
under the kernel's qdisc root lock. Thus, both tc ingress and egress hooks
are executed in a lockless manner in the fast-path. In either case, preemption
is disabled and execution happens under RCU read side.

Typically on egress there are qdiscs attached to netdevices such as ``sch_mq``,
``sch_fq``, ``sch_fq_codel`` or ``sch_htb`` where some of them are classful
qdiscs that contain subclasses and thus require a packet classification
mechanism to determine a verdict where to demux the packet. This is handled
by a call to ``tcf_classify()`` which calls into tc classifiers if present.
``cls_bpf`` can also be attached and used in such cases. Such operation usually
happens under the qdisc root lock and can be subject to lock contention. The
``sch_clsact`` qdisc's egress hook comes at a much earlier point however which
does not fall under that and operates completely independent from conventional
egress qdiscs. Thus for cases like ``sch_htb`` the ``sch_clsact`` qdisc could
perform the heavy lifting packet classification through tc BPF outside of the
qdisc root lock, setting the ``skb->mark`` or ``skb->priority`` from there such
that ``sch_htb`` only requires a flat mapping without expensive packet
classification under the root lock thus reducing contention.

Offloaded tc BPF programs are supported for the case of ``sch_clsact`` in
combination with ``cls_bpf`` where the prior loaded BPF program was JITed
from a SmartNIC driver to be run natively on the NIC. Only ``cls_bpf``
programs operating in ``direct-action`` mode are supported to be offloaded.
``cls_bpf`` only supports offloading a single program and cannot offload
multiple programs. Furthermore only the ingress hook supports offloading
BPF programs.

One ``cls_bpf`` instance is able to hold multiple tc BPF programs internally.
If this is the case, then the ``TC_ACT_UNSPEC`` program return code will
continue execution with the next tc BPF program in that list. However, this
has the drawback that several programs would need to parse the packet over
and over again resulting in degraded performance.

**BPF program return codes**

Both the tc ingress and egress hook share the same action return verdicts
that tc BPF programs can use. They are defined in the ``linux/pkt_cls.h``
system header:

::

    #define TC_ACT_UNSPEC         (-1)
    #define TC_ACT_OK               0
    #define TC_ACT_SHOT             2
    #define TC_ACT_STOLEN           4
    #define TC_ACT_REDIRECT         7

There are a few more action ``TC_ACT_*`` verdicts available in the system
header file which are also used in the two hooks. However, they share the
same semantics with the ones above. Meaning, from a tc BPF perspective,
``TC_ACT_OK`` and ``TC_ACT_RECLASSIFY`` have the same semantics, as well as
the three ``TC_ACT_STOLEN``, ``TC_ACT_QUEUED`` and ``TC_ACT_TRAP`` opcodes.
Therefore, for these cases we only describe ``TC_ACT_OK`` and the ``TC_ACT_STOLEN``
opcode for the two groups.

Starting out with ``TC_ACT_UNSPEC``. It has the meaning of "unspecified action"
and is used in three cases, i) when an offloaded tc BPF program is attached
and the tc ingress hook is run where the ``cls_bpf`` representation for the
offloaded program will return ``TC_ACT_UNSPEC``, ii) in order to continue
with the next tc BPF program in ``cls_bpf`` for the multi-program case. The
latter also works in combination with offloaded tc BPF programs from point i)
where the ``TC_ACT_UNSPEC`` from there continues with a next tc BPF program
solely running in non-offloaded case. Last but not least, iii) ``TC_ACT_UNSPEC``
is also used for the single program case to simply tell the kernel to continue
with the ``skb`` without additional side-effects. ``TC_ACT_UNSPEC`` is very
similar to the ``TC_ACT_OK`` action code in the sense that both pass the
``skb`` onwards either to upper layers of the stack on ingress or down to
the networking device driver for transmission on egress, respectively. The
only difference to ``TC_ACT_OK`` is that ``TC_ACT_OK`` sets ``skb->tc_index``
based on the classid the tc BPF program set. The latter is set out of the
tc BPF program itself through ``skb->tc_classid`` from the BPF context.

``TC_ACT_SHOT`` instructs the kernel to drop the packet, meaning, upper
layers of the networking stack will never see the ``skb`` on ingress and
similarly the packet will never be submitted for transmission on egress.
``TC_ACT_SHOT`` and ``TC_ACT_STOLEN`` are both similar in nature with few
differences: ``TC_ACT_SHOT`` will indicate to the kernel that the ``skb``
was released through ``kfree_skb()`` and return ``NET_XMIT_DROP`` to the
callers for immediate feedback, whereas ``TC_ACT_STOLEN`` will release
the ``skb`` through ``consume_skb()`` and pretend to upper layers that
the transmission was successful through ``NET_XMIT_SUCCESS``. The perf's
drop monitor which records traces of ``kfree_skb()`` will therefore
also not see any drop indications from ``TC_ACT_STOLEN`` since its
semantics are such that the ``skb`` has been "consumed" or queued but
certainly not "dropped".

Last but not least the ``TC_ACT_REDIRECT`` action which is available for
tc BPF programs as well. This allows to redirect the ``skb`` to the same
or another's device ingress or egress path together with the ``bpf_redirect()``
helper. Being able to inject the packet into another device's ingress or
egress direction allows for full flexibility in packet forwarding with
BPF. There are no requirements on the target networking device other than
being a networking device itself, there is no need to run another instance
of ``cls_bpf`` on the target device or other such restrictions.

**tc BPF FAQ**

This section contains a few miscellaneous question and answer pairs related to
tc BPF programs that are asked from time to time.

* **Question:** What about ``act_bpf`` as a tc action module, is it still relevant?
* **Answer:** Not really. Although ``cls_bpf`` and ``act_bpf`` share the same
  functionality for tc BPF programs, ``cls_bpf`` is more flexible since it is a
  proper superset of ``act_bpf``. The way tc works is that tc actions need to be
  attached to tc classifiers. In order to achieve the same flexibility as ``cls_bpf``,
  ``act_bpf`` would need to be attached to the ``cls_matchall`` classifier. As the
  name says, this will match on every packet in order to pass them through for attached
  tc action processing. For ``act_bpf``, this is will result in less efficient packet
  processing than using ``cls_bpf`` in ``direct-action`` mode directly. If ``act_bpf``
  is used in a setting with other classifiers than ``cls_bpf`` or ``cls_matchall``
  then this will perform even worse due to the nature of operation of tc classifiers.
  Meaning, if classifier A has a mismatch, then the packet is passed to classifier
  B, reparsing the packet, etc, thus in the typical case there will be linear
  processing where the packet would need to traverse N classifiers in the worst
  case to find a match and execute ``act_bpf`` on that. Therefore, ``act_bpf`` has
  never been largely relevant. Additionally, ``act_bpf`` does not provide a tc
  offloading interface either compared to ``cls_bpf``.

..

* **Question:** Is it recommended to use ``cls_bpf`` not in ``direct-action`` mode?
* **Answer:** No. The answer is similar to the one above in that this is otherwise
  unable to scale for more complex processing. tc BPF can already do everything needed
  by itself in an efficient manner and thus there is no need for anything other than
  ``direct-action`` mode.

..

* **Question:** Is there any performance difference in offloaded ``cls_bpf`` and
  offloaded XDP?
* **Answer:** No. Both are JITed through the same compiler in the kernel which
  handles the offloading to the SmartNIC and the loading mechanism for both is
  very similar as well. Thus, the BPF program gets translated into the same target
  instruction set in order to be able to run on the NIC natively. The two tc BPF
  and XDP BPF program types have a differing set of features, so depending on the
  use case one might be picked over the other due to availability of certain helper
  functions in the offload case, for example.

**Use cases for tc BPF**

Some of the main use cases for tc BPF programs are presented in this subsection.
Also here, the list is non-exhaustive and given the programmability and efficiency
of tc BPF, it can easily be tailored and integrated into orchestration systems
in order to solve very specific use cases. While some use cases with XDP may overlap,
tc BPF and XDP BPF are mostly complementary to each other and both can also be
used at the same time or one over the other depending which is most suitable for a
given problem to solve.

* **Policy enforcement for containers**

  One application which tc BPF programs are suitable for is to implement policy
  enforcement, custom firewalling or similar security measures for containers or
  pods, respectively. In the conventional case, container isolation is implemented
  through network namespaces with veth networking devices connecting the host's
  initial namespace with the dedicated container's namespace. Since one end of
  the veth pair has been moved into the container's namespace whereas the other
  end remains in the initial namespace of the host, all network traffic from the
  container has to pass through the host-facing veth device allowing for attaching
  tc BPF programs on the tc ingress and egress hook of the veth. Network traffic
  going into the container will pass through the host-facing veth's tc egress
  hook whereas network traffic coming from the container will pass through the
  host-facing veth's tc ingress hook.

  For virtual devices like veth devices XDP is unsuitable in this case since the
  kernel operates solely on a ``skb`` here and generic XDP has a few limitations
  where it does not operate with cloned ``skb``'s. The latter is heavily used
  from the TCP/IP stack in order to hold data segments for retransmission where
  the generic XDP hook would simply get bypassed instead. Moreover, generic XDP
  needs to linearize the entire ``skb`` resulting in heavily degraded performance.
  tc BPF on the other hand is more flexible as it specializes on the ``skb``
  input context case and thus does not need to cope with the limitations from
  generic XDP.

..

* **Forwarding and load-balancing**

  The forwarding and load-balancing use case is quite similar to XDP, although
  slightly more targeted towards east-west container workloads rather than
  north-south traffic (though both technologies can be used in either case).
  Since XDP is only available on ingress side, tc BPF programs allow for
  further use cases that apply in particular on egress, for example, container
  based traffic can already be NATed and load-balanced on the egress side
  through BPF out of the initial namespace such that this is done transparent
  to the container itself. Egress traffic is already based on the ``sk_buff``
  structure due to the nature of the kernel's networking stack, so packet
  rewrites and redirects are suitable out of tc BPF. By utilizing the
  ``bpf_redirect()`` helper function, BPF can take over the forwarding logic
  to push the packet either into the ingress or egress path of another networking
  device. Thus, any bridge-like devices become unnecessary to use as well by
  utilizing tc BPF as forwarding fabric.

..

* **Flow sampling, monitoring**

  Like in XDP case, flow sampling and monitoring can be realized through a
  high-performance lockless per-CPU memory mapped perf ring buffer where the
  BPF program is able to push custom data, the full or truncated packet
  contents, or both up to a user space application. From the tc BPF program
  this is realized through the ``bpf_skb_event_output()`` BPF helper function
  which has the same function signature and semantics as ``bpf_xdp_event_output()``.
  Given tc BPF programs can be attached to ingress and egress as opposed to
  only ingress in XDP BPF case plus the two tc hooks are at the lowest layer
  in the (generic) networking stack, this allows for bidirectional monitoring
  of all network traffic from a particular node. This might be somewhat related
  to the cBPF case which tcpdump and Wireshark makes use of, though, without
  having to clone the ``skb`` and with being a lot more flexible in terms of
  programmability where, for example, BPF can already perform in-kernel
  aggregation rather than pushing everything up to user space as well as
  custom annotations for packets pushed into the ring buffer. The latter is
  also heavily used in Cilium where packet drops can be further annotated
  to correlate container labels and reasons for why a given packet had to
  be dropped (such as due to policy violation) in order to provide a richer
  context.

..

* **Packet scheduler pre-processing**

  The ``sch_clsact``'s egress hook which is called ``sch_handle_egress()``
  runs right before taking the kernel's qdisc root lock, thus tc BPF programs
  can be utilized to perform all the heavy lifting packet classification
  and mangling before the packet is transmitted into a real full blown
  qdisc such as ``sch_htb``. This type of interaction of ``sch_clsact``
  with a real qdisc like ``sch_htb`` coming later in the transmission phase
  allows to reduce the lock contention on transmission since ``sch_clsact``'s
  egress hook is executed without taking locks.

..

One concrete example user of tc BPF but also XDP BPF programs is Cilium.
Cilium is open source software for transparently securing the network
connectivity between application services deployed using Linux container
management platforms like Docker and Kubernetes and operates at Layer 3/4
as well as Layer 7. At the heart of Cilium operates BPF in order to
implement the policy enforcement as well as load balancing and monitoring.

* Slides: https://www.slideshare.net/ThomasGraf5/dockercon-2017-cilium-network-and-application-security-with-bpf-and-xdp
* Video: https://youtu.be/ilKlmTDdFgk
* Github: https://github.com/cilium/cilium

**Driver support**

Since tc BPF programs are triggered from the kernel's networking stack
and not directly out of the driver, they do not require any extra driver
modification and therefore can run on any networking device. The only
exception listed below is for offloading tc BPF programs to the NIC.

**Drivers supporting offloaded tc BPF**

* **Netronome**

  * nfp [2]_

Note that also here examples for writing and loading tc BPF programs are
included in the toolchain section under the respective tools.

.. _bpf_users:

Further Reading
===============

Mentioned lists of docs, projects, talks, papers, and further reading
material are likely not complete. Thus, feel free to open pull requests
to complete the list.

Kernel Developer FAQ
--------------------

Under ``Documentation/bpf/``, the Linux kernel provides two FAQ files that
are mainly targeted for kernel developers involved in the BPF subsystem.

* **BPF Devel FAQ:** this document provides mostly information around patch
  submission process as well as BPF kernel tree, stable tree and bug
  reporting workflows, questions around BPF's extensibility and interaction
  with LLVM and more.

  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/bpf/bpf_devel_QA.rst

..

* **BPF Design FAQ:** this document tries to answer frequently asked questions
  around BPF design decisions related to the instruction set, verifier,
  calling convention, JITs, etc.

  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/bpf/bpf_design_QA.rst

Projects using BPF
------------------

The following list includes a selection of open source projects making
use of BPF respectively provide tooling for BPF. In this context the eBPF
instruction set is specifically meant instead of projects utilizing the
legacy cBPF:

**Tracing**

* **BCC**

  BCC stands for BPF Compiler Collection and its key feature is to provide
  a set of easy to use and efficient kernel tracing utilities all based
  upon BPF programs hooking into kernel infrastructure based upon kprobes,
  kretprobes, tracepoints, uprobes, uretprobes as well as USDT probes. The
  collection provides close to hundred tools targeting different layers
  across the stack from applications, system libraries, to the various
  different kernel subsystems in order to analyze a system's performance
  characteristics or problems. Additionally, BCC provides an API in order
  to be used as a library for other projects.

  https://github.com/iovisor/bcc

..

* **bpftrace**

  bpftrace is a DTrace-style dynamic tracing tool for Linux and uses LLVM
  as a back end to compile scripts to BPF-bytecode and makes use of BCC
  for interacting with the kernel's BPF tracing infrastructure. It provides
  a higher-level language for implementing tracing scripts compared to
  native BCC.

  https://github.com/ajor/bpftrace

..

* **perf**

  The perf tool which is developed by the Linux kernel community as
  part of the kernel source tree provides a way to load tracing BPF
  programs through the conventional perf record subcommand where the
  aggregated data from BPF can be retrieved and post processed in
  perf.data for example through perf script and other means.

  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/perf

..

* **ply**

  ply is a tracing tool that follows the 'Little Language' approach of
  yore, and compiles ply scripts into Linux BPF programs that are attached
  to kprobes and tracepoints in the kernel. The scripts have a C-like syntax,
  heavily inspired by DTrace and by extension awk. ply keeps dependencies
  to very minimum and only requires flex and bison at build time, only libc
  at runtime.

  https://github.com/wkz/ply

..

* **systemtap**

  systemtap is a scripting language and tool for extracting, filtering and
  summarizing data in order to diagnose and analyze performance or functional
  problems. It comes with a BPF back end called stapbpf which translates
  the script directly into BPF without the need of an additional compiler
  and injects the probe into the kernel. Thus, unlike stap's kernel modules
  this does neither have external dependencies nor requires to load kernel
  modules.

  https://sourceware.org/git/gitweb.cgi?p=systemtap.git;a=summary

..

* **PCP**

  Performance Co-Pilot (PCP) is a system performance and analysis framework
  which is able to collect metrics through a variety of agents as well as
  analyze collected systems' performance metrics in real-time or by using
  historical data. With pmdabcc, PCP has a BCC based performance metrics
  domain agent which extracts data from the kernel via BPF and BCC.

  https://github.com/performancecopilot/pcp

..

* **Weave Scope**

  Weave Scope is a cloud monitoring tool collecting data about processes,
  networking connections or other system data by making use of BPF in combination
  with kprobes. Weave Scope works on top of the gobpf library in order to load
  BPF ELF files into the kernel, and comes with a tcptracer-bpf tool which
  monitors connect, accept and close calls in order to trace TCP events.

  https://github.com/weaveworks/scope

..

**Networking**

* **Cilium**

  Cilium provides and transparently secures network connectivity and load-balancing
  between application workloads such as application containers or processes. Cilium
  operates at Layer 3/4 to provide traditional networking and security services
  as well as Layer 7 to protect and secure use of modern application protocols
  such as HTTP, gRPC and Kafka. It is integrated into orchestration frameworks
  such as Kubernetes and Mesos, and BPF is the foundational part of Cilium that
  operates in the kernel's networking data path.

  https://github.com/cilium/cilium

..

* **Suricata**

  Suricata is a network IDS, IPS and NSM engine, and utilizes BPF as well as XDP
  in three different areas, that is, as BPF filter in order to process or bypass
  certain packets, as a BPF based load balancer in order to allow for programmable
  load balancing and for XDP to implement a bypass or dropping mechanism at high
  packet rates.

  http://suricata.readthedocs.io/en/latest/capture-hardware/ebpf-xdp.html

  https://github.com/OISF/suricata

..

* **systemd**

  systemd allows for IPv4/v6 accounting as well as implementing network access
  control for its systemd units based on BPF's cgroup ingress and egress hooks.
  Accounting is based on packets / bytes, and ACLs can be specified as address
  prefixes for allow / deny rules. More information can be found at:

  http://0pointer.net/blog/ip-accounting-and-access-lists-with-systemd.html

  https://github.com/systemd/systemd

..

* **iproute2**

  iproute2 offers the ability to load BPF programs as LLVM generated ELF files
  into the kernel. iproute2 supports both, XDP BPF programs as well as tc BPF
  programs through a common BPF loader backend. The tc and ip command line
  utilities enable loader and introspection functionality for the user.

  https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/

..

* **p4c-xdp**

  p4c-xdp presents a P4 compiler backend targeting BPF and XDP. P4 is a domain
  specific language describing how packets are processed by the data plane of
  a programmable network element such as NICs, appliances or switches, and with
  the help of p4c-xdp P4 programs can be translated into BPF C programs which
  can be compiled by clang / LLVM and loaded as BPF programs into the kernel
  at XDP layer for high performance packet processing.

  https://github.com/vmware/p4c-xdp

..

**Others**

* **LLVM**

  clang / LLVM provides the BPF back end in order to compile C BPF programs
  into BPF instructions contained in ELF files. The LLVM BPF back end is
  developed alongside with the BPF core infrastructure in the Linux kernel
  and maintained by the same community. clang / LLVM is a key part in the
  toolchain for developing BPF programs.

  https://llvm.org/

..

* **libbpf**

  libbpf is a generic BPF library which is developed by the Linux kernel
  community as part of the kernel source tree and allows for loading and
  attaching BPF programs from LLVM generated ELF files into the kernel.
  The library is used by other kernel projects such as perf and bpftool.

  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/lib/bpf

..

* **bpftool**

  bpftool is the main tool for introspecting and debugging BPF programs
  and BPF maps, and like libbpf is developed by the Linux kernel community.
  It allows for dumping all active BPF programs and maps in the system,
  dumping and disassembling BPF or JITed BPF instructions from a program
  as well as dumping and manipulating BPF maps in the system. bpftool
  supports interaction with the BPF filesystem, loading various program
  types from an object file into the kernel and much more.

  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/bpf/bpftool

..

* **gobpf**

  gobpf provides go bindings for the bcc framework as well as low-level routines in
  order to load and use BPF programs from ELF files.

  https://github.com/iovisor/gobpf

..

* **ebpf_asm**

  ebpf_asm provides an assembler for BPF programs written in an Intel-like assembly
  syntax, and therefore offers an alternative for writing BPF programs directly in
  assembly for cases where programs are rather small and simple without needing the
  clang / LLVM toolchain.

  https://github.com/solarflarecom/ebpf_asm

..

XDP Newbies
-----------

There are a couple of walk-through posts by David S. Miller to the xdp-newbies
mailing list (http://vger.kernel.org/vger-lists.html#xdp-newbies), which explain
various parts of XDP and BPF:

4. May 2017,
     BPF Verifier Overview,
     David S. Miller,
     https://www.spinics.net/lists/xdp-newbies/msg00185.html

3. May 2017,
     Contextually speaking...,
     David S. Miller,
     https://www.spinics.net/lists/xdp-newbies/msg00181.html

2. May 2017,
     bpf.h and you...,
     David S. Miller,
     https://www.spinics.net/lists/xdp-newbies/msg00179.html

1. Apr 2017,
     XDP example of the day,
     David S. Miller,
     https://www.spinics.net/lists/xdp-newbies/msg00009.html

BPF Newsletter
--------------

Alexander Alemayhu initiated a newsletter around BPF roughly once per week
covering latest developments around BPF in Linux kernel land and its
surrounding ecosystem in user space.

All BPF update newsletters (01 - 12) can be found here:

     https://cilium.io/blog/categories/BPF%20Newsletter

Podcasts
--------

There have been a number of technical podcasts partially covering BPF.
Incomplete list:

5. Feb 2017,
     Linux Networking Update from Netdev Conference,
     Thomas Graf,
     Software Gone Wild, Show 71,
     http://blog.ipspace.net/2017/02/linux-networking-update-from-netdev.html
     http://media.blubrry.com/ipspace/stream.ipspace.net/nuggets/podcast/Show_71-NetDev_Update.mp3

4. Jan 2017,
     The IO Visor Project,
     Brenden Blanco,
     OVS Orbit, Episode 23,
     https://ovsorbit.org/#e23
     https://ovsorbit.org/episode-23.mp3

3. Oct 2016,
     Fast Linux Packet Forwarding,
     Thomas Graf,
     Software Gone Wild, Show 64,
     http://blog.ipspace.net/2016/10/fast-linux-packet-forwarding-with.html
     http://media.blubrry.com/ipspace/stream.ipspace.net/nuggets/podcast/Show_64-Cilium_with_Thomas_Graf.mp3

2. Aug 2016,
     P4 on the Edge,
     John Fastabend,
     OVS Orbit, Episode 11,
     https://ovsorbit.org/#e11
     https://ovsorbit.org/episode-11.mp3

1. May 2016,
     Cilium,
     Thomas Graf,
     OVS Orbit, Episode 4,
     https://ovsorbit.org/#e4
     https://ovsorbit.benpfaff.org/episode-4.mp3

Blog posts
----------

The following (incomplete) list includes blog posts around BPF, XDP and related projects:

34. May 2017,
     An entertaining eBPF XDP adventure,
     Suchakra Sharma,
     https://suchakra.wordpress.com/2017/05/23/an-entertaining-ebpf-xdp-adventure/

33. May 2017,
     eBPF, part 2: Syscall and Map Types,
     Ferris Ellis,
     https://ferrisellis.com/posts/ebpf_syscall_and_maps/

32. May 2017,
     Monitoring the Control Plane,
     Gary Berger,
     http://firstclassfunc.com/2017/05/monitoring-the-control-plane/

31. Apr 2017,
     USENIX/LISA 2016 Linux bcc/BPF Tools,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2017-04-29/usenix-lisa-2016-bcc-bpf-tools.html

30. Apr 2017,
     Liveblog: Cilium for Network and Application Security with BPF and XDP,
     Scott Lowe,
     http://blog.scottlowe.org//2017/04/18/black-belt-cilium/

29. Apr 2017,
     eBPF, part 1: Past, Present, and Future,
     Ferris Ellis,
     https://ferrisellis.com/posts/ebpf_past_present_future/

28. Mar 2017,
     Analyzing KVM Hypercalls with eBPF Tracing,
     Suchakra Sharma,
     https://suchakra.wordpress.com/2017/03/31/analyzing-kvm-hypercalls-with-ebpf-tracing/

27. Jan 2017,
     Golang bcc/BPF Function Tracing,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2017-01-31/golang-bcc-bpf-function-tracing.html

26. Dec 2016,
     Give me 15 minutes and I'll change your view of Linux tracing,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-12-27/linux-tracing-in-15-minutes.html

25. Nov 2016,
     Cilium: Networking and security for containers with BPF and XDP,
     Daniel Borkmann,
     https://opensource.googleblog.com/2016/11/cilium-networking-and-security.html

24. Nov 2016,
     Linux bcc/BPF tcplife: TCP Lifespans,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-11-30/linux-bcc-tcplife.html

23. Oct 2016,
     DTrace for Linux 2016,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-10-27/dtrace-for-linux-2016.html

22. Oct 2016,
     Linux 4.9's Efficient BPF-based Profiler,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-10-21/linux-efficient-profiler.html

21. Oct 2016,
     Linux bcc tcptop,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-10-15/linux-bcc-tcptop.html

20. Oct 2016,
     Linux bcc/BPF Node.js USDT Tracing,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-10-12/linux-bcc-nodejs-usdt.html

19. Oct 2016,
     Linux bcc/BPF Run Queue (Scheduler) Latency,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-10-08/linux-bcc-runqlat.html

18. Oct 2016,
     Linux bcc ext4 Latency Tracing,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-10-06/linux-bcc-ext4dist-ext4slower.html

17. Oct 2016,
     Linux MySQL Slow Query Tracing with bcc/BPF,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-10-04/linux-bcc-mysqld-qslower.html

16. Oct 2016,
     Linux bcc Tracing Security Capabilities,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-10-01/linux-bcc-security-capabilities.html

15. Sep 2016,
     Suricata bypass feature,
     Eric Leblond,
     https://www.stamus-networks.com/2016/09/28/suricata-bypass-feature/

14. Aug 2016,
     Introducing the p0f BPF compiler,
     Gilberto Bertin,
     https://blog.cloudflare.com/introducing-the-p0f-bpf-compiler/

13. Jun 2016,
     Ubuntu Xenial bcc/BPF,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-06-14/ubuntu-xenial-bcc-bpf.html

12. Mar 2016,
     Linux BPF/bcc Road Ahead, March 2016,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-03-28/linux-bpf-bcc-road-ahead-2016.html

11. Mar 2016,
     Linux BPF Superpowers,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-03-05/linux-bpf-superpowers.html

10. Feb 2016,
     Linux eBPF/bcc uprobes,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-02-08/linux-ebpf-bcc-uprobes.html

9. Feb 2016,
     Who is waking the waker? (Linux chain graph prototype),
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-02-05/ebpf-chaingraph-prototype.html

8. Feb 2016,
     Linux Wakeup and Off-Wake Profiling,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-02-01/linux-wakeup-offwake-profiling.html

7. Jan 2016,
     Linux eBPF Off-CPU Flame Graph,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-01-20/ebpf-offcpu-flame-graph.html

6. Jan 2016,
     Linux eBPF Stack Trace Hack,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2016-01-18/ebpf-stack-trace-hack.html

1. Sep 2015,
     Linux Networking, Tracing and IO Visor, a New Systems Performance Tool for a Distributed World,
     Suchakra Sharma,
     https://thenewstack.io/comparing-dtrace-iovisor-new-systems-performance-platform-advance-linux-networking-virtualization/

5. Aug 2015,
     BPF Internals - II,
     Suchakra Sharma,
     https://suchakra.wordpress.com/2015/08/12/bpf-internals-ii/

4. May 2015,
     eBPF: One Small Step,
     Brendan Gregg,
     http://www.brendangregg.com/blog/2015-05-15/ebpf-one-small-step.html

3. May 2015,
     BPF Internals - I,
     Suchakra Sharma,
     https://suchakra.wordpress.com/2015/05/18/bpf-internals-i/

2. Jul 2014,
     Introducing the BPF Tools,
     Marek Majkowski,
     https://blog.cloudflare.com/introducing-the-bpf-tools/

1. May 2014,
     BPF - the forgotten bytecode,
     Marek Majkowski,
     https://blog.cloudflare.com/bpf-the-forgotten-bytecode/

Talks
-----

The following (incomplete) list includes talks and conference papers
related to BPF and XDP:

44. May 2017,
     PyCon 2017, Portland,
     Executing python functions in the linux kernel by transpiling to bpf,
     Alex Gartrell,
     https://www.youtube.com/watch?v=CpqMroMBGP4

43. May 2017,
     gluecon 2017, Denver,
     Cilium + BPF: Least Privilege Security on API Call Level for Microservices,
     Dan Wendlandt,
     http://gluecon.com/#agenda

42. May 2017,
     Lund Linux Con, Lund,
     XDP - eXpress Data Path,
     Jesper Dangaard Brouer,
     http://people.netfilter.org/hawk/presentations/LLC2017/XDP_DDoS_protecting_LLC2017.pdf

41. May 2017,
     Polytechnique Montreal,
     Trace Aggregation and Collection with eBPF,
     Suchakra Sharma,
     http://step.polymtl.ca/~suchakra/eBPF-5May2017.pdf

40. Apr 2017,
     DockerCon, Austin,
     Cilium - Network and Application Security with BPF and XDP,
     Thomas Graf,
     https://www.slideshare.net/ThomasGraf5/dockercon-2017-cilium-network-and-application-security-with-bpf-and-xdp

39. Apr 2017,
     NetDev 2.1, Montreal,
     XDP Mythbusters,
     David S. Miller,
     https://www.netdevconf.org/2.1/slides/apr7/miller-XDP-MythBusters.pdf

38. Apr 2017,
     NetDev 2.1, Montreal,
     Droplet: DDoS countermeasures powered by BPF + XDP,
     Huapeng Zhou, Doug Porter, Ryan Tierney, Nikita Shirokov,
     https://www.netdevconf.org/2.1/slides/apr6/zhou-netdev-xdp-2017.pdf

37. Apr 2017,
     NetDev 2.1, Montreal,
     XDP in practice: integrating XDP in our DDoS mitigation pipeline,
     Gilberto Bertin,
     https://www.netdevconf.org/2.1/slides/apr6/bertin_Netdev-XDP.pdf

36. Apr 2017,
     NetDev 2.1, Montreal,
     XDP for the Rest of Us,
     Andy Gospodarek, Jesper Dangaard Brouer,
     https://www.netdevconf.org/2.1/slides/apr7/gospodarek-Netdev2.1-XDP-for-the-Rest-of-Us_Final.pdf

35. Mar 2017,
     SCALE15x, Pasadena,
     Linux 4.x Tracing: Performance Analysis with bcc/BPF,
     Brendan Gregg,
     https://www.slideshare.net/brendangregg/linux-4x-tracing-performance-analysis-with-bccbpf

34. Mar 2017,
     XDP Inside and Out,
     David S. Miller,
     https://github.com/iovisor/bpf-docs/raw/master/XDP_Inside_and_Out.pdf

33. Mar 2017,
     OpenSourceDays, Copenhagen,
     XDP - eXpress Data Path, Used for DDoS protection,
     Jesper Dangaard Brouer,
     https://github.com/iovisor/bpf-docs/raw/master/XDP_Inside_and_Out.pdf

32. Mar 2017,
     source{d}, Infrastructure 2017, Madrid,
     High-performance Linux monitoring with eBPF,
     Alfonso Acosta,
     https://www.youtube.com/watch?v=k4jqTLtdrxQ

31. Feb 2017,
     FOSDEM 2017, Brussels,
     Stateful packet processing with eBPF, an implementation of OpenState interface,
     Quentin Monnet,
     https://fosdem.org/2017/schedule/event/stateful_ebpf/

30. Feb 2017,
     FOSDEM 2017, Brussels,
     eBPF and XDP walkthrough and recent updates,
     Daniel Borkmann,
     http://borkmann.ch/talks/2017_fosdem.pdf

29. Feb 2017,
     FOSDEM 2017, Brussels,
     Cilium - BPF & XDP for containers,
     Thomas Graf,
     https://fosdem.org/2017/schedule/event/cilium/

28. Jan 2017,
     linuxconf.au, Hobart,
     BPF: Tracing and more,
     Brendan Gregg,
     https://www.slideshare.net/brendangregg/bpf-tracing-and-more

27. Dec 2016,
     USENIX LISA 2016, Boston,
     Linux 4.x Tracing Tools: Using BPF Superpowers,
     Brendan Gregg,
     https://www.slideshare.net/brendangregg/linux-4x-tracing-tools-using-bpf-superpowers

26. Nov 2016,
     Linux Plumbers, Santa Fe,
     Cilium: Networking & Security for Containers with BPF & XDP,
     Thomas Graf,
     http://www.slideshare.net/ThomasGraf5/clium-container-networking-with-bpf-xdp

25. Nov 2016,
     OVS Conference, Santa Clara,
     Offloading OVS Flow Processing using eBPF,
     William (Cheng-Chun) Tu,
     http://openvswitch.org/support/ovscon2016/7/1120-tu.pdf

24. Oct 2016,
     One.com, Copenhagen,
     XDP - eXpress Data Path, Intro and future use-cases,
     Jesper Dangaard Brouer,
     http://people.netfilter.org/hawk/presentations/xdp2016/xdp_intro_and_use_cases_sep2016.pdf

23. Oct 2016,
     Docker Distributed Systems Summit, Berlin,
     Cilium: Networking & Security for Containers with BPF & XDP,
     Thomas Graf,
     http://www.slideshare.net/Docker/cilium-bpf-xdp-for-containers-66969823

22. Oct 2016,
     NetDev 1.2, Tokyo,
     Data center networking stack,
     Tom Herbert,
     http://netdevconf.org/1.2/session.html?tom-herbert

21. Oct 2016,
     NetDev 1.2, Tokyo,
     Fast Programmable Networks & Encapsulated Protocols,
     David S. Miller,
     http://netdevconf.org/1.2/session.html?david-miller-keynote

20. Oct 2016,
     NetDev 1.2, Tokyo,
     XDP workshop - Introduction, experience, and future development,
     Tom Herbert,
     http://netdevconf.org/1.2/session.html?herbert-xdp-workshop

19. Oct 2016,
     NetDev1.2, Tokyo,
     The adventures of a Suricate in eBPF land,
     Eric Leblond,
     http://netdevconf.org/1.2/slides/oct6/10_suricata_ebpf.pdf

18. Oct 2016,
     NetDev1.2, Tokyo,
     cls_bpf/eBPF updates since netdev 1.1,
     Daniel Borkmann,
     http://borkmann.ch/talks/2016_tcws.pdf

17. Oct 2016,
     NetDev1.2, Tokyo,
     Advanced programmability and recent updates with tc’s cls_bpf,
     Daniel Borkmann,
     http://borkmann.ch/talks/2016_netdev2.pdf
     http://www.netdevconf.org/1.2/papers/borkmann.pdf

16. Oct 2016,
     NetDev 1.2, Tokyo,
     eBPF/XDP hardware offload to SmartNICs,
     Jakub Kicinski, Nic Viljoen,
     http://netdevconf.org/1.2/papers/eBPF_HW_OFFLOAD.pdf

15. Aug 2016,
     LinuxCon, Toronto,
     What Can BPF Do For You?,
     Brenden Blanco,
     https://events.linuxfoundation.org/sites/events/files/slides/iovisor-lc-bof-2016.pdf

14. Aug 2016,
     LinuxCon, Toronto,
     Cilium - Fast IPv6 Container Networking with BPF and XDP,
     Thomas Graf,
     https://www.slideshare.net/ThomasGraf5/cilium-fast-ipv6-container-networking-with-bpf-and-xdp

13. Aug 2016,
     P4, EBPF and Linux TC Offload,
     Dinan Gunawardena, Jakub Kicinski,
     https://de.slideshare.net/Open-NFP/p4-epbf-and-linux-tc-offload

12. Jul 2016,
     Linux Meetup, Santa Clara,
     eXpress Data Path,
     Brenden Blanco,
     http://www.slideshare.net/IOVisor/express-data-path-linux-meetup-santa-clara-july-2016

11. Jul 2016,
     Linux Meetup, Santa Clara,
     CETH for XDP,
     Yan Chan, Yunsong Lu,
     http://www.slideshare.net/IOVisor/ceth-for-xdp-linux-meetup-santa-clara-july-2016

10. May 2016,
     P4 workshop, Stanford,
     P4 on the Edge,
     John Fastabend,
     https://schd.ws/hosted_files/2016p4workshop/1d/Intel%20Fastabend-P4%20on%20the%20Edge.pdf

9. Mar 2016,
    Performance @Scale 2016, Menlo Park,
    Linux BPF Superpowers,
    Brendan Gregg,
    https://www.slideshare.net/brendangregg/linux-bpf-superpowers

8. Mar 2016,
    eXpress Data Path,
    Tom Herbert, Alexei Starovoitov,
    https://github.com/iovisor/bpf-docs/raw/master/Express_Data_Path.pdf

7. Feb 2016,
    NetDev1.1, Seville,
    On getting tc classifier fully programmable with cls_bpf,
    Daniel Borkmann,
    http://borkmann.ch/talks/2016_netdev.pdf
    http://www.netdevconf.org/1.1/proceedings/papers/On-getting-tc-classifier-fully-programmable-with-cls-bpf.pdf

6. Jan 2016,
    FOSDEM 2016, Brussels,
    Linux tc and eBPF,
    Daniel Borkmann,
    http://borkmann.ch/talks/2016_fosdem.pdf

5. Oct 2015,
    LinuxCon Europe, Dublin,
    eBPF on the Mainframe,
    Michael Holzheu,
    https://events.linuxfoundation.org/sites/events/files/slides/ebpf_on_the_mainframe_lcon_2015.pdf

4. Aug 2015,
    Tracing Summit, Seattle,
    LLTng's Trace Filtering and beyond (with some eBPF goodness, of course!),
    Suchakra Sharma,
    https://github.com/iovisor/bpf-docs/raw/master/ebpf_excerpt_20Aug2015.pdf

3. Jun 2015,
    LinuxCon Japan, Tokyo,
    Exciting Developments in Linux Tracing,
    Elena Zannoni,
    https://events.linuxfoundation.org/sites/events/files/slides/tracing-linux-ezannoni-linuxcon-ja-2015_0.pdf

2. Feb 2015,
    Collaboration Summit, Santa Rosa,
    BPF: In-kernel Virtual Machine,
    Alexei Starovoitov,
    https://events.linuxfoundation.org/sites/events/files/slides/bpf_collabsummit_2015feb20.pdf

1. Feb 2015,
    NetDev 0.1, Ottawa,
    BPF: In-kernel Virtual Machine,
    Alexei Starovoitov,
    http://netdevconf.org/0.1/sessions/15.html

0. Feb 2014,
    DevConf.cz, Brno,
    tc and cls_bpf: lightweight packet classifying with BPF,
    Daniel Borkmann,
    http://borkmann.ch/talks/2014_devconf.pdf

Further Documents
-----------------

- Dive into BPF: a list of reading material,
  Quentin Monnet
  (https://qmonnet.github.io/whirl-offload/2016/09/01/dive-into-bpf/)

- XDP - eXpress Data Path,
  Jesper Dangaard Brouer
  (https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/index.html)
