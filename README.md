# MVVM에 대하여

앞선 파트에서 저의 마지막 결론은 **일단 지금의 지식으론 MVVM을 사용하자.** 입니다.

그렇다면 이 프로그래머에서 분분한 **MVVM**

왜 이렇게 분분한가에대한 내용을 기록하였습니다.

### 먼저 가장 중요한 AAC의 ViewModel과 MVVM의 ViewModel은 같은것인가 입니다.

이 [포스팅](https://github.com/tnvnfdla1214/MVVM-VM-AAC-VM-)을 보신 다면 이해가 빠르실 겁니다.

MVVM의 ViewModel은 view에 viewmodel의 **데이터를 주입하기 위한 관계 규명과 같은 관계로서 존재하는 것이고 View와 ViewModel 연결 최소화**해야합니다.
그러나 **안드로이드 아키텍쳐 컴포넌트의 뷰모델은 생명주기에서 벗어나기 위한 것** 입니다.

그러므로 명확히 MVVM을 이용하기 위해서는 AAC의 ViewModel을 사용해서 MVVM의 ViewModel로서 사용하면 안됩니다.
 
**승욱 아저씨 : 뷰모델을 뷰가 옵저빙하고 있다 > 상관관계가 있다. 뷰는 xml이지 액티비티나 프레그먼트가 아니다. > 이걸 하려면 데이터 바인딩을 해줘야한다.**

<img src="https://user-images.githubusercontent.com/48902047/146144848-7629b689-510f-434c-b67d-7f6227d83c3d.png"></img>

위의 그림은 view와 ViewModel의 상관관계로 진행됩니다. 그러나 View와 ViewModel의 연결을 최소화 해야 하기때문에 아래와 같은 모습으로 바뀌어야 합니다.

<img src="https://user-images.githubusercontent.com/48902047/146144740-dea54991-4c1b-496f-8a7b-2a073b8210dc.png"></img>

다시 그림으로 설명하자면
<img src="https://user-images.githubusercontent.com/48902047/146145119-50f95996-4358-4a76-bd37-f67769e357ac.png"></img>

전 포스팅의 그림은 아래와 같이 바뀌어야 합니다.

<img src="https://user-images.githubusercontent.com/48902047/146145132-fa50a48b-680d-4ddb-97e5-ad03c6e4681f.png"></img>

이러하므로 결론은 [**DataBinding**]()을 사용해야합니다.
