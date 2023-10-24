# 리눅스를 main desktop으로 사용하기

빈번하게 실패하면서 또 시도하게 되는, 그러나 이제는 할 수 있을 것 같은 시도와 내용을 기록하여 둔다. mac은 메모리에 따라 비용 차이가 너무 심하고, 너무 상업적이라 마음에 들지 않음. 윈도우는 WSL2 로 올라가면서 많이 편리해 졌으나, native linux가 아니고 별도의 vm이라 간혹 문제가 있다.

이 문서에서는 ubuntu 22.04 + KDE plasma desktop 을 기반으로 설명한다.

arch linux, manjaro등 더 훌륭하다고 얘기되는 배포판들이 있으나 적응에 실패하였다.

## 준비하기

1. Dell또는 Thinkpad의 랩탑이 가장 적당함 -> 가성비 좋고 안정성 높다.
2. 리눅스 전문 브랜드(framework laptop. system76 laptop...)의 랩탑 -> 좋은데 비용이 다소 높다.
3. 중국 통팡사의 OEM 랩탑 등 중국랩탑 -> 저렴하고 고성능

문제는 어디서나 발생할 수 있으며 계속 부딪혀야 한다.
기본적인 절전모드, 전원사용, 재기동 등 시스템의 여러가지 기능들에 대하여 1번 형태의 랩탑이 아닐 경우 문제가 발생할 확률이 높다.
사용하는 프로그램이 없을수도, 대안조차 존재하지 않을 수 있다.

## 설치하기

Ubuntu 22.04 LTS(최신) 버전 설치
- desktop으로는 kde plasma (안정성이 높다)
- 그 밖에 설치해야 할 프로그램들 - vscode, intellij , dbeaver, slack, enpass, ms edge, chrome
- vagrant, libvirt(virt-manager) - 가상머신, 클러스터 테스트용
- openjdk-17-jdk, nvim, ghostwriter, tlp(전원관리), Kate(편집기-기능이 많은-)

kde desktop의 기본 프로그램인 Dolphin, Konsole 등은 대체제가 없을 정도로 훌륭한 기능들이 많음

아직은 절전모드 등에서 벗어날 때 오류가 나거나, 배터리가 많이 빠지는 현상 등이 발생할 수 있음.

ubuntu 설치 후 kde plasma desktop을 설치하면 nerwork manager가 보이지 않는다 - sudo apt install plasma-nm으로 설치 후 재기동한다.

## 사용하기

### 기본도구

터미널(kconsole) 사용 단축키
- Ctrl+(-세로분할,Ctrl+)-가로분할
- Ctrl+{-현재창왼쪽으로축소,Ctrl+}-현재창우측으로확장
- 단축키의 중심이 되는 메타 키는 Ctrl + Shift임

Dolphin에서 F4를 누르면 하단에 Konsole이 생기며 터미널 화면이 보여진다. 쓸데 없을 듯 하나 간혹 대단히 유용


### 쉘

쉘 환경 구성(oh-my-zsh) 및 유용한 도구
- zsh,oh-my-zsh 및 플러그인(syntax-highlighting,autosuggestions
,fasd) 설치

```
# 2020.06 업데이트 from https://github.com/ohmyzsh/ohmyzsh
$ sh -c "$(curl -fsSL https://raw.githubusercontent .com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
# zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
# zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
# fasd
sudo apt install fasd
# 추가 후 .zshrc 파일의 plugins부분을 다음과 같이 수정
plugins=(git zsh-autosuggestions zsh-syntax-highlighting fasd)
```

fasd의 다른 기능은 잘 모르겠는데 z xxxx 로 이동했었던 디렉토리로 바로 점프하는 기능은 유용

### 폰트

