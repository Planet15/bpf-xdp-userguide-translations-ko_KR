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

3. **허용되는 전역 변수가 없습니다.**

  섹션 1번에서 언급 한 이유로 BPF는 일반적인 C 프로그램에서 자주 사용되는
  전역 변수를 가질 수 없습니다.

  그러나 프로그램에서 단순히 ``BPF_MAP_TYPE_PERCPU_ARRAY`` 타입의 BPF 맵
  을 임의의 값 크기의 단일 슬롯과 함께 사용할 수 있다는 해결 방법이 있습
  니다.이것은 실행 중에 BPF 프로그램이 결코 커널에 의해 선점되지 않도록
  보장되므로 단일 맵 항목을 임시 데이터를 위한 스크래치 버퍼로 사용할
  수 있으며,  예를 들어서, 스택 제한 범위를 넘어서는 확장 입니다. 이것
  은 또한 선점과 관련하여 동일한 보장이 있기 때문에,  tail 호출을 건
  너서 동작 합니다.

  그렇지 않으면 여러 BPF 프로그램 실행에서 상태를 유지하기 위해 정상적인
  BPF 맵을 사용할 수 있습니다.

4. **const 문자열이나 배열이 허용되지 않습니다.**

  BPF C 프로그램에서 ``const`` 문자열 이나 다른 배열을 정의하는 것은
  섹션 1과 3에서 지적한 내용과 같이 즉, 로더쪽으로 ABI의 일부가 아니
  기 때문에 로더가 거부하는 ELF 파일에 재배치 항목이 생성 되는 이유로
  동작 하지 않습니다. (로더는 이미 컴파일 된 BPF 순서를 다시 작성
  해야하므로 이러한 항목을 수정할 수 없습니다).

  앞으로 LLVM은 이러한 발생을 감지하고 사용자에게 오류를 일찍 알수
  있습니다.

  ``trace_printk()`` 과 같은 helper 함수는 다음과 같이 처리 할 수 있습
  니다:

  ::

    static void BPF_FUNC(trace_printk, const char *fmt, int fmt_size, ...);

    #ifndef printk
    # define printk(fmt, ...)                                      \
        ({                                                         \
            char ____fmt[] = fmt;                                  \
            trace_printk(____fmt, sizeof(____fmt), ##__VA_ARGS__); \
        })
    #endif

  그런 다음 프로그램은 ``printk("skb len:%u\n", skb->len);`` 와 같이 자연
  스럽게 매크로를 사용할 수 있습니다. 그러면 출력이 추적 파이프에 기록됩
  니다. ``tc exec bpf dbg`` 를 사용하여 거기에서 메시지를 검색 할 수 있
  습니다.

  ``trace_printk()`` helper 함수의 사용에는 몇 가지 단점이 있으므로 프로덕션
  용도로 권장되지 않습니다. helper 함수가 호출 될 때마다 ``"skb len:%u\n"``
  과 같은 상수 문자열을 BPF 스택에 로드 해야 하지만 BPF helper 함수는 최대
  5 개의 인수로 제한됩니다.이것은 dumping을 위해 전달 될 수 있는 추가
  변수 3 개만 남겨 둡니다.

  따라서 빠른 디버깅에 도움이 되지만 네트워킹 프로그램에서 ``skb_event_output()``
  또는 ``xdp_event_output()`` helper 함수를 사용 하는 것이 좋습니다. BPF 프로그램
  의 사용자 지정 구조체를 선택적 패킷 샘플과 함께 ``perf`` 이벤트 링 버퍼로 전달할
  수 있습니다. 예를 들어, Cilium의 모니터는 디버깅 프레임 워크, 네트워크 정책
  위반에 대한 알림 등을 구현하기 위해 이 helper 함수 를 사용합니다.이러한 helper
  함수들은 잠금없는 메모리 매핑 된 CPU 당 성능 링 버퍼를 통해 데이터를 전달 하므로
  ``trace_printk()`` 보다 훨씬 빠릅니다.

5. **memset()/memcpy()/memmove()/memcmp()에 LLVM 내장 함수를 사용합니다.**

  BPF 프로그램은 BPF helper 가 아닌 다른 함수 호출을 수행 할 수 없기 때문에 공용
  라이브러리 코드를 인라인 함수로 구현해야합니다. 또한 LLVM은 프로그램이 일정한
  크기(여기서는 ``n`` )로 사용할 수있는 내장 함수를 제공 라며, 이 함수는 항상
  인라인 됩니다:

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

  ``memcmp()`` 내장함수는 백엔드에서 LLVM 문제로 인해 인라인이 발생하지 않는
  일부 특수한 사례가 있으므로 문제가 해결 될 때까지는 사용하지 않는 것이
  좋습니다.

6. **사용할 수 있는 루프가 없습니다(미완성).**

  커널의 BPF verifier는 BPF 프로그램이 다른 제어 흐름 그래프 검증 외에 가능한
  모든 프로그램 경로의 깊이 우선 검색을 수행하여 루프를 포함하고 있지 않은지
  확인합니다.목적은 프로그램이 항상 종료되도록 보장하는 것입니다.

  ``#pragma unroll`` 지시문을 사용하여 상한 루프 경계에서 매우 제한된 루핑
  형식을 사용할 수 있습니다. BPF로 컴파일 된 예제 코드:

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

  또 다른 가능성은 같은 프로그램에 다시 호출하고 로컬 스크래치 공간을 갖는
  ``BPF_MAP_TYPE_PERCPU_ARRAY`` 맵을 사용하여 tail 호출을 사용하는 것입니
  다. 동적 인 반면,이 루핑 형식은 최대 32 회 반복으로 제한됩니다.

  앞으로 BPF는 네이티브이지만 제한된 형태의 구현 루프를 가질 수 있습니다.

7. **tail 호출로 프로그램 분할.**

  tail call은 하나의 BPF 프로그램에서 다른 BPF 프로그램으로 점프하여 런타임
  중에 프로그램 동작을 atomically으로 변경할 수있는 유연성을 제공합니다.
  다음 프로그램을 선택하기 위해 tail call은 프로그램 배열 map
  (``BPF_MAP_TYPE_PROG_ARRAY``)을 사용하고 map뿐만 아니라 다음 프로그램에게
  인덱스를 다음 프로그램으로 점프 되도록 인덱스를 전달 합니다. 점프가 수행
  된 이후에는 이전 프로그램으로 리턴되지 않으며 주어진 map 인덱스에 프로그램
  이 없는 경우 처음 수행한 프로그램에서 실행을 계속됩니다.

  예를 들어, 이것은 파서의 다양한 단계를 구현하는 데 사용할 수 있으며, 이러
  한 단계는 런타임 중에 새로운 구문 분석 기능으로 업데이트 될 수 있습니다.

  또 다른 사용 사례는 이벤트 알림 이며, 예를 들어, Cilium은 런타임 중에 패킷
  drop 알림을 선택 할 수 있으며, ``skb_event_output()`` 함수 콜은 tail 호출된
  프로그램 안에 위치합니다. 따라서 일반적인 동작에서는 프로그램이 관련 map 인
  덱스에 추가되지 않으면 fall-through 경로가 항상 실행 되며, 여기에서 프로그
  램은 메타 데이터를 준비하고 사용자 공간에 있는 데몬에 이벤트 통지를 트리거
  합니다.

  프로그램 배열 map은 매우 유연하므로 각 map 인덱스에 있는 프로그램에 대해
  개별 액션을 구현할 수 있습니다. 예를 들어서, XDP 혹은 tc에 연결된 최상위
  프로그램은 프로그램 배열 맵의 인덱스 0에서 초기 tail 호출을 수행하여,
  트래픽 샘플링을 수행 한 다음 프로그램 배열 맵의 인덱스 1로 이동하며, 여기
  서 방화벽 정책을 적용하며, 프로그램 맵 배열 인덱스 2에서 패킷에 대해 drop
  또는 추후 처리중 하나를 선택하며, 여기서 패킷이 mangled 되고, 인터페이스
  에서 다시 전동 됩니다. 물론 프로그램 배열 맵의 점프는 임의적 일 수 있습
  니다. 최대 taill call 제한에 도달하면 커널은 결국 fall-through path 를
  실행합니다.

  tail 호출을 사용하는 최소 예제 발쵀는 아래와 같습니다:

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

  이 toy 프로그램을 로드 할 때, tc는 프로그램 배열을 생성하고 ``jmp_map``
  아래의 전역 네임 스페이스의 BPF 파일 시스템에 고정 시킵니다. 또한
  iproute2의 BPF ELF 로더는 ``__section_tail()`` 으로 표시 된 섹션을 인식
  합니다. ``struct bpf_elf_map`` 의 제공된 ``id`` 는 ``__section_tail()``
  의 id 마커, 즉 ``JMP_MAP_ID`` 와 일치하므로 프로그램은 사용자 지정 프로
  그램 배열 map 인덱스 (이 예제에서는 ``0`` )에서 로드 됩니다. 결과적으로
  제공된 모든 tail call 섹션은 iproute2 로더에 의해 해당 맵에 채워 집니다.
  이 메커니즘은 tc에만 해당되는 것이 아니라 iproute2가 지원하는 다른 BPF
  프로그램 유형(예 :XDP, lwt)에도 적용 할수 있습니다.

  생성 된 elf에는 map ID와 해당 map 내의 항목을 설명하는 섹션 헤더가 있습
  니다:

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

  이 경우 ``section 1/0`` 은 ``looper()`` 함수 가 ``0`` 위치의 map ID ``1``
  에 상주 함을 나타냅니다.

  고정 된 map은 새로운 프로그램으로 맵을 업데이트하기 위해 사용자 공간
  애플리케이션(예 : Cilium 데몬)에 의해 검색 될뿐만 아니라 tc 자체에 의해
  검색 될 수 있습니다. 업데이트는 atomically 으로 발생하며, 다양한 하위
  시스템에서 처음 트리거되는 초기 엔트리 프로그램도 자동 업데이트 됩니다.

  tail call map 업데이트를 수행하는 tc의 예:

  ::

    # tc exec bpf graft m:globals/jmp_map key 0 obj new.o sec foo

  iproute2가 고정 된 프로그램 배열을 갱신 할 경우, ``graft`` 명령을 사용할
  수 있습니다. ``globals/jmp_map`` 을 가리키면 tc는 인덱스/키 ``0`` 의 맵을
  ``foo`` 섹션 아래 ``new.o`` 오브젝트 파일에 있는 새 프로그램으로 갱신합니
  다.

8. **최대 512 바이트의 제한된 스택 공간.**

  BPF 프로그램의 스택 공간은 512 바이트로 제한되어 있으므로 C로 BPF 프로그
  램을 구현할 때는 신중하게 고려해야합니다. 그러나 앞의 3 번 항목에서 설명
  한 것처럼 스크래치 버퍼 공간을 확장하기 위해 단일 항목이있는
  ``BPF_MAP_TYPE_PERCPU_ARRAY`` 맵을 사용할 수 있습니다.

9. **BPF 인라인 어셈블리를 사용할 수 있습니다.**

  또한 LLVM은 필요할 수있는 드문 경우를 위해 BPF 용 인라인 어셈블리를 사용
  할 수 있습니다. 다음 (말도 안되는) toy 예제는 64 비트 atomic 추가를 보여
  줍니다. 문서가 없기 때문에 ``lib/Target/BPF/BPFInstrInfo.td`` 및
  ``test/CodeGen/BPF/`` 에있는 LLVM 소스 코드가 몇 가지 추가 예제를 제공하
  는 데 도움이 될 수 있습니다. 테스트 코드:

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

  위의 프로그램은 다음과 같은 BPF 명령어 순서로 컴파일 됩니다:

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

bcc, perf, iproute2 및 기타와 같은 BPF 프로그램을 커널에 로드 하기위한 다양
한 프런트 엔드가 있습니다. Linux 커널 소스 트리는 ``tools/lib/bpf/`` 디렉토
리 아래에 사용자 공간 라이브러리를 제공하는데, 이는 주로 BPF 추적 프로그램
을 커널에 로드하기 위해 perf에 의해 사용됩니다. 그러나 라이브러리 자체는
범용이며 perf에만 제한되지 않습니다. bcc는 BPF C 코드가 내장 된 Python 인터
페이스를 통해 ad-hoc으로 로드 되는 많은 유용한 BPF 프로그램을 주로 제공하는
툴킷 입니다. BPF 프로그램을 구현하기위한 구문 및 의미는 일반적으로 프론트
엔드간에 약간 다릅니다. 게다가, 생성된 오브젝트 파일을 구문 분석 하고 코드
를 시스템 콜 인터페이스에 직접 로드 하는 BPF 샘플 코드들이 커널 소스 트리
``samples/bpf/`` 아래에 있습니다.

Cilium 프로그램은 주로 BPF 로더에 대해 구현 되므로,현재 세션 및 이전 섹션은
주로 XDP, tc 또는 lwt 유형의 네트워킹 프로그램을 로드 하기 위한 iproute2
제품군의 BPF 프런트 엔드에 중점을 둡니다. 앞으로 Cilium에는 기본 BPF 로더가
갖추어져 있지만, 개발 및 디버깅을 용이하게 하기 위해 프로그램은 iproute2
제품군을 통해 로드 될 수 있도록 여전히 호환됩니다.

iproute2가 지원하는 모든 BPF 프로그램 타입은 공통 로더 백엔드를 라이브러리
(iproute2 소스 트리의 ``lib/bpf.c``)로 구현 하므로 동일한 BPF 로더 로직을
공유 합니다.

LLVM의 이전 섹션에서는 BPF C 프로그램 작성과 관련된 일부 iproute2 부분을
다루었으며, 이 문서의 뒷부분은 프로그램 작성시 tc 및 XDP 특정 측면과 관련
이 있습니다. 따라서 현재 섹션에서는 로더의 일반 메커니즘 뿐만 아니라
iproute2로 객체 파일을 로드하는 데 사용 예제에 초점을 맞춥니다. 모든 세부
사항을 완벽하게 다루지는 않지만 시작하기에 충분합니다.

**1. XDP BPF 객체 파일 로드.**


  BPF 객체 파일 ``prog.o`` 가 XDP 용으로 컴파일 된 경우, 다음 명령을
  사용하여 ``em1`` 이라는 XDP 지원 netdevice에 ``ip`` 명령어를 통해 로드
  할 수 있습니다:

  ::

    # ip link set dev em1 xdp obj prog.o

  위의 명령은 프로그램 코드가 XDP의 경우 ``prog`` 라고하는 기본 섹션에
  있다고 가정합니다. 이런 경우가 아니고 섹션의 이름이 다르게 지정되면
  (예 : ``foobar``) 프로그램을 다음과 같이 로드 해야 합니다:

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

  네트워크 인터페이스에 XDP 프로그램이 이미 연결되어 있어 실수로 덮어 쓰지
  않도록 기본적으로 ``ip`` 명령어는 오류를 발생 시킵니다. 현재 실행중인
  XDP 프로그램을 새 프로그램으로 바꾸려면 ``-force`` 옵션을 사용해야
  합니다:

  ::

    # ip -force link set dev em1 xdp obj prog.o

  오늘날 대부분의 XDP 지원 드라이버는 트래픽 중단 없이 기존 프로그램을
  새로운 것으로 대체합니다. 여기에는 성능상의 이유로 XDP 지원 드라이버에
  연결된 단일 프로그램 만 있으므로 프로그램 체인이 지원되지 않습니다.
  그러나 이전 섹션에서 설명한 것처럼 필요한 경우 유사한 사례를 얻기 위해
  tail 호출을 통해 프로그램을 나뉠 수 있습니다.

  인터페이스에 XDP 프로그램이 연결되어 있으면 ``ip link`` 명령에 ``xdp``
  플래그가 표시됩니다. 따라서 ``"ip link | grep xdp"`` 를 사용하여 XDP가
  실행되는 모든 인터페이스를 찾을 수 있습니다. ``ip link`` 덤프에 표시된
  BPF 프로그램 ID를 기반으로 연결된 프로그램에 대한 정보를 검색하는 데
  ``ip -d`` 링크가있는 자세히 보기를 통해 추가 자가 검사 기능이 제공 되며
  ``bpftool`` 을 사용하여 이 정보를 검색 할 수 있습니다.

  인터페이스에서 기존 XDP 프로그램을 제거하려면 다음 명령을 실행해야합
  니다:

  ::

    # ip link set dev em1 xdp off

  드라이버의 동작 방식을 non-XDP에서 네이티브 XDP로 또는 그 반대로 전환
  하는 경우, 일반적으로 드라이버는 수신 된 패킷이 BPF가 읽고 쓸 수있는
  단일 페이지 내에서 선형적으로 설정되도록 수신(및 전송) 링을 재구성해
  야합니다. 그러나, 일단 완료되면, 대부분의 드라이버는 BPF 프로그램을
  스왑 하도록 요청할 때 프로그램 자체의 atomic 교체를 수행하면 됩니다.

  전체적으로 XDP는 iproute2가 구현하는 세 가지 작동 방식을 지원 합니다
  : ``xdpdrv``, ``xdpoffload``, 그리고 ``xdpgeneric``.

  ``xdpdrv`` 는 네이티브 XDP를 의미하며,  즉, BPF 프로그램은 소프트웨어
  의 가능한 가장 빠른 시점에 드라이버의 수신 경로에서 직접 실행됩니다.
  이것은 일반 / 일반 XDP 방식이며, XDP 지원을 구현 하는 데 드라이버가
  필요하며, 업스트림 리눅스 커널의 모든 주요 10G/40G/+ 네트워킹 드라
  이버가 이미 제공합니다.

  ``xdpgeneric`` 은 일반 XDP를 나타내며 아직 네이티브 XDP를 지원하지
  않는 드라이버를 위한 실험용 테스트 베드로 사용됩니다.진입 경로의
  일반적인 XDP hook이 패킷이 이미 스택의 메인 수신 경로인 ``skb``
  로 들어 가는 훨씬 늦은 시점에 제공되며, ``xdpdrv`` 방식에서 처리
  하는 것보다 성능이 훨씬 낮습니다.그러므로 ``xdpgeneric`` 은 대부분
  실험적인 측면에서만 흥미롭우며 실제 운영 환경 에서는 사용이 작습
  니다.

  마지막으로, ``xdpoffload`` 모드는 Netronome의 nfp 드라이버가 지원
  하는 SmartNIC에서 구현되며 전체 BPF/XDP 프로그램을 하드웨어로 오프
  로드 할 수 있으므로 프로그램은 카드의 각 패킷 수신시 직접 실행됩니다.
  네이티브 XDP 와 비교하여 모든 BPF map 유형 또는 BPF helper 함수를
  사용할 수 있는 것은 아니지만 네이티브 XDP에서 실행하는 것보다 훨씬
  높은 성능을 제공합니다. 이 경우 BPF verifier는 프로그램을 거부하고
  그리고 지원되지 않는 것을 사용자에게 보고합니다. 지원되는 BPF 기능
  및 도우미 기능의 영역에 머무르는 것 외에 BPF C 프로그램을 작성할
  때는 특별한 주의를 기울이 지 않아도됩니다

  ``ip link set dev em1 xdp obj [...]`` 와 같은 명령이 사용되면 커널
  은 먼저 프로그램을 기본 XDP로 로드 하려고 시도하고 그리고 드라이버가
  네이티브 XDP를 지원하지 않으면 자동으로 일반 XDP로 되돌아갑니다.
  따라서 예를 들어 ``xdp`` 대신 명시적으로 ``xdpdrv`` 를 사용하면 커널
  은 프로그램을 raw XDP로 로드 하려고 시도하고 드라이버가 지원하지 않는
  경우 실패 하여 일반 XDP가 모두 회피 되는 보장을 제공 합니다.

  네이티브 XDP 방식에서 로드 할 BPF/XDP 프로그램 실행, 링크 세부 정보
  덤프 및 프로그램 언로드 예제:

  ::

     # ip -force link set dev em1 xdpdrv obj prog.o
     # ip link show
     [...]
     6: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp qdisc mq state UP mode DORMANT group default qlen 1000
         link/ether be:08:4d:b6:85:65 brd ff:ff:ff:ff:ff:ff
         prog/xdp id 1 tag 57cd311f2e27366b
     [...]
     # ip link set dev em1 xdpdrv off

  드라이버가 기본 XDP를 지원하고 bpftool을 통해 삽입된 더미 프로그램의
  BPF 명령어를 추가로 덤프하는 경우에도 일반 XDP를 강제 실행하는 것과
  같은 예제가 있습니다:

  ::

    # ip -force link set dev em1 xdpgeneric obj prog.o
    # ip link show
    [...]
    6: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpgeneric qdisc mq state UP mode DORMANT group default qlen 1000
        link/ether be:08:4d:b6:85:65 brd ff:ff:ff:ff:ff:ff
        prog/xdp id 4 tag 57cd311f2e27366b                <-- BPF 프로그램 ID 4
    [...]
    # bpftool prog dump xlated id 4                       <-- em1에서 실행중인 명령어 덤프
    0: (b7) r0 = 1
    1: (95) exit
    # ip link set dev em1 xdpgeneric off

  마지막이긴 하나 중요한 오프로드 된 XDP는 일반 메타 데이터 검색을 위해
  bpftool을 통해 프로그램 정보를 추가로 덤프합니다:

  ::

     # ip -force link set dev em1 xdpoffload obj prog.o
     # ip link show
     [...]
     6: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpoffload qdisc mq state UP mode DORMANT group default qlen 1000
         link/ether be:08:4d:b6:85:65 brd ff:ff:ff:ff:ff:ff
         prog/xdp id 8 tag 57cd311f2e27366b
     [...]
     # bpftool prog show id 8
     8: xdp  tag 57cd311f2e27366b dev em1                  <-- 또한 em1에 오프로드 된 BPF 프로그램을 나타냅니다
         loaded_at Apr 11/20:38  uid 0
         xlated 16B  not jited  memlock 4096B
     # ip link set dev em1 xdpoffload off

  ``xdpdrv`` 와 ``xdpgeneric`` 또는 다른 방식를 동시에 사용할 수는 없으며,
  이는 XDP 동작 방식 중 하나만 선택해야 함을 의미합니다.

  다른 XDP 모드 간의 전환은 예를 들어 generic에서 native로 또는 그 반대
  로도 atomically으로 가능하지 않습니다:

  ::

     # ip -force link set dev em1 xdpgeneric obj prog.o
     # ip -force link set dev em1 xdpoffload obj prog.o
     RTNETLINK answers: File exists
     # ip -force link set dev em1 xdpdrv obj prog.o
     RTNETLINK answers: File exists
     # ip -force link set dev em1 xdpgeneric obj prog.o    <-- xdpgeneric로 인해 성공
     #

  방식을 전환하려면 먼저 현재 작동 모드를 종료 한 후 새로운 방식으로 전환
  해야합니다:

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

**2. tc BPF 오브젝트 파일의 로딩.**

  ``prog.o`` 가 tc 용으로 컴파일 된 BPF 오브젝트 파일이 고려될때, tc 명령을
  통해 netdevice에 로드 할 수 있습니다. XDP와 달리 장치에 BPF 프로그램을
  연결하는 데 필요한 드라이버 의존성이 없습니다. 여기서 netdevice는 ``em1``
  이라고하며 다음 명령을 사용하여 프로그램을 em1의 네트워킹 ``ingress``
  경로에 연결할 수 있습니다:

  ::

    # tc qdisc add dev em1 clsact
    # tc filter add dev em1 ingress bpf da obj prog.o

  첫 번째 단계는 ``clsact`` qdisc (리눅스 대기 행렬의 규칙)를 설정하는
  것입니다. ``clsact`` 는 ``ingress`` qdisc와 비슷한 더미 qdisc이며,
  classifier 및 action 만 보유 할 수 있지만 실제 대기열을 작동 하지는
  않습니다. ``bpf`` classifier를 연결하기 위해 필요합니다. ``clsact``
  qdisc는 ``ingress`` 및 ``egress`` 라고 하는 두 개의  특수 hook를 제공
  하며, 이 hook는 classifier를 연결할 수 있습니다. ``ingress`` 및
  ``egress`` hook은 장치의 모든 패킷이 통과하는 네트워킹 데이터 경로의
  중심 수신 및 전송 위치에 있습니다. ``ingress`` hook은 커널의 함수의
  ``__netif_receive_skb_core () -> sch_handle_ingress ()`` 호출되며,
  ``egress`` hook은 ``__dev_queue_xmit () -> sch_handle_egress ()``
  에서 호출됩니다.

  ``egress`` hook에 프로그램을 연결하는 것에 해당하는 것은 다음과 같
  습니다:

  ::

    # tc filter add dev em1 egress bpf da obj prog.o

  ``clsact`` qdisc는 ``ingress`` 및 ``egress`` 방향에서 lockless로 처리
  되며, 컨테이너를 연결하는 ``veth`` 장치와 같은 가상, 대기열없는 장치
  에도 연결할 수 있습니다.

  hook 다음에, ``tc filter`` 명령은 ``da`` (direct-action mode)
  방식에서 사용 할 ``bpf`` 를 선택합니다. ``da`` mode가 권장이 되며
  그리고 항상 명시 해야 합니다. 기본적으로 ``bpf`` classifier는 모든
  패킷 mangling 및 포워딩 또는 다른 동작이 이미 single bpf 프로그램
  내부에 서 수행 될 수 있기 때문에 ``bpf`` 필요하지 않은 외부 tc
  action 모듈을 호출 할 필요가 없다는 것을 의미하며, 연결이 될때
  상당히 빨라 집니다.

  이 시점에서 프로그램이 연결되어 패킷이 장치를 통과하면 프로그램이 실행
  됩니다. XDP처럼, 기본 섹션 이름을 사용하지 않으면 로드 중에 ``foobar``
  섹션과 같이 지정할 수 있습니다:

  ::

    # tc filter add dev em1 egress bpf da obj prog.o sec foobar

  iproute2의 BPF 로더는 프로그램 타입에 따라 동일한 명령 행 구문을 사용
  할 수 있으므로 ``obj prog.o sec foobar`` 는 앞에서 언급 한 XDP와 같은
  구문입니다.

  연결 된 프로그램은 다음 명령을 통해 나열 될 수 있습니다:

  ::

    # tc filter show dev em1 ingress
    filter protocol all pref 49152 bpf
    filter protocol all pref 49152 bpf handle 0x1 prog.o:[ingress] direct-action id 1 tag c5f7825e5dac396f

    # tc filter show dev em1 egress
    filter protocol all pref 49152 bpf
    filter protocol all pref 49152 bpf handle 0x1 prog.o:[egress] direct-action id 2 tag b2fd5adc0f262714

  ``prog.o : [ingress]`` 의 출력은 프로그램 섹션 ``ingress`` 가 ``prog.o``
  파일에서 로드 되었으며, ``bpf`` 가 직접-실행(direct-action mode) 방식
  에서 동작 하는 것을 나타냅니다.프로그램 ``id`` 및 ``tag`` 는 각각 값
  이며, 마지막 명령어 스트림에 대한 hash 나타내며, 오브젝트 파일 혹은
  stack trace가 있는 ``perf`` reports 등과 같은 관련을 될수 있습니다.
  마지막이긴 하나 중요한 id는 ``bpftool`` 과 함께 사용하여 첨부 된 BPF
  프로그램을 추가로 검사하거나 덤프 할 수있는 시스템 전체의 고유 한 BPF
  프로그램 식별자를 나타냅니다.

  tc는 단 하나의 BPF 프로그램 이상을 연결 할 수 있으며, tc는 함께 chained
  할 수 있는 다양한 다른 classifier들을 제공합니다.하지만, BPF 프로그램
  자체가 이미 ``TC_ACT_OK``, ``TC_ACT_SHOT`` 및 기타와 같은 tc 작업 결정을
  반환한다는 것을 의미하는 ``da`` (직접-실행, ``direct-action`` )방식 덕분
  에 모든 패킷 조작이 프로그램 자체에 포함될 수 있기 때문에 단일 BPF 프로그
  램을 연결 만으로 충분합니다. 최적의 성능과 유연성을 위해 권장되는
  사용법 입니다.

  위의 ``show`` 명령에서 tc는 BPF 관련 출력 옆에 ``pref 49152`` 및 ``handle 0x1``
  도 표시합니다. 둘 다 명령 줄을 통해 명시적으로 제공되지 않는 경우 자동
  생성됩니다. ``pref`` 는 우선 순위 번호를 나타내며, 이는 다중 classifier들이
  연결 된 경우 오름차순 우선 순위에 따라 실행되는 것을 의미하며, handle은 같은
  classifier의 다중 인스턴스가 동일한 ``pref`` 아래 로드 된 경우 식별자를 나타
  냅니다. BPF의 경우 하나의 프로그램으로 충분하기 때문에 일반적으로 ``pref``
  와 ``handle`` 을 무시할 수 있습니다.

  연결된 BPF 프로그램들을 atomically하게 교체할 계획이 있는 경우 에만, 초기
  로드시 연역적으로 ``pref`` 및 ``handle`` 을 명시적으로 지정하는 것을 추천하며,
  따라서 나중에 ``replace`` 작업을 위해 따로 쿼리할 필요는 없습니다.
  따라서 창작물은 다음과 같이 됩니다:

  ::

    # tc filter add dev em1 ingress pref 1 handle 1 bpf da obj prog.o sec foobar

    # tc filter show dev em1 ingress
    filter protocol all pref 1 bpf
    filter protocol all pref 1 bpf handle 0x1 prog.o:[foobar] direct-action id 1 tag c5f7825e5dac396f

  그리고 atomic 교체을 위해 ``foobar`` 섹션의 ``prog.o`` 파일에서 새로운 BPF
  프로그램으로 ``ingress`` hook에서 기존 프로그램을 업데이트하기 위해 다음을
  발행 할 수 있습니다:

  ::

    # tc filter replace dev em1 ingress pref 1 handle 1 bpf da obj prog.o sec foobar

  마지막으로 중요한 것은 첨부 된 각 프로그램을 ``ingress`` 및 ``egress`` hook에서
  모두 제거하려면 다음을 사용할 수 있습니다:

  ::

    # tc filter del dev em1 ingress
    # tc filter del dev em1 egress

  netdevice에서 ``clsact`` qdisc 전체 즉 암시적으로 ``ingress`` 및 ``egress`` hook에
  모든 연결된 프로그램을 제거하려면 다음과 같은 명령이 제공 됩니다 :

  ::

    # tc qdisc del dev em1 clsact

  tc BPF 프로그램은 만약 NIC 및 드라이버가 XDP BPF 프로그램과 유사하게 지원 되는 경
  우 오프로드 할 수 있습니다. Netronome의 nfp 지원 NIC는 두 가지 유형의 BPF 오프로드
  를 제공합니다.

  ::

    # tc qdisc add dev em1 clsact
    # tc filter replace dev em1 ingress pref 1 handle 1 bpf skip_sw da obj prog.o
    Error: TC offload is disabled on net device.
    We have an error talking to the kernel

  위의 오류가 표시되면 ethtool의 ``hw-tc-offload`` 설정을 통해 장치의 tc 하드웨어
  오프로드를 먼저 활성화 해야 합니다:

  ::

    # ethtool -K em1 hw-tc-offload on
    # tc qdisc add dev em1 clsact
    # tc filter replace dev em1 ingress pref 1 handle 1 bpf skip_sw da obj prog.o
    # tc filter show dev em1 ingress
    filter protocol all pref 1 bpf
    filter protocol all pref 1 bpf handle 0x1 prog.o:[classifier] direct-action skip_sw in_hw id 19 tag 57cd311f2e27366b

  ``in_hw`` 플래그는 프로그램이 NIC로 오프로드 되었음을 확인합니다.

  tc와 XDP의 BPF 오프로드를 동시에 로드 할 수 없으므로 tc 또는 XDP 오프로드
  옵션을 선택해야합니다.

**3. netdevsim 드라이버를 통해 BPF 오프로드 인터페이스를 테스트 합니다.**

  Linux 커널의 일부인 netdevsim 드라이버는 XDP BPF 및 tc BPF 프로그램 용 오프
  로드 인터페이스를 구현하는 더미 드라이버를 제공하며 커널의 UAPI에 대해 직접
  control plane을 구현하는 커널 변경 또는 low-level 사용자 공간 프로그램 테스
  트를 가능 하게합니다.

  netdevsim 장치는 다음과 같이 생성 될 수 있습니다 :

  ::

    # ip link add dev sim0 type netdevsim
    # ip link set dev sim0 up
    # ethtool -K sim0 hw-tc-offload on
    # ip l
    [...]
    7: sim0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
        link/ether a2:24:4c:1c:c2:b3 brd ff:ff:ff:ff:ff:ff

  이 단계가 끝난 후 XDP BPF 또는 tc BPF 프로그램은 앞서 다양한 예제에 표시
  된대로 로드 할 수 있습니다:

  ::

    # ip -force link set dev sim0 xdpoffload obj prog.o
    # ip l
    [...]
    7: sim0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 xdpoffload qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
        link/ether a2:24:4c:1c:c2:b3 brd ff:ff:ff:ff:ff:ff
        prog/xdp id 20 tag 57cd311f2e27366b

이 두 워크 플로는 iproute2를 사용하여 XDP BPF 및 tc BPF 프로그램을 로드하는
기본 작업입니다.

BPF 로더에는 XDP와 tc에 모두 적용되는 여러 가지 고급 옵션이 있으며 그 중
일부는 여기에 나열되어 있습니다. 예제에서 XDP는 단순화를 위해 제시된 것입
니다.

**1. 성공시에도 자세한 로그 출력.**

  오류가 발생하지 않은 경우에도 verifier 로그를 덤프하기 위해 ``verb`` 옵션
  을 추가하여 프로그램을 로드 할 수 있습니다:

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

**2. BPF 파일 시스템에 이미 고정되어 있는 프로그램을 로드 하십시오.**

  오브젝트 파일에서 프로그램을 로드하는 대신에, iproute2는 BPF 파일 시스템
  에서 프로그램을 검색 할 수 있으며, 외부 엔티티가 거기에 고정시키고 장치에
  연결하는 경우입니다:

  ::

  # ip link set dev em1 xdp pinned /sys/fs/bpf/prog

  iproute2는 감지 된 BPF 파일 시스템의 마운트 지점과 관련된 짧은 형식을 사용
  할 수도 있습니다:

  ::

  # ip link set dev em1 xdp pinned m:prog

BPF 프로그램을 로드 할 때, iproute2는 노드의 고정을 수행하기 위해 마운트 된
파일 시스템 인스턴스를 자동으로 감지합니다. 마운트 된 BPF 파일 시스템 인스턴
스가 없는 경우, tc는 이를 자동으로 ``/sys/fs/bpf/`` 아래의 기본 위치에 마운트
합니다.

인스턴스가 이미 발견 된 경우, 인스턴스가 사용되며 추가 마운트가 수행되지 않습
니다:

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

기본적으로 tc는 위와 같이 모든 서브 시스템 사용자는 ``globals`` 이름 공간에 대한
기호 링크를 통해 동일한 위치를 가리키는 초기 디렉토리 구조를 생성하며, 고정
된 BPF 맵을 iproute2의 다양한 BPF 프로그램 유형간에 재사용 할 수 있습니다.
파일 시스템 인스턴스가 이미 마운트 되었고 기존 구조가 이미 존재하는 경우,
tc는 파일 시스템 인스턴스를 오버라이드 하지 않습니다. 이는 ``globals`` 을 공유하
지 않기 위해 ``lwt``, ``tc`` 및 ``xdp`` map을 분리하는 경우가 발생 할 수 있습니
다.

이전 LLVM 섹션에서 간단히 설명 했듯이, iproute2는 설치 시 BPF 프로그램에 의해
표준 include 경로를 통해 포함될 수있는 헤더 파일을 설치합니다:

  ::

    #include <iproute2/bpf_elf.h>

이 헤더 파일의 목적은 프로그램에 사용되는 map과 기본 섹션 이름에 대한 API를 제공
하는 것입니다. 그것은 iproute2와 BPF 프로그램 사이의 안정적인 계약입니다.

iproute2에 대한 map 정의는 ``struct bpf_elf_map`` 입니다. 그것의 구성은 이 문서
의 LLVM 부분에서 더 일찍 다루어졌습니다.

BPF 객체 파일을 구문 분석 할 때 iproute2 로더는 모든 ELF 섹션을 처리합니다.
처음에는 ``maps`` 및 ``license`` 와 같은 보조 섹션을 가져옵니다. ``maps`` 의 경우
``struct bpf_elf_map`` 배열의 유효성을 검사하고 필요할 때마다 호환성 우회 방법이
수행 됩니다. 결과적으로 모든 map은 사용자가 제공 한 정보로 생성되며, 고정 된 객체
로 검색되거나 새로 생성 된 다음 BPF 파일 시스템에 고정됩니다. 다음으로 로더는 map
에 대한 ELF 재배치 엔트리가 포함된 프로그램 세션을 처리 하며, 이 의미는 map 파일
디스크립터를 레지스터리에 로드하는 BPF 명령어로 다시 작성되어, 해당 map 파일 디스
크립터가 immediate 값으로 인코딩이 되고, 이는 커널이 >나중에 커널 포인터로 변환
할 수 있도록 하기 위해서 입니다. 이후 모든 프로그램 자체가 BPF 시스템 호출을 통
해 생성되고 tail 호출된 map이 있는 경우, 이 프로그램의 파일 디스크립터로 업데이트
됩니다.

bpftool
-------

bpftool은 BPF 주변의 주요 자가 검사 및 디버깅 도구이며 ``tools/bpf/bpftool/``
아래의 Linux 커널 트리와 함께 개발 및 제공됩니다.

이 도구는 현재 시스템에 로드 된 모든 BPF 프로그램과 맵을 덤프하거나, 특정
프로그램에서 사용하는 모든 BPF 맵을 나열하고 연관시킬 수 있습니다.
또한 전체 맵의 키/값 쌍을 덤프하거나 개별 맵을 조회, 업데이트, 삭제할 수
있을 뿐만 아니라 맵에서 키의 인접 키를 검>색 할 수 있습니다. 이러한 작업은
BPF 프로그램 또는 맵 ID를 기반으로 수행하거나, BPF 파일 시스템 고정 프로그램
또는 맵의 위치를 지정하여 수행 할 수 있습니다. 또한 이 도구는 map을 고정하는
옵션 또는 프로그램을 BPF 파일 시스템에 제공합니다.

현재 호스트에 로드 된 모든 BPF 프로그램의 개요 내용을 확인하려면 다음 명령을
호출하십시오:

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

마찬가지로 모든 활성 map의 개요를 보려면 다음을 수행하십시오:

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

각 명령에 대해 bpftool은 명령 줄 끝에 ``--json`` 을 추가하여 json 기반
출력을 지원합니다. 추가적인 ``--pretty`` 는 사람이 읽을 수 있는 출력을
향상 시킵니다.

  ::

     # bpftool prog --json --pretty

특정 BPF 프로그램의 verifier 후 BPF 명령어 이미지를 덤프 하는 경우,
예를 들어 tc ingress hook에 연결된 특정 프로그램을 검사 할 수 있습니다.

  ::

     # tc filter show dev cilium_host egress
     filter protocol all pref 1 bpf chain 0
     filter protocol all pref 1 bpf chain 0 handle 0x1 bpf_host.o:[from-netdev] \
                         direct-action not_in_hw id 406 tag e0362f5bd9163a0a jited

객체 파일 ``bpf_host.o`` 프로그램은 ``from-netdev`` 섹션에서 ``id 406``
에 표시 된대로, 프로그램 ID ``406`` 을 가지고 있습니다. 이 정보를 바탕
으로 bpftool은 프로그램과 관련된 몇 가지 상위 레벨 메타 데이터를 제공
할 수 있습니다:

  ::

     # bpftool prog show id 406
     406: sched_cls  tag e0362f5bd9163a0a
          loaded_at Apr 09/16:24  uid 0
          xlated 11144B  jited 7721B  memlock 12288B  map_ids 18,20,8,5,6,14

ID 406의 프로그램은 ``sched_cls`` (``BPF_PROG_TYPE_SCHED_CLS``) 타입이며,
``e0362f5bd9163a0a`` (명령 시퀀스에 대한 sha 합계) ``tag`` 를 가지고 있으
며, ``Apr 09/16:24`` 에 root ``uid 0`` 에 의해 로드 되었습니다. BPF 명령
시퀀스는 ``11,144 바이트`` 이며, 주입된 이미지는 ``7,721 바이트`` 입니다.
프로그램 자체 (map 제외)는 사용자 ``uid 0`` 에 대해 계산된/소비된
accounted / charged ``12,288 바이트`` 를 사용합니다. 그리고 BPF 프로그램은
ID가 ``18``, ``20``, ``8``, ``5``, ``6`` 및 ``14`` 인 BPF 맵을 사용 합니다.
이후의 ID는 정보를 얻거나 맵을 덤프 하는데 더 사용될 수 있습니다.

또한 bpftool은 프로그램이 실행하는 BPF 명령의 덤프 요청을 실행 할 수 있습
니다:

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
     54: (85) call bpf_skb_event_output#5656112  <-- BPF helper 호출
     55: (69) r1 = *(u16 *)(r6 +192)
     [...]

bpftool은 BPF helper 또는 다른 BPF 프로그램에 대한 호출 뿐만 아니라 위의 그림과
같이 BPF map ID를 명령 스트림에 상관 관계가 있습니다.

명령 덤프는 커널의 BPF verifier와 동일한 'pretty-printer'를 다시 사용 합니다.
위의 ``xlated`` 명령에서 생성된 프로그램이 주입 되어 실제 JIT 이미지가 실행 되
므로 bpftool을 통해서 덤프 될 수 있습니다:

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

BPF JIT 개발자를 중심으로, 실제 네이티브 opcode로 디스 어셈블리를 interleave
하는 옵션도 있습니다:

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

커널에서 디버깅 할 때 유용 할 수있는 일반적인 BPF 명령어에 대해 동일한
interleave을 수행 할 수 있습니다:

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

``graphviz`` 의 도움을 받아 프로그램의 기본 블록을 시각화 할 수도 있습
니다. 이 목적을 위해 bpftool은 나중에 png 파일로 변환 할 수 있는 일반
BPF ``xlated`` 명령 덤프 대신 도트 파일을 생성하는 ``visual`` 덤프 모드
를 가지고 있습니다:

  ::

     # bpftool prog dump xlated id 406 visual &> output.dot
     $ dot -Tpng output.dot -o output.png

또 다른 옵션은 dot 파일을 뷰어인 dotty로 전달 하는 것이며, 즉
``dotty output.dot`` 하는 경우 , ``bpg_host.o`` 프로그램의 결과는 다음
과 같습니다(작은 발췌).:

.. image:: images/bpf_dot.png
    :align: center

``xlated`` 명령어 덤프는 만약 BPF 인터프리터를 통해 실행될 경우와 같은
명령어를 덤프한다는 의미의 verifier후 BPF 명령어 이미지를 제공합니다.
커널에서 verifier 는 BPF 로더가 제공 한 본래 명령어의 다양한 다시 쓰기
을 수행합니다.

다시 쓰기의 한가지 예는 런타임 성능을 향상 시키기 위해, helper 함수의
인라이닝 (inline) 이며, 여기 hash 테이블에 대한 map 조회 경우 입니다:

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

bpftool은 모듈이 참조할 수 있는 커널 내부의 함수나 변수의 심볼 정보
인 kallsyms 를 통해, helper 함수 또는 BPF 호출에서 BPF 호출에 연결
합니다. 따라서 주입되어 BPF 프로그램이 kallsyms(``bpf_jit_kallsyms``
파라메터를 통해)에 노출 되어 있고, kallsyms주소가 난독화되지 않는지
확인하십시오 호출은 ``call bpf_unspec#0`` 로 표시됩니다):

  ::

     # echo 0 > /proc/sys/kernel/kptr_restrict
     # echo 1 > /proc/sys/net/core/bpf_jit_kallsyms

BPF 호출에서 BPF 호출은 JIT 경우뿐만 아니라 인터프리터와 상관 관계
가 있습니다. 뒤에서는 서브 프로그램의 태그가 호출 대상으로 표시됩
니다. 각각의 경우, ``pc+2`` 는 호출 대상의 pc-상대 오프셋이며, 서브
프로그램을 나타냅니다.

  ::

     # bpftool prog dump xlated id 1
     0: (85) call pc+2#__bpf_prog_run_args32
     1: (b7) r0 = 1
     2: (95) exit
     3: (b7) r0 = 2
     4: (95) exit

덤프의 주입된 변형:

  ::

     # bpftool prog dump xlated id 1
     0: (85) call pc+2#bpf_prog_3b185187f1855c4c_F
     1: (b7) r0 = 1
     2: (95) exit
     3: (b7) r0 = 2
     4: (95) exit

tail 호출의 경우, 커널은 내부적으로 하나의 명령어로 매핑을 하며,
bpftool은 디버깅을 쉽게하기 위해 helper 호출과 연관 시킵니다:

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

``map dump`` 서브 명령을 사용하여, 현재의 모든 맵 요소를 반복하고
키 / 값 쌍으로 16진 수로 덤프하는 전체 맵을 덤프할 수 있습니다.

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

특정 키에 대한 map에서 조회, 업데이트, 삭제 및 `다음 키 얻기` 작업 은 bpftool을
통해서도 수행 할 수 있습니다.

BPF sysctls
-----------

리눅스 커널에서는 몇가지 BPF와 관련된 sysctl에 대한 제공을 하며, 이 장에서는
이 내용을 다룹니다.

* ``/proc/sys/net/core/bpf_jit_enable``: BPF JIT 컴파일러를 활성화 혹은 비활성화
                                         하도록 설정합니다.

  +-------+-------------------------------------------------------------------+
  | 값    | 설명                                                              |
  +-------+-------------------------------------------------------------------+
  | 0     | JIT를 비활성화 하며, 인터프리터만 사용합니다 (커널 기본값)        |
  +-------+-------------------------------------------------------------------+
  | 1     | JIT 컴파일러를 활성화 합니다.                                     |
  +-------+-------------------------------------------------------------------+
  | 2     | JIT를 활성화 하며,  커널 로그에 emit 디버깅 추적을 남깁니다.      |
  +-------+-------------------------------------------------------------------+

  다음 절에서 설명하는 것 처럼, ``bpf_jit_disasm`` 도구는 JIT 컴파일러가 디버깅
  모드 (옵션 ``2``)로 설정된 경우 디버깅 추적을 처리하는 데 사용 할 수 있습니다.

* ``/proc/sys/net/core/bpf_jit_harden``: BPF JIT 하드닝를 활성화 혹은 비활성화
  하도록 설정합니다. 하드닝 기능을 사용하면 성능이 저하되지만, BPF 프로그램의
  즉각적인 값을 무시함으로써 JIT 스프레이 공격에 대해 완화 할 수 있습니다.
  인터프리터를 통해 처리되는 프로그램의 경우 즉시 값의 블라인드가 필요하지 않거
  나 수행되지 않습니다.

  +-------+-------------------------------------------------------------------+
  | 값    | 설명                                                              |
  +-------+-------------------------------------------------------------------+
  | 0     | JIT 하드닝 비활성화 (커널의 기본값)                               |
  +-------+-------------------------------------------------------------------+
  | 1     | 허가받은 사용자들에 대해서 JIT 하드닝 활성화                      |
  +-------+-------------------------------------------------------------------+
  | 2     | 모든 사용자들에 대해서 JIT 하드닝 활성화                          |
  +-------+-------------------------------------------------------------------+

* ``/proc/sys/net/core/bpf_jit_kallsyms``: 주입된 프로그램을 ``/proc/kallsyms`` 에
  대한 커널 심볼로 내보내기를 활성화하거나 비활성화 하도록 설정하며, ``perf`` 도구
  와 함께 사용할 수 있으며 이러한 주소가 스택 해제를 위해 커널에 인식 되며, 예를
  들어 스택 추적을 덤프라는 데 사용됩니다. 심볼 이름에는 BPF 프로그램 태그
  (``bpf_prog_<tag>``)가 포함됩니다. 만약 ``bpf_jit_harden`` 파라메터 값이 활성화
  되어 있다면, 이 기능은 비활성화 됩니다.

  +-------+-------------------------------------------------------------------+
  | 값    | 설명                                                              |
  +-------+-------------------------------------------------------------------+
  | 0     | JIT kallsyms 내보내기 비활성화(커널 기본값)                       |
  +-------+-------------------------------------------------------------------+
  | 1     | 허가받은 사용자들에 대해서 JIT kallsyms 내보내기 활성화           |
  +-------+-------------------------------------------------------------------+

* ``/proc/sys/kernel/unprivileged_bpf_disabled``: ``bpf(2)`` 시스템 호출의 권한없는
  사용을 활성화하거나 비활성화 하도록 설정합니다. Linux 커널은 기본적으로 ``bpf(2)``
  를 사용하는 권한 없는 사용에 대해서 활성화 되어 있으며, 이 옵션이 바뀌면, 다음
  재부팅 때까지 권한이 없는 사용이 영구적으로 비 활성화됩니다. 이 sysctl 설정은 일회성
  스위치 이며, 즉 일단 설정되면, 응용프로그램이나 관리자가 더 이상 값을 재설정 할수
  없습니다. 이 설정은 프로그램을 커널에 로드하기 위해 ``bpf(2)`` 시스템 호출을 사용하지
  않는 seccomp 또는 기존 소켓 필터와 같은 cBPF 프로그램에는 영향을 주지 않습니다.

  +-------+-------------------------------------------------------------------+
  | 값    | 설명                                                              |
  +-------+-------------------------------------------------------------------+
  | 0     | 권한없는 bpf 시스템 호출 사용 활성화 (커널 기본값)                |
  +-------+-------------------------------------------------------------------+
  | 1     | 권한없는 bpf 시스템 호출 사용 비활성화                            |
  +-------+-------------------------------------------------------------------+

Kernel Testing
--------------

Linux 커널은 커널 소스 트리에서 ``tools/testing/selftest/bpf`` 아래에 BPF
자가 테스트 모음을 제공 합니다.

::

    $ cd tools/testing/selftests/bpf/
    $ make
    # make run_tests

테스트 모음는 BPF verifier에 대한 테스트 케이스, 프로그램 태그, BPF map 인터페이스 및
map 유형에 대한 다양한 테스트를 포함합니다. LLVM 백엔드 확인을위한 C 코드와 인터프리터
및 JIT 테스트를 위해 커널에서 실행되는 cBPF asm 코드 뿐만 아니라 eBPF의 다양한 런타임
테스트가 포함되어 있습니다.

JIT 디버깅
----------

감사를 수행 하거나 확장 기능을 수행하는 JIT 개발자의 경우, 각 컴파일러 실행은 생성된
JIT 이미지를 다음을 통해 커널 로그에 출력 할수 있습니다
::

    # echo 2 > /proc/sys/net/core/bpf_jit_enable

새 BPF 프로그램이 로드 될 때마다 JIT 컴파일러는 출력을 덤프하고 ``dmesg`` 를 통해
점검 할 수 있으며, 예를 들면 다음과 같습니다:

::

    [ 3389.935842] flen=6 proglen=70 pass=3 image=ffffffffa0069c8f from=tcpdump pid=20583
    [ 3389.935847] JIT code: 00000000: 55 48 89 e5 48 83 ec 60 48 89 5d f8 44 8b 4f 68
    [ 3389.935849] JIT code: 00000010: 44 2b 4f 6c 4c 8b 87 d8 00 00 00 be 0c 00 00 00
    [ 3389.935850] JIT code: 00000020: e8 1d 94 ff e0 3d 00 08 00 00 75 16 be 17 00 00
    [ 3389.935851] JIT code: 00000030: 00 e8 28 94 ff e0 83 f8 01 75 07 b8 ff ff 00 00
    [ 3389.935852] JIT code: 00000040: eb 02 31 c0 c9 c3

``flen`` 는 BPF 프로그램 여기서는 6개 BPF 명령어이며, ``proglen`` 는 opcode 이미지
에 대해 JIT에서 생성된 70 byte 를 알려 줍니다. ``pass`` 는 이미지가 3 단계 compiler
passes 에서 생성 된 것을 의미 하며, 예를 들어 x86_64는 가능한 경우에 이미지 크기를
줄이기 위해 다양한 최적화 단계를 가질수 있습니다. ``image`` 는 생성 된 JIT 이미지의
주소를 포함하고 있으며, ``from`` 및 ``pid`` 는 컴파일 프로세스를 트리거 한 사용자
영역 응용 프로그램 이름과 PID입니다. eBPF 및 cBPF JIT의 덤프 출력은 동일한 형식입니다.

``tools/bpf/`` 아래의 커널 트리에는 ``bpf_jit_disasm`` 불리는 도구가 있습니다. 최신
덤프를 읽고 추가 검사를 위해 디스 어셈블리를 출력합니다:

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

또는 이 도구는 디스어셈블 와 함께 관련 opcode를 덤프 할 수도 있습니다.

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

보다 최근에 ``bpftool`` 은 이미 시스템에 로드 된 BPF 프로그램 ID를 기반으로
BPF JIT 이미지를 덤프하는 것과 동일한 기능을 채택했습니다 (bpftool 섹션 참조
부탁드립니다).

주입된 BPF 프로그램의 성능 분석을 위해 일반적으로 ``perf`` 를 사용할 수 있습
니다. 전제 조건으로, 주입된 프로그램은 kallsyms 인프라를 통해 export 되어야
합니다.

::

    # echo 1 > /proc/sys/net/core/bpf_jit_enable
    # echo 1 > /proc/sys/net/core/bpf_jit_kallsyms

``bpf_jit_kallsyms`` 파라메터를 활성화 또는 비 활성화에는 관련 BPF 프로그램을
다시 로드 할 필요가 없습니다. 다음으로 BPF 프로그램을 프로파일링하기 위한 작은
작업 흐름 예제가 제공됩니다. 만들어진 tc BPF 프로그램은 ``bpf_clone_redirect()``
helper 함수에서 perf가 실패한 할당을 기록하는 데 사용됩니다. 직접 쓰기를 사용하기
때문에, ``bpf_try_make_head_writable()`` 함수에 실패하여 복제 된 ``skb`` 를 다시
해제하고 오류 메시지와 함께 리턴합니다. ``perf`` 는 모든 ``kfree_skb`` 이벤트를
기록합니다.

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

``perf`` 로 기록 된 스택 추적은 함수 호출 추적의 일부로 ``bpf_prog_8227addf251b7543()``
심볼을 표시하며. 태그 ``8227addf251b7543`` 을 가진 BPF 프로그램이 ``kfree_skb`` 이벤트
와 관련이 있다는 것을 의미하며, 앞서 말한 프로그램은 tc에 표시된 것처럼 ingress hook의
netdevice ``em1`` 에 연결되었습니다.

Introspection
-------------

리눅스 커널은 BPF 와 XDP 주변의 다양한 tracepoint들을 제공하는데, 이는 추가적인
introspection를 뜻하며, 예를 들어 사용자 영역 프로그램과 bpf 시스템 콜의 대화를
추적 할 수 있습니다

BPF를 위한 추적점:

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

``perf`` 를 사용한 예제 사용법(여기에 사용 된 ``sleep`` 예제 대신 ``tc``
와 같은 특정 응용 프로그램을 사용할 수 있습니다):

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

BPF 프로그램의 경우, 개별 프로그램 태그가 표시됩니다.

디버깅을 위해 XDP에는 예외가 발생할 때 트리거되는 추적점도 있습니다:

::

    # perf list | grep xdp:
    xdp:xdp_exception                                  [Tracepoint event]

예외는 다음 시나리오에서 트리거 됩니다:

* BPF 프로그램은 유효하지 않은/ 알 수없는 XDP 액션 코드를 반환 했습니다.
* BPF 프로그램은 비정상 종료를 나타내는 ``XDP_ABORTED`` 를 반환 했습니다.
* BPF 프로그램은 ``XDP_TX`` 와 함께 반환 되었지만, 예를 들어 포트가 작동
  하지 않거나, transmit ring이 가득 차거나, 할당 실패 로 인해, 전송 시
  오류가 발생했습니다.

두개의 tracepoint class는 BPF 프로그램 자체가 하나 이상의 추적점에 연결
되어 검사될 수 있으며, 예를 들어, ``bpf_perf_event_output()`` helper
함수를 통해 map에 추가 정보를 수집하거나 사용자 영역 수집기에 이러한
이벤트를 punting 할수 있습니다.

Miscellaneous
-------------

BPF 프로그램과 map는 ``RLIMIT_MEMLOCK`` 을 기준으로 한 메모리 이며,
``perf`` 와 비슷합니다. 메모리에 잠길 수 있는 시스템 페이지 단위의
현재 사용 가능한 크기는 ``ulimit -l`` 을 통해 검사 할 수 있습니다.
setrlimit 시스템 콜 man 페이지는 자세한 내용을 제공합니다.

더 복잡한 프로그램이나 큰 BPF map을 로드하는 것에 있어서 기본 한계값
은 충분하지 않으며, 그래서 BPF 시스템 콜은 ``EPERM`` 의 ``errno`` 를
반환활 것 입니다. 이런 상황에서는 ``ulimit -l ulimited`` 혹은 충분히
큰 제한 값을 수행이 될수 있습니다. ``RLMIT_MEMLOCK`` 는 주로 권한이
없는 사용자에게 제한이 적용됩니다. 설정에 따라, 권한있는 사용자에
대해 더 높은 제한을 설정하는 것이 종종 허용 됩니다.

프로그램 유형
=============

이 글을 쓸 당시, 사용 가능한 18 가지의 다른 BPF 프로그램의 유형이 있으며,
네트워킹을 위한 주요 유형 중 두가지는 XDP BPF 프로그램 과 tc BPF 프로그램
에 대해 하위 세션에서 자세히 설명합니다. LLVM, iproute2 또는 기타 도구에
대한 두 가지 프로그램 유형에 대한 광범위한 사용 예는 툴체인 섹션에 전달
되어 있으며 여기서는 다루지 않습니다. 대신이 이 섹션에서는 아키텍처, 개념
및 사용 사례에 중점을 둡니다.

XDP
---

XDP는 eXpress Data Path의 약자이며 리눅스 커널에서 고성능 프로그램 가능한
패킷 처리를 가능하게 하는 BPF를 위한 프레임 워크를 제공합니다.XDP는 BPF
프로그램을 소프트웨어의 가장 빠른 시점, 즉 네트워크 드라이버가 패킷을받는
순간 실행합니다.

고속 경로의 이시점에서, 드라이버는 패킷등을
GRO(generic receive offload)로 푸시 하지 않고, 패킷을 네트워킹 스택 보다
더 위로 올리기 위한 ``skb`` 할당과 같은 비용이 많이 드는 작업을 수행하지
않고, 수신 링에서 패킷을 수신합니다. 따라서, XDP BPF 프로그램은 처리를
위해 CPU에서 사용 할수 있는 가장 빠른 시점에 실행됩니다.

XDP는 리눅스 커널 및 인프라와 함께 동작 하므로, 사용자 영역에서만 작동하는
다양한 네트워크 프레임 워크처럼 커널을 우회하는 동작을 하지 않습니다. 패킷
을 커널 공간에 유지하면 다음과 같은 몇 가지 주요한 이점이 있습니다:


* XDP는 BPF helper 호출 자체에서 라우팅 테이블, 소켓등과 같이 업스트립에서
  개발된 모든 커널 드라이버, 사용자 영역 툴 또는 다른 사용 가능한 커널 내부
  구조를 재사용 할수 있습니다.
* 커널 공간에 존재하는 XDP는 하드웨어에 접근을 위한 커널의 나머지 부분과
  동일한 보안 모델을 가지고 있습니다.
* 처리 된 패킷이 이미 커널에 있기 때문에 컨테이너 혹은 커널의 네트워킹 스택
  자체에서 사용되는 네임 스페이스와 같은 다른 커널 내부 엔터티로 패킷을
  유연하게 전달할 수 있으므로 커널 / 사용자 영역 경계를 넘을 필요가 없습니다.
  이것은 멜트다운과 스펙터의 시기와 관련이 있습니다.
* XDP에서 커널의 robust로 패킷을 퍼팅(punting)하여 광범위하게 사용되는 TCP/IP
  스택을 쉽게 사용할 수 있으며, 완전히 재사용 할 수 있으며 사용자 공간 프레임
  워크와 별도로 TCP/IP 스택을 유지 관리 할 필요가 없습니다.
* BPF를 이용하여 커널의 시스템 call ABI 호출하는 것과 동일한
  'naver-break-user-space' 보장을 통해 안정적인 ABI를 유지하며, 모듈과 비교
  하여 커널 동작의 안정성을 보장하는 BPF verifier 덕분에 안전 장치를 제공하는
  완전한 프로그래밍 기능을 사용 할수 있습니다.
* XDP를 사용하면 네트워크 트래픽 중단이나 커널 / 시스템 재부팅 없이 런타임에
  프로그램을 atomically하게 스왑할 수 있습니다.
* XDP를 사용하면 커널에 통합된 워크로드를 유연한 구조화 할 수 있습니다. 예를
  들어, "busy polling" 또는 "interrupt driven" 모드에서 작동 할 수 있습니다.
  명시 적으로 CPU를 XDP 전용으로 지정할 필요는 없습니다. 특별한 하드웨어 요구
  사항은 없으며, hugepages에 의존하지 않습니다.
* XDP에는 타사 커널 모듈이나 라이센스가 필요하지 않습니다. 이것은 장기적인
  아키텍처 솔루션으로 리눅스 커널의 핵심 부분이며 커널 커뮤니티에 의해 개발
  되었습니다.
* XDP는 4.8 이상에 해당하는 커널을 실행하는 주요 배포판을 통해 모든 곳에서
  이미 활성화되고 제공되며 대부분의 주요 10G 이상의 네트워킹 드라이버를 지원
  합니다.

드라이버에서 BPF를 실행하기 위한 프레임워크로서, XDP는 패킷이 선형적으로 배치
되어 BPF 프로그램이 읽고 쓸 수 있는 단일 DMA의 페이지에 맞도록 보장합니다.
또한 XDP는 ``bpf_xdp_adjust_head()`` BPF helper 함수를 사용하여,
사용자 정의 캡슐화 헤더를 구현하거나 ``bpf_xdp_adjust_meta()`` 를 통해 패킷
앞에 사용자 정의 메타 데이터를 추가하는 프로그램에서 256 바이트의 추가 헤드
룸을 사용할 수 있도록합니다.

프레임워크에서 BPF 프로그램이 드라이버에게 패킷 진행 방법을 지시하기 위해
반환 할 수 있는 XDP 액션 코드가 아래의 절에 자세히 설명되어 있으며, XDP
계층에서 실행되는 BPF 프로그램을 atomically 하게 바꿀 수 있습니다. XDP는
고성능을 위해 디자인 되었습니다. BPF는 '직접 패킷 액세스
(direct packet access)'를 통해 패킷 데이터에 액세스 할 수 있으며,  이는
프로그램이 레지스터에서 직접 데이터 포인터를 보유하고, 내용을 레지스터에
로드하고, 레지스터로 부터 패킷으로 씁니다.

BPF 컨텍스트로 BPF 프로그램에 전달 된 XDP의 패킷 표현은 다음과 같습니다:

::

    struct xdp_buff {
        void *data;
        void *data_end;
        void *data_meta;
        void *data_hard_start;
        struct xdp_rxq_info *rxq;
    };

``data`` 는 페이지의 패킷 데이터의 시작을 가르키며, 이름에서 알 수 있듯이
``data_end`` 는 패킷 데이터의 끝을 가르킵니다. XDP는 헤드 룸을 허용하기
때문에 ``data_hard_start`` 는 페이지에서 가능한 최대 헤드 룸 시작을 가리
키며, 즉, 패킷을 캡슐화 해야 할 때 ``data`` 가 ``bpf_xdp_adjust_head()`` 를
통해 ``data_hard_start`` 쪽으로 더 가까이 이동합니다. 같은 BPF helper 함수는
``data`` 를 ``data_hard_start`` 에서 더 멀리 이동시키는 경우 역캡슐화를 허용
합니다.

``data_meta`` 는 처음에는 ``data`` 와 같은 위치를 가리키고 ``bpf_xdp_adjust_meta()``
는 일반적인 커널 네트워킹 스택에서는 볼 수 없는 사용자 정의 메타 데이터 공간을 제공
하기 위해 포인터를 ``data_hard_start`` 방향으로 이동할 수 있지만, tc BPF 프로그램은
XDP에서 ``skb`` 로 전송되기 때문에 읽을 수 있습니다. 그 반대의 경우에는
``data_hard_start`` 에서 ``data_meta`` 를 다시 이동하여, 동일한 BPF helper 함수를
통해, 사용자 정의 메타 데이터의 크기를 제거하거나 줄일 수 있습니다. ``data_meta`` 는
tc BPF 프로그램에서 액세스 할 수있는 ``skb->cb[]`` 제어 블럭의 경우와 유사하게 tail
호출 간에 상태를 전달하는 용도로만 사용할 수도 있습니다.

이것은 ``struct xdp_buff`` 패킷포인터에 대해 각가 다음과 같은 관계를 갖습니다:
``data_hard_start`` <= ``data_meta`` <= ``data`` < ``data_end``.

``rxq`` 필드는 링 설정 시간(XDP 런타임이 아님)에 채워지는 수신 큐 메타 데이터 당
몇 가지 추가 정보를 가리 킵니다:

::

    struct xdp_rxq_info {
        struct net_device *dev;
        u32 queue_index;
        u32 reg_state;
    } ____cacheline_aligned;

BPF 프로그램은 ``ifindex`` 와 같은 netdevice 자체에서 추가 데이터 뿐만 아니라,
``queue_index`` 를 검색 할 수 있습니다.

**BPF program return codes**

XDP BPF 프로그램을 실행 한 후 드라이버에게 다음 패킷 처리 방법을 알리기 위해 프로
그램에서 결정이 반환됩니다. ``linux/bpf.h`` 시스템 헤더 파일에서 사용 가능한 모든
반환 결과가 나열됩니다:

::

    enum xdp_action {
        XDP_ABORTED = 0,
        XDP_DROP,
        XDP_PASS,
        XDP_TX,
        XDP_REDIRECT,
    };

이름에서 알 수 있듯이 ``XDP_DROP`` 은 추가 리소스를 낭비하지 않고 드라이버 수준
에서 바로 패킷을 삭제합니다. 이것은 특히 DDoS 완화 메커니즘 또는 방화벽을 일반적
으로 구현하는 BPF 프로그램에 유용합니다. ``XDP_PASS`` 리턴 코드는 패킷이 커널의
네트워킹 스택에 전달 될 수 있음을 의미합니다. 이 의미는 이 패킷을 처리하고 있던
현재 CPU는 이제 ``skb`` 를 할당하고 채우고 GRO 엔진으로 전달합니다. 이것은 XDP가
없는 기본 패킷 처리 동작과 동일합니다. ``XDP_TX`` 를 사용하면 BPF 프로그램은 방금
도착한 동일한 NIC에서 네트워크 패킷을 전송할 수 있는 효율적인 옵션을 제공합니다.
이는 일반적으로 클러스터에서 후속 로드 밸런싱을 사용하는 방화벽과 같이, XDP BFP에
서 재작성 후 들어오는 패킷을 다시 스위치로 밀어 넣는 헤어핀(loopback) 로드 밸런서
의 역활을 하는 몇 개의 노드가 구현되는 경우에 유용합니다. ``XDP_REDIRECT`` 는 XDP
패킷을 전송할 수 있지만 다른 NIC를 통해 전송할 수 있다는 점에서 ``XDP_TX`` 와 유사
합니다. ``XDP_REDIRECT`` 의 또 다른 옵션은 BPF cpumap으로 리디렉션하는 것이며, 이
의미는, NIC의 수신 대기열에서 XDP를 제공하는 CPU는 계속해서 상위 커널 스택을 처리
하기 위해 패킷을 원격 CPU로 푸시 할 수 있습니다. 이는 ``XDP_PASS`` 와 유사하지만
XDP BPF 프로그램이 상위 계층으로 푸시하기 위해, 일시적으로 현재 패킷에 대한 작업을
수행하는 것과 달리 들어오는 높은 부하를 계속 서비스 할 수있는 기능을 갖추고 있습니다.
마지막으로 프로그램에서 상태와 같은 예외를 표시하는 역할을 하는 ``XDP_ABORTED`` 가
있으며, ``XDP_DROP`` 와 동일한 동작을하며, ``XDP_ABORTED`` 는 추가로 잘못 감지 된
``trace_xdp_exception tracepoint`` 을 전달합니다.

**XDP 사용 사례**

XDP의 주요 사용 사례 중 일부 이 하위 섹션에 나와 있습니다. 이 목록은 포괄적인 것은
아니며 XDP 및 BPF가 프로그래밍 할 수있는 효율성과 효율성을 고려할 때 매우 구체적인
사용 사례를 해결하기 위해 쉽게 적용 할 수 있습니다.

* **DDoS 완화, 방화벽 기능**

  기본적인 XDP BPF 기능 중 하나이며 드라이버가 초기 단계에서 ``XDP_DROP`` 를 사용하
  여 패킷을 drop하도록 지시하는 것으로 패킷 당 비용이 매우 낮아 모든 종류의 효율적인
  네트워크 정책 시행을 허용합니다. 이것은 모든 종류의 DDoS 공격애 대처할 필요가 있을
  때 이상적이며, BPF에서 오버레드가 거의 없는 모든 종류의 방화벽 정책을 구현 할 수
  있으며, 예를 들어 독립형 어플라이언스 (예를 들어, ``XDP_TX`` 를 통한 'clean' 트래픽
  제거) 또는 마지막 호스트 자체를 보호하는 노드 (``XDP_PASS`` 또는 cpumap ``XDP_REDIRECT``
  를 통한 양호한 트래픽)에 널리 배치 됩니다. 오프로드 된 XDP는 라인 속도로 처리하면서,
  이미 패킷 당 비용을 전체적으로 NIC로 완전히 옮김으로써 한 단계 더 나아갑니다.

..

* **포워딩 및 로드밸런싱**

  XDP의 또 다른 주요 사용 사례는 ``XDP_TX`` 또는 ``XDP_REDIRECT`` 동작을 통한 패킷 전달
  및 로드 밸런싱 입니다. 패킷은 XDP 계층에서 실행되는 BPF 프로그램에 의해 중간에 mangle
  될수 있으며, BPF 도우미 함수 기능은 패킷을 캡슐화 해제하고 다시 캡슐화 하기 전에 캡슐
  화하기 위해패킷의 headroom을 늘리거나 줄일 수 있습니다. 원래 도착한 동일한 네트워크
  장치에서 패킷을 푸시 하는 ``XDP_TX`` hairpinned(혹은 loopback) 로드 밸런서를 사용하거나
  ``XDP_REDIRECT`` 동작을 통해 전송을 위해 다른 NIC로 전달할 수 있습니다. 이후의 리턴
  코드는 BPF의 cpumap과 함께  로컬 스택을 통과 하기위한 패킷을 로드 밸런싱하기 위해 사용
  될 수 있지만 원격의 non-XDP 프로세싱 CPU에서는 사용할 수 없습니다

..

* **사전 스택 필터링 / 처리**

  정책 적용 외에도 XDP는 ``XDP_DROP`` 의 경우, 커널의 네트워크 스택을 강화하는 데 사용
  할 수 있으며, 이 의미는 네트워크 스택이 알기 전에 로컬 노트에 대해 관련성 없는 패킷을
  가능한 빨리 삭제 할 수 있으며, 예를 들어 노드가 TCP 트래픽 만 처리하는 것을 알면,
  모든 UDP, SCTP 또는 기타 L4 트래픽을 버릴 수 있습니다. 이것은 패킷이 GRO 엔진 과 같은
  커널의 흐름 분석기 및 다른 것들을 가로지를  필요가 없이  패킷을 버릴수 있기 때문에
  커널의 공격 영역을 줄일 수 있다는 이점이 있습니다. XDP의 조기 처리 단계 덕분에,
  이것은 효과적으로 커널의 네트워킹 스택을 가장하며, 이러한 패킷은 네트워킹 장치에서
  확인된적은 없습니다. 또한 스택의 수신 경로에 잠재적 인 버그가 발견되어 'ping of death'
  같은 시나리오가 발생하면 XDP를 사용하여 커널을 재부팅하거나 서비스를 다시 시작하지
  않고도 해당 패킷을 즉시 삭제할 수 있습니다. 이러한 프로그램을 atomically으로 스왑
  하여 불량 패킷을 버리는 기능으로 인해 호스트에서 네트워크 트래픽이 중단되지 않습니다.

  사전 스택 처리를 위한 또 다른 사용 사례는 커널이 아직 패킷에 대한 ``skb`` 를 할당하지
  않은 경우, BPF 프로그램은 패킷을 수정 할 수 있으며, 이 방법으로 네트워크 장비로 부터
  스택에 수신된 패킷 인척 합니다. 이렇게하면 GRO 는 사용자 정의 프로토콜를 인식 못하기
  때문에 어떠한 종료의 통계도 수행 할수 없으며, GRO aggregation을 들어가기 전에 패킷을
  캡슐화 해제할 수있는 사용자 정의  패킷 맹 글링 및 캡슐화 프로토콜이 있는 경우가 허용
  됩니다. 이는 일반적인 커널 스택에 '보이지' 않으며, GRO(메타 데이터 매칭을 위해)를
  집계 하고, 나중에 예를 들어 사용할 수있는 다양한 skb 필드를 설정 하여, ``skb``
  컨텍스트를 갖는 tc ingress BPF 프로그램과 함께 처리 할 수 있습니다.

..

* **flow 샘플링, 모니터링**

  또한 XDP는 패킷 모니터링, 샘플링 또는 기타 네트워크 분석과 같은 경우에 사용할 수
  있으며, 예를 들어 경로의 중간 노드 또는 끝 호스트에서 앞서 언급 한 사용 사례와
  함께 사용할 수 있습니다.복잡한 패킷 분석을 위해 XDP는 네트워크 패킷 (생략된 또는
  전체 페이로드 포함) 및 사용자 지정 메타 데이터를  Linux perf 인프라에서 사용자
  공간 응용 프로그램으로 제공되는 CPU 메모리 매핑 링 버퍼 당 빠른 lock없이 신속하게
  푸시 할 수있는 기능을 제공합니다. 이는 또한 흐름의 초기 데이터 만 분석하고 모니터링
  을 우회하는 양호한 트래픽으로 판단되는 경우도 허용합니다. BPF가 제공하는 유연성으로
  인해 사용자 정의 모니터링이나 샘플링을 구현할 수 있습니다.

..

프로덕션 사용의 한 예로 L4로드 균형 조정 및 DDoS 대응을 구현하는 Facebook SHIV및
Droplet infrastructure 가 있습니다. 프로덕션 인프라를 Netfilter의 IPVS (IP Virtual
Server)에서 XDP BPF로 마이그레이션하면 이전 IPVS 설정에 비해 속도가 10 배 빨라졌습
니다. 이것은 netdev 2.1 컨퍼런스에서 처음 발표되었습니다:

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

**XDP 동작 모드**

XDP에는 세 가지 작동 모드가 있습니다. 'Native' XDP가 기본 모드입니다.
XDP에 관해 이야기 할 때 일반적으로 이 모드가 암시됩니다.



* **Native XDP**

  이 모드는 XDP BPF 프로그램이 네트워킹 드라이버의 초기 수신 경로에서
  직접 실행되는 기본 모드입니다. 10G 이상을 위해 널리 사용되는 NIC는
  이미 Native XDP를 지원합니다.

..

* **오프로드된 XDP**

  오프로드 XDP 모드에서 XDP BPF 프로그램은 호스트 CPU에서 실행되는 대신에
  NIC로 직접 오프로드됩니다. 따라서 패킷 당 비용이 매우 낮아 호스트 CPU에서
  완전히 푸시되고 NIC에서 실행되므로 Native XDP에서 실행하는 것 보다 훨씬
  높은 성능을 제공합니다. 이 오프로드는 일반적으로 커널 내 JIT 컴파일러가
  BPF를 후자의 기본 명령어로 변환하는 멀티 스레드, 멀티 코어 플로우 프로세서
  를 포함하는 SmartNIC에 의해 구현됩니다. 또한 오프로드된 XDP를 지원하는
  드라이버는 일부 BPF helper가 아직 또는 native 모드에서만 사용할 수없는 경우
  에 대비해 Native XDP를 지원합니다.

..

* **Generic XDP**

  native 또는 오프로드된 XDP를 아직 구현되지 않는 드라이버의 경우, 커널은
  네트워킹 스택에서 많이 늦는 시점에서 실행되는 드라이버 변경이 필요없는
  generic XDP 옵션을 제공합니다. 이 설정은 주로 커널의 XDP API에 대해
  프로그램을 작성 및 테스트하고 원시 또는 오프로드된 모드의 성능 속도로
  동작하지 않는 개발자를 대상으로합니다. 프로덕션 환경에서의 XDP 사용의
  경우 native 또는 오프로드된 모드가 더 적합하며 XDP를 실행하는 데
  권장되는 방법입니다.

..

**Driver support**

BPF와 XDP는 기능 및 드라이버 지원 측면에서 빠르게 진화하고 있기 때문에
다음은 커널 4.17에서의 네이티브 및 오프로드 된 XDP 드라이버 목록입니다.

**native XDP를 지원하는 드라이버**

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

XDP 프로그램 작성 및 로드 예제는 해당 도구 아래의 툴체인 섹션에 각각 도구
에 포함되어 있습니다.

.. [1] sfc에 대한 XDP 지원은 커널 4.17에서 트리 드라이버를 통해 사용할 수
   있지만 곧 업스트림에 반영 예정에 있습니다.
.. [2] 현재 CPU 번호 검색과 같은 일부 BPF helper 기능은 오프로드 된 설정
   에서 사용할 수 없습니다.


tc (트래픽 제어)
----------------

XDP와 같은 다른 프로그램 유형 과는 별도로 BPF는 네트워킹 데이터 경로의 커널
tc(트래픽 제어) 계층에서도 사용할 수 있습니다. 상위 수준에서 XDP BPF 프로그램
과 tc BPF 프로그램을 비교할 때 세 가지 주요 차이점이 있습니다:

* BPF 입력 컨텍스트는 ``xdp_buff`` 가 아닌 ``sk_buff`` 입니다. 커널의 네트워킹
  스택이 패킷을 받으면 XDP 계층 다음에 버퍼를 할당하고 패킷을 구문 분석하여
  패킷에 대한 메타 데이터를 저장합니다. 이 표현을 ``sk_buff`` 라고합니다. 이후
  이 구조는 BPF 입력 컨텍스트에서 노출되므로 tc ingress 레이어의 BPF 프로그램이
  스택에서 패킷에서 추출한 메타 데이터를 사용할 수 있습니다.이것은 유용 할 수
  있지만,이러한  할당 및 메타 데이터 추출을 수행하는 스택의 관련 비용과 함께
  tc hook에 도달 할 때까지 패킷을 처리합니다. 정의에 따르면 ``xdp_buff`` 는 이
  작업이 완료되기 전에, XDP hook가 호출되기 때문에 이 메타 데이터에 액세스
  할 수 없습니다. 이는 XDP와 tc hook 간의 성능 차이에 크게 관여됩니다.

  따라서 tc BPF hook에 연결 된 BPF 프로그램은 예를 들어, 읽기 혹은 쓰기 skb의 ``mark``
  , ``pkt_type``, ``protocol``, ``priority``, ``queue_mapping``, ``napi_id``, ``cb[]`` 배열,
  ``hash``, ``tc_classid`` 혹은 ``tc_index`` , vlan 메타데이터, XDP로 전송 된 사용자
  정의 메타 데이터 및 다양한 다른 정보를 제공 할 수 있습니다. tc BPF에서 사용되는
  ``struct __sk_buff`` BPF 컨텍스트의 모든 멤버는 ``linux/bpf.h`` 시스템 헤더에 정의되어
  있습니다.

  일반적으로 ``sk_buff`` 는 장점과 단점이 모두 있는 ``xdp_buff`` 와 완전히 다릅니다.
  예를 들어 ``sk_buff`` 의 경우에는 연관된 메타 데이터를 mangle하는 것이 훨씬 간단하지만,
  많은 프로토콜 관련 정보 (예 : GSO 관련 상태) 를 가진 sk_buff가 단순히 패킷 데이터를
  다시 작성하여 프로토콜을 간단하게 전환하는 되는 것은 어렵습니다. 이것은 매번 패킷
  내용에 액세스하는 비용이 들지 않고 메타 데이터를 기반으로 패킷을 처리하는 스택 때문
  입니다. 따라서 ``sk_buff`` 내부가 올바르게 변환되도록 주의하면서, BPF helper 함수에서
  추가 변환이 필요합니다. 그러나 xdp_buff의 경우 커널이 아직 sk_buff를 할당하지 않은
  처음 단계에 있기 때문에 이러한 문제는 발생하지 않으므로 모든 종류의 패킷 재 작성을
  쉽게 실현할 수 있습니다. 그러나 ``xdp_buff`` 의 경우에는이 단계에서 mangling에
  sk_buff 메타 데이터를 사용할 수 없다는 단점이 있습니다.후자는 XDP BPF에서 tc BPF로
  사용자 지정 메타 데이터를 전달함으로써 해결이 됩니다. 이러한 방식으로, 각 프로그램
  유형의 한계는 사용 사례를 요구하는 두 유형의 보완 프로그램을 동작>하는 것으로 해결
  될 수 있습니다.

..

* XDP와 비교하여 네트워킹 데이터 경로의 ingress 및 egress 지점에서 tc BPF 프로그램이
  트리거 될 수 있으며, XDP의 경우는 반대로 ingress 만 가능합니다.

  커널에서 ``sch_handle_ingress()`` 및 ``sch_handle_egress()`` 라는 두 개의 hook 지점
  은 각각 ``__netif_receive_skb_core()`` 및 ``__dev_queue_xmit()`` 에서 트리거 됩니다.
  후자의 두 가지는 데이터 경로의 주요 수신 및 전송 기능으로, XDP를 별도로 설정하면
  노드에서 들어오고 나가는 모든 네트워크 패킷에 대해 트리거되어 이러한 hook 지점에서
  tc BPF 프로그램을 완벽하게 볼 수 있습니다.

..

* tc BPF 프로그램은 네트워킹 스택의 일반 레이어에서 연결 지점에서 실행 되므로 드라이버
  변경이 필요하지 않습니다. 따라서 모든 유형의 네트워킹 장치에 연결할 수 있습니다.

  이는 유연성을 제공하지만 네이티브 XDP 계층에서 실행하는 것과 비교하여 성능을
  저하 시킵니다. 그러나 tc BPF 프로그램은 GRO이 실행 된 후 모든 프로토콜 처리,
  기존 iptables 방화벽이 실행되기 **이전** 에 일반 커널의 네트워킹 데이터 경로의
  가장  초기 단계에 있으며, 예를 들어, iptables PREROUTING 또는 nftables ingress hooks
  또는 다른 패킷 처리 와 같은 항목이 발생합니다. 마찬가지로 egress에서 tc BPF
  프로그램은 전송을 위해 패킷을 드라이버에 전달 하기전 마지막 부분에서 실행이 되며
  이 의미는 iptables POSTROUTING과 같은 전통적인 iptables firewall hook **이후**
  패킷을 커널의 GSO 엔진에게 넘겨주기 전 부분을 설명하는내용입니다.

  그러나 드라이버 변결을 요구하는 예외 사항 중 하나는 tc BPF 프로그램을 오프로드
  하는 것이며, 일반적으로 BPF 입력 컨텍스트, helper 기능 및 결정 코드의 차이로 인해
  기능이 다르기 때문에 오프로드된 XDP와 유사한 방식으로 SmartNIC에서 제공합니다.

..

tc 계층에서 실행되는 BPF 프로그램은 ``cls_bpf`` classifier에서 실행됩니다.
tc 용어는 BPF 연결 지점을 "classifier" 로 설명하지만 ``cls_bpf`` 가 수행 할 수
있는 것을 충분하지 않는 표현을 하기 때문에 다소 오해의 소지가 있습니다.
즉, 완전히 프로그래밍 가능한 패킷 프로세서는 ``skb`` 메타 데이터 및 패킷 데이터
를 읽을 수있을뿐만 아니라 임의로 mangle하고 동작 판정으로 tc 처리를 종료 할
수 있습니다. 따라서 ``cls_bpf`` 는 tc BPF 프로그램을 관리하고 실행하는 독립적인
엔티티로 간주 될 수 있습니다.

``cls_bpf`` 는 하나 이상의 tc BPF 프로그램을 보유 할 수 있습니다. Cilium이
``cls_bpf`` 프로그램을 배포하는 경우 ``직접 실행 모드`` 에서 지정된 hook에
대해 단일 프로그램 만 연결합니다. 일반적으로 전통적인 TC 체계에는 classfier와
일치하는 하나 이상의 동작이 연결 된 classfier 와 동작 모듈간에 구분이 있으며,
classfier 에게는 일치가 있는 경우 트리거됩니다. 소프트웨어 데이터 경로에서
tc를 사용하는 현대 세계에서 이 모델은 복잡한 패킷 처리에 대해서는 확장 되지
않습니다. ``cls_bpf`` 에 연결 된 tc BPF 프로그램이 완전히 독립적 인 경우, 파싱
및 액션 프로세스를 효과적으로 단일 단위로 통합 합니다.
``cls_bpf`` 의 ``직접 행동 모드`` 덕분에, 그것은 tc 액션 결정을 반환하고,
처리 파이프 라인을 즉시 종료합니다. 이를 통해 작업의 선형 반복을 피함 으로써
네트워킹 데이터 경로에서 확장 가능한 프로그래밍 가능한 패킷 처리를 구현할 수
있습니다. ``cls_bpf`` 는 이러한 고속 경로가 가능한 tc 계층의 유일한 "classifier"
모듈 입니다.

XDP BPF 프로그램과 마찬가지로 tc BPF 프로그램은 네트워크 트래픽을 방해하거나
서비스를 다시 시작하지 않고 ``cls_bpf`` 를 통해 런타임에 atomically하게 업데이트
될 수 있습니다.

``cls_bpf`` 자체가 연결 될 수 있는 tc ingress 와 egress 훅은 ``sch_clsact`` 라는
preseudo qdisc에 의해 관리됩니다. 이는 ingress 및 egress tc hook를 모두 관리
할 수 있으므로, ingress qdisc의 drop-in 대체 및 적절한 상위 집합 입니다.
``__dev_queue_xmit()`` 에서 tc의 egress hook의 경우 커널의 qdisc root lock 아래
에서 실행되지 않는다고 강조하는 것이 중요합니다. 따라서, tc ingress 및 egress hook
은 고속 경로에서 잠금없는 방식으로 실행됩니다. 두 경우 모두 선점이 비활성화되고
RCU 읽기 측에서 실행이 발생합니다.

일반적으로 egress에는 넷디바이스에 연결된 qdisc가 있으며,예를 들어 ``sch_mq``,
``sch_fq``, ``sch_fq_codel`` 또는 ``sch_htb`` 와 같이 하위클래스 를 포함하는
classical qdisc 인 패킷을 역다중화 판정을 결정하는 패킷 분류 메커니즘이 필요
합니다. 이것은 tc classier를 호출하는 ``tcf_classify()`` 호출에 의해 처리
됩니다.이러한 경우 ``cls_bpf`` 를 연결하여 사용할 수도 있습니다. 이러한 작업은
대부분 qdisc root lock 아래에서 수행 되며 lock 경합의 영향을 받을 수 있습니다.
``sch_clsact`` qdisc의 egress hook는 훨씬 빠르지만, 기존의 egress qdisc와는
완전히 독립적으로 작동합니다. 따라서 ``sch_htb`` 와 같은 경우 ``sch_clsact``
qdisc는 qdisc root lock 외부의 tc BPF를 통해 과도한 패킷 분류를 수행 할 수
있습니다. ``skb-> mark`` 또는 ``skb-> priority`` 를 설정하면 ``sch_htb`` 는
root lock 아래 비용이 많이 발생하는 패킷 분류 없이 평면 매핑 만 필요 하므로
경합이 줄어 듭니다.

오프로드 된 tc BPF 프로그램은 이전 로드 된 BPF 프로그램이 SmartNIC 드라이버
에서 주입되어 NIC에서 기본적으로 실행되는 ``cls_bpf`` 와 함께 ``sch_clsact``
의 경우에 지원됩니다. ``직접 실행 모드`` 에서 작동하는 ``cls_bpf`` 프로그램
만이 오프로드 되도록 지원됩니다. ``cls_bpf`` 는 단일 프로그램의 오프로딩 만
지원 하며 다중 프로그램을 오프로드 할 수 없습니다. 또한 ingress hook만으로
BPF 프로그램을 오프로드 할 수 있습니다.

하나의 ``cls_bpf`` 인스턴스는 여러 tc BPF 프로그램을 내부적으로 가질수 있습
니다. 이 경우 ``TC_ACT_UNSPEC`` 프로그램 리턴 코드는 해당 목록의 다음 tc BPF
프로그램으로 계속 실행됩니다. 그러나 이는 여러 프로그램이 패킷을 반복해서
구문 분석해야 성능이 저하된다는 단점이 있습니다.

**BPF 프로그램 반환 코드들**

tc ingress 및 egress hook는 tc BPF 프로그램이 사용할 수있는 동일한 작업
결정 반환 공유합니다. 이 내용에 대해서는 ``linux/pkt_cls.h`` 시스템 헤더에
정의되어 있습니다 :

::

    #define TC_ACT_UNSPEC         (-1)
    #define TC_ACT_OK               0
    #define TC_ACT_SHOT             2
    #define TC_ACT_STOLEN           4
    #define TC_ACT_REDIRECT         7

두개의 hook에서도 사용되는 시스템 헤더 파일에서 사용 할수 있는 몇가지 액션
``TC_ACT_*`` 결정이 더 있습니다. 그러나 언급한 몇가지 행동은 위의 것과
동일한 의미를 공유합니다. tc BPF 관점에서 의미는 ``TC_ACT_OK`` 와
``TC_ACT_RECLASSIFY`` 는 세 가지 ``TC_ACT_STOLEN``, ``TC_ACT_QUEUED`` 및
``TC_ACT_TRAP`` opcode와 동일한 의미를 가집니다. 따라서 이 경우 두 그룹에
대한 ``TC_ACT_OK`` 및 ``TC_ACT_STOLEN`` 연산 코드만 설명합니다.

결정 코드에 대해서 ``TC_ACT_UNSPEC`` 으로 시작합니다. 이것은 "지정되지 않은 동작"
의 의미를 가지며 다음과 같은 세 가지 경우에 사용됩니다. i) 오프로드 된 tc BPF
프로그램이 연결되고 오프로드 된 프로그램에 대한 ``cls_bpf`` 표현이 ``TC_ACT_UNSPEC``
을 반환 하는 tc ingress hook이 실행되는 경우, ii) 다중 프로그램의 경우 ``cls_bpf``
에있는 다음 tc BPF 프로그램을 계속 진행되는 경우, 후자는 오프로드 된 경우에만
실행되는 다음 tc BPF 프로그램에서 ``TC_ACT_UNSPEC`` 이 계속되는 시점인 i)에서
오프로드 된 tc BPF 프로그램과 함께 작동합니다. 마지막으로 중요한 것은, iii)
``TC_ACT_UNSPEC`` 은 단일 프로그램의 경우에도 커널에 부가적인 부작용없이
``skb`` 계속 진행 하도록 알려 주는 용도로 사용됩니다. ``TC_ACT_UNSPEC`` 은
``TC_ACT_OK`` 결정 코드 와 매우 유사 하며, 즉, ingress 에서 스택의 상위 레이어로
``skb`` 를 전달하거나, egress에서 전송하기 위해 네트워킹 디바이스 드라이버로
내리는 행동을 합니다. ``TC_ACT_OK`` 의 유일한 차이점은 ``TC_ACT_OK`` 가 tc BPF
프로그램 세트의 classid를 기반으로 ``skb->tc_index`` 를 설정 한다는 것입니다.
후자는 BPF 컨텍스트에서 ``skb->tc_classid`` 를 통해 tc BPF 프로그램 자체에서
설정됩니다.

