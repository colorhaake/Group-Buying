## 理想的情況底下，我覺得client的架構應該有下面幾個特點
### Pure View/Component
所謂的Pure View/Component指的是外部給他什麼樣的資料，他就顯示什麼樣的View  
裡面沒有儲存state  
(state指的是任何一種資料, 像是我們可能會存說現在訂單的資料、Leader是誰，有下標的人是誰等等)  

用React舉例來說以下就是一個Pure View/Component
```
// Clock是一個Pure View/Component, 他只根據外部給的props，來顯示View
function Clock(props) {
  return (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {props.date.toLocaleTimeString()}.</h2>
    </div>
  );
}
```

而下面的Clock就不是一個Pure View/Component，因為他裡面存了state
```
class Clock extends React.Component {
  constructor(props) {
    super(props);
    // 儲存date
    this.state = {date: new Date()};
  }

  // 每呼叫一次tick都會變更state, 儲存新的date
  tick() {
    this.setState({
      date: new Date()
    });
  }

  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.state.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}
```
## 為什麼Pure View/Component是比較好的呢？
以上面的例子來看，如果有其他的function去呼叫到`tick()` 那麼state的值就會被改變, 顯示的View也會跟著改  
如果這個時候顯示出來的View不是你要的話，你就要花時間一個一個去檢查變更state的邏輯。看到底是哪裡出錯了  

而如果是Pure View/Component的話，就只要檢查給的Props值是不是對的，然後在看顯示View的邏輯有沒有問題就好  

