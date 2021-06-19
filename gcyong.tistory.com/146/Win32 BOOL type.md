# Win32 BOOL 타입에 대한 이야기

## 시작하면서
대부분의 현대 프로그래밍 언어에서 제공하는 자료형 중 진위형(boolean)이 있다. 이 자료형의 도메인은 참(true)과 거짓(false)이 있으며, 진위형으로 선언된 변수는 두 값 중 하나를 갖는다.

또한 `if`나 `while`등 조건을 나타내는 표현식에서 평가될 수 있는 자료형이며, 자바나 C#같은 언어에서는 진위형으로 평가되지 않는 표현식은 조건식에 위치할 수 없다.

다만 과거 K&R C에서는 이러한 자료형은 존재하지 않았고, C와 비슷한, 혹은 오래된 역사를 갖는 다른 언어에서는 진위형이 없는 경우도 충분히 많다. 대신, 이러한 언어도 어떤 표현식이 참인지(=유효한지, 조건에 맞는지) 또는 거짓인지(=유효하지 않은지, 조건에 맞지 않는지)에 대한 여부는 존재한다.

가령, K&R C에서는 C++에서 제공하는 `bool` 타입과 같은 별도의 진위형 타입은 제공하지 않지만, `i > 10`과 같은 표현식은 얼마든지 있을 수 있고, 이 표현식은 논리적으로 참(true) 또는 거짓(false)을 의미한다. C++나 C#마냥 이를 `true` 또는 `false`라는 별도의 진위형 값으로 평가하지 않을 뿐이다.

그렇다면, 전통적으로 C에서는 어떻게 참과 거짓을 나타냈을까? C에서는 평가되는 표현식 결과의 모든 비트가 `0`일 경우 거짓을 간주하는 반면, 어느 비트가 하나라도 `0`일 아닐 경우 참으로 간주한다.

```C
#include <stdio.h>

int main(int argc, char* argv[])
{
    if ( 0x00000000 )   // 거짓
    {
        puts("#1");
    }

    if ( 0x00000001 )   // 참
    {
        puts("#2");
    }

    if ( 0x10000000 )   // 참
    {
        puts("#3");
    }

    int count = 5;
    while ( count )     // 참일 동안 (= count가 0으로 평가되지 않을 동안)
    {
        fputc('0' + (count--), stdout);
    }

    return 0;
}
```

위 소스코드의 결과는 다음과 같다.

```
#2
#3
54321
```

## Win32 `BOOL` 타입
위와 같은 이유로, Win32에서는 C에서 사용할 `BOOL` 타입과 `BOOL` 타입의 도메인인 `TRUE`와 `FALSE`를 따로 정의했는데, Windows SDK 10.0.19041.0 에서는 다음과 같이 정의되어 있다.

```C
// minwindef.h
// Windows SDK 10.0.19041.0

// 생략...
#ifndef FALSE
#define FALSE               0
#endif

#ifndef TRUE
#define TRUE                1
#endif
// 중략
typedef int                 BOOL;
// 후략
```

`BOOL`은 `int` 의 별칭으로 타입 선언을 했으며, `FALSE`는 `0`으로, `TRUE`는 `1`로 치환되게끔 매크로 정의를 했다.

아주 직관적인데, 현재 사실상 C를 버렸다고도 볼 수 있는 현재로서는 C++의 `bool`, `true`, `false`로 정의할 법도 하지만 과거 호환성을 고려하여 이 정의를 유지했다고 볼 수 있다.

수많은 Win32 함수들이 `BOOL` 타입과 `TRUE`, `FALSE` 매크로를 사용하고 있는데, 대표적으로 윈도우의 위치나 크기 등을 조절할 수 있는 `SetWindowPos` 함수가 있다.

```C
// SetWindowPos function
// https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowpos

BOOL SetWindowPos(
  HWND hWnd,
  HWND hWndInsertAfter,
  int  X,
  int  Y,
  int  cx,
  int  cy,
  UINT uFlags
);
```

이 함수는 `hWnd` 윈도우 핸들을 가지고 크기나 위치, `hWndInsertAfter`를 참고하여 Z-Order를 조절하되, `uFlags`로 조절하는 범위를 제한할 수 있다. (크기나 위치나 Z-Order만 조절할 수 있다.)