``TC_ACT_SHOT`` 명령은 커널에게 패킷을 버리게 하며, 즉 네트워킹 스택의
상위 레이어는 진입시 ``skb`` 를 전혀 볼 수 없으며 유사하게 패킷은 egress에서
전송을 위해 제출되지 않습니다. ``TC_ACT_SHOT`` 및 ``TC_ACT_STOLEN`` 은
근본적으로 유사하지만 몇 가지 차이점이 있습니다: ``TC_ACT_SHOT`` 는
``kfree_skb()`` 를 통해 ``skb`` 가 해제 되었고, 즉각적인 피드백을 위해
발신자에게 ``NET_XMIT_DROP`` 를 반환하지만 ``TC_ACT_STOLEN`` 은
``consume_skb()`` 를 통해 ``skb`` 를 해제하고 ``NET_XMIT_SUCCESS`` 를 통해
전송이 성공했다고 상위 계층처럼 가장합니다. 따라서 ``kfree_skb()`` 의 흔적을
기록하는 perf 모니터는 ``TC_ACT_STOLEN`` 의 버리는 표시를 보지 않으며,
이 의미는 ``skb`` 가 "사용"되거나 대기하지만 확실히 "삭제"되지 않았기 때문에
입니다.

마지막으로 tc BPF 프로그램에서 사용할 수있는 ``TC_ACT_REDIRECT`` 동작도
있습니다. 이렇게 하면 ``skb`` 를 ``bpf_redirect()`` helper와 함께 동일한
또는 다른 장치의 ingress 또는 egress 경로로 리디렉션 할 수 있습니다.
패킷을 다른 장치의 ingress 또는 egress 방향으로 주입 할 수 있기 때문에
BPF를 사용한 패킷 전달에서 완전한 유연성을 얻을 수 있습니다. 네트워킹 장치
자체가 아닌 대상 네트워킹 장치에 대한 요구 사항이 없으므로, 대상 장치에서
``cls_bpf`` 의 다른 인스턴>스 또는 다른 제한 사항을 실행할 필요가 없습니다.

