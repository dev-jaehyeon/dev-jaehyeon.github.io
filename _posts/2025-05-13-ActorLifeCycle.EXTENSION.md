---
title: Actor Life Cycle의 만료
date: 2025-05-13 14:32:00 +0900
categories: [Unreal]
tags: [actor, lifecycle, memory, gc]     # TAG names should always be lowercase
math: true
mermaid: true
---

출처 및 참고 자료<br/>
<https://unreal.gg-labs.com/wiki-archives/common-pitfalls/how-to-prevent-crashes-due-to-dangling-actor-pointers>
<https://dev.epicgames.com/documentation/ko-kr/unreal-engine/unreal-engine-actor-lifecycle>
<https://algorfati.tistory.com/75>

## Actor Life Cycle (액터 생명 주기)

공식 문서를 보면 액터의 생명주기를 자세히 알 수 있다. 간단히 요약하면 게임 시작 시 여러 함수들을 거쳐 ``AActor::BeginPlay()``까지 호출된다. 액터의 라이프사이클이 종료될 때는 마찬가지로 ``AActor::EndPlay()``가 호출되면서 마무리된다.

런타임에서 액터를 없앨 때, ``AActor::Destroy()``를 호출하게 되는데, 문제는 ``AActor::Destroy()``를 호출한다고 바로 액터가 소멸되는 것이 아니고 ``RF_PendingKill`` 플래그가 표시되어 다음 가비지 컬렉션에서 사라지게 된다. 이 동안은 해당 액터는 nullptr이 아니기 때문에 null 체크만 하면 크래시가 나기 십상이다.

## null, valid, weakptr 체크

언리얼 메모리관리와 스마트포인터, 순환 참조 문제는 위 블로그에 아주 잘 나와있다.
간단하게 액터를 스폰하고 파괴하는 코드를 짜본다.

```cpp
public:
	UFUNCTION(BlueprintCallable)
	void CreateActor();
	UFUNCTION(BlueprintCallable)
	bool CheckActor();
	UFUNCTION(BlueprintCallable)
	void DestroyActor();

public:
	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	TSubclassOf<AActor> ActorClass;

private:
	AActor* ActorInstance;
```

```cpp

void UPointerCheckComponent::CreateActor()
{
	FActorSpawnParameters param;
	ActorInstance = GetWorld()->SpawnActor<AActor>(ActorClass,  param);
	WeakPtr = ActorInstance;
}

bool UPointerCheckComponent::CheckActor()
{
	bool returnal = true;
  //null 체크
	if (ActorInstance == nullptr)
	{
		UE_LOG(LogTemp, Warning, TEXT("1. NULL Check: null"));
		returnal = false;
	}
	else
	{
		UE_LOG(LogTemp, Warning, TEXT("1. NULL Check: NOT null"));
	}
  //valid 체크
	if (IsValid(ActorInstance))
	{
		UE_LOG(LogTemp, Warning, TEXT("2. Valid Check: valid"));
	}
	else
	{
		UE_LOG(LogTemp, Warning, TEXT("2. Valid Check: NOT valid"));
		returnal = false;
	}
  //pending kill 체크
	if (ActorInstance && ActorInstance->IsPendingKill() == true)
	{
		UE_LOG(LogTemp, Warning, TEXT("3. PendingKill Check: true"));
	}
	else
	{
		UE_LOG(LogTemp, Warning, TEXT("3. PendingKill Check: false"));
	}
  //lowlevel valid 체크
	if (ActorInstance->IsValidLowLevel() == true)
	{
		UE_LOG(LogTemp, Warning, TEXT("4. Low Valid Check: valid"));
	}
	else
	{
		UE_LOG(LogTemp, Warning, TEXT("4. Low Valid Check: NOT valid"));
		returnal = false;
	}
	return returnal;
}

void UPointerCheckComponent::DestroyActor()
{
	if (CheckActor() == true)
	{
		UE_LOG(LogTemp, Warning, TEXT("Destroying Actor.. is valid.."));
		ActorInstance->Destroy();
	}
	else
	{
		UE_LOG(LogTemp, Warning, TEXT("Destroying Actor.. is not valid.."));
	}
}
```

이렇게 해서 ``CreateActor()`` -> ``DestroyActor()`` -> ``CheckActor()``를 호출하면 아래와 같이 출력된다.

![img-description](/assets/ActorLifeCycle/result1.png){: .normal w="400" h="250"}

``AActor::Destroy()``를 호출했지만 여전히 null이 아니며 valid하진 않지만 valid in lowlevel에서는 여전히 유효하다. 이 상황에서 다른 오브젝트가 참조할 때 null만 체크해버리면 크래시가 난다. PendingKill이 true가 되었는데, 이것은 다음 가비지 컬렉션에서 메모리 할당 해제되는 플래그이다.

## 가장 깔끔한 방법: TWeakObjectPtr
해당 오브젝트가 유효한지 아닌지 알아보는 안전한 방법은 TWeakObjectPtr을 사용하는 것이다.
```cpp

void UPointerCheckComponent::CreateActor()
{
	FActorSpawnParameters param;
	ActorInstance = GetWorld()->SpawnActor<AActor>(ActorClass,  param);
	WeakPtr = ActorInstance;
}

bool UPointerCheckComponent::CheckActor()
{
	if (WeakPtr.IsValid() == true)
	{
		UE_LOG(LogTemp, Warning, TEXT("5. WeakPtr Check: valid"));
	}
	else
	{
		UE_LOG(LogTemp, Warning, TEXT("5. WeakPtr Check: NOT valid"));
		returnal = false;
	}
}
```
이렇게 하게되면 액터가 파괴되었을 때 즉각 유효하지 않다고 반환해준다.

## 결론
이러나 저러나 자주 생성하고 사라지는 액터들을 참조하는 경우는 언리얼 공식 문서에 나와있는대로<br/>
``
발생 과정과 상관없이, 액터는 RF_PendingKill 로 표시되어 다음 가비지 컬렉션 주기 동안 UE가 메모리에서 할당 해제합니다. 또한, 보류 중인 킬을 수동으로 확인하는 대신 FWeakObjectPtr<AActor> 를 사용하는 것이 더 깔끔합니다.
``<br/>
강한 참조는 필요한 부분에만 쓰고 나머진 WeakObjectPtr로 참조하면 참조 그래프에 부담도 덜하고 크래시도 피할 수 있을 것이다.