## `BOOL` 타입을 통한 참 또는 거짓 판단
하지만, 얼핏 보면 `SetWindowPos` 함수는 조절에 성공하면 `TRUE`를, 실패하면 `FALSE`를 리턴한다고 볼 수 있는데, 실제 이 함수의 MS Docs를 보면 다음과 같이 설명하고 있다.

>Type: BOOL
>
>If the function succeeds, the return value is nonzero.
>
>If the function fails, the return value is zero. To get extended error information, call GetLastError.

설명이 약간 미묘한데, MS Docs에서는 `TRUE` 또는 `FALSE` 라고 명확히 설명하지 않고, zero 또는 nonzero라는 표현을 사용한다.

zero야 0이고, `FALSE`가 `0`으로 치환되니 별 문제가 안되는데, 문제는 nonzero이다. `TRUE`는 `1`로 치환되고, 1은 nonzero지만, 그렇다고 nonzero와 1이 완전히 같은 것은 아니기 때문이다. nonzero는 0이 아닌 나머지 정수(`BOOL`타입이 `int`의 별칭이므로 `int` 범위에서 0이 아닌 나머지 정수)에 해당하는 집합의 개념(또는 해당 집합에 포함된 임의의 원소)인 반면, `1`은 nonzero 집합의 한 원소일 뿐이다.

MS Docs에서는 `SetWindowPos` 함수 외에도 `BOOL` 타입을 리턴하는 많은 함수에서 이와 같이 zero와 nonzero라는 표현을 자주 사용하는데, 이렇게 표현하는 이유는 다음과 같은 이유 때문일 것으로 추측해본다. (사실, 방어적인 표현으로 '추측'이라는 단어를 사용했지 본인은 거의 '확신'하고 있는 이유다.)

 1. C/C++에서 참을 의미하는 값이 정수 중에서는 0이 아닌 정수이기 때문이다. 이는 상기한 사항과 일치하며, MS Docs에서도 C/C++의 이러한 성질을 염두해두고 문서를 작성했다면 nonzero라고 충분히 말할 만 하다.
 2. `BOOL` 타입이 `int`라서 `TRUE`와 `FALSE` 외에도 임의의 정수를 리턴할 수 있기 때문이다. `GetMessage` 함수가 대표적이다. 이 함수는 `BOOL` 타입을 리턴한다고 되어 있지만, 실제 리턴되는 값은 세 종류이며 (0, -1, 0과 -1이 아닌 정수), 세 종류의 값에 따라 다른 의미를 갖는다.

이러한 이유로, `BOOL` 값을 리턴받아 사용할 때는 주의를 기울여야 한다. 최소한 MS Docs 문서를 참고하여 정말 진위형의 의미로 써도 될지, 쓴다면 `TRUE`와 `FALSE`로 나타낼 수 있는지 등을 면밀히 살펴보고 사용해야 한다.

## `BOOL` 사용에 있어 주의점 하나 : 비교(Comparison)
일부 소스코드에서는 Win32 함수를 사용하는 데 있어 다음과 같은 (불필요한) 비교를 한다.

```C
if ( IsWindow(hWnd) == TRUE )
{
    // hWnd는 윈도우다!
}
else
{
    // hWnd는 윈도우가 아니다!
}
```

`IsWindow`라는 Win32 함수는 `HWND` 타입의 윈도우 핸들을 하나 받는데, 이 윈도우 핸들이 올바른 윈도우를 나타내는 핸들이라면 참을 리턴할 것이고, 그렇지 않다면 거짓을 리턴할 것이다.

그런데 위 소스코드는 이 함수를 사용하여 윈도우 처리를 하고 있는데, 리턴하는 타입이 `BOOL` 이다보니 나름 명확하게 비교한다는 의미에서 `TRUE`와 비교했다.

결론부터 말하자면, 이는 매우 불필요하면서 버그를 만들어 낼 수 있는 위험한 코드일 수 있다. 왜냐하면,

 1. 해당 함수는 리턴 값을 그대로 `if` 조건식에 넣어도 참과 거짓을 나타낼 수 있어 다시한 번 비교하는 것 자체가 연산을 낭비하기 때문이다. 이는 `bool` 타입에서도 마찬가지 이며, `== true` 또는 `== false`도 마찬가지로 낭비이다.
 2. `IsWindow` 함수는 [MS Docs](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-iswindow) 문서를 읽어보면 `SetWindowPos`와 마찬가지로 zero와 nonzero를 리턴한다고 되어있기 때문이다.