**tc BPF FAQ**

이 절에는 자주 질문하는 tc BPF 프로그램과 관련된 몇 가지 질문과 대답이 있습니다.

* **질문:** 액션 모듈로 ``act_bpf`` 는 무엇입니까? 여전히 관련이 있습니까?
* **답변:** 그렇지 않습니다. ``cls_bpf`` 와 ``act_bpf`` 는 tc BPF 프로그램에
  대해 동일한 기능을 공유하지만 ``cls_bpf`` 는 ``act_bpf`` 의 적절한 수퍼 세트
  이며, 더 유연합니다. tc가 작동하는 방식은 tc 액션을 tc 분류 자에 연결되어야
  한다는 것입니다. ``cls_bpf`` 와 동일한 유연성을 얻으려면 ``act_bpf`` 를
  ``cls_matchall`` 분류 자에 연결 해야합니다. 이름에서 알 수 있듯이, 이것은
  연결 된 tc 작업 처리를 위해 모든 패킷에서 일치합니다. ``act_bpf`` 의 경우
  ``직접 실행 모드`` 에서 ``cls_bpf`` 를 직접 사용하는 것보다 패킷 처리가
  덜 효율적 입니다. ``act_bpf`` 가 ``cls_bpf`` 또는 ``cls_matchall`` 이 아닌
  다른 분류 자의 설정에서 사용 되면, tc 분류 자의 동작 특성으로 인해 더 나쁜
  성능을 보입니다. 분류자 A가 불일치 하는 경우, 패킷은 분류자 B로 전달되며,
  패킷등을 다시 패싱 되므로, 일반적인 경우에 패킷은 이>러한 일치를 찾아
  ``act_bpf`` 를 실행하기 위해 최악의 경우 N개의 분류기를 통과해야 하는 선형
  처리가 있게 됩니다. 따라서 ``act_bpf`` 는 그다지 중요하지 않습니다. 또한
  ``act_bpf`` 는 ``cls_bpf`` 와 비교하여 tc 오프 로딩 인터페이스를 제공하지
  않습니다.