폰트는 nerd-font(https://www.nerdfonts.com/) 중 MesloLGMDZ NFM, Fatasque sans mono 추천함, 터미널 상에서 기본적으로 표현 불가능한 문자까지 나타내어 준다.

해당 폰트를 설치한 뒤 vscode, atom 등에서 다음과 같이 입력함

- vscode : font 선택한 뒤 "MesloLGMDZ NFM, Malgun Gothic, monospace"으로 설정
  - 실제 파일명은 "Meslo LG M DZ Regular Nerd Font Complete Mono Windows Compatible"임
- ATOM : MesloLGMDZ NFM, DejaVu Sans Mono, Consolas, monospace


### 테마

```zsh
git clone https://github.com/denysdovhan/spaceship-prompt.git "$ZSH_CUSTOM/themes/spaceship-prompt"
ln -s "$ZSH_CUSTOM/themes/spaceship-prompt/spaceship.zsh-theme" "$ZSH_CUSTOM/themes/spaceship.zsh-theme"
# 추가 후 .zshrc 파일의 ZSH_THEME부분을 다음과 같이 수정
ZSH_THEME="spaceship"
```

### 구글 드라이브 연결

- 구글 드라이브 쪽에서 공식적으로 제공하는 Linux용 클라이언트는 없음
- google-drive-ocamlfuse사용법(비공식 지원도구) - https://wikidocs.net/168277, 잘 동작하나 속도가 매우느린편임

```zsh
#실행하기
google-drive-ocamlfuse /home/ska/googledrive
#재시작시에도 동일하게 동작하기 위해 다음과 같이 crontab 에 등록
@reboot google-drive-ocamlfuse /home/ska/googledrive
```


### 터치패드

mac에 비빌 수준은 아니나, 윈도우에서도 많이 발전했듯이 Linux에서도 많이 발전했다.

- fusuma 설치 및 적용 - https://shanepark.tistory.com/257

위 블로그에서는 대부분 정리되어 있으나 핀치인, 핀치아웃이 거꾸로 되어 있어 그것들만 반대로 저장하면 된다.

```
#vi .config/fusuma/config.yml
swipe:
  3:
    left:
      command: "xdotool key alt+Right" # History forward
    right:
      command: "xdotool key alt+Left" # History back
    begin:
      command: xdotool mousedown 1
    update:
      command: xdotool mousemove_relative -- $move_x, $move_y
      interval: 0.01
      accel: 2
    end:
      command: xdotool mouseup 1
  4:
    up:
      command: "xdotool key super" # Activity
    down:
      command: "xdotool key super" # Activity
    left:
      command: "xdotool key super+0xff55" # Switch to previous workspace , pageUp
    right:
      command: "xdotool key super+0xff56" # Switch to next workspace , pageDown

pinch:
  2:
    in:
      com9Jz3UaT@5Fjqumand: "xdotool keydown ctrl click 5 keyup ctrl"
    out:
      command: "xdotool keydown ctrl click 4 keyup ctrl"

threshold9Jz3UaT@5Fjqu:
  swipe: 0.5
  pinch: 0.3

interval:
  swipe: 0.5
  pinch: 0.5
```

시작 프로그램에 스크립트를 등록해 주어야 재기동 후에도 사용 할 수 있다.
- 등록할 내용 : /usr/local/bin/fusuma -d


## linux desktop의 좋은점

- (기본적으로 MacOS나 Windows보다는) 자원을 적게 사용한다.
- 비용이 적게 든다.(50만원이면 쓸만한 Thinkpad 랩탑을 구할 수 있다.)
- 리눅스 기반 개발(Cloud Native)에 최적화 되어 있다. 별도 VM을 설치하거나 할 필요가 없음
- 오류를 자주 마주쳐,해결을 하는 실력이 늘어날 수 있다.(과연 이게 실력일까? 스트레스일까?)

## PainPoint

- 윈도우를 써야 할 상황이 간혹 생긴다, 업무를 하다 보면 삼성에서 개발한 knox meeting에 들어가야 한다거나.. 다만 mac사용자가 늘면서 그런 상황은 많이 줄었다.

- 오피스 - 조회만이면 Libre Office또는 Google Docs, Microsoft 365(Web)으로 가능. 제안서 작업 등 본격적으로 파워포인트 등을 쓰려면 윈도우 랩탑이 필요함

- 원격연결 - 클라이언트로만 사용, 접속시에 desktop환경(gnome,xfce,kde 등) 선택에 따라 못쓸정도로 느릴수도 있고, GUI사용시 동일ID의 타 User로 인식하여 문제가 있을 수 있다.

- 화면캡춰 - Flameshot 을 사용

- 공공기관,증명서 발급 - 사용불가, 원격으로 윈도우 노트북에 붙어서 사용해야 함
