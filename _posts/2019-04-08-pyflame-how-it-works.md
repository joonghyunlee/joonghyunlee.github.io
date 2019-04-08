---
layout: post
cover: 'assets/images/cover7.jpg'
navigation: True
title: Pyflame - How it works
date: 2019-04-08 12:00:00
tags: python
subclass: 'post tag-test tag-content'
logo: 'assets/images/ghost.png'
categories: python
---

# Pyflame: Uber Engineering's Ptracing Profiler for Python

# Deterministic Profilers
파이썬은 몇가지 빌트인 deterministic profiler들을 제공합니다.
`profile` 이나 `cProfile` 모듈이 바로 그 것입니다.
이러한 프로파일러들은 모두 `sys.settrace()` 함수를 통해 구현됩니다.
이 함수는 함수의 시작과 끝 부분이나 특정 코드의 시작 지점과 같이 분석할 부분에 추적을 위한 함수를 삽입할 수 있게끔 해줍니다.
이 `sys.settrace()` 함수를 통한 프로파일링 기법은 높은 수준의 프로파일링 결과를 얻을 수 있지만 몇 가지 단점이 있습니다.

## High Overhead
이벤트 기반 프로파일링 기법의 가장 큰 단점은 바로 실제 동작을 느리게 만든다는 것입니다.
경우에 따라서 분석할 프로그램이 최대 2배 이상 느려지기도 했습니다.
이러한 성능 부하는 결국 정확한 프로파일링 결과를 얻는데 걸림돌이 됩니다.
특히 아주 짧은 함수가 여러번 호출되는 경우 추적 함수에 의한 부하로 인해 정확한 시계열 데이터를 얻을 수 없게 됩니다.

## Lack of Full Call Stack Information
이벤트 기반 프로파일링 기법의 두 번째 문제는 전체 호출 스택을 얻을 수 없다는 것입니다.
아시다시피 추적 함수는 필요한 부분에만 삽입하기 때문에 전체 호출 스택을 보려면 모든 함수의 호출자를 따라가며 삽입해야 합니다.
만약 분석하고자 하는 모든 함수의 모든 부분마다 추적 함수를 삽입한다고 했을때,
경우에 따라 불필요한 부분의 정보가 많아져 실제 분석에 어려움을 겪기도 합니다.
Python의 `decorator`나 JAVA의 `annotation`과 같이 자주 호출되는 함수가 대표적인 예입니다.

## Lack of Services Written for Profiling
이벤트 기반 프로파일링 기법의 마지막 문제는 프로그램을 분석하기 위해서는 실제 코드를 수정해야 한다는 점입니다.
대부분의 어플리케이션은 이러한 프로파일링을 최소한 Production 환경에서는 염두에 두지 않습니다.
앞서 언급한 성능 부하가 원인입니다.
어렵게 프로파일링을 통해 유의미한 자료를 추출한다고 하더라도 외부로 전달하는 것은 쉽지 않습니다.
가장 기초적으로 파일로 내려보낸다고 하더라도 실제 동작과 무관한 IO 작업을 야기 하며 이는 또다른 부하를 야기 하게 됩니다.
또한 일반적인 개발 사이클 (코드 작성 - 코드 리뷰 - 테스트 - 배포) 과정을 거치게 되면 프로파일링은 더더욱 어려운 일이 됩니다.

# Sampling Profilers
이벤트 기반 프로파일러와 반대되는 개념으로 샘플링 방식 프로파일러가 있습니다.
샘플링 방식 프로파일러의 경우 주기적으로 대상 프로세스를 멈추고 프로파일링 정보 수집을 위한 핸들러를 실행합니다.
이 경우 샘플링 주기를 조절하여 필요에 따라 프로파일링 데이터의 정확도를 조절할 수 있습니다.
그렇다고 샘플링 주기를 무한정 높게 하면 성능 부하가 발생할 것입니다.
<br>
물론 샘플링 방식 프로파일러도 몇 가지 단점이 있습니다.
보통은 Python으로 작성되었다는 것, 그리고 역시 코드 수정이 불가피하다는 것입니다.

# Pyflame to the Rescue

 우리는 Pyflame을 개발하면서 다음과 같은 장점을 유지하려고 노력했습니다.
 - 전체 Python 스택을 수집할 수 있을 것
 - 수집한 데이터를 시각화 할 수 있는 포맷으로 변경할 수 있을 것
 - 낮은 부하를 가질 것
 - 코드 재작성이 필요하지 않을 것

가장 중요한 것은 기존 프로파일러들이 갖는 단점들을 극복하는 것입니다.
아무런 희생 없이 위에 열거한 기능을 갖추는 것은 불가능해 보일지도 모릅니다.
그러나 실제로 그렇게까지 불가능한 것은 아니었습니다.

# Using `ptrace` for Python Profiling
대부분의 Unix 호환 시스템들은 `ptrace`라고 불리는 특수한 시스템콜을 제공합니다.
이 시스템콜은 특정 프로세스의 가상 메모리 영역이나 esp와 같은 CPU 레지스터들을 읽고 쓸 수 있도록 해줍니다.
오랫동안 디버깅 도구로 사용된 GDB 역시 바로 `ptrace`를 통해 구현되었습니다.


이 `ptrace`를 통해 Python profiler를 사용하는 것은 간단한 일입니다.
기본 Idea는 다음과 같습니다.
  - 주기적으로 대상 프로세스에 'attach' 한다 (PTRACE_ATTACH)
  - 프로세스의 가상 메모리 영역을 훔쳐보아 Python stack을 얻어온다 (PTRACE_PEEKDATA)
  - 프로세스에서 `detach` 한다. (PTRACE_DETACH)

이론적으로 위 방식은 매우 직관적입니다만 
메모리 훔쳐보기를 통해 얻어진 데이터를 통해 Python stack을 복구해낸다는 것은 매우 어렵습니다.
왜냐하면 PTRACE_PEEKDATA를 얻어온 데이터는 단순히 HEX 데이터에 불과하기 때문입니다.

먼저 PTRACE_PEEKDATA가 어떻게 동작하는지를 살펴보겠습니다.
`ptrace` 함수의 원형은 다음과 같습니다.

```c++
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
```

| Parameter | Value |
|--|--|
| request | PTRACE_PEEKDATA |
| pid | 대상 프로세스 PID |
| addr | 훔쳐볼 메모리 주소 |
| data | Unused (NULL) |



#  References
https://eng.uber.com/pyflame/