..

* **질문:** ``직접 행동 모드`` 가 아닌 ``cls_bpf`` 를 사용하는 것이 좋습니까?
* **답변:** 아니요. 답변은 복잡한 처리를 위해 확장 할 수 없다는 점에서 위의 것과
  비슷합니다. tc BPF는 이미 효율적으로 자체적으로 필요한 모든 것을 수행 할 수
  있으므로 ``직접 실행 모드`` 이외의 다른 기능은 필요하지 않습니다.

..

* **질문:** 오프로드 된 ``cls_bpf`` 와 오프로드 된 XDP에서 성능 차이가 있습니까?
* **답변:** 아니요. 둘 다 SmartNIC에 대한 오프로드를 처리하는 커널의 동일한
  컴파일러를 통해 주입이 됩니다. 두 가지 모두의 로딩 메커니즘도 비슷합니다.
  따라서 BPF 프로그램은 NIC에서 기본적으로 실행될 수 있도록 동일한 대상 명령
  세트로 변환됩니다. 두 개의 tc BPF 및 XDP BPF 프로그램 유형은 서로 다른 기능
  세트를 가지므로 사용 사례에 따라 다른 서비스 유형 (예: 오프로드 경우)의
  가용성으로 인해 다른 서비스 유형을 선택할 수 있습니다.

