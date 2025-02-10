---
title: Unreal Insight로 Android trace하기
date: 2024-04-21 09:00:00 +0900
categories: [Unreal, Study]
tags: [UnrealInsight]     # TAG names should always be lowercase
math: true
mermaid: true
---
## 이야기
회사 프로젝트 작업을 하던 중, Meta Quest 플랫폼으로 패키징을 하면 다르게 동작하는 부분들이 있어 프로파일링을 찾아보게 되었다.
매뉴얼에는 안드로이드에서 프로파일링을 하는 방법이 꽤나 부실하게 설명되어 있는데, 우연히 좋은 영상을 찾아서 정리한다.

## Unreal Insight - Android

Unreal Insight는 언리얼 엔진을 소스코드로 다운 받고 그 안의 프로그램에서 실행할 수 있다.
![Untitled](/assets/UnrealInsightAndroid/UnrealInsight.png)

그런데 안드로이드에서는 어떻게 프로파일링을 할까?

참고 링크: <https://www.youtube.com/watch?v=IHC9dn8yl1A>

- 우선 안드로이드 플랫폼으로 패키징하여 apk로 만든다.
- adb - 아마 안드로이드 패키징 때문에 설치한 안드로이드 스튜디오에 의해 자동으로 설치되었을 것이다.
- 당연한 이야기지만 안드로이드 기기를 준비하고 개발자모드를 켠다.
- cmd에서 입력하기:
참고 매뉴얼(<https://docs.unrealengine.com/4.27/ko/TestingAndOptimization/PerformanceAndProfiling/UnrealInsights/Overview/>)


```text
adb.exe reverse tcp:1980 tcp:1980
adb shell setprop debug.ue.commandline -tracehost=127.0.0.1
```
- UnrealInsight를 켠다.
- 안드로이드 기기에서 앱을 켠다.
- UnrealInsight에서 LIVE, Game이라는 세션으로 새로 생성된다. 그것을 실행한다.

## Test

안드로이드 apk 파일이 준비되었고 개발자 모드가 켜진 안드로이드 기기에 설치했다고 치고 테스트를 해본다.
cmd를 켠 후 위 명령어를 입력한다.
![Untitled](/assets/UnrealInsightAndroid/cmdadb.png)

안드로이드 기기에서 앱을 실행한다.
![Untitled](/assets/UnrealInsightAndroid/AppInAndroid.jpg)

그 뒤, Unreal Insight를 켜주면 새로운 항목과 함께 LIVE가 붙은 세션이 보인다. Platform이 Android이며, Build Target은 Game이다. 이것을 실행한다.
![Untitled](/assets/UnrealInsightAndroid/LiveInsight.png)

정상적으로 Unreal Insight가 실행된다.
![Untitled](/assets/UnrealInsightAndroid/InsightMain.png)

쓰레드를 테스트하다가 발견한 것인데, 데스크탑에 비해 ForegroundWorker 및 BackgroundWorker가 매우 적은 것을 확인할 수 있다.
![Untitled](/assets/UnrealInsightAndroid/InsightLessWorkers.png)

Unreal Insight - Android를 끄고 싶다면, Unreal Insight Session Browser에서 Connection탭으로 간 후, Connection을 누르면 Successfully Connected To 메시지가 출력되고, 에디터를 실행하거나 게임플레이를 누르면 다시 에디터에서 Insight를 trace하게 된다.
![Untitled](/assets/UnrealInsightAndroid/Successful.png)

다만 여기서 다시 Android를 켤 경우, 이유는 모르겠지만 똑같이 커맨드를 입력하고 실행하면 안드로이드로 Insight가 연결되지 않고 계속 에디터를 trace하는 현상이 발생했다.
해결 방법이 조금 이상하지만 이럴 때는 Unreal Insight와 에디터 모두 다 끄고
- 안드로이드 기기 usb 다시 꽂기
- adb kill-server 후 다시 adb start-server
- 언리얼 에디터를 실행하기(중요, Unreal Insight를 실행하기 전에 에디터를 먼저 켜야함)
- adb 명령어 입력
- Unreal Insight 실행
- 안드로이드에서 앱 실행
을 하면 된다.

## 마무리
원래 AsyncTask 테스트를 하던 도중, 생각보다 Unreal Insight에 시간을 더 쓰게되어 따로 정리했다. AsyncTask를 사용할 때, 여러 종류의 쓰레드를 사용할 수가 있는데, 에디터에서 할 때는 잘 되던 것이 패키징만 하고나면 작동을 안해서 코드가 어느 쓰레드에서 작동하는지 확인하려고 Unreal Insight를 쓰게 되었다. 어느 정도 예상대로 데스크탑에서 할 때는 ForgroundWorker와 BackgroundWorker가 엄청 많은데(심지어 노는 애들도 많다) 안드로이드는 그 Worker의 수가 엄청 적다. 아직 확인은 못했지만 가설을 세우자면, BackgroundWorker에 할당되어 실행되는 기능이 있었는데, AsynchTask로 Background에서 부하가 큰 작업을 실행시켜버렸다가 큐에 쌓이게 되어 제대로 작동하지 않았을 것 같다. 조만간 확인할 것이다.