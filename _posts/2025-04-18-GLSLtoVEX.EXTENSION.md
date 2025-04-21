---
title: ShaderToy를 VEX로 구현하기
date: 2025-04-18 09:00:00 +0900
categories: [Houdini]
tags: [glsl, vex]     # TAG names should always be lowercase
math: true
mermaid: true
---
출처 및 참고 자료<br/>
ShaderToy
<https://www.shadertoy.com/view/MdXyzX>

## 개요
ShaderToy는 셰이더로 여러가지 영상이나 이미지를 만들고 올리는 사이트이다. GLSL 언어로 되어 있으며, 들었던 강좌 중에서는 GLSL을 HLSL로 변환하여 DirectX에서 실행하는 예제도 있다.<br/>
Houdini에서 사용하는 프로그래밍 언어는 VEX인데, Attribute Wrangle 노드에서 사용되는 VEX는 그 작동 방식이 셰이더와 아주 유사하다.(각 픽셀, 혹은 포인트에서 실행되는 점)<br/>
사실 별 의미는 없는 일이지만 VEX에 익숙해지는데 괜찮은 것 같아 포스팅 해본다.

## 1. ShaderToy
ShaderToy에서 적당한 작품 하나를 선정한다. 여기선 따로 복잡하지 않고 짧은 예제를 선택했다.(위 링크)
![](/assets/GLSLtoVEX/ocean.png){: w="800" h="200" .w-95 .normal}

이 바다는 무려 코드로 만들어져 있다.
```glsl
// Main
void mainImage(out vec4 fragColor, in vec2 fragCoord) {
  // get the ray
  vec3 ray = getRay(fragCoord);
  if(ray.y >= 0.0) {
    // if ray.y is positive, render the sky
    vec3 C = getAtmosphere(ray) + getSun(ray);
    fragColor = vec4(aces_tonemap(C * 2.0),1.0);   
    return;
  }
  //엄청 길다
}

```
## 2. Houdini Setting
Houdini에서 픽셀의 역할을 할 뷰포트를 만들기 위해 grid 노드를 꺼내고 row와 column을 맞춰준다. 여기선 800x600으로 설정했다.
![](/assets/GLSLtoVEX/grid.png){: w="600" h="450" .w-100 .normal}

그다음 transform 노드로 위 grid 해상도의 절반만큼 평행이동 해준다.
![](/assets/GLSLtoVEX/translate.png){: w="600" h="450" .w-100 .normal}

이제 Wrangle를 달아주고 코드 작업을 시작한다.

## 3. Attribute Wrangle
GLSL에서 사용되는 함수들이 거의 VEX에서도 똑같이 사용되어서 생각보다 바꿀 것이 크게 많지 않다. 하지만 문법적으로 다른 부분이 있어서 그 부분들을 조정해준다.

```glsl
vec2 wavedx(vec2 position, vec2 direction, float frequency, float timeshift) {
  float x = dot(direction, position) * frequency + timeshift;
  float wave = exp(sin(x) - 1.0);
  float dx = wave * cos(x);
  return vec2(wave, -dx);
}

vector2 wavedx(vector2 position; vector2 direction; float frequency; float timeshift) {
  float x = dot(direction, position) * frequency + timeshift;
  float wave = exp(sin(x) - 1.0);
  float dx = wave * cos(x);
  vector2 output = set(wave, -dx);
  return output;
}
```

- glsl은 vec2, vec3 를 쓰지만 vex는 vector2, vector를 쓴다.
- vec는 생성자의 개념이 없는듯 하다. 따라서 ``return vector(wave, -dx);``같은 문법이 오류가 난다. 따로 ``vector2 output = set(wave, -dx);``를 선언하여 반환하는 식으로 해야한다.
- vex는 uv를 직접 생성해야 한다.(인라인 코드에 vex는 없어서 일단 glsl로 해놓음) fit을 이용해 각 포인트의 위치값을 0에서 1까지로 해주면 uv가 쉽게 만들어진다.
```glsl
#define RESX 800  
#define RESY 450
float u = fit(@P.x, 0, RESX, 0, 1);
float v = fit(@P.z, 0, RESY, 0, 1);
u+= -0.5; //상황 봐가면서 조절
v+= -0.5; //상황 봐가면서 조절
vector2 fragCoord = set(u * RESX, v*RESY);
```
- 그 외 이름 다른 함수들의 이름을 바꿔준다. (예를 들어 mix는 lerp로)
- ShaderToy의 iTime은 @Frame으로 대체한다. 만약 함수들이 전역변수인 iTime을 인자로 받는다면 vex에서는 그 함수들이 iTime을 받을 수 있게 변경해줘야 한다.

## 4. 결과
![](/assets/GLSLtoVEX/vexocean.gif){: w="600" h="450" .w-100 .normal}
색감이 약간 안개낀 것 같은 느낌이고 uv도 조절 해야하지만 그럴 이유가 있는 작업은 아닌지라 일단 여기서 멈춘다.