**tc BPF의 사용 사례**

tc BPF 프로그램의 주요 사용 사례 중 일부 내용이 하위 절에 나와 있습니다.
또한 여기에서는 목록이 전체적이지는 않지만, 프로그래밍 가능성 과 효율성을
감안할 때 매우 특정한 사용자 사례를 해결하기 위해 오케스트레이션 시스템에
맞춤화되고 통합될 수 있습니다. XDP 사용 사례가 일부 겹칠 수 있지만 tc BPF와
XDP BPF는 상호 보완적이며 동시에 해결할 주어진 문제에 가장 적합한 두 가지
방법을 주어진 문제를 풀기에 가장 적합하게 동시에 또는 한 번에 사용할 수
있습니다.

* **컨테이너에 대한 정책 시행**

  tc BPF 프로그램이 적합한 한 응용 프로그램은 정책 집행, 사용자 정의 방화벽
  또는 컨테이너 또는 포드에 대한 유사>한 보안 대책을 각각 구현하는 것입니다.
  이전의 경우, 컨테이너 격리는 호스트의 초기 네임 스페이스와 전용 컨테이너의
  네임 스페이스를 연결하는 네트워크 장치가 있는 네트워크 네임 스페이스를
  통해 구현됩니다. veth 인터페이스 쌍의 한쪽 끝은 컨테이너의 네임 스페이스로
  이동 되었지만 다른 쪽 끝은 호스트의 초기 네임 스페이스에 남아 있기 때문에,
  컨테이너로부터의 모든 네트워크 트래픽은 호스트 측면의 tc ingress 및
  egress hook에 연결 가능한 veth 디바이스를 통과 해야 합니다. 컨테이너로
  들어오는 네트워크 트래픽은 호스트를 향한 경로로 전달되지만, 컨테이너에서
  들어오는 네트워크 트래픽은 호스트를 향한 경로로 전달됩니다. 컨테이너로
  들어오는 네트워크 트래픽은 호스트 측면 veth의 egress hook을 통과하지만
  컨테이너에서 오는 네트워크 트래픽은 호스트 측면 veth의 ingress hook을
  통과합니다.

  veth 장치와 같은 가상 장치의 경우, XDP는 커널이 여기의 ``skb`` 에서만
  작동하고 일반 XDP는 복제 된 ``skb`` 와 함께 작동하지 않는 몇 가지 제한
  사항이 있으므로 이 경우 적합하지 않습니다. 후자는 generic XDP hook이
  단순히 우회되는 재전송을 위한 데이터 세그먼트를 유지하기 위해 TCP/IP
  스택에서 많이 사용됩니다. 또한 generic XDP는 전체 ``skb`` 를 선형화 해야
  성능이 크게 저하됩니다. 반면에 tc BPF는 ``skb`` 입력 컨텍스트 케이스를
  전문으로 하므로 더 유연하며, 따라서 generic XDP의 한계에 염두할 필요가
  없습니다.

