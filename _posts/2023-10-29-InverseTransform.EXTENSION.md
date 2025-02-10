---
title: Inverse Transform과 Grandparent에 대한 RelativeTransform
date: 2023-10-29 09:00:00 +0900
categories: [Unreal, Study]
tags: [Transform]     # TAG names should always be lowercase
math: true
mermaid: true
---
## 이야기
딱히 행렬에 대해 깊게 알지 못해도 Unity나 Unreal에서 오브젝트를 움직이고 Transform 자식 구조를 바꾸고 위치를 계산하고 원하는 결과를 만드는데에는 큰 무리가 없다. 워낙 API가 잘 만들어져 있고 우리는 그 기능들이 무슨 결과를 가져다주는지 정도만 파악해도 개발에 지장이 있는건 아니다. 그러나 종종 더 복잡한 기능이 필요할 때가 있는데, 예를 들어 부모의 부모로부터의 Transform이나 복잡한 Transform 체계를 가진 오브젝트의 특정 상대좌표 등이다.

회사프로젝트에서 관전자 플레이어가 있는데, 이 관전자 플레이어는 게임 플레이어의 카메라를 따라가도록 되어 있었다. 즉, 게임 플레이어의 특정 SceneComponent의 RelativeTransform이 레플리케이션되고, 관전자 플레이어는 게임 플레이어의 Pawn에 Snap되고, 관전자 플레이어의 카메라가 레플리케이션되는 RelativeTransform을 따라가게 만들었다.(처음엔 Snap하지 않고 WorldTransform을 레플리케이션하여 따라가게 했는데 그렇게 하니 오차가 너무 커지고 계속 뚝뚝 끊기게 되어서 방식을 변경했다) 그런데 문제는 RelativeTransform을 레플리케이션하기 때문에 게임플레이어의 SceneComponent의 Transform 체계가 바뀌면 게임 플레이어와 관전자 플레이어가 공유해야하는 Transform값이 표면상으로만 같아지고 실제 값은 게임 플레이어의 Transform 체계가 바뀐 만큼 달라진다는 것이다. 그렇다고 게임플레이어의 Transform 체계가 바뀔 때마다 이를 일일이 RPC를 호출하자니 매우 번거롭고 애초에 게임 플레이어는 내가 만든 것도 아니었다.

그래서 Snap을 함과 동시에 Transform 체계가 바뀌더라도 Root에 대한 타겟 SceneComponent의 RelativeTransform만 계산하여 레플리케이션하기로 했다.

## RelativeTransform from Grandparent
Transform은 그 자체로 행렬이며 변환, 즉 함수이기 때문에 부모와 부모의 부모까지 올라가는 RelativeTransform은 Transform의 곱셈으로 간단히 구할 수 있다.

```cpp
FTransform AInverseTester::GetTransformFrom(USceneComponent* _Head, USceneComponent* _Root)
{
	FTransform Accumalated = _Head->GetRelativeTransform();
	USceneComponent* CurrentSceneComp = _Head->GetAttachParent();
	while (CurrentSceneComp != _Root)
	{
		Accumalated *= CurrentSceneComp->GetRelativeTransform();
		CurrentSceneComp = CurrentSceneComp->GetAttachParent();
		if (CurrentSceneComp == nullptr)
		{
			UE_LOG(LogTemp, Warning, TEXT("There is no %s in parents of %s"), *(_Root->GetName()), *(_Head->GetName()));
			break;
		}
	}
	return Accumalated;
}
```
특정 SceneComponent와 Root를 주고, SceneComponent로부터 Root까지의 모든 RelativeTransform들을 곱하여 궁극적으로 Root에서 Head까지의 RelativeTransform을 구하게 된다.

## Test

![Untitled](/assets/InverseTransform/TransformSystem.png)

