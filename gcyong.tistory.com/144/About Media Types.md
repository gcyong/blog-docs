# 미디어 타입에 관하여

*미디어 타입* 은 미디어 스트림 형식을 표현한다. 마이크로소프트 미디어 파운데이션에서의 미디어 타입이라 함은 [`IMFMediaType`](https://docs.microsoft.com/en-us/windows/desktop/api/mfobjects/nn-mfobjects-imfmediatype) 인터페이스로 나타내는 것을 의미한다. 이 인터페이스는 [`IMFAttributes`](https://docs.microsoft.com/en-us/windows/desktop/api/mfobjects/nn-mfobjects-imfattributes) 인터페이스를 상속하는데, 임의의 미디어 타입은 속성으로서 특정하기 때문이다.

새로운 미디어 타입을 만들기 위해 [`MFCreateMediaType`](https://docs.microsoft.com/en-us/windows/desktop/api/mfapi/nf-mfapi-mfcreatemediatype) 함수를 호출한다. 이 함수는 [`IMFMediaType`](https://docs.microsoft.com/en-us/windows/desktop/api/mfobjects/nn-mfobjects-imfmediatype) 인터페이스에 대한 포인터를 리턴하는데, 기본적으로 미디어 타입은 어떤 속성도 가지고 있지 않다. 대신, 자세한 형식을 지정하기 위해 관련 속성을 설정한다.

미디어 타입 속성에 대한 목록은 [미디어 타입 속성](https://docs.microsoft.com/ko-kr/windows/win32/medfound/media-type-attributes)을 참고하시라.

## 주 타입과 세부 타입

임의의 미디어 타입에 대한 두 가지 중요한 정보는 주 타입(major type)과 세부 타입(subtype)이다.

 * *주 타입* 은 미디어 스트림에서 데이터의 포괄적인 분류를 정의하는 GUID이다. 주 타입은 비디오와 오디오를 포함한다. 주 타입을 나타내기 위해 [`MF_MT_MAJOR_TYPE`](https://docs.microsoft.com/ko-kr/windows/win32/medfound/mf-mt-major-type-attribute) 속성을 설정한다. [`IMFMediaType::GetMajorType`](https://docs.microsoft.com/en-us/windows/desktop/api/mfobjects/nf-mfobjects-imfmediatype-getmajortype) 함수는 이 속성 값을 반환한다.
 * *세부 타입* 은 미디어 형식을 추가 정의한다. 가령, 비디오 주 타입에서 세부 타입은 RGB-24, RGB-32, YUV2 등이 있다. 한편, 오디오에서는 PCM 오디오, IEEE 부동 소수점 오디오 등이 있다. 세부 타입은 주 타입보다 더 상세한 정보를 제공하는데, 해당 미디어 형식에 대한 모든 점을 정의하지는 않는다. 가령, 비디오의 세부 타입은 이미지 크기나 프레임 비율을 정의하지 않는다. 세부 타입을 나타내기 위해 [`MF_MT_SUBTYPE`](https://docs.microsoft.com/ko-kr/windows/win32/medfound/mf-mt-subtype-attribute) 속성을 설정한다.

모든 미디어 타입은 주 타입 GUID와 세부 타입 GUID을 가지고 있어야한다. 주 타입과 세부 타입의 GUID 목록은 [미디어 타입 GUID](https://docs.microsoft.com/ko-kr/windows/win32/medfound/media-type-guids)을 참고하시라.

## 미디어 타입을 속성으로 나타내는 이유

타입을 속성으로 나타내는 점은 DirectShow나 Windows Media Format SDK와 같은 이전 기술에서 사용했던 형식 구조체(format structures)에 비해 갖는 몇 가지 장점이 있다.

 * 모르거나 상관하지 않는 값을 나타내기 더 쉽다. 가령, 비디오 변환 프로그램을 작성한다고 생각해보자. 이 경우, 변환을 지원할 형식이 RGB와 YUV인지는 미리 알 수 있으나 비디오 프레임의 차원(dimensions)의 경우 실제 비디오 소스를 받아보기 전 까지는 알아내기 힘들다. 비슷하게, 비디오 원색(video promaries)와 같은 부분에 대해서는 별로 중요하게 생각하지 않을 수도 있다. 형식 구조체에서는 모든 멤버를 적당한 값으로 채워야 한다. 따라서, 모르는 값이나 기본 값을 나타내는 0을 자주 사용하곤 한다. 이런 경향은 다른 구성 요소가 0을 의미있는 값으로 나타낼 경우 오류를 유발할 수 있다. 속성으로 나타내는 경우 구성 요소에서 모르거나 없어도 상관 없는 속성은 단순히 제외하면 된다.
 * 시간이 지나 요구 사항이 바뀔 때, 형식 구조체는 기존의 구조체 끝에다 추가할 데이터를 덧붙이면서 확장했었다. 가령, **`WAVEFORMATEXTENSIBLE`** 구조체는 **`WAVEFORMATEX`** 구조체를 확장한다. 이런 경향은 구성 요소가 구조체 포인터를 다룬 다른 구조체 타입으로 형 변환 해야 하기 때문에 오류가 발생하기 쉽다. 속성 기반에서는 안전하게 확장할 수 있다.
 * 상호 배재되는 불안정한 형식 구조체를 정의할 가능성이 있다. 가령, DirectShow는 **`VIDEOINFOHEADER`** 와 **`VIDEOINFOHEADER2`** 구조체를 정의한다. 속성은 상호 독립적으로 설정하기 때문에 이런 문제가 발생하지 않는다.

## 관련 주제

[미디어 타입 속성](https://docs.microsoft.com/ko-kr/windows/win32/medfound/media-type-attributes)

[미디어 타입](https://docs.microsoft.com/ko-kr/windows/win32/medfound/media-types)

## 원본

[About Media Types](https://docs.microsoft.com/ko-kr/windows/win32/medfound/about-media-types)