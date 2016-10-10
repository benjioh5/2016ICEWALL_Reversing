# 2 주차

일단 1주차 과제부터 살펴보도록하죠. -_-a

1. 코드 엔진 챌린지 레벨 1 - 5 혹은 11 - 15 풀어오기가 과제였습니다 :P
2. 주어진 C 코드 컴파일 해보고, 도대체 어떤식으로 작동되는지 gdb/strace로 확인해보기



이 두 가지를 모두 해오신 분들도 계시지만, 안 그런분들도 계시고, 과제를 한 걸 확인하니 역시나 제 생각대로 배우신 분들도 계시고, 안 그런분들도 계셔서, 전체적인 설명을 처음부터 다시해야하지 않나 싶습니다.

```c
// add.c
#include <stdio.h>

int main()
{
  int a, b, c;
  a = 1;
  b = 2;
  c = a + b;
  printf("%d\n", c);
  return 0;
}
```

일단 add.c의 코드입니다. `gcc -o add add.c` 로 컴파일을 해보고, lldb나 gdb로 디스어셈블을 해봅시다.

``` shell
$ lldb add
(lldb) target create "add"
Current executable set to 'add' (x86_64).
(lldb) disass -n main
add`main:
add[0x100000f40] <+0>:  push   rbp
add[0x100000f41] <+1>:  mov    rbp, rsp
add[0x100000f44] <+4>:  sub    rsp, 0x20
add[0x100000f48] <+8>:  lea    rdi, [rip + 0x57]         ; "%d\n"
add[0x100000f4f] <+15>: mov    dword ptr [rbp - 0x4], 0x0
add[0x100000f56] <+22>: mov    dword ptr [rbp - 0x8], 0x1
add[0x100000f5d] <+29>: mov    dword ptr [rbp - 0xc], 0x2
add[0x100000f64] <+36>: mov    eax, dword ptr [rbp - 0x8]
add[0x100000f67] <+39>: add    eax, dword ptr [rbp - 0xc]
add[0x100000f6a] <+42>: mov    dword ptr [rbp - 0x10], eax
add[0x100000f6d] <+45>: mov    esi, dword ptr [rbp - 0x10]
add[0x100000f70] <+48>: mov    al, 0x0
add[0x100000f72] <+50>: call   0x100000f84               ; symbol stub for: printf
add[0x100000f77] <+55>: xor    esi, esi
add[0x100000f79] <+57>: mov    dword ptr [rbp - 0x14], eax
add[0x100000f7c] <+60>: mov    eax, esi
add[0x100000f7e] <+62>: add    rsp, 0x20
add[0x100000f82] <+66>: pop    rbp
add[0x100000f83] <+67>: ret