이러한 시스템이 있고, Root로부터 Child0~3까지 모두 상대위치를 10, 10, 10으로 주었다. Head는 StaticMeshComponent로 그냥 위치 파악용으로 놨다. HeadTracker는 Root의 바로 아래 Child로 있는 StaticMeshComponent다. 여기서 Head의 Root로 부터의 좌표는 40,40,40이 될 것이다.(Child0~3 총 4개가 각각 상대좌표가 10, 10, 10이므로)
![Untitled](/assets/InverseTransform/TransformSystem2.png)

HeadTracker에 넣어봤다. 예상대로 HeadTracker의 Root로부터의 상대좌표는 40, 40, 40이 되었다.

![Untitled](/assets/InverseTransform/TransformSystem3.png)

그런데 만약, HeadTracker의 Transform이 단순히 Root의 자식이 아닌 다른 Transform 체계를 가지고 있다면 어떨까? 

![Untitled](/assets/InverseTransform/InverseTransform0.png)

알고 있는 정보는 오직 Head의 Root로부터의 상대좌표 뿐이고 그 좌표를 적용시켜야하는 HeadTracker는 Head와 다른 Transform 체계를 가지고 있다면 어떻게 접근해야할까?

## Inverse Transform

Inverse Transform으로 문제를 해결했다. Inverse Transform, 역행렬로 이해하면 된다.

$$ AA^{-1} = A^{-1}A = I $$

예를 들어 현재 Head가

$$A * B * C * D * H$$ 로 RelativeTransform을 가지고,

HeadTracker가

$$A * E * HT$$ 로 RelativeTransform을 가진다고 하자.

여기서 $HT$의 절대좌표가 $H$와 일치하게 하고 싶은데, 네트워크 레플리케이션 특성상 $H$의 Root에 대한 상대좌표(40,40,40) 밖에 모르는 상황이다. 다시 말해 $A$에서 $H$로 가는 Transform 밖에 모르고, $HT$는 이미 $E$라는 SceneComponent의 Child인 상황이다. 해결 방법은 의외로 간단한데, $A$에서 $H$로 가는 Trnasform에서 $A$에서 $HT$의 부모로 가는 Transform을 빼면 $HT$의 부모에서 $H$로 가는 Transform이 될 것이고, 이것을 다시 $HT$의 RelativeTransform으로 설정해주면 $HT$의 Transform을 $H$로 만들 수 있을 것이다. Vector의 뺄셈 연산과 똑같다. 다만, 이것은 행렬이라 역행렬을 구해서 곱해줘야 된다는 것만 다르다.

```cpp
FTransform AInverseTester::GetTransformTo(FTransform _Desired, USceneComponent* _Target, USceneComponent* _Root)
{
	//루트에서 HeadTracker의 부모까지가는 Transform
	FTransform RootToTargetParent = GetTransformFrom(_Target->GetAttachParent(), _Root);
	//HeadTracker의 부모에서 _Desire로 가는 Transform
	FTransform FromTo = _Desired * RootToTargetParent.Inverse();
	return FromTo;
}
```

![Untitled](/assets/InverseTransform/InverseTransform1.png)

아까 HeadTracker의 부모 OtherChild를 만들고, RelativeTransform에서 Location을 15,15,15로 설정했다. Head와 Head의 모든 부모의 상대좌표를 더하면 40,40,40이었으니, Head를 따라가는 15,15,15의 상대좌표를 가진 OtherChild의 자식인 HeadTracker의 상대좌표는 25,25,25가 되어야 한다.

![Untitled](/assets/InverseTransform/InverseTransform2.png)
잘 작동한다.

## 마무리
만약 작년의 나였다면 부모를 해제하고 직접 Root로부터의 좌표를 구해다가 넘기고 다시 부모설정하고 그랬을 것이다. 게임 수학에 나오는 개념들을 전부 다 활용해본 것은 아니지만 이렇게 하나둘씩 적용해보면 다 유용하게 쓰일 곳이 있을 것 같다. 사실 긴가민가 했었는데 ChatGPT한테 물어보고나서 이거 되겠는데? 하고 테스트 해봤고, 잘 작동했다.