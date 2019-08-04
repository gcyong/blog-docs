# 미디어 파운데이션 속성

속성은 키와 값으로 이루어진 쌍인데, 키는 GUID이고 값은 `PROPVARIANT`이다. 속성은 마이크로소프트 미디어 파운데이션 전반에 사용되는데, 객체를 설정할 뿐만 아니라 미디어 서식을 나타내고 객체 속성을 질의하며, 그 밖에 다른 목적으로도 사용할 수 있다.

이번 주제에서는 하기하는 항목에 대해 설명한다.

 * 속성에 대하여
 * 속성 직렬화 하기
 * `IMFAttributes` 구현하기
 * 관련 주제

## 속성에 대하여

속성은 키와 값으로 이루어진 쌍인데, 키는 GUID이고 값은 `PROPVARIANT`이다. 속성값은 다음 데이터 타입으로 한정한다.

 * 부호 없는 32비트 정수 (`UINT32`)
 * 부호 없는 64비트 정수 (`UINT64`)
 * 64비트 부동 소수점수
 * GUID
 * 와이드 문자로 이루어진 널 종료 문자열 (Null-terminated wide-character string)
 * 바이트 열
 * `IUnknown` 포인터

이런 타입은 [`MF_ATTRIBUTE_TYPE`](https://docs.microsoft.com/en-us/windows/desktop/api/mfobjects/ne-mfobjects-_mf_attribute_type) 열거형에서 정의한다. 속성 값을 설정하거나 조사하기 위해서 [`IMFAttributes`](https://docs.microsoft.com/en-us/windows/desktop/api/mfobjects/nn-mfobjects-imfattributes) 인터페이스를 사용한다. 이 인터페이스는 특정 데이터 타입으로 이루어진 값을 얻고 설정하기 위한 메소드를 포함하는데, 이런 메소드들은 타입에 안전하다. 가령, 32비트 정수를 설정하기 위해 [`IMFAttributes::SetUINT32`](https://docs.microsoft.com/en-us/windows/desktop/api/mfobjects/nf-mfobjects-imfattributes-setuint32) 메소드를 호출하시라. 속성에서 키는 속성을 관리하는 객체 내에서 유일하다. 만일 같은 키로 서로 다른 두 값을 설정할 경우 뒤에 설정하는 값이 앞서 설정한 값을 덮어 쓸 것이다.

일부 미디어 파운데이션 인터페이스는 [`IMFAttributes`](https://docs.microsoft.com/en-us/windows/desktop/api/mfobjects/nn-mfobjects-imfattributes)를 상속한다. 이 인터페이스를 갖는 객체는 필수적이거나 추가적인 속성을 가지는데, 애플리케이션이 해당 객체에서 반드시 설정하거나, 또는 애플리케이션이 조사할 수 있는 속성을 가질 수 있도록 한다. 또한, 일부 메소드와 함수는 `IMFAttritubes` 포인터를 파라미터로 전달하도록 요구하는데, 이는 애플리케이션이 환경 설정 정보를 설정할 수 있도록 한다. 애플리케이션은 반드시 환경 설정 속성을 가지도록 속성을 저장해두는 장소를 만들어 두어야 한다. 빈 속성 저장소를 만들기 위해 `MFCreateAttributes` 함수를 호출하시라.

다음 코드는 두 개의 함수를 보여주고 있다. 첫 번째 함수는 새로운 속성 저장소를 만들고 문자열 값을 저장하는 `MY_ATTRIBUTE`라고 이름 붙인 가상의 속성을 설정한다. 두 번째 함수는 이 속성 값을 조사한다.

```C++
extern const GUID MY_ATTRIBUTE;

HRESULT ShowCreateAttributeStore(IMFAttributes **ppAttributes)
{
    IMFAttributes *pAttributes = NULL;
    const UINT32 cElements = 10;  // Starting size.

    // Create the empty attribute store.
    HRESULT hr = MFCreateAttributes(&pAttributes, cElements);

    // Set the MY_ATTRIBUTE attribute with a string value.
    if (SUCCEEDED(hr))
    {
        hr = pAttributes->SetString(
            MY_ATTRIBUTE,
            L"This is a string value"
            );
    }

    // Return the IMFAttributes pointer to the caller.
    if (SUCCEEDED(hr))
    {
        *ppAttributes = pAttributes;
        (*ppAttributes)->AddRef();
    }

    SAFE_RELEASE(pAttributes);

    return hr;
}

HRESULT ShowGetAttributes()
{
    IMFAttributes *pAttributes = NULL;
    WCHAR *pwszValue = NULL;
    UINT32 cchLength = 0;

    // Create the attribute store.
    HRESULT hr = ShowCreateAttributeStore(&pAttributes);

    // Get the attribute.
    if (SUCCEEDED(hr))
    {
        hr = pAttributes->GetAllocatedString(
            MY_ATTRIBUTE,
            &pwszValue,
            &cchLength
            );
    }

    CoTaskMemFree(pwszValue);
    SAFE_RELEASE(pAttributes);

    return hr;
}
```

미디어 파운데이션 속성 리스트를 보고 싶다면, [미디어 파운데이션 속성](https://docs.microsoft.com/en-us/windows/desktop/medfound/media-foundation-attributes)을 참고하시라. 각 속성이 가지고 있어야 할 데이터 타입이 문서에 잘 설명되어 있다.

## 속성 직렬화하기

미디어 파운데이션은 직렬화한 속성을 저장하는 데 두 가지 방법의 함수를 제공한다. 하나는 바이트 배열로 속성을 기록하는 방법이고, 다른 하나는 `IStream` 인터페이스를 지원하는 스트림으로 속성을 기록하는 방법이다. 각 함수는 동일한 방식으로 데이터를 적재하는 대응 함수가 존재한다.

|동작|바이트 배열|`IStream`|
|-|-|-|
|저장|`MFGetAttributesAsBlob`|`MFSerializeAttributesToStream`|
|로드|`MFInitAttributesFromBlob`|`MFDeserializeAttributesFromStream`|

저장된 속성 값을 바이트 배열에 기록하려면 `MFGetAttributesAsBlob` 함수를 호출한다. 이 때, `IUnKnown` 포인터 타입의 속성은 제외한다. 이렇게 직렬화한 배열을 다시 속성으로 불러오려면 `MFInitAttributesFromBlob` 함수를 호출한다.

한편, 속성 값을 스트림으로 기록하려면 `MFSerializeAttributesToStream` 함수를 호출한다. 이 함수는 `IUnKnown` 포인터 값을 포함하여 직렬화 한다. 이 함수를 사용할 때는 반드시 `IStream` 인터페이스를 구현한 스트림 객체가 있어야 한다. 스트림에서 다시 속성으로 불러오려면 `MFDeserializeAttributesFromStream` 함수를 호출한다.

## `IMFAttributes` 구현하기

미디어 파운데이션은 `IMFAttributes`을 이미 구현하여 제공하는데, 구현한 타입의 객체는 `MFCreateAttributes` 함수를 호출하여 얻을 수 있다. 대부분의 경우, 직접 구현하지 않고 제공하는 구현체를 그대로 사용해야 한다.

다만, `IMFAttributes`를 상속하는 2차 인터페이스를 구현한다면 `IMFAttributes` 인터페이스를 직접 구현할 수 있긴 하다. 이 경우, 2차 인터페이스가 상속하는 `IMFAttributes` 메소드를 반드시 구현해야 한다.

이 때, 2차 인터페이스는 이미 미디어 파운데이션에서 구현한 것을 감싸고 있도록 설계하는 것이 좋다. 다음 코드는 `IMFAttributes` 포인터를 가지고 있으면서 `IUnKnown` 메소드를 제외한 모든 `IMFAttributes` 메소드를 감싸는 클래스 템플릿을 보여준다.

```C++
#include <assert.h>

// Helper class to implement IMFAttributes. 

// This is an abstract class; the derived class must implement the IUnknown 
// methods. This class is a wrapper for the standard attribute store provided 
// in Media Foundation.

// template parameter: 
// The interface you are implementing, either IMFAttributes or an interface 
// that inherits IMFAttributes, such as IMFActivate

template <class IFACE=IMFAttributes>
class CBaseAttributes : public IFACE
{
protected:
    IMFAttributes *m_pAttributes;

    // This version of the constructor does not initialize the 
    // attribute store. The derived class must call Initialize() in 
    // its own constructor.
    CBaseAttributes() : m_pAttributes(NULL)
    {
    }

    // This version of the constructor initializes the attribute 
    // store, but the derived class must pass an HRESULT parameter 
    // to the constructor.

    CBaseAttributes(HRESULT& hr, UINT32 cInitialSize = 0) : m_pAttributes(NULL)
    {
        hr = Initialize(cInitialSize);
    }

    // The next version of the constructor uses a caller-provided 
    // implementation of IMFAttributes.

    // (Sometimes you want to delegate IMFAttributes calls to some 
    // other object that implements IMFAttributes, rather than using 
    // MFCreateAttributes.)

    CBaseAttributes(HRESULT& hr, IUnknown *pUnk)
    {
        hr = Initialize(pUnk);
    }

    virtual ~CBaseAttributes()
    {
        if (m_pAttributes)
        {
            m_pAttributes->Release();
        }
    }

    // Initializes the object by creating the standard Media Foundation attribute store.
    HRESULT Initialize(UINT32 cInitialSize = 0)
    {
        if (m_pAttributes == NULL)
        {
            return MFCreateAttributes(&m_pAttributes, cInitialSize); 
        }
        else
        {
            return S_OK;
        }
    }

    // Initializes this object from a caller-provided attribute store.
    // pUnk: Pointer to an object that exposes IMFAttributes.
    HRESULT Initialize(IUnknown *pUnk)
    {
        if (m_pAttributes)
        {
            m_pAttributes->Release();
            m_pAttributes = NULL;
        }


        return pUnk->QueryInterface(IID_PPV_ARGS(&m_pAttributes));
    }

public:

    // IMFAttributes methods

    STDMETHODIMP GetItem(REFGUID guidKey, PROPVARIANT* pValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetItem(guidKey, pValue);
    }

    STDMETHODIMP GetItemType(REFGUID guidKey, MF_ATTRIBUTE_TYPE* pType)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetItemType(guidKey, pType);
    }

    STDMETHODIMP CompareItem(REFGUID guidKey, REFPROPVARIANT Value, BOOL* pbResult)
    {
        assert(m_pAttributes);
        return m_pAttributes->CompareItem(guidKey, Value, pbResult);
    }

    STDMETHODIMP Compare(
        IMFAttributes* pTheirs, 
        MF_ATTRIBUTES_MATCH_TYPE MatchType, 
        BOOL* pbResult
        )
    {
        assert(m_pAttributes);
        return m_pAttributes->Compare(pTheirs, MatchType, pbResult);
    }

    STDMETHODIMP GetUINT32(REFGUID guidKey, UINT32* punValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetUINT32(guidKey, punValue);
    }

    STDMETHODIMP GetUINT64(REFGUID guidKey, UINT64* punValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetUINT64(guidKey, punValue);
    }

    STDMETHODIMP GetDouble(REFGUID guidKey, double* pfValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetDouble(guidKey, pfValue);
    }

    STDMETHODIMP GetGUID(REFGUID guidKey, GUID* pguidValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetGUID(guidKey, pguidValue);
    }

    STDMETHODIMP GetStringLength(REFGUID guidKey, UINT32* pcchLength)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetStringLength(guidKey, pcchLength);
    }

    STDMETHODIMP GetString(REFGUID guidKey, LPWSTR pwszValue, UINT32 cchBufSize, UINT32* pcchLength)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetString(guidKey, pwszValue, cchBufSize, pcchLength);
    }

    STDMETHODIMP GetAllocatedString(REFGUID guidKey, LPWSTR* ppwszValue, UINT32* pcchLength)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetAllocatedString(guidKey, ppwszValue, pcchLength);
    }

    STDMETHODIMP GetBlobSize(REFGUID guidKey, UINT32* pcbBlobSize)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetBlobSize(guidKey, pcbBlobSize);
    }

    STDMETHODIMP GetBlob(REFGUID guidKey, UINT8* pBuf, UINT32 cbBufSize, UINT32* pcbBlobSize)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetBlob(guidKey, pBuf, cbBufSize, pcbBlobSize);
    }

    STDMETHODIMP GetAllocatedBlob(REFGUID guidKey, UINT8** ppBuf, UINT32* pcbSize)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetAllocatedBlob(guidKey, ppBuf, pcbSize);
    }

    STDMETHODIMP GetUnknown(REFGUID guidKey, REFIID riid, LPVOID* ppv)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetUnknown(guidKey, riid, ppv);
    }

    STDMETHODIMP SetItem(REFGUID guidKey, REFPROPVARIANT Value)
    {
        assert(m_pAttributes);
        return m_pAttributes->SetItem(guidKey, Value);
    }

    STDMETHODIMP DeleteItem(REFGUID guidKey)
    {
        assert(m_pAttributes);
        return m_pAttributes->DeleteItem(guidKey);
    }

    STDMETHODIMP DeleteAllItems()
    {
        assert(m_pAttributes);
        return m_pAttributes->DeleteAllItems();
    }

    STDMETHODIMP SetUINT32(REFGUID guidKey, UINT32 unValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->SetUINT32(guidKey, unValue);
    }

    STDMETHODIMP SetUINT64(REFGUID guidKey,UINT64 unValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->SetUINT64(guidKey, unValue);
    }

    STDMETHODIMP SetDouble(REFGUID guidKey, double fValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->SetDouble(guidKey, fValue);
    }

    STDMETHODIMP SetGUID(REFGUID guidKey, REFGUID guidValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->SetGUID(guidKey, guidValue);
    }

    STDMETHODIMP SetString(REFGUID guidKey, LPCWSTR wszValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->SetString(guidKey, wszValue);
    }

    STDMETHODIMP SetBlob(REFGUID guidKey, const UINT8* pBuf, UINT32 cbBufSize)
    {
        assert(m_pAttributes);
        return m_pAttributes->SetBlob(guidKey, pBuf, cbBufSize);
    }

    STDMETHODIMP SetUnknown(REFGUID guidKey, IUnknown* pUnknown)
    {
        assert(m_pAttributes);
        return m_pAttributes->SetUnknown(guidKey, pUnknown);
    }

    STDMETHODIMP LockStore()
    {
        assert(m_pAttributes);
        return m_pAttributes->LockStore();
    }

    STDMETHODIMP UnlockStore()
    {
        assert(m_pAttributes);
        return m_pAttributes->UnlockStore();
    }

    STDMETHODIMP GetCount(UINT32* pcItems)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetCount(pcItems);
    }

    STDMETHODIMP GetItemByIndex(UINT32 unIndex, GUID* pguidKey, PROPVARIANT* pValue)
    {
        assert(m_pAttributes);
        return m_pAttributes->GetItemByIndex(unIndex, pguidKey, pValue);
    }

    STDMETHODIMP CopyAllItems(IMFAttributes* pDest)
    {
        assert(m_pAttributes);
        return m_pAttributes->CopyAllItems(pDest);
    }

    // Helper functions
    
    HRESULT SerializeToStream(DWORD dwOptions, IStream* pStm)      
        // dwOptions: Flags from MF_ATTRIBUTE_SERIALIZE_OPTIONS
    {
        assert(m_pAttributes);
        return MFSerializeAttributesToStream(m_pAttributes, dwOptions, pStm);
    }

    HRESULT DeserializeFromStream(DWORD dwOptions, IStream* pStm)
    {
        assert(m_pAttributes);
        return MFDeserializeAttributesFromStream(m_pAttributes, dwOptions, pStm);
    }

    // SerializeToBlob: Stores the attributes in a byte array. 
    // 
    // ppBuf: Receives a pointer to the byte array. 
    // pcbSize: Receives the size of the byte array.
    //
    // The caller must free the array using CoTaskMemFree.
    HRESULT SerializeToBlob(UINT8 **ppBuffer, UINT32 *pcbSize)
    {
        assert(m_pAttributes);

        if (ppBuffer == NULL)
        {
            return E_POINTER;
        }
        if (pcbSize == NULL)
        {
            return E_POINTER;
        }

        *ppBuffer = NULL;
        *pcbSize = 0;

        UINT32 cbSize = 0;
        BYTE *pBuffer = NULL;

        HRESULT hr = MFGetAttributesAsBlobSize(m_pAttributes, &cbSize);

        if (FAILED(hr))
        {
            return hr;
        }

        pBuffer = (BYTE*)CoTaskMemAlloc(cbSize);
        if (pBuffer == NULL)
        {
            return E_OUTOFMEMORY;
        }

        hr = MFGetAttributesAsBlob(m_pAttributes, pBuffer, cbSize);

        if (SUCCEEDED(hr))
        {
            *ppBuffer = pBuffer;
            *pcbSize = cbSize;
        }
        else
        {
            CoTaskMemFree(pBuffer);
        }
        return hr;
    }
    
    HRESULT DeserializeFromBlob(const UINT8* pBuffer, UINT cbSize)
    {
        assert(m_pAttributes);
        return MFInitAttributesFromBlob(m_pAttributes, pBuffer, cbSize);
    }

    HRESULT GetRatio(REFGUID guidKey, UINT32* pnNumerator, UINT32* punDenominator)
    {
        assert(m_pAttributes);
        return MFGetAttributeRatio(m_pAttributes, guidKey, pnNumerator, punDenominator);
    }

    HRESULT SetRatio(REFGUID guidKey, UINT32 unNumerator, UINT32 unDenominator)
    {
        assert(m_pAttributes);
        return MFSetAttributeRatio(m_pAttributes, guidKey, unNumerator, unDenominator);
    }

    // Gets an attribute whose value represents the size of something (eg a video frame).
    HRESULT GetSize(REFGUID guidKey, UINT32* punWidth, UINT32* punHeight)
    {
        assert(m_pAttributes);
        return MFGetAttributeSize(m_pAttributes, guidKey, punWidth, punHeight);
    }

    // Sets an attribute whose value represents the size of something (eg a video frame).
    HRESULT SetSize(REFGUID guidKey, UINT32 unWidth, UINT32 unHeight)
    {
        assert(m_pAttributes);
        return MFSetAttributeSize (m_pAttributes, guidKey, unWidth, unHeight);
    }
};
```

다음 코드는 상기한 클래스 템플릿을 상속하는 방법을 보여주고 있다.

```C++
#include <shlwapi.h>

class MyObject : public CBaseAttributes<>
{
    MyObject() : m_nRefCount(1) { }
    ~MyObject() { }

    long m_nRefCount;

public:

    // IUnknown
    STDMETHODIMP MyObject::QueryInterface(REFIID riid, void** ppv)
    {
        static const QITAB qit[] = 
        {
            QITABENT(MyObject, IMFAttributes),
            { 0 },
        };
        return QISearch(this, qit, riid, ppv);
    }

    STDMETHODIMP_(ULONG) MyObject::AddRef()
    {
        return InterlockedIncrement(&m_nRefCount);
    }

    STDMETHODIMP_(ULONG) MyObject::Release()
    {
        ULONG uCount = InterlockedDecrement(&m_nRefCount);
        if (uCount == 0)
        {
            delete this;
        }
        return uCount;
    }

    // Static function to create an instance of the object.

    static HRESULT CreateInstance(MyObject **ppObject)
    {
        HRESULT hr = S_OK;

        MyObject *pObject = new MyObject();
        if (pObject == NULL)
        {
            return E_OUTOFMEMORY;
        }

        // Initialize the attribute store.
        hr = pObject->Initialize();

        if (FAILED(hr))
        {
            delete pObject;
            return hr;
        }

        *ppObject = pObject;
        (*ppObject)->AddRef();

        return S_OK;
    }
};
```

속성을 저장하기 위해 이 클래스 타입의 객체를 생성할 경우 `CBaseAttributes::Initialize` 함수를 반드시 호출해야 한다. 이전 예제에서는 정적으로 선언된 객체 생성 함수 안에서 호출했었다.

템플릿 인수는 인터페이스 타입인데, 기본값은 `IMFAttributes`이다. 하지만 `IMFActivate`와 같이 `IMFAttributes`를 상속하는 임의의 인터페이스를 구현한다면 템플릿 인수에 해당 인터페이스의 이름을 넣어주면 된다.

## 관련 주제

[미디어 파운데이션 기본 타입](https://docs.microsoft.com/en-us/windows/desktop/medfound/media-foundation-primitives)

## 원본

[Attributes in Media Foundation](https://docs.microsoft.com/en-us/windows/desktop/medfound/attributes-and-properties)