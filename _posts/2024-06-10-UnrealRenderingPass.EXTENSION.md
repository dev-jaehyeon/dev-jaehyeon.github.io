---
title: Unreal Rendering Pass
date: 2024-04-28 09:00:00 +0900
categories: [Unreal, Study]
tags: [RenderDoc, RenderingPass]     # TAG names should always be lowercase
math: true
mermaid: true
---
## 이야기
참고 자료(<https://unrealartoptimization.github.io/book/profiling/passes/>)<br/>
참고 자료(<https://interplayoflight.wordpress.com/2017/10/25/how-unreal-renders-a-frame/>)<br/>
참고 자료(<https://www.youtube.com/watch?v=EF0YpKHfbAw>)<br/><br/>
프로젝트 최적화를 위해 Unreal Insight를 이리저리 만져보는 중, 최적화 과정에서 Rendering Pass라는 것에 대해 알게 되었다. Unreal이 어떻게 하나의 프레임을 그리는지 그 과정을 연구한 여러 글들을 참고했다.

Unreal에서 Pass란, 여러 드로우콜의 집합이다. 하나의 프레임을 렌더링하는데 필요한 일련의 과정에서 순서대로 실행되며 반투명 메쉬를 그린다거나 포스트 프로세싱을 적용하는 등 최종 렌더링 결과를 얻기 위해 호출된다.
<br/>
View Mode에서 Buffer Visualization을 선택하면 현재 어떤 버퍼(각 패스에서 렌더링된 결과들)이 그려지고 있는지 볼 수 있다.

![Untitled](/assets/RenderPass/ViewMode.png){: .w-50 .normal}
![Untitled](/assets/RenderPass/GBuffers.png)

위 링크에서 RenderDoc을 이용해서 Unreal Rendering Pass에 대해 설명한 글이 있는데, 흥미로운 정보라 한번 리뷰해봤다.


## RenderDoc

RenderDoc은 오픈소스 디버거로, 게임이나 그래픽을 사용하는 프로그램에서 프레임을 분석하는데 사용할 수 있는 도구다. 각종 API를 사용하는 대부분의 프로그램에 사용할 수 있다. 프로파일링 뿐만아니라 학습 차원에서 RenderDoc을 이용해 언리얼이 한 프레임을 렌더링하기 위해 어떤 과정을 거치는지도 분석할 수 있다.

RenderDoc에서 개인적으로 흥미로웠던 점은 공부했던 렌더링 파이프라인(입력조립기, 버텍스 셰이더, 레스터라이제이션 등등)을 기준으로 어떤 일이 일어나는지 보여준다는 것이다. 우선 공부용으로 간단히 만든 DX11 프로그램으로 텍스처가 입혀진 큐브를 렌더링한다.


![Untitled](/assets/RenderPass/firecube.png){: .w-50 .normal}

그리고 RenderDoc으로 캡처를 하면 이 큐브를 그리기 위한 과정을 볼 수 있다.

![Untitled](/assets/RenderPass/RenderDocMain.png)

특히 각 EID(Event Id)를 누르면 어떤 함수가 호출되는지도 볼 수 있는데, DrawIndexed를 보면 실제로 DX를 통해 호출한 함수들이 보인다!

![Untitled](/assets/RenderPass/RenderDocCallList.png){: .w-50 .normal}

위 큐브들은 아래의 함수들을 거쳐서 렌더링되는데, IASetVertexBuffer 등 호출 했던 함수들이 모두 캡처된 것을 볼 수 있다.

```cpp

void ModelClass::RenderBuffers(ID3D11DeviceContext* deviceContext)
{
	unsigned int stride;
	unsigned int offset;

	//Vertex Buffer stride, offset 설정
	stride = sizeof(VertexType);
	offset = 0;

	// Vertex Buffer가 렌더링될 수 있게 Input Assembler에서 활성화
	deviceContext->IASetVertexBuffers(0, 1, &m_vertexBuffer, &stride, &offset);

	// Vertex Buffer가 렌더링될 수 있게 Input Assembler에서 활성화
	deviceContext->IASetIndexBuffer(m_indexBuffer, DXGI_FORMAT_R32_UINT, 0);

	// Vertex Buffer로부터 그려질 primitive(삼각형)을 설정
	deviceContext->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);

	return;
}


void ShaderClass::RenderShader(ID3D11DeviceContext* deviceContext, int indexCount)
{
	//설정한 Input Layout을 입력조립기(InputAssembler)에서 활성화
	deviceContext->IASetInputLayout(m_layout);

	//Vertex Shader, Pixel Shader 설정
	deviceContext->VSSetShader(m_vertexShader, NULL, 0);
	deviceContext->PSSetShader(m_pixelShader, NULL, 0);

	//삼각형 그리기
	deviceContext->DrawIndexed(indexCount, 0, 0);

	return;
}
```

이렇게 RenderDoc을 이용해 언리얼의 렌더링 패스를 분석할 수 있는 것이다.


## Unreal RenderDoc  

언리얼에서는 편리하게 RenderDoc 플러그인만 활성화하면 어렵지 않게 씬을 캡처하여 RenderDoc으로 보낼 수 있다.

![Untitled](/assets/RenderPass/RenderDocPlugin.png){: .w-50 .normal}

활성화 후 에디터를 재시작 하면 우측 상단에 RenderDoc 캡처 버튼이 보인다. 이 버튼을 누르면 바로 캡처되어 RenderDoc이 실행된다.

![Untitled](/assets/RenderPass/UnrealEditorRenderDoc.png){: .w-50 .normal} 

실행하면 간단하게 DX로 렌더링하는 것보다 훨씬 많은 EID가 보인다. 여기서 EID 350~3755로 가장 많이 차지하는 Scene을 보면 렌더링 패스를 확인할 수 있다.

![Untitled](/assets/RenderPass/UnrealRenderDoc1.png)

Scene 하위로 또 Scene이 있는데, 여기를 열면 언리얼의 Rendering Pass들, PrePass, BuildHZB, BasePass, Light 등이 보인다.

![Untitled](/assets/RenderPass/EventList.png)

## 마무리
언리얼 안에서 stat GPU로도 각 패스가 현재 얼마나 시간이 걸리는지를 확인할 수 있다.

![Untitled](/assets/RenderPass/GPUDebug.png)

하지만 그 패스 안에서 어떤 항목에서 병목이 걸리는지는 역시 RenderDoc으로 더 자세히 볼 수 있다.(위 캡처는 루멘이 켜져 있어 루멘 관련 여러 패스가 추가되었음을 볼 수 있다.)
RenderDoc은 꽤 흥미로운 툴이고, 차후 그래픽스 공부할 때 게속 써볼 예정이다. 동시에 시간 날 때마다 Rendering Pass 한 항목씩 파볼 예정이다.