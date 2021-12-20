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

**각 Unit 을 노드로 구분, Node가 구성될때 ViewModel을 바인딩**

#### Activity
```kotlin
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
#### 사진의 5번 TaxiTypeNode
```kotlin
class TaxiTypeNode(parentView:()->ViewGroup): BindingNode{
    @Inject lateinit var vm: ViewModel
    fun build() {
        depdency.inject(this)
        binding(parenteView(), vm)
    }
}
```
#### BindingNode
```kotlin
open class BindingNode {
    fun binding(parentView: ViewGroup, vm: ViewModel) {
        val binding = ViewBinding.inflate(vm.layoutId, parentView)
        binding.setVariable(BR.vm, vm)
    }
}
```
#### Root Parent XML
```kotlin
<FrameLayout>
    <FrameLayout android:id="@+id/parent1" />
	<FrameLayout android:id="@+id/parent2" />
</FrameLayout>
```
#### 노드의 장,단점
기존에는 큰 레이아웃에 여러개의 뷰가 있고 가상으로 만들어놀거나 커스텀 뷰에 뷰모델을 매핑하는 방식을 사용하였다. 하지만 발표자는 레이아웃자체를 플러그인 하는 방식으로 하였다고 합니다. (임플레이트(?)해서 넣는 방식) 임플레이트하는 코드는 노드와 따라다니는 것이라 보고 노드는 뷰모델을 기본적으로 수집(?)하는것이기 때문에 저희가 어느 뷰에 그 노드를 집어넣을 건지 지정을 하면 되는 형태가 되는 것입니다. 그렇기에 뷰가 굉장히 자유롭게 움직일 수 있습니다.
- 장점 : Node 제어로 View 플러그인화 가능
- 단점 : BackKey, Save/Restore Instance 등 다양한 처리 구현
위 구조는 작은 사이즈(Mininum Value Product = MVP)에서 중간 규모의 앱에서는 과도하며, 인터랙션이 많거나 View 자유도, 재활용이 높은 앱에 권장한다고 합니다.

### 발표자의 정리

+ View는 XML
+ xml(View)과 ViewModel을 연결 하기 위해 DataBinding 필수요소
+ 감지하기 어려운 뷰 변화는 ViewUsecase
+ LifeCycle, Result 위한 추가적인 처리 필요 (그랩에서는 상위에서 메이징(?)하는 매퍼를 만들어 처리하는 방식)
+ Resource(Context) 접근은 뷰모델에서 Context에 접근하는것을 막기 위해서 Wrapper 처리
+ Node 예제는 Optional

### Q&A

Q : 그랩에서는 AAC 사용 안하는지?

LiveDatasms Room과 연동하여 사용하나 어플리케이션 자체가 실시간성이기 때문에 
데이터 용도로 맞지 않고 라이프사이클 처리도 직접 함으로 Room, LiveData, AAC의 ViewModel 등은 사용 안함.

Q : 노드 사용 사례는?

루트 노드에서 라우터가 하위 노드를 컨트롤하며, 하위 노드를 구성하도록 지시함.

Q : 이재원님(MS Expert)의 안드로이드의 MVVM 오류에 대한 내용이 반영되었는지?

반영된 내용임.

Q : RxBinding이 그랩에서 자체적으로 만든건지?

외부(Trello)에서 아이디어만 가져오고 자체 구현함.

Q : Activity에서 ViewModel을 직접 참조할 일이 없다는데? Activity에서 반응에 대한 부분을 ViewModel에 전할때는?

데이터 바인딩으로 처리됨.

Q : 노드의 상대적인 위치는 어떻게 지정?

루트노드에서 Parent 대비 상대적인 위치가 지정되어 있음.

Q : 전통적인 Activity와의 차이는?

레이아웃 부분이 static으로 지정되어 있지만, 노드에서는 노드 단위로 다른 화면에서도 재활용이 가능함.

Q : 빈 레이아웃에 attach/dettach 하는 방식이 Fragment와 비슷한데?

그랩에서는 Fragment를 사용안함. onRestore 시 등 재생성 충돌등의 문제가 있어 별도로 노드 처리함.

Q : 기본 선수 지식은?

DataBinding(2-way까지), MVP에 대한 깊은 경험, MVP에서 MVVM으로 변화에 대한 이해 필요.

Q : ViweModel에서 Resource ID를 가지는 이유는?

ViewModel에서 어떤 리소스와 매칭되는지 판단을 했지만, 편의상 Node보다 ViewModel에서 가지고 있음.

Q : 2-way 바인딩 사용 사례는?

2-way 바인딩이 한번 이상 재사용시에는 2-way 바인딩을 빼서 사용을 권고함.

Q : DataBinding이 MVVM에 필수인데 Anko의 경우는?

Anko같은 경우는 네이티브 코드이고 Anko로 작성한다 해도 중간에 추상화 코드가 나오는 게 아니라 현재 구현된 상태가 최선인 듯. 현재로썬 Anko와는 궁합이 좋지 않음.

Q : AlertDialog는 어떻게 처리?

AlertDialog도 노드로 처리

Q : 싱글 액티비티에서 히스토리 관리는?

상위 노드에서 관리하며, 히스토리 관리하는 모듈이 있음.

Q : 그랩은 모두 싱글액티비티임?

메인 플로우들은 싱글 액티비티이나 페이먼트, 리워드 등은 아님. 그래서 onResult 처리가 필요했음.

Q : Trello RxLifeCycle에서 추가된 기능은?

자체적으로 필요한 부분만 추가되고 Trello에서는 필요한 것만 빼서 씀

Q : MS 제안한 MVVM 의도대로 분업이 되었는지? (XML은 디자이너, ViewModel은 엔지니어)

그랩에선 디자이너는 UX/비주얼(7:3정도) 디자이너로 나뉘며, xml이나 ViewModel 모두 엔지니어가 작업

Q : Save Instance 상태는 어떻게 관리하는지?

노드별로 저장해서 상위 노드에 전달하고 복원 시작시 하위 노드에 다시 전달함. 복잡한 부분이라 라이브러리화해서 사용 중.

