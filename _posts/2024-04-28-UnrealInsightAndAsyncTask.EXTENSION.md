---
title: AsyncTask와 Unreal Insight
date: 2024-04-28 09:00:00 +0900
categories: [Unreal, Study]
tags: [UnrealInsight, AsyncTask]     # TAG names should always be lowercase
math: true
mermaid: true
---
## 이야기
회사 프로젝트 작업을 하던 중, 소켓 통신을 해야할 일이 있었다. 그런데 Connect 함수가 한번 호출에 많은 시간이 걸려 게임이 멈추는 문제가 있었다. 그래서 언리얼의 멀티쓰레드 기능들을 이용하기로 했고, 그 중 하나가 AsyncTask다.


AsyncTask는 여러 쓰레드를 이용하는데, 하필 AsyncTask가 사용한 쓰레드에서 다른 기능들이 사용되고 있어서 소켓의 Connect함수가 다 실행되고 나서야 그 쓰레드에서 다른 기능들이 작동하는 문제가 발생했다. 즉, 멀티쓰레딩이 만능은 아니라는 것이다. 이런 현상을 겪고나니 AsyncTask가 실행될 때 언리얼 안에서는 무슨일이 일어나는지 궁금해졌다.

## AsyncTask

"Convenience function for executing code asynchronously on the Task Graph"

언리얼 매뉴얼에 있는 AsyncTask의 설명이다. 비동기로(즉, 다른 동작이 끝나고 실행되는 것이 아닌 따로 실행되는) 코드를 실행하는 편리한 함수. 우선 액터를 하나 만들고 소켓 통신을 하는 함수를 선언 및 정의한다. 아래의 Socket->Connect(*addr) 코드가 많은 시간을 소모하는 코드다. Tick에서 2초마다 실행되도록 한다.(TimerHandle을 쓰면 Unreal Insight에 좀 어렵게 표시된다)
  
AsyncTask를 간단하게 사용하자면 아래와 같다. HardWorker라는 액터를 하나 만들고 AsyncTask를 사용하는 코드다.

```cpp
void AHardWorker::WorkHardAsync() 
{
	AsyncTask(ENamedThreads::AnyNormalThreadNormalTask, [this]()
		{
			//여기서 코드 실행
		}
	);
}
```

여기서 AsyncTask는 ENamedThreads enum과 람다 함수로 실행할 수 있다. 더 세련되게 사용하는 방법이 있는데, 멀티쓰레딩의 바이블과도 같은 포스트가 있다.
<br/>
참고 자료(<https://forums.unrealengine.com/t/multithreading-and-performance-in-unreal/1216417>)


## Test
AsyncTask가 실행되는 동안 Unreal Insight에서 trace하기 위한 간단한 코드를 작성했다. 액터를 하나 만들고 소켓을 생성 및 연결을 하는 코드다.
<br/>
여기서 Socket->Connect(*addr) 코드가 많은 시간을 잡아먹는 코드다.


```cpp

void AHardWorker::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
	Timer += DeltaTime;
	if (Timer > 2.0f)
	{
		WorkHard();
		Timer = 0.0f;
	}
}

void AHardWorker::WorkHard()
{
	FSocket* Socket = ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM)->CreateSocket(TEXT("Stream"), TEXT("Client Socket"));

	// IP를 FString으로 입력받아 저장
	FString address = TEXT("127.0.0.1");
	FIPv4Address ip;
	FIPv4Address::Parse(address, ip);

	int32 port = 6000;	// 포트는 6000번

	// 포트와 소켓을 담는 클래스
	TSharedRef<FInternetAddr> addr = ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM)->CreateInternetAddr();
	addr->SetIp(ip.Value);
	addr->SetPort(port);

	GEngine->AddOnScreenDebugMessage(-1, 5.f, FColor::Red, FString::Printf(TEXT("Trying to connect@@@@@.")));

	// 연결시도, 결과를 받아옴
	bool isConnetcted = Socket->Connect(*addr);
}

```

이렇게하면 게임이 뚝뚝 끊기게되고, Unreal Insight를 켜면 아래처럼 나온다. 한 프레임에 2초가 걸리는 것을 확인할 수 있다.
![Untitled](/assets/AsyncTask/2sTick.png)


이제 AsyncTask를 추가하고 Tick에선 WorkHard() 대신 WorkHardAsync를 실행하도록 한다. WorkHard() 함수를 비동기로 실행하는 함수를 선언 및 정의한다.

```cpp

void AHardWorker::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
	Timer += DeltaTime;
	if (Timer > 2.0f)
	{
		WorkHardAsync();
		Timer = 0.0f;
	}
}

void AHardWorker::WorkHardAsync()
{
	AsyncTask(ENamedThreads::AnyNormalThreadNormalTask, [this]()
		{
			WorkHard();
		}
	);
}
```

실행하면 게임은 더 이상 끊기지 않으며(Game Thread가 10ms 이하로 안정적으로 나온다) Foreground 및 Background Thread에서 코드가 실행됨을 확인할 수 있다. 다만, 추적된 코드실행이 ExecuteForgroundTask 이런식으로 되어 있는데, 확실하게 이것이 무엇을 의미하는지는 아직 모른다. 다만, AsyncTask의 인자로 AnyBackgroundTask 같은 쓰레드를 사용하면 해당 코드가 해당 쓰레드에서 실행되는 것 또한 확인할 수 있다.


아래는 AnyNormalThreadNormalTask를 사용한 것이고, Foreground 및 Background에서 코드가 실행됨을 확인할 수 있다. (부하가 큰 코드를 실행해야 확인이 가능하다. 다른 task들도 이름이 똑같이 추적되기 때문에...)
![Untitled](/assets/AsyncTask/Foreground.png)


ENamedThread를 AnyBackgroundThreadNormalTask로 하게 되면, Background Thread에서 실행되는 것을 확인할 수 있다. 아래 코드는 소켓 연결이 아닌 LineTrace로 부하가 큰 코드를 실행하여 889ms가 걸렸다.
```cpp
void AHardWorker::WorkHard2()
{
	FHitResult HitResult;
	for (int32 i = 0; i < 1500000; i++)
	{
		FVector Start = GetActorLocation();
		FVector End = GetActorLocation() + FVector(0.0f, 0.0f, 1.0f + (i * 0.0001f));
		GetWorld()->LineTraceSingleByChannel(HitResult, Start, End, ECollisionChannel::ECC_WorldStatic);
	}
}
```
![Untitled](/assets/AsyncTask/Background.png)


800ms이 넘는 시간이 걸렸음에도 Game Thread는 16ms 정도로, 거의 60fps를 유지하는 것 또한 확인할 수 있다.
![Untitled](/assets/AsyncTask/GameAndBackground.png)







## 마무리
AsyncTask를 테스트하고 나서 FRunnable도 테스트를 해보았지만 FRunnable은 Unreal Insight에서 제대로 확인할 수가 없었다. FRunnable은 위 링크에서 자세히 설명되어 있지만 제대로 다룰 수 있으려면 좀 더 연구해봐야 될 것 같고, 프로파일링 방법도 찾아봐야할 것이다.