當然，非常重要一點，並不可能所有的View都是Pure View，一般我們還是會在View裡面存一些state  
盡可能的是UI state，像是存說Checkbox現在是on/off。每點擊一次Checkbox，變更裡面的UI state  
更進階一點的，我們會分成[Presentational and Container Components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0#.6q34yupfg)，這個就有空在看看


### Single State
Single state的意思是，我們整個client全部儲存一個共用的state，裡面存說整個client會用到的全部資料  
更實際一點的來說，一個state就是一個大大的JSON Object就是了  
```
// for example
{
	orders: [],
	users: [],
	...
}
```
會用Single state的原因是，就是為了要集中管理，不要將state放置各處，只會造成麻煩而已
當然並不是所有的東西都要存在這個Single state上，盡可能的存整個client會用到的資料就好。一些簡單的UI state就交給Container component去暫時儲存就好

### 只能透過Action去變更Single State
要變更State也不是說可以亂變更，一般會發送一個Action, Action裡面會定義說要做的行為是什麼，也可以夾帶一些資料  
接著處理Action的部分(在redux中稱作reducer)，會根據收到的Action，做變更State的動作。  
Action在Javascript中也只是一個Object, 裡面定義他的type是什麼，還有額外要帶的資料。  
```
// Action example
{
	type: 'CHANGE_USER_NAME'
	...payload // payload是你額外想帶的資料
}
```
變更的flow大致上為
View -> (透過Action) -> (update)Model -> View -> ...

### Signle State必須是Immutable
我們要設計成任何變更State的動作，都會回傳一個新的State Object  
所謂的Immutable, 更明確的說是Immutable Object, 指的是資料是不可變更的，如果要變更就要回傳一個新的Object  
新的Object和舊的Object所指向的memory位置是不一樣的。  
舉例來說
```
// a is mutable
var a = {data: 10}
var b = a
// a把data設為100之後, a指到的memory位置還是一樣。
a.data = 100
a === b // a等於b為true

--------------

// a is immutable
var a = {data: 10}
var b = a
// 複製了一個全新的a, 並且把data的值改為100
a = Object.assgin({}, a, {data: 100})
a === b // a等於b為false
```
為什麼要Immutable的設計？原因是因為Mutable的問題很多，舉一個簡單的例子
b和a指向同一個memory位置，b或a任何一個變數變更了裡面的值，都會影響到彼此
```
// a is mutable
var a = {data: 10}
var b = a
// b把data設為100, 也會跟著影響到a
b.data = 100

```
雖然產生一個新的Object效率會比較差一點，不過可以抵掉Mutable產生的問題。
詳細可以看一下[Mutable vs immutable objects](http://stackoverflow.com/questions/214714/mutable-vs-immutable-objects)

### 任何變更Single State的動作就僅僅變更Single State和定義Side effects，不能直接執行Side effects
這個部分比較抽象一點，最後我會給一個完整的例子，會比較清楚完整的架構  
先定義什麼是Side effects。任何變更state的行為就稱為Side effects，或者可以參考[wiki的定義](https://en.wikipedia.org/wiki/Side_effect_(computer_science))  
談到Side effects就會提到pure funciton
```
function pure(number) {
	return number + 1
}

function nonPure(number) {
	var result = callApi(number)
	return result
}
```
上面的pure function，他的回傳值完全取決於function的參數  
而下面的nonPure function，callApi是一個不確定的因素，server可能會回傳各種結果  
導致nonPure function不能輕易的被預測，很容易產生bug。nonPure function也就沒有辦法根據他給的參數，來決定他的回傳值
所以說一般像call api、寫log、寫檔讀檔的IO動作、需要外部動作跟這個function無關的行為，我們會稱他作是Side effects。

回到任何變更Single State的動作，我們希望是一個pure function。function回傳的結果是一個全新的state和你要執行的Side effects。Framework會去執行的你定義的side effects。執行完畢會去在執行相對應的action
```
// click button action
{
	type: 'CLICK_BUTTON',
	...payload
}

// 假設我們已經傳送了ClickButton的Action, 並且執行了相對應的handleActionClickButton
// payload是傳送action時，你想額外帶的資料。state是整個client用到的single state
function handleActionClickButton(payload, state) {
	var newState = update(state) // 看你要怎麼update state都可以
	return [
		newState,
		// 定義你要執行的side effects，以及執行side effects完之後，要執行的action
		callAPI(payload).perform(successAction, failureAction)
	]
}

// 這邊的payload為call api的response
function handleSuccessAction(payload, state) {
	var newState = Object.assign({}, state, {result: payload})
	return [newState] // 沒有要執行新的side effects就不用定義
}

// 這邊的payload為call api的response
function handleFailureAction(payload, state) {
	var newState = Object.assign({}, state, {error: payload})
	return [
		newState,
		sendErrorToServer(payload)
		.perform(successSendErrorAction, failureSendErrorAction)
	]
}
```
	
所以說整個Client的架構會變得很簡單  
View就是一個pure View/Component，View只管兩件事  
1. state變化，顯示什麼樣的View.  
2. 先點擊View就送Action出去  
他也不知道Action會做什麼樣的事，因此View就變得非常的單純，沒有額外的負任。

負責處理Action的部分就會更新state，並且回傳新的state和他要做的side effects  
Framework會去執行side effects，執行完畢會去執行side effects完的action  
action會在去變更state。View聽到state的變化，就會在重新顯示新的View
所以完整的flow就是  
```
View -> (透過Action) -> handle action and update Model -> View -> ...
													  -> do side effects -> handle action and update Model -> ...
```													
上述我講的所有東西大致上就是[Elm](http://elm-lang.org/)的[架構](https://guide.elm-lang.org/architecture/)  
Redux是受到Elm的啟發所開發出來的Framework, 但是處理side effects的部分我覺得沒有很好。而[Redux-Loop](https://github.com/redux-loop/redux-loop)為模擬Elm中處理Side effects的部分，可以用Redux-Loop完成處理Side effects

所以說Redux + Redux-Loop = Elm  
Client我會以Redux + Redux-Loop去實現  

## 參考
如果要學習Redux，強力建議先看下面的Tutorial Video，他的講解清析易懂  
[Redux Tutorial Video](https://egghead.io/courses/getting-started-with-redux)  
[Redux中文教學](https://chentsulin.github.io/redux/index.html)  
[Redux GitHub](https://github.com/reactjs/redux)  
[Redux-Loop](https://github.com/redux-loop/redux-loop)  
[Elm](http://elm-lang.org/)  
[Elm 架構](https://guide.elm-lang.org/architecture/)
