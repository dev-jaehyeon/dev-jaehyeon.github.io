---
title: Unreal UnityBuild
date: 2023-10-23 14:00:00 +0900
categories: [Unreal]
tags: [unitybuild]     # TAG names should always be lowercase
math: true
mermaid: true
---
## 더 볼만 자료:

넷마블 기술블로그, [https://netmarble.engineering/unity-build-can-pump-up-build-speed/](https://netmarble.engineering/unity-build-can-pump-up-build-speed/)
언리얼 공식 문서, [https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/BuildTools/UnrealBuildTool/TargetFiles/](https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/BuildTools/UnrealBuildTool/TargetFiles/)

## 이야기

회사 프로젝트가 점점 커지고 어딘가 헤더 파일이 꼬이는 경우가 있는데, 도저히 원인을 찾을 수 없는 경우가 있다. 근본적인 해결책은 아니지만 UnityBuild를 false하는 것으로 임시로 해결했다.

오류 상황을 보면 도저히 찾을 수 없는 곳에서 에러가 뜨는데, 해당 cpp파일에서 그냥 빈 칸 하나 추가하는 등 수정을 가하면 자동으로 UnityBuild에서 제외되고 새로 컴파일하게 된다. 그렇게라도 빌드가 되면 다행이지만 끝까지 오류가 남는 경우는 아예 UnityBuild를 끄는 것으로 임시로 해결할 수 있다. 현 프로젝트에서는 급하게 시연을 준비해야할 것이 있어 임시로 조치를 취했다.

## Unity Build

UnityBuild (Unity Engine 아님)는 C와 C++ 소스를 빌드할 때, 빌드 시간을 단축해주는 아이디어이다. cpp 파일이 헤더를 include할 때, 여러 곳에서 같은 헤더를 include하면 컴파일러 체인에 의해 헤더가 여러 번 처리되고 그만큼 정직하게 용량과 빌드 시간이 늘어나게 된다.

하지만 UnityBuild 개념을 적용해 헤더를 한 번만 컴파일하게 하면 용량도 줄어들고 빌드 시간도 단축할 수 있다. 직접 UnityBuild를 적용하는 방법은 UnityBuild용 cpp파일에 몰아 넣기 등으로 적용할 수 있다.

언리얼 엔진에서는 기본적으로 UnityBuild가 적용되어 있다. 하지만 언리얼 엔진에서 UnityBuild를 끄는 방법은 여러가지가 있는데, 그중 하나로, 엔진 디렉토리에서 Engine\Saved\UnrealBuildTool 의 BuildConfiguration.xml에 UnityBuild 비활성화 옵션을 추가하는 것이다.

출처: [https://gist.github.com/leegoonz/b98f440e6329836ea18bd94e4b778fe2](https://gist.github.com/leegoonz/b98f440e6329836ea18bd94e4b778fe2)

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Configuration xmlns="https://www.unrealengine.com/BuildConfiguration">
	<BuildConfiguration>
		<bUseUnityBuild>false</bUseUnityBuild>
		<bUsePCHFiles>false</bUsePCHFiles>
	</BuildConfiguration>
</Configuration>
```

## 테스트

UnityBuild가 켜진 상태(디폴트, 아무것도 건드리지 않음)에서 프로젝트 빌드를 하면

![Untitled](/assets/UnityBuild/Before.png)

이렇게 2000대의 컴파일 프로세스가 있지만, UnityBuild를 끄면 원래 2000대였던 컴파일 프로세스가 16000으로 늘어난 것을 볼 수 있다.

![Untitled](/assets/UnityBuild/After.png)

## 마무리

그러나 UnityBuild에 의존하는 플러그인들도 간혹 있다. 현재 회사 프로젝트에는 Logitech Wheel 플러그인이 설치되어 있는데, 이 플러그인이 UnityBuild에 의존하며 만들어져서 그런지 Logitech Wheel 플러그인이 활성화되어 있으면 빌드가 되지 않는 문제가 있었다. UnityBuild를 해제해야만 빌드가 되도록하는 헤더파일 꼬임 현상을 근본적으로 해결할 필요가 있다.