..

* **포워딩 그리고 로드벨런싱**

  포워딩 및로드 밸런싱 사용 사례는 서버와 클라이언트 간의 트래픽 보다는
  컨테이너 와 컨테이너 사이의 워크로드를 타겟으로 하며, (두 기술이 어느
  경우에도 사용될 수 있지만,) XDP 와 매우 유사합니다. XDP는 ingress 측에서만
  사용할 수 있으므로 tc BPF 프로그램은 특히 egress에서 적용되는 추가 사용
  사례를 허용 하며, 예를 들어 컨테이너 기반 트래픽은 초기 네임 스페이스에서
  BPF를 통해 송신 측에서 이미 NAT된 및 로드 밸런싱 될 수 있으며, 이것은
  컨테이너 자체에 투명하게 이루어 집니다. egress 트래픽은 커널의 네트워킹
  스택의 특성 때문에 ``sk_buff`` 구조체를 기반으로 하므로 패킷 재작성 및
  리디렉션는 tc BPF에서 적합합니다. BPF는 ``bpf_redirect()`` helper 함수를
  활용하여 포워딩 로직를 대신 받아 패킷을 다른 네트워킹 장치의 ingress 또는
  egress 경로로 푸시 할 수 있습니다. 따라서 모든 브리지 형 장치는
  forwarding fabric으로 tc BPF 활용으로 불필요하게 됩니다