첫 번째 이유는 로직상의 문제(버그)는 아니고, 효율성의 문제라고 할 수 있다. 이미 참 또는 거짓으로 평가될 수 있는 값을 리턴했는데, 이를 굳이 한번 더 비교연산을 하는 것 자체가 비효율적이면서 어색하기까지 하다.

두 번째 이유가 이번에 말하고 싶은 이유인데, `IsWindow` 함수는 `BOOL` 타입의 값을 리턴한다. 그런데 이는 결국 `int` 타입의 값과 도메인이 동일하기 때문에 이러한 상등 비교는 결국 **정수 타입에 대한 일치 비교** 가 된다. 즉, `TRUE`는 `1`이라는 특정 값으로 치환되는 반면, `IsWindow` 함수는 0이 아닌 어떤 정수를 리턴하기 때문에 둘 다 참/거짓을 평가하는 수식에서는 모두 참으로 평가될 수 있으나, `TRUE`와 `IsWindow`의 리턴값을 비교하는 연산을 추가하면 거짓으로 평가될 수 있다. 즉 둘 다 "참"으로 퉁칠수 있는 상황에 굳이 현미경을 들이밀어서 거짓으로 평가하겠다는 의미라고 할 수 있다.

따라서, C/C++에서 전통적으로 보장하는 참/거짓 수식 평가를 그대로 따르면서 비교 없이 무조건 `BOOL` 타입의 리턴 값으로 나타내던지, 정 비교를 하고 싶다면 (`BOOL` 타입에서 `bool` 타입으로 변환하고 싶다면) `FALSE`와 비교해야 한다. `FALSE`는 `0`이고, zero도 결국 정수 범위에서는 0밖에 없으니 둘은 비교했을 때 항상 같음을 보장할 수 있다.

다만 비교 시 `==` 연산자가 아닌 `!=` 연산자를 사용하는 편이 직관적이다. 왜냐하면, `FALSE`와 `==` 비교를 했을 때 (C++ 표현으로) `true`가 나온다면 실제로는 거짓이고, `false`가 나온다면 실제로는 참이기 때문이다. 즉, 비교되는 값과 정 반대로 나오기 때문에 논리 오류를 일으킬 수 있다.

```C
if ( ::IsWindow(hWnd) != FALSE )
{
    // hWnd는 윈도우다!
}

// 또는

if ( ::IsWindow(hWnd) )
{
    // hWnd는 윈도우다!
}
```

이해를 돕기 위해, 재밌는 상황을 한번 연출해본다. A라는 회사에서 윈도우용 DLL 라이브러리를 만드는데, 다음과 같이 만들었다고 가정하자.

```C
// genius.h
#pragma once

#ifdef DLL_PROJECT
#define EXPORTSPEC __declspec(dllexport)
#else
#define EXPORTSPEC __declspec(dllimport)
#endif

#include <Windows.h>

EXPORTSPEC BOOL AreYouGenius();
```

```C
// genius.c
#define DLL_PROJECT
#include "genius.h"

EXPORTSPEC BOOL AreYouGenius()
{
    return TRUE;
}
```

신개념 차세대 4차혁명 빅데이터 AI 딥러닝을 이용하여 사용자가 천재인지 아닌지 판단해주는 `AreYouGenius`라는 함수를 만들었다고 치자. 이 함수는 호출한 사용자가 천재라면 `TRUE`를 리턴하는데, 너무 신기술인 나머지 미리 `TRUE`를 예측한 듯 싶다.

이제 이 라이브러리를 사용자가 (큰 기대를 갖고) 연동했다고 치자.

```C
#include "genius.h"
#pragma comment(lib, "genius")

#include <stdio.h>

#ifdef TRUE
#undef TRUE
#define TRUE 2 // 모종의 사유로 TRUE가 바뀌었다고 가정
#endif

int main()
{
    puts("Are you genius???");
    if ( AreYouGenius() == TRUE )
    {
        puts("Yes.");
    }
    else
    {
        puts("No.");
    }

    return 0;
}
```

