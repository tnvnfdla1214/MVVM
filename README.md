# MVVM에 대하여

앞선 파트에서 저의 마지막 결론은 **일단 지금의 지식으론 MVVM을 사용하자.** 입니다.

그렇다면 이 프로그래머에서 분분한 **MVVM**

왜 이렇게 분분한가에대한 내용을 기록하였습니다.

### 먼저 가장 중요한 AAC의 ViewModel과 MVVM의 ViewModel은 같은것인가 입니다.

이 [포스팅](https://github.com/tnvnfdla1214/MVVM-VM-AAC-VM-)을 보신 다면 이해가 빠르실 겁니다.

MVVM의 ViewModel은 view에 viewmodel의 **데이터를 주입하기 위한 관계 규명과 같은 관계로서 존재하는 것이고 View와 ViewModel 연결 최소화**해야합니다.
그러나 **안드로이드 아키텍쳐 컴포넌트의 뷰모델은 생명주기에서 벗어나기 위한 것** 입니다.

그러므로 명확히 MVVM을 이용하기 위해서는 AAC의 ViewModel을 사용해서 MVVM의 ViewModel로서 사용하면 안됩니다.

### MVVM의 잘못된 이해
국내 MVVM 이해 오류가 많습니다.
예로 아래의 경우 뷰에서 옵저빙 하는 부분이 문제입니다.
```kotlin
class MainViewModel {
    val title LiveData<String>
}
class MainActivity {
    val viewModel: ViewModel
    val textView: TextView
    fun onCreated() {
        viewModel.title.observe(this) { //Activity에서 observe 해서는 안됨
            textView.text =it
        }
    }
}
```
위에 코드는 아래와 같이 변경 되어 실제 Activity에서는 DataBinding 하여 ViewModel과 View를 연결 해 주어야 합니다.
```kotlin
class MainActivity { 
    val viewModel: ViewModel    
    fun onCreated() {
        dataBinding.setVariable(BR.vm, viewModel)
    }
}
```

<img src="https://user-images.githubusercontent.com/48902047/146144848-7629b689-510f-434c-b67d-7f6227d83c3d.png"></img>

위의 그림은 view와 ViewModel의 상관관계로 진행됩니다. 그러나 View와 ViewModel의 연결을 최소화 해야 하기때문에 아래와 같은 모습으로 바뀌어야 합니다.

<img src="https://user-images.githubusercontent.com/48902047/146144740-dea54991-4c1b-496f-8a7b-2a073b8210dc.png"></img>

다시 그림으로 설명하자면
<img src="https://user-images.githubusercontent.com/48902047/146145119-50f95996-4358-4a76-bd37-f67769e357ac.png"></img>

전 포스팅의 그림은 아래와 같이 바뀌어야 합니다.

<img src="https://user-images.githubusercontent.com/48902047/146145132-fa50a48b-680d-4ddb-97e5-ad03c6e4681f.png"></img>

이러하므로 결론은 [**DataBinding**]()을 사용해야합니다.

### 안드로이드에서 DataBinding 사용에 문제 사항들
안드로이드에는 문제인 사항들이 있습니다. 아래와 같은 사항으로 View와 ViewModel의 결합성을 지속적으로 끊어내면서 어떻게 코드를 짜야할 것인지 알아야 합니다.
1. LifeCycle
2. Databinding으로 다 표현하기 힘든 View 이벤트
3. Resource 등 Context를 접근해야 하는 경우

### LifeCycle
그랩에서는 LifeCycle 문제를 위해 RxBinder를 만들어 사용했습니다. ([Trello](https://github.com/trello/RxLifecycle)의 RxBinder 참조)
아래의 코드는 "특정 이벤트가 들어올때까지 계속 해서 동작이 가능하도록 작동을 하겠다" 입니다.
해당 발표에서는 ViewModel을 사용하지 않는다고 합니다.
```kotlin
class ViewModel(val rxbinder: RxBinder) {
    init {
        rxbinder.bind(ON_DESTROYED) {
            Observable....
        }
    }
}
...
class RxBinder {
    val map: Map<Event, CompositDiposable>
    fun apply(lifeEvent: Event) {
        map[lifeEvent]?.clear()
    }
    fun bind(lifeEvent: Event, body:()->Disposable) {
        map[lifeEvent].add(body())
    }
}
...
open class RxActivity{
    val rxbinder: RxBinder
    fun onCreated() {
        rxbinder.apply(ON_CREATED)
    }
}
```
### Databinding으로 다 표현하기 힘든 View 이벤트
DataBinding이 좋지만 GlobalLayoutChangeListener 등은 데이터 바인딩으로 구현이 어렵고 너무나도 많은 코드가 들어갑니다. (View가 attach 되있는지 안되어있는지 리스너하는것, 단발성으로 리스너 해야하는 것)

위의 경우 2-way 바인딩 구현해야 하며 3개의 function을 구현해야 하며 잘 연결이 되어 있는지 확인하기 위해서 믿을 수 있는 방법은 Compile방법밖에 없습니다.

이를 위해 2-way 바인딩 구현보다는 usecase를 만들어 회피 사용합니다.(실제 그랩에서는 ViewUseCase라고 부르는 UseCase를 만든다고 합니다.)
```kotlin
class GetVisibleAreaUsecase(view1) {
    fun observe(): Observable<Rect> {
        return Observable.create {e->
                                  e.onNext(view1.height)
                                  view1.addGlobalLayoutChange{
                                      e.onNext(view1.height)
                                  }
		}.distictUntilChanged()
    }
}
...
class ViewModel(usecase:GetVisibleAreaUsecase) {
    init{
        usecase.observe().subscribe{
            /* do something */
        }.bindUntil(rxbinder)
    }
}
```
위에 코드는 View에 대해 특정 이벤트를 기다렸다가 그 View의 특정 이벤트를 리스너로 등록을 하고 리스너가 들어오면 Rx를 이용하는 방식을 한다고 합니다.

### Resource 등 Context를 접근해야 하는 경우
ResouceProvider 라는 랩핑을 만들어 context를 전달해 사용합니다.
```kotlin
class ResourceProvider(context: Context) {
    fun string(resId: String) = context.resource.getSting(resId)
}
```
ActivityResult는 매니저를 만들어 Activity에서 메소드 실행시 ViewModel에서 구독됩니다.
```kotlin
class Activity {
    val resultManager
    fun onActivityResult() {
        resultManager.onResult()
    }
}
...
class ResultManager{
    fun listen(observer)
    fun onResult()
}
...
class ViewModel(resultManager) {
    init{
        resultManager.listen {
            // do something
        }
    }
}
```
위의 모든 것이 구현된다면 대략 이러한 형태가 됩니다.
```kotlin
class ViewModel(rxbinder, usercase, resultManager) {
    init{
        // many things are here
    }
}
...
class Activity: ResultableRxActivity {
    fun onCreated() {
        dataBinding.setVariable(BR.vm, viewModel)
    }
}
```
이렇게 되면 아래와 같이 껍데기 뿐인 액티비티에서 DataBinding과 DI만 선언하는 역할로 취하게 되어 액티비티가 필요가 없어지게 됩니다.
```kotlin
class MainActivity: RxResultableActivity{
    @Inject lateinit var vm: MainViewModel
    fun onCreated() {
        dataBinding.setVariable(BR.vm, vm)
    }
}
```
이렇게 되면 맥티비티가 View를 선언하고 DataBinding만 연결하는 방식만 취하게 됩니다. 그러기에 그랩은 한단계 더 나아가 보기로 했습니다.
보통은 xml 하나에 databinding당 하나의 viewmodel이 생기게 됩니다.
그러나 그랩에서는 단일 Layout을 조각내기로 했습니다.

아래 예시는 실제 그랩의 앱 사진입니다. 아래사진은 몇개의 뷰 조각이 있을까요? 총 7조각으로 되어 있습니다.

<img src="https://user-images.githubusercontent.com/48902047/146646795-aabfb0a1-b096-4251-a5f8-48815a581ba2.png"></img>

그랩에서는 한 레이아웃을 뷰를 나눠 각각을 노드화하여 처리합니다. 각 뷰 별로 바인딩 노드가 되고, 각 노드에 ViewModel이 인젝션되게 합니다. 바인딩 노드는 부모뷰와 ViewModel을 받습니다.
이렇게 되면 코드는 아래와 같습니다.

#### Activity
```kotlin
//Activity(21:55)
class ConfirmActivity{
    fun onCreated() {
        BackButtonNode({parentView}).dependency(root).build()
        TaxiTypeNode({parentView}).dependency(root).build()
        ExtraInfoNode({parentView}).dependency(root).build()
        PayConfirmNode({parentView}).dependency(root).build()
        //...etc...
    }
}
```