..

* **흐름 샘플링, 모니터링**

  XDP의 경우와 마찬가지로 플로우 샘플링 및 모니터링은 BPF 프로그램이 맞춤
  데이터, 전체 또는 절단 된 패킷 내용, >또는 사용자 공간 응용 프로그램에
  이르기까지 푸시 할 수있는 고성능의 잠금없는 CPU 당 메모리 매핑 된
  perf ring 버퍼를 통해 실현 될 수 있습니다. tc BPF 프로그램에서 이것은
  ``bpf_xdp_event_output()`` 과 같은 대표 함수 와 의미를 가진
  ``bpf_skb_event_output()`` BPF help 함수를 통해 실현됩니다. 주어진
  tc BPF 프로그램은 XDP BPF 사용 사례에서 ingress 만 아닌 ingress와
  egress에 연결 될수 있으며, 두 개의 tc 후크는 (일반) 네트워킹 스택의
  가장 낮은 계층에 있으며,이를 통해 특정 노드의 모든 네트워크 트래픽을
  양방향 모니터링 할 수 있습니다. 이것은 tcpdump와 wireshark가 ``skb``
  를 복제할 필요 없으며, 프로그래밍 가능성 측면에서 보다 유연한 cBPF 사용
  사례와 다소 관련될 수 있으며, 예를 들어 BPF는 이미 링 버퍼에 푸시 된
  패킷에 대해 사용자 공간 뿐만 아니라 사용자 annotations까지 모두를
  푸시하는 대신에 커널 내 aggregation을 수행 할 수 있습니다. 후자는 또한
  Cilium에서 많이 사용되는데, 패킷 레이블에 컨테이너 레이블을 연관
  시키려면 패킷 주석을 추가로 주석을 붙여야하며, 다양한 패킷을 처리
  해야하는 이유(정책 위반 등) 때문에 다양한 context을 제공 해야합니다.