(lldb)
```

네, 해괴한 것들이 나오기 시작하네요. -_-; 일단 찬찬히 뜯어보면서, 알아봅시다.

```shell
add[0x100000f40] <+0>:  push   rbp
add[0x100000f41] <+1>:  mov    rbp, rsp
add[0x100000f44] <+4>:  sub    rsp, 0x20
```

함수의 프롤로그입니다. 사실 얘가 얼마나 중요한지에 대해서는 나중에 배우실테고, 간단하게 설명을 하자면, 스택 프레임을 만드는 과정입니다. 컴퓨터 구조 시간에 이거를 자세히 공부할 시간이 있을테니, 궁금하시면 찾아보시길 바랍니다.

```shell
add[0x100000f48] <+8>:  lea    rdi, [rip + 0x57]         ; "%d\n"
```

일단 printf("%d\n", c) 에서 사용할 "%d\n" 이 녀석을 rid 레지스터에 저장하는 모습입니다.

```shell
add[0x100000f4f] <+15>: mov    dword ptr [rbp - 0x4], 0x0 ; v4 = 0
add[0x100000f56] <+22>: mov    dword ptr [rbp - 0x8], 0x1 ; v8 = 1
add[0x100000f5d] <+29>: mov    dword ptr [rbp - 0xc], 0x2 ; vc = 2
add[0x100000f64] <+36>: mov    eax, dword ptr [rbp - 0x8] ; eax = v8
add[0x100000f67] <+39>: add    eax, dword ptr [rbp - 0xc] ; eax += vc
add[0x100000f6a] <+42>: mov    dword ptr [rbp - 0x10], eax ; v10 = eax
```

이 이후는 `a = 1`, `b = 2`, `c = a + b`를 각각 나타냅니다. 일단 rbp (베이스 포인터) 값에서 0x8 만큼 뺀 위치에 1을 저장을 하고, 0xc 만큼 뺀 위치에 2를 저장을 합니다. 뭔 말인지 모르겠다면, 그냥 메모리에 데이터를 저장해야하는데, 그것의 위치를 상대적으로 (rbp와 0x4, 0x8, 0xc, 0x10 등등… 잘 보면 4 단위로 커지는 걸 알 수 있습니다. int형 변수니까요 ㅋㅋ) 나타냈다고만 생각하시면 됩니다. IDA에서 Hexray를 돌리거나 Hopper를 쓰시게 된다면, 일반적으로 v_4, v_8 (variable_rbp-0x8) 형태로 변수명을 명명하는 것을 볼 수 있습니다. 저도 편의상 그렇게 칭하도록 하죠.

일단, 이것을 어셈블리에서 간단한 c 코드로 바꾸도록 해보죠.

```c
v4 = 0;
v8 = 1;
vc = 2;
eax = v8;
eax += vc;
v10 = eax;
```

음?

이제 좀 익숙해보이는 코드가 나왔습니다. 원래 코드와 비슷한 (?) 동작을 하는 녀석이 나왔다는 것을 알 수 있습니다. v8은 아마도 a 인거 같고, vc는 b네요. 근데 왜 불편하게, eax에 v8 값을 대입하고, vc 값을 더하고, eax를 다시 v10(아마도 c겠죠?)에 저장을 할까요? 그건 CPU와 메모리는 독립된 존재이기 때문에 그렇습니다. CPU는 거대한 계산기입니다. 그리고 메모리는 말 그대로 기억을 하는 저장소이죠. CPU에서 eax, ebx 같은 레지스터들을 이용하여 계산을 하고, 계산된 결과를 레지스터에서 메모리에 쓰고, 필요할 때 메모리에서 다시 레지스터로 읽어오는 형태인 것입니다. 아아, 너무 불편하군요. 좀 더 개선된 방법이 있을까요? 뭐 이건 나중에 차차 설명하도록 하죠.

```shell
add[0x100000f6d] <+45>: mov    esi, dword ptr [rbp - 0x10]
add[0x100000f70] <+48>: mov    al, 0x0
add[0x100000f72] <+50>: call   0x100000f84               ; symbol stub for: printf
```

이제 call 명령어를 이용하여 printf() 함수를 호출하는 것을 끝을 맺습니다. printf 에 두 개의 인자를 던져줘야 할텐데, 첫번째 인자인 "%d\n"은 rdi 레지스터에 담겨있고, 두번째 인자인 c는 v10에서 esi 레지스터로 mov 명령어를 통해 넘겨 받습니다. 즉, printf 함수는 edi와 esi 레지스터의 값을 받아 "3\n"을 출력하게 됩니다.

```shell
add[0x100000f77] <+55>: xor    esi, esi
add[0x100000f79] <+57>: mov    dword ptr [rbp - 0x14], eax
add[0x100000f7c] <+60>: mov    eax, esi
add[0x100000f7e] <+62>: add    rsp, 0x20
add[0x100000f82] <+66>: pop    rbp
add[0x100000f83] <+67>: ret
```

이 이후는 에필로그입니다. 프롤로그가 있으면 에필로그가 있는 법이지요. :) 지금은 별로 생각을 안 하셔도 됩니다만, 그냥 이것도 공부해두면 좋긴 합니다.



이제, add.c를 -O1 옵션을 줘서 최적화를 해봅시다. `gcc -o add_O1 -O1 add.c` 이런식으로 명령어를 치면 됩니다. 그리고, 디스어셈블을 해봅시다.

```shell
$ lldb add_o1
(lldb) target create "add_o1"
Current executable set to 'add_o1' (x86_64).
(lldb) disass -n main
add_o1`main:
add_o1[0x100000f70] <+0>:  push   rbp
add_o1[0x100000f71] <+1>:  mov    rbp, rsp
add_o1[0x100000f74] <+4>:  lea    rdi, [rip + 0x33]         ; "%d\n"
add_o1[0x100000f7b] <+11>: mov    esi, 0x3
add_o1[0x100000f80] <+16>: xor    eax, eax
add_o1[0x100000f82] <+18>: call   0x100000f8c               ; symbol stub for: printf
add_o1[0x100000f87] <+23>: xor    eax, eax
add_o1[0x100000f89] <+25>: pop    rbp
add_o1[0x100000f8a] <+26>: ret

(lldb)
```

?!

엄청 짧아졌습니다. 컴파일러가 컴파일을 하면서, 알아챈거죠. __"c 값만 출력하면 되는데, c 값은 a + b이고,  a 값과 b 값은 각각 1과 2이다. 그렇다면, c 값은 3일테고, 그냥 바로 3을 넘겨줘도 무방하겠다!"__라는 것을요. 그래서, 좀 전과 달리 esi 레지스터에 바로 0x3을 던져줍니다! 그러니까, C코드로 보면 이런식으로 바꿔버린거죠.