연동을 하긴 했는데, 다른 라이브러리에서 임의로 또는 Win32의 정책 변경 등의 사유로 `TRUE` 매크로가 `1`에서 `2`로 바뀌었다. (Win32에서는 어지간해선 바꾸지 않지만, 어떤 서드파티에서 (이기적이고 막무가내로) 값을 바꿀 가능성은 충분히 생각해 볼 수 있다. 가령 `UNKNOWN`이라는 값을 `1`로 할당하기 위해 `TRUE`를 `2`로 바꾼다거나...)

이렇게 만들고 실제 구동해보면 우리가 원하는 값은 "Yes." 이나, 실제 출력되는 값은 "No." 이다.

![01.png](01.png)

하지만 `AreYouGenius` 함수가 리턴하는 값을 `if`의 조건식에 그대로 평가한다면 분명히 C/C++에서는 참으로 간주하는 것은 자명하기 때문에, 다음처럼 사용한다면 전혀 문제되지 않는다.

```C
int main()
{
    puts("Are you genius???");
    if ( AreYouGenius() ) // 또는 AreYouGenius() != FALSE
    {
        puts("Yes.");
    }
    else
    {
        puts("No.");
    }

    return 0;
}
```

![02.png](02.png)

결론은, `BOOL`로 리턴하는 함수는 비교를 통한 참, 거짓 판정을 자제하되, 굳이 비교를 하고 싶을 경우는 거짓과 다름을 비교하라는 것이다.

## `BOOL` 사용에 있어 주의점 둘 : 논리적으로 정말 진위형인가
위에서도 잠시 언급했지만 `GetMessage` 함수는 리턴 타입이 `BOOL`임에도 실제 참, 거짓으로 판정하기 애매한 제 3의 상태가 존재한다. 다음은 `GetMessage` 함수의 [MS Docs](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getmessage) 내용의 일부이다.

>If the function retrieves a message other than WM_QUIT, the return value is nonzero.
>
>If the function retrieves the WM_QUIT message, the return value is zero.
>
>If there is an error, the return value is -1. For example, the function fails if hWnd is an invalid window handle or lpMsg is an invalid pointer. To get extended error information, call GetLastError.

`GetMessage` 함수는 호출에 성공한다면 nonzero를 리턴하되, `WM_QUIT` 메시지를 받으면 zero를 리턴한다. 따라서 nonzero일 경우는 해당 스레드에서 계속 메시지 펌핑하도록, zero를 리턴할 때 메시지 루프를 탈출하도록 윈도우 개발자들이 흔히 설계한다.

```C
void MessagePumpW()
{
    MSG msg = {0};
    while ( GetMessage(&msg, NULL, 0, 0) )
    {
        TranslateMessage(&msg);
        DispatchMessageW(&msg);
    }
}
```

단, `GetMessage` 함수에서 nonzero는 0 외에도 -1을 제외한 값이며, nonzero와 zero 모두 함수 호출 자체는 성공한 경우이다. 그런데 이 함수는 내부적으로 함수 호출이 실패할 가능성이 존재하여 (메시지 스레드나 큐가 잘못된 경우가 대표적이다.) 실패할 때는 실패했다는 의미의 특정한 값을 리턴해야 호출부에서 어떤 경우인지 구분지을 수 있다.

MS에서는 `GetMessage` 함수를 설계할 때 실패할 경우 -1을 리턴하도록 약속했으며, 이는 [MS Docs](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getmessage)에서도 확인 가능하다.

이를 반영하여 다음과 같이 설계할 수 있다.

```C
void MessagePumpW()
{
    MSG msg = {0};
    int ret = 0;

    while ( TRUE )
    {
        ret = (int)GetMessage(&msg, NULL, 0, 0);

        if(ret == -1)
        {
            // 함수 호출 실패
            // 예외 처리 또는 별도 처리 하지 않고 스킵
            break;
        }
        else if (ret == 0)
        {
            // 함수 호출 성공
            // 단, WM_QUIT 메시지를 수신한 경우이므로 루프 탈출
            break;
        }
        else
        {
            // 함수 호출 성공
            // 일반적인 경우로, 메시지 프로시저로 메시지 전달 또는 일부 메시지 조합은 추가 메시지 생성
            TranslateMessage(&msg);
            DispatchMessageW(&msg);
        }
    }
}
```

