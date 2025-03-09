---
title: RenderDoc으로 보는 Rendering Pipeline - RS(Rasterizer)와 PS(Pixel Shader)
date: 2024-10-11 09:00:00 +0900
categories: [Graphics]
tags: [renderingpipeline, renderdoc]     # TAG names should always be lowercase
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

## Rasterizer (래스터화)

VS 다음은 테셀레이션 등이 있지만 필수적으로 필요한 단계도 아니고 이 예제에서는 구현하지 않으므로 생략한다. 그 다음은 Rasterizer 단계인데, 래스터화(Rasterizer) 단계에서는 프로그래머가 래스터화의 구성만 설정 가능하다. 래스터화 단계에서 어떤 일이 생기는지는 자료가 많으므로 여기선 RenderDoc에서 관찰할 수 있는 부분만 남긴다.

```cpp
bool D3DClass::Initialize(int screenWidth, int screenHeight, bool vsync, HWND hwnd, bool fullscreen, float screenDepth, float screenNear)
{
	//.
	//.
	//.
	D3D11_RASTERIZER_DESC rasterDesc;
	rasterDesc.AntialiasedLineEnable = false;
	rasterDesc.CullMode = D3D11_CULL_BACK;
	rasterDesc.DepthBias = 0;
	rasterDesc.DepthBiasClamp = 0.0f;
	rasterDesc.DepthClipEnable = true;
	rasterDesc.FillMode = D3D11_FILL_SOLID;
	rasterDesc.FrontCounterClockwise = false;
	rasterDesc.MultisampleEnable = false;
	rasterDesc.ScissorEnable = false;
	rasterDesc.SlopeScaledDepthBias = 0.0f;

	// 래스터화기 상태 설정
	result = m_device->CreateRasterizerState(&rasterDesc, &m_rasterState);
	if (FAILED(result))
	{
		return false;
	}

	m_deviceContext->RSSetState(m_rasterState);

	//뷰포트 설정
	m_viewport.Width = (float)screenWidth;
	m_viewport.Height = (float)screenHeight;
	m_viewport.MinDepth = 0.0f;
	m_viewport.MaxDepth = 1.0f;
	m_viewport.TopLeftX = 0.0f;
	m_viewport.TopLeftY = 0.0f;

	m_deviceContext->RSSetViewports(1, &m_viewport);
	//.
	//.
	//
}
```

RenderDoc에서도 코드로 설정한 RasterizerState가 출력되고 설정한 뷰포트 역시 확인할 수 있다.
![Untitled](/assets/RenderingPipeline/RSState.png)

## Pixel Shader(픽셀 셰이더)
픽셀 셰이더는 프로그래머가 작성하는 GPU 프로그램으로서 각 픽셀이 출력할 색상을 계산한다. RenderDoc의 API Inspector를 보면 PS에 해당하는 함수들을 확인할 수 있다.
![Untitled](/assets/RenderingPipeline/PSCommands.png)

#### ShaderResourceView (SRV, 셰이더 리소스 뷰)
Pixel Shader 단계에 해당하는 함수 호출 중에 `SetShaderResourceView`가 있다. SRV는 간단히 말해 셰이더가 읽을 수 있는 형식의 텍스처를 말한다. 예제대로 하면 targa파일을 불러서 초기화 하는데, 여기선 png를 불러오기 위해 [**DirectXTK**](https://github.com/Microsoft/DirectXTK)을 사용한다. DirectXTK의 `CreateWICTextureFromFile`를 이용하면 간단하게 png 파일을 불러올 수 있다.

```cpp

bool TextureClass::InitializeWIC(ID3D11Device* _device, const wchar_t* _filename)
{
	//헤더 파일에 SRV 미리 선언: ID3D11ShaderResourceView* m_textureView; 

	ID3D11Resource* resource;
	HRESULT hResult = CreateWICTextureFromFile(_device, _filename, &resource, &m_textureView);
	
	if (FAILED(hResult))
	{
		MessageBox(m_hwnd, L"Loading Texture FAiled", L"Error", MB_OK);
		return false;
	}
	return true;
}

```
SRV를 생성했으면 `PSSetShaderResources`로 픽셀 셰이더에 텍스처를 넘겨준다.

```cpp
bool ShaderClass::SetTextureShaderParameters(ID3D11DeviceContext* deviceContext,
	XMMATRIX worldMatrix, XMMATRIX viewMatrix, XMMATRIX projectionMatrix, ID3D11ShaderResourceView* texture)
{
	//Vertex Shader Constant Buffer 설정
	//.
	//.

	//SRV 설정
	deviceContext->PSSetShaderResources(0, 1, &m_textureView);

	//.
	//.
}
```

#### SamplerState
Sampling, 샘플링은 텍스처 좌표에서 필요한 색상 값을 가져오는 것을 말한다. 즉, 여러 데이터에서 몇 개만 가져오는 것을 의미한다. SampleState를 통해 원본 텍스처로부터 어떤 픽셀들이 사실 상 그려지는지를 알아낼 수 있다. 이 SampleState도 PS 단계에서 설정된다.

```cpp
bool ShaderClass::InitializeTextureShader(ID3D11Device* _device, HWND _hwnd, const wchar_t* _texFilename)
{
	//.
	//.
	//헤더파일에서 샘플러명세 선언: D3D11_SAMPLER_DESC samplerDesc;

	samplerDesc.Filter = D3D11_FILTER_MIN_MAG_MIP_LINEAR;
	samplerDesc.AddressU = D3D11_TEXTURE_ADDRESS_WRAP;
	samplerDesc.AddressV = D3D11_TEXTURE_ADDRESS_WRAP;
	samplerDesc.AddressW = D3D11_TEXTURE_ADDRESS_WRAP;
	samplerDesc.MipLODBias = 0.0f;
	samplerDesc.MaxAnisotropy = 1;
	samplerDesc.ComparisonFunc = D3D11_COMPARISON_ALWAYS;
	samplerDesc.BorderColor[0] = 0;
	samplerDesc.BorderColor[1] = 0;
	samplerDesc.BorderColor[2] = 0;
	samplerDesc.BorderColor[3] = 0;
	samplerDesc.MinLOD = 0;
	samplerDesc.MaxLOD = D3D11_FLOAT32_MAX;

	result = _device->CreateSamplerState(&samplerDesc, &m_sampleState);

	//.
	//.
}
```

간단히 ResourceView(Texture)와 SamplerState가 설정되면 셰이더에서도 선언하여 사용할 수 있게 된다.
```hlsl
Texture2D shaderTexture : register(t0);
SamplerState SampleType : register(s0);
```
hlsl에서 선언한 텍스처와 샘플스테이트가 RenderDoc에서 확인된다.

![Untitled](/assets/RenderingPipeline/PSPipelineState.png)