```c
#include <stdio.h>

int main()
{
  printf("%d\n", 3);
  return 0;
}
```



자 이제 add2.c를 봅시다.

```c
// add2.c
#include <stdio.h>

int main()
{
   int a, b, c;
   a = 1;
   b = 2;
   c = a + b;
   printf("%d + %d = %d\n", a, b, c);
   return 0;
}
```

요번에는 조금 달라졌습니다. 일단, c만 사용하는 것이 아닌, a, b, c 모두 사용을 합니다. 이제 이를 컴파일 해보고 뜯어봅시다.

```shell
$ lldb add2
(lldb) target create "add2"
Current executable set to 'add2' (x86_64).
(lldb) disass -n main
add2`main:
add2[0x100000f40] <+0>:  push   rbp
add2[0x100000f41] <+1>:  mov    rbp, rsp
add2[0x100000f44] <+4>:  sub    rsp, 0x20
add2[0x100000f48] <+8>:  lea    rdi, [rip + 0x5b]         ; "%d + %d = %d\n"
add2[0x100000f4f] <+15>: mov    dword ptr [rbp - 0x4], 0x0
add2[0x100000f56] <+22>: mov    dword ptr [rbp - 0x8], 0x1
add2[0x100000f5d] <+29>: mov    dword ptr [rbp - 0xc], 0x2
add2[0x100000f64] <+36>: mov    eax, dword ptr [rbp - 0x8]
add2[0x100000f67] <+39>: add    eax, dword ptr [rbp - 0xc]
add2[0x100000f6a] <+42>: mov    dword ptr [rbp - 0x10], eax
add2[0x100000f6d] <+45>: mov    esi, dword ptr [rbp - 0x8]
add2[0x100000f70] <+48>: mov    edx, dword ptr [rbp - 0xc]
add2[0x100000f73] <+51>: mov    ecx, dword ptr [rbp - 0x10]
add2[0x100000f76] <+54>: mov    al, 0x0
add2[0x100000f78] <+56>: call   0x100000f8a               ; symbol stub for: printf
add2[0x100000f7d] <+61>: xor    ecx, ecx
add2[0x100000f7f] <+63>: mov    dword ptr [rbp - 0x14], eax
add2[0x100000f82] <+66>: mov    eax, ecx
add2[0x100000f84] <+68>: add    rsp, 0x20
add2[0x100000f88] <+72>: pop    rbp
add2[0x100000f89] <+73>: ret

(lldb)
```

어셈 코드는 익숙하리라 생각을 하고, 바로 O1 옵션 준 녀석도 봅시다.

```shell
$ lldb add2_O1
(lldb) target create "add2_O1"
Current executable set to 'add2_O1' (x86_64).
(lldb) disass -n main
add2_O1`main:
add2_O1[0x100000f60] <+0>:  push   rbp
add2_O1[0x100000f61] <+1>:  mov    rbp, rsp
add2_O1[0x100000f64] <+4>:  lea    rdi, [rip + 0x3b]         ; "%d + %d = %d\n"
add2_O1[0x100000f6b] <+11>: mov    esi, 0x1
add2_O1[0x100000f70] <+16>: mov    edx, 0x2
add2_O1[0x100000f75] <+21>: mov    ecx, 0x3
add2_O1[0x100000f7a] <+26>: xor    eax, eax
add2_O1[0x100000f7c] <+28>: call   0x100000f86               ; symbol stub for: printf
add2_O1[0x100000f81] <+33>: xor    eax, eax
add2_O1[0x100000f83] <+35>: pop    rbp
add2_O1[0x100000f84] <+36>: ret

```

예상대로군요. 그냥 귀찮으니, 계산도 안하고, 바로 0x1, 0x2, 0x3을 때려박습니다. 즉, `printf("%d + %d = %d", 1, 2, 3)`과 동치라는거죠!



이 과제의 의미는 정말 간단했습니다. 최적화가 소스코드를 얼마나 단순화를 시키느냐라는 걸 알았으면 하는 거였고, 두번째로는 어셈블리랑 좀 친해져보란 의미로 과제를 낸 것입니다. 특히 코드엔진 문제를 풀어보면 아시겠지만, add나 mov 뿐만 아니라 cmp, jz, jne 같은 (if, else, else if와 같은) 조건 분기를 위한 명령어, call과 같은 함수 호출을 위한 명령어 등등을 보셨을테고, 이것이 C언어와 어떤 관계를 맺고 있는지에 대해서 아주 조금 알게 되셨을 것입니다. C언어는 다수 개의 어셈블리어와 대응되는 언어입니다. 그리고, 컴퓨터(정확히는 컴파일러)는 상황에 따라서 필요없다고 판단하면, 구문을 축약하거나, 생략하여 프로그램의 실행 속도를 높이려고 합니다. 이런 것들을 알고 시작을 하신다면, 리버싱을 시작하는데 마딱드리게되는 복잡한 로직들을 조금 더 쉽게 이해를 할 수 있을 것이라 생각됩니다.

