---
title: Houdini에서 RBD 시뮬레이션을 Unreal로 가져오기 -Alembic
date: 2025-04-15 09:00:00 +0900
categories: [Houdini]
tags: [alembic]     # TAG names should always be lowercase
math: true
mermaid: true
---


## 개요
Houdini에서는 다양한 파티클과 물리 시뮬레이션을 구현할 수 있다. 이 포스트는 Houdini로 만든 RBD 시뮬레이션을 Unreal로 가져오는 방법에 대한 정리다.<br/>
Houdini와 Unreal이 서로 같은 3D 에셋을 사용하기 위해서는 맞춰야 하는 일련의 설정들이 있다.

## 1. 닫힌 메쉬 사용
Houdini에서 RBD 시뮬레이션을 하기 위해서는 닫힌 메쉬를 사용해야 한다.<br/>
일반적으로 게임 엔진에서는 Backface를 렌더링하는 경우는 특수한 경우를 빼고는 거의 없다. 특히 레벨에 배치된 Static Object들은 더더욱 그렇다. 플레이 중에 눈에 보이지도 않고 움직이지도 않기 때문에 한 면만 만들고 반대편은 렌더링하지도 않는다.

![](/assets/HoudiniToUnreal/FrontAndBack.png){: w="600" h="200" .w-75 .normal}

<br/>
하지만 Houdini에서 RBD 시뮬레이션을 하려면 **닫힌** 메쉬여야 한다. Houdini에서 Fracture를 적용할 때 메쉬가 닫혀있지 않으면 제대로 나눠지지 않는다. 노말도 이상하게 적용될 뿐더러 찢어진 메쉬까지 보인다.
![](/assets/HoudiniToUnreal/NonClosed.png){: w="250" h="200" .w-75 .normal}

Extrude, fill 등의 기능으로 면을 채워준다.
![](/assets/HoudiniToUnreal/BlenderClose.png){: w="600" h="200" .w-75 .normal}
_Houdini에서도 닫힌 메쉬로 만들 수 있지만 개인적으로 Blender 같은 툴에서 작업하는게 더 편했다_
<br/>

## 2. Triangulate
Unreal은 Alembic 파일을 읽을 때 삼각형 또는 사각형 메쉬밖에 읽지 못한다.
> 면이 5버텍스로 구성되어 있어 메시를 임포트할 수 없습니다. 트라이앵글(3) 또는 쿼드(4)여야 합니다.

fbx 파일을 읽을 때는 알아서 Triangulate를 해주지만, abc 파일은 처음부터 Triangulate가 되어있어야 한다. Triangulate를 하는 방법은:
- Houdini에서 `divide` 노드에서 Maximum edge = 3
- Houdini에서 `remesh` 노드에서 Iteration = 0

![](/assets/HoudiniToUnreal/divide.png){: w="800" h="600" .w-75 .normal}

Blender에서도 Edit Mode에서 Ctrl + T로 간단히 Triangulate가 되지만 Houdini에서 Fracture를 적용한 다음에 어차피 다각형이 생기기 때문에 위 노드 중 하나는 꼭 넣어줘야 Unreal에서 읽을 수 있다.

## 3. 작업 (RBD Simulation)
메쉬가 준비되었으면 시뮬레이션 작업을 한다. 여기선 간단하게 리지드바디만 적용하여 무너지는 집을 시뮬레이션 했다.<br/>
![](/assets/HoudiniToUnreal/CollapsingHouseOfHoudini.gif)

## 4. Material 보존
Houdini로 가져올 에셋에 Material이 할당되어 있었다면, prim Attribute에 shop_materialpath라는 Attribute가 생성되어 있다. 그렇지 않다면 직접 Material들이 할당될 슬롯들을 Houdini에서 만들어야 하는데, 이는 개인적으로 Blender같은 툴들이 더 편하기 때문에 여기선 미리 Material이 할당되어 있는 모델로 진행한다.

![](/assets/HoudiniToUnreal/shopmaterialpath.png){: w="600" h="400" .w-75 .normal}

<br/>
dop network 등으로 물리 시뮬레이션을 했다면 sop network에 `dop import` 노드가 생성되어 있다. `dop import` 밑에 `convert` 및 `name` 노드를 차례로 연결해준다.<br/>
`name` 노드에서 할당된 Material 만큼 name을 만들어준다.

![](/assets/HoudiniToUnreal/namenode.png){: w="600" h="400" .w-75 .normal}

<br/>
그 다음 `rop alembic` 노드를 이용해 alembic파일로 저장하면 된다. 2초짜리 시뮬레이션이라 0에서 48프레임까지의 범위로 저장하였다. path attribute 항목에 생성했던 name 을 넣어준다. 그 후 save disk를 눌러주면 abc(alembic)파일이 생성된다.

![](/assets/HoudiniToUnreal/ropalembic.png){: w="600" h="400" .w-75 .normal}

## 5. Unreal로 Import
생성한 abc(alembic) 파일을 임포트하면 된다.
![](/assets/HoudiniToUnreal/Import.png){: w="250" h="200" .w-75 .normal}

Import Setting에서
- Alembic->임포트 타입 : `Skeletal Mesh`
- 컨버전->스케일 : `1.0, 1.0, 1.0`

으로 해주면 된다. 만약 메쉬가 누워있거나 그러면 회전에서 값을 조정해주면 된다. 이번 작업에서는 컨버전->회전 : `90, -90, 0` 으로 하니 잘 맞게 임포트되었다.
<br/>
![](/assets/HoudiniToUnreal/ImportSettings.png){: w="600" h="400" .w-75 .normal}

<br/>
임포트하면 `스켈레탈 메쉬(Skeletal Mesh)` 와 `애니메이션 시퀀스(Animation Sequence)` 그리고 `스켈레톤(Skeleton)` 이 생성된다. `스켈레탈 메쉬` 에 모프 타겟이 엄청나게 많이 생긴 것을 볼 수 있다.(용량도 엄청나다..)<br/>
![](/assets/HoudiniToUnreal/ImportedMesh.png){: w="600" h="400" .w-75 .normal}

<br/>
모든 모프 타깃이 0으로 설정되어 있어 메쉬가 엉망으로 보이지만 `애니메이션 시퀀스`를 열어보면 모든 모프 타겟의 그래프가 설정되어 있는 것을 볼 수 있다.
![](/assets/HoudiniToUnreal/Importedanim.png){: w="600" h="400" .w-75 .normal}

<br/>
다시 `스켈레탈 메쉬`에서 Material들을 할당해주면(Material 이름들은 따라오지 않아 직접 넣어가며 뭐가 맞는지 일일이 넣는 수 밖에 없었다..) 준비가 완료된다.
![](/assets/HoudiniToUnreal/materialslots.png){: w="600" h="400" .w-75 .normal}

<br/>
이제 씬으로 `애니메이션 시퀀스`를 드래그 앤 드롭(혹은 `스켈레탈 메쉬`를 드래그 앤 드롭 한 후 애니메이션 할당)한 후 게임을 실행하면 건물이 무너지는 것을 볼 수 있다.
![](/assets/HoudiniToUnreal/CollapsingHouse.gif){: w="600" h="400" .w-75 .normal}

## 마무리
이후는 블루프린트, 시퀀서, 코드 등을 이용해 애니메이션을 제어하면 된다. alembic 캐시 때문에 용량이 엄청난 것은 염두하고 개발해야할 것 같다.



