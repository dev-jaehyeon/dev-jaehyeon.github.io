---
title: Net Cull Distance
date: 2023-10-21 14:32:00 +0900
categories: [Unreal, Study]
tags: [network, net, replication]     # TAG names should always be lowercase
math: true
mermaid: true
---

## 이야기

  회사에서 작업하던 도중, 도저히 모르겠던 버그가 하나 있었다. 프로젝트에는 보이스챗 기능이 들어가 있는데, 그동안 잘 작동하던 것이 갑자기 작동을 하지 않았다. 애초에 보이스챗 플러그인을 설치해서 사용하고 있었고 몇 달 전에 만들어 놓고 그동안 건드리지 않았는데 갑자기 작동하지 않아서 커밋을 하나하나 확인한 결과, 플레이어 폰의 위치가 바뀐 커밋에서 보이스챗 기능이 작동하지 않았다.

  경고 메시지를 거슬러 올라가 원인을 찾아보니 보이스챗 액터의 컴포넌트들이 갑자기 null이 되었다면서 보이스챗이 작동하지 않은 것이었는데, 이게 왜 갑자기 작동을 하지 않았던 것일까?

  원인은 Net Cull Distance에 있었다.

## Net Cull Distance
![Untitled](/assets/UENetCullDistance/NetReplicationExplain.png)

Net Cull Distance의 설명을 보면 일정 거리 안에 있어야 레플리케이션이 이루어진다고 되어 있다. 이 옵션은 액터의 디테일 패널에 있고 Net Cull Distance Squared라고 되어 있다.

![Untitled](/assets/UENetCullDistance/NetCullDistanceOption.png)

넷 컬 디스턴스 제곱이라고 되어 있는데, 저 225,000,000의 제곱근은 15,000 언리얼 유닛이고 해당 액터와 클라이언트 사이의 거리가 저 안에 있어야 레플리케이션이 된다는 것이다.

## 테스트

  간단하게 Dedicated Server를 빌드 후, 클라이언트와 에디터로 들어간다.  캐릭터는 int32 번수 ReplicatedNum을 가지고 있고 1초에 1씩 증가시킨 값을 서버로 RPC 호출한다.
  또한, 자신이 아닌 플레이어를 찾아 그 플레이어의 폰과 자신의 폰과의 거리를 갱신한다.

```cpp
if (IsLocallyControlled() == true)
{
	 Server_SetReplicatedNum(ReplicatedNum +1); //RPC
}
```

```cpp
double Dist = FVector::Distance(GetActorLocation(), OtherPawn->GetActorLocation());
Distance = (float)Dist;
```

![Untitled](/assets/UENetCullDistance/ConfirmingReplication.png)

이제 거리가 15,000을 넘어가면 Net Cull Distance를 넘어서 더 이상 레플리케이션이 되지 않을 것이다.

![Untitled](/assets/UENetCullDistance/NetCullDistanceTest.gif)

## 마무리

  결국 Net Cull Distance 때문에 플레이어의 폰이 보이스챗 액터와 멀어짐에 따라 레플리케이션으로 서버와 클라이언트에 보이스챗 기능을 제공하던 액터가 작동을 하지 않게 되어 보이스챗 기능이 작동하지 않았던 것이었다. 이는 간단하게 보이스챗 액터가 소유 플레이어의 폰을 따라다니게 만들어 해결하였다.