`BOOL`이 `int`임에도 굳이 형 변환 하는 이유는 가독성 때문이다. 물론 `BOOL`은 `int`의 별칭이라 형 변환이 의미가 없지만, 그럼에도 `BOOL` 자체는 진위형으로 오해할 수 있어 왜 숫자 비교를 하는지에 대해 다른 작업자가 궁금해 할 수 있다. 그래서 "의도적으로 바꾼거니 태클은 걸지 마시오!" 하기 위해 형 변환을 했다.

여기서 중요한 것은 리턴 타입이 `BOOL`이라고 해도 실제 진위형처럼 참 또는 거짓만 갖는다는 보장은 없다는 것이다. 가령, `GetMessage`가 함수 호출 실패 시 리턴하는 -1이 있다.

그런데 -1은 정수 범위에서 0이 아니기 때문에 nonzero 범위에 들어갈 수 있어, 자칫 오해하여 세부적인 값을 비교하지 않고 `BOOL`을 리턴했으니 진위여부만 따져도 되겠지 하는 생각은 치명적인 오류로 더 이상 진행할 수 없는 경우임에도 계속 진행하여 무한 루프에 빠질 논리적 오류를 수반할 수 있다.

그리고 사실 이런 경우에는 `BOOL` 이라는 타입을 쓰는 것 자체가 가독성에 좋은 것은 아니다. 이를 MS에서도 모르는 것은 아닐 터, 그럼에도 `BOOL`을 리턴 타입으로 정의한 것은 아마 비슷한 기능을 하는 `PeekMessage` 함수와 프로토타입을 비슷하게 맞추려는 의도가 있지 않았나 생각해본다.

여하튼 결론은, `BOOL` 타입을 리턴한다고 해서 반드시 진위 여부만 따지는 것은 위험할 수 있고, MS Docs에 기재된 내용을 참고하여 적절히 판단해야 할 것이다. 정확히 둘로 나뉘어 nonzero가 참을, zero가 거짓을 나타내는 경우에는 리턴 타입이 `BOOL`인 함수의 리턴값을 바로 조건식에 넣어 활용할 수 있다. 하지만 그렇지 않은 경우 세부적으로 나누어 (`int`같은 정수 타입으로 간주하여) 분기를 나누어야 할 것이다.

## 참고 : C99과 stdbool.h
맨 위 문단을 보면 C라고 표현하지 않고 K&R C라고 표현했다. 사실 K&R C는 아주 오래전에 만들어진 C이고, K&R C가 공표된 이후 ANSI, ISO 등에서 추가적인 표준화를 통해 C89 같은 표준이 나오기도 했다.

이 때, 굳이 K&R C라고 표현한 이유는 C99부터 stdbool.h라는 라이브러리로 진위형을 타입으로 제공하고 있기 때문이다.

stdbool.h는 전처리 매크로를 통해 `bool` 타입을 제공하고 있으며, `true`와 `false`는 Win32의 `TRUE`와 `FALSE`와 동일하게 각각 `1`과 `0`으로 매크로 정의되고 있다.

C 역시 C++처럼 정식으로 타입으로 제공하기에는 워낙 오래된 언어이기 때문에 식별자로 쉽게 사용될 수 있는 `bool`과 `true`, `false`를 하루아침에 키워드 또는 예약어로 등록하기에는 이전 코드와의 호환성을 깨트릴 수 있다. 만일 당장 사용하지 않더라도 미리 예약어로 등록해 두었다면 라이브러리가 아닌 정식 타입으로 제공할 가능성도 있겠다.

즉, 과거 수많은 코드에서 흔히 식별자로 사용할 만 하기 때문에 라이브러리로 표준을 도입한 것이다. 필요한 사람은 stdbool.h를 선언해서 사용할 수 있고, 이름 충돌이 발생한다면 stdbool.h를 사용하지 않는 등 선택의 여지를 준 것이다.

## 참고자료
 - https://en.cppreference.com/w/c/language/history
 - https://en.cppreference.com/w/c/language/arithmetic_types#Boolean_type
 - https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getmessage
 - https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-iswindow
 - https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowpos
 - https://en.cppreference.com/w/c/types