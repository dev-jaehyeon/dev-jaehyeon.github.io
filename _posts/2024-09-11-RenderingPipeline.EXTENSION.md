---
title: RenderDoc으로 보는 Rendering Pipeline - IA(Input Assembly)
date: 2024-09-11 09:00:00 +0900
categories: [Graphics, Study]
tags: [renderingpipeline, RenderDoc]     # TAG names should always be lowercase
math: true
mermaid: true
---
출처 및 참고 자료<br/>
<https://unrealartoptimization.github.io/book/profiling/passes/><br/>
어서와 GPU 최적화는 처음이지 - 비엘북스<br/>
유니티로 배우는 게임 수학 - 한빛미디어<br/>
DirectX 12를 이용한 3D 게임 프로그래밍 입문 - 한빛미디어<br/>
<https://velog.io/@syiee><br/>
<https://www.rastertek.com/dx11win10tut04.html><br/>

## RenderDoc으로 보는 렌더링 파이프라인
프랭크 루나 책의 표현을 가져오면 3차원 장면의 기하학적 서술과 가상 카메라의 위치 및 방향이 주어졌을 때, 현재 가상 카메라에 비친 3차원 장면의 모습에 근거해서 2차원 이미지를 생성하는데 필요한 일련의 단계들 전체를 말한다.<br/> 
한마디로 지오메트리, 텍스쳐 등 여러 자원들의 데이터를 이용해 모니터 스크린의 픽셀에 출력하는 과정이라고 할 수 있다.<br/>

아래는 간단한 메쉬(Box)에 텍스처를 입혀서 출력하는 예제를 Renderdoc으로 캡쳐한 것이다.<br/>
Pipeline State로 가보면 어떤 단계를 거쳐 화면이 렌더링되었는지 볼 수 있다.

![Untitled](/assets/RenderingPipeline/PipelineState.png)

- IA(Input Assembly): 입력 조립기
- VS(Vertex Shader): 정점 셰이더
- Hull Shader: 덮개 셰이더
  - Tesselator: 테셀레이터  
- DS(Domain Shader): 영역 셰이더
- Rasterizer
- PS(Pixel Shader): 픽셀 셰이더
- OM(Output Merger): 출력 병합기

간단한 렌더링 예제는 Hull Shader, Tesselation이 없이도(DX11에서 추가되었고 그 전엔 없었다고 한다) 렌더링 할 수 있다.

## Input Assembler (입력 조립기)
### 개요
IA 단계에서는 메모리에서 Vertex와 Index 등을 읽어 Primitive(삼각형, 선분, 점 등)을 조립한다.<br/>
API Inspector로 보면 IA관련 함수들이 호출된 것을 볼 수 있다.
![Untitled](/assets/RenderingPipeline/IACommands.png)

`IASetVertexBuffer`, `IASetIndexBuffer`, `IASetPrimitiveTopology` 그리고 `IASetInputLayout` 함수를 호출하여 최종적으로 정점의 레이아웃대로 버퍼들을 활성화하면 비로소 셰이더에서 사용할 준비가 되는 것이다.<br/>
버퍼들은 일반적으로 모델 파일로부터 채운다.

