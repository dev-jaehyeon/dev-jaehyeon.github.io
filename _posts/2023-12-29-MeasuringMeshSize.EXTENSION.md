---
title: Mesh의 Unreal Unit Size 구하기
date: 2023-12-29 09:00:00 +0900
categories: [Unreal]
tags: [unrealunit]     # TAG names should always be lowercase
math: true
mermaid: true
---
## 이야기
Procedural Ladder 만들기 파트를 공부하던 중, 오브젝트의 메쉬가 절차적으로 생성되게 하려면 오브젝트의 정확한 크기를 알아야 한다. 언리얼에서는 Unreal Unit(=cm)을 사용한다.

사다리를 만드는데, 절차적으로 생성되게 하려면 메쉬의 정확한 크기를 알고 거기에 맞게 다음 메쉬를 생성해야 할 것이다. 강의를 듣던 중, 메쉬 크기를 확인하는 방법을 메모해두면 좋을 것 같아서 포스트를 적어본다.

## Unreal Unit
Unreal에서 기본 단위는 Unreal Unit이고 기본적으로 1cm를 나타낸다. 이것은 Project Setting에서 바꿀 수 있다. 옛날 문서를 보면 Unreal Tournament 게임에서는 2cm이기도 했고, 어떤 게임 에서는 2Unreal Unit이 1 inch이기도 했다고 한다. 하지만 요즘은 이 단위는 잘 안 건드리는 것 같다.

언리얼에서 Mesh들의 크기를 보는 방법은 첫 번째로 StaticMesh 창에서 확인할 수 있다.
![Untitled](/assets/MeasurementMeshSize/StaticMeshMeasurement.png)
Approx Size에서 7x85x10이라고 되어 있는 부분인데, 이 Mesh는 x축으로 7, y축으로 85, z축으로 10의 크기를 가진 Mesh인 것이다.

두 번째 방법으로는 레벨이 올려 놓고 4분할 perspective view를 통해서 직접 확인할 수 있다. 우측 위 분할 버튼을 눌러 4분할한 다음, 마우스 가운데 버튼으로 직접 측정할 수 있다.
![Untitled](/assets/MeasurementMeshSize/MeasurmentMesh.gif)
![Untitled](/assets/MeasurementMeshSize/MeasurmentMesh2.gif)

실제로 높이는 10, 너비는 85인 것을 볼 수 있다.

하지만 이 수치가 cm 단위로 딱딱 떨어지게 만들어졌다면 이렇게 측정해도 상관은 없겠지만, 부동소수점까지 사용해서 그 수치가 integer로 전부 출력되기 힘든 경우가 있다. 그럴 때는 BoundingBox로부터 그 크기를 구한다. Mesh들은 BoundingBox를 반환할 수 있다.

```cpp
//LadderBase.h
//UStaticMesh* LadderPoleMesh로 사용해도 되지만 언리얼에서 TObjectPtr로 대체하도록 권장하고 있다고 한다. 이유는 아직 잘 모르겠지만 아래 참조
//https://husk321.tistory.com/377

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Ladder Base Setting")
TObjectPtr<UStaticMesh> LadderPoleMesh;

//LadderBase.cpp
void ALadderBase::BeginPlay()
{
	Super::BeginPlay();
	if (LadderPoleMesh)
	{
		UE_LOG(LogTemp, Warning, TEXT("%s"), *(LadderPoleMesh->GetBoundingBox().GetSize().ToString()));
	}
}
```

![Untitled](/assets/MeasurementMeshSize/LadderPoleSize.png)

높이에 해당하는 Z축으로의 길이가 8.404로 출력되는 것을 볼 수 있다. 강좌에서는 그냥 Approx Size의 8을 사용하던데, 이렇게 BoundingBox로부터 구한 정확한 길이를 사용하는게 더 맞을 것 같다.

## Test
그렇다면 실제로 메쉬를 만들어서 적용하면 어떻게 될까? 블렌더로 기본 큐브를 만들어서 임포트를 해보았다. 블렌더 상에서는 Dimension이 2m씩으로 설정되어 있다.
![Untitled](/assets/MeasurementMeshSize/BlenderCube.png)

언리얼로 가져와보면 Approx Size가 200x200x200으로 정확한 사이즈로 임포드된 것을 확인할 수 있다.
![Untitled](/assets/MeasurementMeshSize/BlenderCubeUnreal.png)


## 마무리
Unity에도 이렇게 정확하게 크기를 측정하는 기능이 있는지는 모르겠다. 일단 언리얼에서 이렇게 상당히 정확하게 크기를 측정할 수 있다면, Procedural한 오브젝트 생성에 위치 계산하기가 훨씬 편할 것이다.