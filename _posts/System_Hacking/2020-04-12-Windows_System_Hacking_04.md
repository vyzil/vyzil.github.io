---
title: 윈도우 시스템 해킹 - UAF
category:
    - System Hacking
tags:
    - Windows
    - System
    - Hacking
---
[윈도우 시스템 해킹 가이드 / 김현민 저] <br/>
위의 책을 보면서 실습한 내용을 정리함. <u>실습 환경과 동일하지 않아 Exploit을 제대로 성공시키지 못해서 진행했던 과정들만 정리함.</u>

이번 내용은 어느정도 기반지식이 필요한 것 같아서 아래와 같은 추가적인 자료를 찾아보았음. 특히 웹은 아예 배운적이 없기 때문에 기초부터 공부함.

- Html & CSS & JavaScript, [생활코딩](https://opentutorials.org/course/1)
- Heap Feng Shui in JavaScript, [Black Hat](https://www.blackhat.com/presentations/bh-europe-07/Sotirov/Presentation/bh-eu-07-sotirov-apr19.pdf)
- WinDBG & Mona & Web Browser Exploit, [Hackability](https://hackability.kr/entry/%EC%9D%B5%EC%8A%A4%ED%94%8C%EB%A1%9C%EC%9E%87-%EA%B0%9C%EB%B0%9C-01-WinDbg?category=726281)

마지막 링크는 사실 WinDBG 사용법정도만 참고하였음.



---
1. Memory Violatoin Check
2. call stack을 확인 - `k`
3. Violation된 메모리부분을 추적 - `!heap -p -a 0x0f0ecffa`
4. 현재 EIP 위치 디스어셈블 - `u`

2번에서 `formobj.reset()` 부분에서 에러가 발생함을 짐작가능.

3번에서 해당 메모리 부분은 Free되었음을 확인 가능 -> 즉, Free 된 메모리에 접근함

=> 2번의 결과를 확인해 보면, mshtml부분에서 violation이 일어났으므로 IDA로 이 부분을 확인하고자함.

---

mshtml.dll을 host로 가져와서 IDA로 분석함. 가장먼저, `COptionElement::IsSelectable`을 확인.

```c
BOOL __thiscall COptionElement::IsSelectable(COptionElement *this)
{
  return !(*((_BYTE *)this + 50) & 4) && (*(int (**)(void))(*(_DWORD *)this + 508))();
}
```

이 함수를 호출하는 부분이 어디인지 확인하기 위해 함수명을 클릭하고 `X`키를 눌러 참조하는 부분을 확인하자.

물론, 함수호출 부분에서 확인했던 직전함수인 `CSelectElement::GetFirstSelectableOptionIndex` 부분을 확인할 것이다.

확인해보면 Select객체 + 0x68부터 객체의 주소가 저장되어 있고, 하나씩 가져와서 `IsSelectable`함수로 검사를 하는 방식. 따라서 일단 IsSelectable에 breakpoint를 걸어서 확인하자. 책에서는 `bp !mshtml+0xFEB40`으로 하였지만, 나는 주솟값을 어떻게 찾는지 잘 몰라서 `bp mshtml!COptionElement::IsSelectable`로 bp 설정하였음.

---
이후 책을 따라하였으나 실습 환경과 완전히 동일한 환경을 구축하지 않아서 Exploit에 실패함. 또한, 책에서도 이후 내용은 디테일하게 다루지 않아서 WinDBG로 혼자 분석해보고 원리만 이해하고 넘어갔다.

원리를 이해한대로 설명하자면 다음과 같다. `optobj`를 가리키고 있던 `selobj`의 innerText를 설정함으로써 간접적으로 free 시켜주었으나, `formobj`는 여전히 해당 부분을 가리키고 있게 되어 UAF가 가능한 점을 이용함. 따라서 위의 상황에서 Heap Spray를 통해 shellcode주소와 shellcode를 뿌려주고 vtable을 `fakeobj`로 바꾸어 실행흐름을 바꾸어줌.