### Vertex Buffer, Index Buffer
Vertex, Index Buffer들은 일반적으로는 모델 파일로부터 가져오지만 수동으로 넣는 방법도 있다.
```cpp
VertexType* vertices;
unsigned long* indices;
D3D11_BUFFER_DESC vertexBufferDesc, indexBufferDesc;
D3D11_SUBRESOURCE_DATA vertexData, indexData;
HRESULT result;
int i;

vertices = new VertexType[m_vertexCount];

indices = new unsigned long[m_indexCount];

// Vertex 배열과 Index 배열 채우기, 예제에서는 직접 텍스트 파일로 정점을 채우고 색인은 그냥 순차적으로 채워넣었다

// Vertex Buffer Desc 채우기
vertexBufferDesc.Usage = D3D11_USAGE_DEFAULT;
vertexBufferDesc.ByteWidth = sizeof(VertexType) * m_vertexCount;
vertexBufferDesc.BindFlags = D3D11_BIND_VERTEX_BUFFER;
vertexBufferDesc.CPUAccessFlags = 0;
vertexBufferDesc.MiscFlags = 0;
vertexBufferDesc.StructureByteStride = 0;

vertexData.pSysMem = vertices;
vertexData.SysMemPitch = 0;
vertexData.SysMemSlicePitch = 0;

//Vertex Buffer 만들기기
result = device->CreateBuffer(&vertexBufferDesc, &vertexData, &m_vertexBuffer);

//Index Buffer도 만들기
//.
//.

```

### Input Layout
Vertex와 Index Buffer들을 채우고 Vertex Shader를 컴파일을 하고나면 Vertex Shader가 사용할 Input Layout을 설정한다. 이 예제에서는 POSITION, TEXCOORD, NORMAL을 사용하므로 3개의 레이아웃을 만든다.
```cpp
D3D11_INPUT_ELEMENT_DESC polygonLayout[2];

polygonLayout[0].SemanticName = "POSITION";
polygonLayout[0].SemanticIndex = 0;
polygonLayout[0].Format = DXGI_FORMAT_R32G32B32_FLOAT;
polygonLayout[0].InputSlot = 0;
polygonLayout[0].AlignedByteOffset = 0;
polygonLayout[0].InputSlotClass = D3D11_INPUT_PER_VERTEX_DATA;
polygonLayout[0].InstanceDataStepRate = 0;

//SementicName이 TEXCOORD, NORMAL인 Layout을 더 만든다
//
//

numElements = sizeof(polygonLayout) / sizeof(polygonLayout[0]);

//Vertex Input Layout 만들기
result = device->CreateInputLayout(polygonLayout, numElements, vertexShaderBuffer->GetBufferPointer(), 
								   vertexShaderBuffer->GetBufferSize(), &m_layout);
```
<br/>
이렇게 만들고 나면 RenderDoc에서도 Input Layouts에 POSITION, TEXCOORD, NORMAL 등이 보인다.
![Untitled](/assets/RenderingPipeline/IAInputLayout.png)

<br/>
마지막으로 `IASetVertexBuffer`, `IASetIndexBuffer`, `IASetPrimitiveTopology` 그리고 `IASetInputLayout` 들을 호출해주면 Vertex Shader 단계에서 사용할 데이터가 준비된다.

```cpp
void ModelClass::RenderBuffers(ID3D11DeviceContext* deviceContext)
{
	unsigned int stride;
	unsigned int offset;

	stride = sizeof(VertexType);
	offset = 0;

	//Vertex Buffer 설정
	deviceContext->IASetVertexBuffers(0, 1, &m_vertexBuffer, &stride, &offset);
	//Index Buffer 설정
	deviceContext->IASetIndexBuffer(m_indexBuffer, DXGI_FORMAT_R32_UINT, 0);
	//기본도형 위상구조 설정, D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST를 변경하여 점 목록, 선 띠, 선 목록 등으로 조립하라고 지정할 수 있다.
    //여기서는 삼각형으로 조립하라고 지정한다.(TRIANGLELIST)
	deviceContext->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);

	return;
}

void ShaderClass::RenderShader(ID3D11DeviceContext* deviceContext, int indexCount)
{
	// Vertex Input Layout 설정
	deviceContext->IASetInputLayout(m_layout);
    //VS, PS, SampleState 설정 후 DrawIndexed를 호출하여 렌더
    //
    //
}
```

## 마무리
Input Layout이 설정되면 다음 단계인 Vertex Shader 단계에서 Input Layout을 토대로 프로그래머가 작성한 Vertex Shader를 호출하여 VS_OUT 결과물을 만들게 된다. 