자 이제, if, else, if else와 같은 구분을 보도록하죠.

```c
// if.c
#include <stdio.h>

int main()
{
   int input;
   scanf("%d", &input);
   if(input == 1)
      printf("hello");
   return 0;
}
```

```shell	
$ gcc -o if -O1 if.c
$ lldb if
(lldb) target create "if"
dCurrent executable set to 'if' (x86_64).
(lldb) disass -n main
if`main:
if[0x100000f40] <+0>:  push   rbp
if[0x100000f41] <+1>:  mov    rbp, rsp
if[0x100000f44] <+4>:  sub    rsp, 0x10
if[0x100000f48] <+8>:  lea    rdi, [rip + 0x59]         ; "%d"
if[0x100000f4f] <+15>: lea    rsi, [rbp - 0x4]
if[0x100000f53] <+19>: xor    eax, eax
if[0x100000f55] <+21>: call   0x100000f7c               ; symbol stub for: scanf
if[0x100000f5a] <+26>: cmp    dword ptr [rbp - 0x4], 0x1
if[0x100000f5e] <+30>: jne    0x100000f6e               ; <+46>
if[0x100000f60] <+32>: lea    rdi, [rip + 0x44]         ; "hello"
if[0x100000f67] <+39>: xor    eax, eax
if[0x100000f69] <+41>: call   0x100000f76               ; symbol stub for: printf
if[0x100000f6e] <+46>: xor    eax, eax
if[0x100000f70] <+48>: add    rsp, 0x10
if[0x100000f74] <+52>: pop    rbp
if[0x100000f75] <+53>: ret

(lldb)
```

코드를 좀 짧게하기 위해서 O1 옵션을 줘봤습니다. 보면 알겠지만, C언어 코드와 거의 대응이 잘 되는 걸 볼 수 있습니다. 그 중 if문을 구성하는 cmp, jne 문을 보도록하죠.

```shell
if[0x100000f5a] <+26>: cmp    dword ptr [rbp - 0x4], 0x1
if[0x100000f5e] <+30>: jne    0x100000f6e               ; <+46>
if[0x100000f60] <+32>: lea    rdi, [rip + 0x44]         ; "hello"
if[0x100000f67] <+39>: xor    eax, eax
if[0x100000f69] <+41>: call   0x100000f76               ; symbol stub for: printf
if[0x100000f6e] <+46>: xor    eax, eax
```

```c
if (input == 1)
  printf("hello");
```

이렇게 대응하는 걸 볼 수 있습니다. 일단 cmp문으로 (v4 == 0x1) 인지 확인하고, jne (jump not equal) 문을 이용해서, 같다면 점프를 하지 않고 (그러면 다음 라인의 printf 문이 실행되겠죠?), 같지 않으면 <+46> 으로 점프를 하는 걸 볼 수 있습니다. 즉 call 0x100000f76 (printf)를 실행하지 않고 프로그램을 종료 시키는 것을 볼 수 있습니다. :)

이런식으로 코드를 까보기 시작하면, 아시겠지만, 이런 cmp나 jz, jmp, jne 같은 점프 j로 시작하는 점프 구문들을 꼼꼼히 확인하면 if/else/else if/switch-case/goto 같은 분기문을 확인할 수 있고, 이를 C언어 형태로 바꾸거나, 플로우를 그려서 어떻게 작동하는지를 바로바로 알아볼 수 있을 것입니다.

기본적으로 알아야할 어셈블리를 알아보고 싶다면, 

1. http://thar.tistory.com/1
2. http://www.hackerschool.org/HS_Boards/data/Lib_prog/Asm.pdf
3. http://dakuo.tistory.com/7

를 참고하시면 될 거 같습니다. 기본적으로 C언어의 연산자(=, +, -, *, /, <<, >>, ^, |, ++, --….) 모두 대응하는 어셈블리어가 있으며, 대소 비교를 하는 >, <, >=, <= 같은 경우도 각각 대응하는 cmp 문들이 존재합니다. 또한, 메모리를 접근하는 명령어나, 편의를 위해 사용하는 stack을 다루는 push, pop 명령어들도 존재하죠. 이런 것들은 공부를 하면서 점점 알게 될 것입니다. 특히, 포인터 개념을 공부할 때 어셈블리를 직접보면서 공부하는 걸 권장하죠 :P