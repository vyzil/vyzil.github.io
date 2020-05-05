---
title: Python 환경변수 설정
category:
    - ETC
tags:
    - Python
---

Python3를 주로 사용하는데, python2를 base로 동작하는 프로그램들이 많아서 환경변수를 설정해준 상황을 정리함.

## Python3
```
C:\Users\Shin\AppData\Local\Programs\Python\Python37\
C:\Users\Shin\AppData\Local\Programs\Python\Python37\Scripts\
```
기존에 위와 같이 계정 환경변수가 설정되어 있어서 `python`, `pip`, `pip3`를 입력하면 python3가 실행되었음.

```
mklink C:\Windows\python3.exe  C:\Users\Shin\AppData\Local\Programs\Python\Python37\python.exe

mklink C:\Windows\pip3.exe  C:\Users\Shin\AppData\Local\Programs\Python\Python37\Scripts\pip.exe
```

환경변수를 제거해주고, 위와 같이 Windows 폴더에 Link를 건 파일을 생성해줌. 실행해보니 `python37.dll`이 없다는 문구가 떠서 해당 경로에 추가해 주었음. `python3`, `pip3`가 python3로 실행이 됨.

## Python2
```
C:\Users\Shin\AppData\Local\Programs\Python\Python27\
C:\Users\Shin\AppData\Local\Programs\Python\Python27\Scripts\
```
계정 환경변수에 위와 같이 추가를 해 주었음. `python`, `pip`가 python2로 실행이됨.

### Immunity Debugger
위와 같이 python2 최신 release를 설치해 주었더니, Immunity Debugger에서 pycommands가 정상적으로 작동하지 않았음. Immunity Debugger에서는 2.7.1 버전을 사용하므로 상위 버전과 호환이 안되는 것 같음. 따라서 일단 상위 버전을 제거해 주었고, 2.7.1v을 설치함. 환경변수는 `C:\Python27`을 추가해 주었음.

찾아보니 python2 하위 버전은 pip가 기본적으로 내장되어 있지 않아서 따로 설치해 주어야 하는데 SSL 관련 문제가 발생하여 실패함. 상황을 정리하자면 아래와 같은데, 일단 Python 2.7.1을 pip없이 사용하기로함.

- Python 2.7.1 => pip 설치에 문제 발생.
- Python 상위버전 => Immunity Debugger에서 Pycommand를 사용할 수 없음.
