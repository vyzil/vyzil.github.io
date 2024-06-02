---
title: Pokerogue 해보기
category:
    - ETC
tags:
    - Game
---

Pokerogue라는 게임을 해보기로 했다.  
Web 기반 게임이라는데, 나무위키 페이지를 보니 github에서 코드를 clone해서 할 수도 있다고 한다.
따라서, clone 해서 local 서버에서 구동시켜봤다.

web에서 굳이 하지 않는 이유
+ 정식 게임사가 아니라서 혹시 중간에 지원 안할까봐
+ "그냥"


## 설치과정

우선 [Github: PokeRogue](https://github.com/pagefaultgames/pokerogue) 를 참조해서 설치


```bash
git clone https://github.com/pagefaultgames/pokerogue
cd ./pokerogue/
npm install
npm run start:dev

```

당연하게도 나는 웹관련 작업을 해본적이 없어서 실행할 수 있는 기본 패키지가 없었다.  
따라서 우선은 npm을 설치해주고 진행

```bash
sudo apt-get install npm
npm install
npm run start:dev

```

여기서 어떤 오류가 떳었는데, 잘 기억은 안난다.  
대부분은 버전 문제이기 떄문에 버전을 확인해보니, 권장 버전이 아니었다.  
특히 wsl ubuntu 에서 기본으로 설치된 node 버전이 맞지 않았던 것 같다.

```bash
node -v # this should be 20.13.1 (Prerequisites)
npm -v
```

요즘 자주 사용하는 ChatGPT-4o에게 질문. (세상이 너무 좋다)

![alt text](../../assets/image/image-2.png)


요약하자면, 
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash

export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

nvm install 20.13.1

nvm use 20.13.1
nvm alias default 20.13.1

node -v
npm -v

```

`export` 명령어는 `install.sh`에서 수행하라는 명령어와 동일했다.  
아무튼 이후 `npm run start:dev` 명령어를 통해 실행해주면 된다.

## 추가 설정

참고로, 나는 vscode를 통해 서버를 돌리므로 이를 끄면 서버가 꺼진다.  
따라서 `screen`으로 세션이 유지되도록 설정했다.

```bash
screen -S pokerogue
npm run start:dev

# [Ctrl + A, D] -> detach

screen -r pokerogue # attach
```

끝.