---
title: Houdini에서 RBD 시뮬레이션을 Unreal로 가져오기 -VAT
date: 2025-03-03 09:00:00 +0900
categories: [Houdini]
tags: [VAT]     # TAG names should always be lowercase
math: true
mermaid: true
---
출처 및 참고 자료<br/>
Houdini Youtube 채널
<https://www.youtube.com/watch?v=3ep9mkwiOjU>

## 개요
Houdini에서는 다양한 파티클과 물리 시뮬레이션을 구현할 수 있다. 이 포스트는 Houdini로 만든 RBD 시뮬레이션을 Unreal로 가져오는 방법에 대한 정리다.<br/>
기존에 Alembic 파일로 가져오는 방법은 결국 Unreal 상에서 Skeletal Mesh로 가져오는데, 최적화에 더 도움이 되는 방법으로 가져올 수 있다. 바로 Virtual Animation Texture, VAT이다.



## 1. VAT
Virtual Animation Texture의 약자로, 기본 원리는 텍스처에 각 Vertex의 프레임별 위치를 저장해두고 셰이더에서 각 Vertex의 위치를 조정해주는 것이다. 해당 텍스처는 이런식으로 생겼다:
![](/assets/HoudiniToUnreal_VAT/VATRot.png){: w="600" h="200" .w-75 .normal}

이는 Skeletal Mesh나 유니티의 Skinned Mesh의 애니메이션들을 베이킹할 때도 사용된다.

## 2. Houdini SideFX Lab
VAT는 Houdini의 SideFX Lab에서 추출할 수 있다. SideFX Lab은 Houdini를 설치할 때 추가할 수 있고, Houdini Github에서도 받을 수 있다. 그러면 좌측 Shelf 탭에 SideFX Lab과 VAT가 보인다.
![](/assets/HoudiniToUnreal_VAT/Shelf.png){: w="800" h="200" .w-95 .normal}

VAT를 누르면 out에 Vertex Animation Texture 노드가 생성된다.
![](/assets/HoudiniToUnreal_VAT/VATNode.png){: w="300" h="200" .w-75 .normal}

## 3. Vertex Animation Texture Import
RBD 파괴 시뮬레이션을 준비한다.
![](/assets/HoudiniToUnreal_VAT/CollapsingHouseOfHoudini_VAT.gif){: w="600" h="450" .w-100 .normal}

그 후, Vertex Animation Texture 노드에서 세팅을 해준다.
- Input Geometry-> RBD 아웃의 경로

### 3-1 Setting 탭
![](/assets/HoudiniToUnreal_VAT/Setting.png){: w="300" h="400" .w-75 .normal}
- Rigid-Body Dynamics에서 Pivot Accuracy를 ``Maximum (Must Turn On Use Full Precision UVs in Unreal)``
- All Modes에서 ``Input Geometry is Cached to Integer Frames: false`` (만약 캐시를 했다면 체크)

### 3-2 Input 탭
![](/assets/HoudiniToUnreal_VAT/Input.png){: w="300" h="400" .w-75 .normal}
- Attribute Auto Generation에서 UV Generation을 ``Compute Missing UVs using UV Unwrap``

이렇게 세팅하고 Render All을 눌러주면 fbx와 VAT들이 생성된다. 생성 경로는 ``Export`` 탭에서 확인한다.

## 4. Unreal에서 SideFX Lab 플러그인 설치
Unreal에서도 SideFX Lab을 설치해줘야 되는데, 만약 직접 Vertex Animation Texture의 데이터로 Material에서 Vertex를 움직이고자 한다면 딱히 필요는 없지만 매우 번거롭기 때문에 플러그인을 설치하는 것이 좋다.<br/>
Houdini의 ``Vertex Animation Texture``노드에서 ``Real-Time Shaders`` 탭을 보면 ``Unreal Engine Content Plugin and Guides``가 있다. <br/>
![](/assets/HoudiniToUnreal_VAT/Plugin.png){: w="300" h="400" .w-75 .normal}

이 버튼을 클릭하면 Unreal의 버전에 맞는 플러그인들이 있다.<br/>
![](/assets/HoudiniToUnreal_VAT/Plugin2.png){: w="300" h="400" .w-85 .normal}

개인적으로 Unreal 5.3을 쓰고 있는데, 여기에 그냥 5.0 플러그인을 설치해도 크게 문제는 없었다.<br/>
![](/assets/HoudiniToUnreal_VAT/SideFXPlugin.png){: w="700" h="400" .w-95 .normal}

## 5. Unreal에 Import
생성한 geo와 tex 폴더를 가져오면 된다.<br/>
![](/assets/HoudiniToUnreal_VAT/geotex.png){: w="300" h="400" .w-85 .normal}

tex폴더 안에 있는 텍스처들의 설정을 바꿔줘야 되는데, 각 텍스처를 열어서 Compression의 Compression Settings를 ``HDR High Precision RGBA32F``로 해준다.
![](/assets/HoudiniToUnreal_VAT/compress.png){: w="300" h="400" .w-85 .normal}

## 6. VAT Material
전용 머티리얼을 만들어야 한다. SideFX Lab 플러그인이 제대로 설치 되었다면 전용 Material Function들이 보인다.
![](/assets/HoudiniToUnreal_VAT/VATMF.png){: w="300" h="400" .w-85 .normal}

Material Result Node Detail에서 Material탭의 Advanced에서 ``Num Customized UVs``를 5로 설정해준다.<br/>
![](/assets/HoudiniToUnreal_VAT/UV5.png){: w="300" h="400" .w-85 .normal}

``MF_VAT_RigidBodyDynamics``노드를 꺼내서 World Position Offset과 Customized UV1~4까지 연결해준다. Result Node의 Customized UV0은 비워둔다.<br/>
![](/assets/HoudiniToUnreal_VAT/connection.png){: w="500" h="550" .w-85 .normal}

그외 나머지 텍스처들을 연결해주면 된다.<br/>
![](/assets/HoudiniToUnreal_VAT/rest.png){: w="500" h="550" .w-85 .normal}

Material Instance를 만들면 Vertex Animation Texture를 넣을 수 있는 부분들이 보인다. position과 rotation 텍스처를 채워준다.<br/>
![](/assets/HoudiniToUnreal_VAT/mi.png){: w="700" h="900" .w-95 .normal}

이제 머티리얼을 메쉬에 채워주면 완성이다.<br/>
![](/assets/HoudiniToUnreal_VAT/CollapsingHouseOfHoudini_VATResult.gif){: w="700" h="900" .w-95 .normal}

## 마무리
VAT를 이용하면 Static Mesh로도 파괴 시뮬레이션을 돌릴 수 있다. 이미 Skeletal Mesh Animation을 베이킹하는 방법도 많으니 복잡하게 제어할 필요가 없는 오브젝트라면 VAT를 활용해서 Static Mesh로 굽는 것도 좋은 방법이다.