..

* **패킷 스케줄러 사전 처리**

  ``sch_clsact`` 의 egress hook 인 ``sch_handle_egress()`` 는 커널의
  qdisc 루트 잠금을 취하기 바로 전에 실행 되며, 따라서 tc BPF 프로그램은
  패킷이 ``sch_htb`` 와 같은 진짜 완전한 qdisc로 전송되기 전에
  heavy lifting packet classification 및 mangling을 동작하는 데 사용될
  수 있습니다. 이러한 ``sch_clsact`` 와 ``sch_htb`` 와 같은 실제 qdisc와의
  상호 작용은 나중에 ``sch_clsact`` 의 egress hook가 잠금을 사용하지 않고
  실행되기 때문에 전송 단계에서 나중에 발생하는 lock contention을 줄일 수
  있습니다.

..

tc BPF뿐만 아니라 XDP BPF 프로그램의 한 구체적인 예는 Cilium입니다.
Cilium은 Docker 및 Kubernetes와 같은 Linux 컨테이너 관리 플랫폼을
사용하여 배포 된 응용 프로그램 서비스 간의 네트워크 연결을 투명하게
보호 하기위한 오픈 소스 소프트웨어이며 Layer 3 및 Layer 7에서 작동
합니다. Cilium의 핵심은 BPF를 사용하여 정책 시행과 로드 밸런싱 및
모니터링을 구현합니다.

* Slides: https://www.slideshare.net/ThomasGraf5/dockercon-2017-cilium-network-and-application-security-with-bpf-and-xdp
* Video: https://youtu.be/ilKlmTDdFgk
* Github: https://github.com/cilium/cilium

**드라이버 지원**

tc BPF 프로그램은 커널의 네트워킹 스택에서 시작되고 드라이버에서
직접 시작되지 않으므로 추가 드라이버 수정이 필요하지 않으므로
모든 네트워킹 장치에서 실행할 수 있습니다. 아래 나열된 유일한
예외는 tc BPF 프로그램을 NIC에 오프 로딩하는 것을 뜻합니다.

**오프로드 된 tc BPF를 지원하는 드라이버**

* **Netronome**

  * nfp [2]_

tc BPF 프로그램을 작성하고 로딩하는 예제는 각 도구 아래의 toolchain
섹션에 포함되어 있습니다.

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
