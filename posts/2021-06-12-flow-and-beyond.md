---
title: "Flow and beyond"
date: 2021-06-12 17:00:00 +0000
image: 'https://gsculerlor.s-ul.eu/yMzpYCK4'
---

*Note: this document is based on Kotlin and kotlinx-coroutines version 1.4.3. Things may change on the latest release of Kotlin (1.5.0). This document is used for research resume on my workplace, placed here only for my blog parser test*

<!-- end -->

*original post and presentation: 2021-05-12 at Menara Jamsostek lt.23*


# Flow

Flow is a sequential emission of values that at some points stopped or end because either completes normally or something wrong happens and throwing an exception. A flow is an asynchronous version of [Sequences](https://kotlinlang.org/docs/sequences.html), a type of collection whose values are lazily produced. A flow can have an infinite number of values and each value is produced on-demand (whenever the value is needed). Flow produces values one at a time that can generate values from asynchronous operations like database calls and network request calls. By default, flows are sequential and all flow operations are executed sequentially in the same coroutine, with an exception for a few operations specifically designed to introduce concurrency into flow execution.

There are three entities involved in streams of data:

* A producer that produces data and adds it to the stream.
* Intermediaries that can modify each value emitted into the stream or the stream itself (optional).
* A consumer consumes the values from the stream.

## Creating a flow

To create flow, use the flow builder APIs. The builder function creates a new flow where you can emit something into it using the emit function. Intermediaries then can be used to modify the stream of data without consuming the values using the _intermediate operator_. A _terminal operator_ is used to trigger the flow to start listening for values. To collect all the values in the stream as they’re emitted, use collect block. We will talk about _intermediate operators_ and _terminal operators_ later on the flow operators. Note that collect is a suspend function, so it needs to be executed inside a coroutine and the coroutines that call collect may suspend until the flow is closed.

```Kotlin
// flow building block
flow {
    // emit values
    emit(1)
    emit(2)
    emit(3)
}.map { intVal ->
    // transform value into string
    "transforming $intVal into a string!"
}.collect { stringVal ->
    // collect every value on stream and print it
    println(stringVal)
}
```


You can also create a flow from the Iterable using asFlow or flowOf.

```Kotlin
listOf(1, 2, 3).asFlow()
flowOf(1, 2, 3).onEach { //do something }
```


There is also a channelFlow builder that used to construct arbitrary flows from potentially concurrent calls to the send function.

```Kotlin
channelFlow {
// send from one coroutine
launch(Dispatchers.IO) {
    send(1)
}
// send from another coroutine
launch(Dispatchers.Default) {
    send(2)
}
```


## Properties of flow

### Context preservation

As stated in the documentation: “it encapsulates its own execution context and never propagates or leaks it downstream. To put it simply, the context where the flow is emitting the value is never leaked to the receiver and the values are produced in one coroutines scope.

There is only one way to change the context of a flow: the flowOn operator that changes the upstream context. 

Let’s take a look at this code snippet:

```Kotlin
flow {
  emit(1)
  // still on the same coroutines
  coroutineScope { emit(2) }
  coroutineScope { 
    // will throw an exception
    launch { emit(3) }
  }
}.collect {
  println(it)
}
```


If you run this snippet it will throw an exception. At the end of the exception, you can see the solution to mitigate this restriction using `channelFlow` instead of `flow`.

### Exception transparency

Exception handling in flows shall be performed with catch operator and it is designed to only catch exceptions coming from upstream flows while passing all downstream exceptions. It is intended to not have try-catch wrapping emit or emitAll calls.

Let’s take a look at this code snippet:

```Kotlin
val flow = flow {
  try { emit(1); emit(2) }
    catch (e: Exception) { emit(3) }
}.collect {
  if (it == 2) throw CancellationException("2 is a bad number :(")
  else println(it)
}
```


If you run this snippet it will throw another exception, this time `IllegalStateException`.

Flow enforces exception transparency at runtime and throws `IllegalStateException` on any attempt to emit a value if an exception has been thrown on the previous attempt.

### Not stable for inheritance

The Flow interface is not stable for inheritance in 3rd party libraries, as new methods might be added to this interface in the future, but is stable for use. (also applied to interfaces that implement `Flow` interface such as `StateFlow<T>`)

## Operators

As mentioned before, there are two types of operators, _terminal_ and _intermediates_ operators.

### Terminal operators

Terminal operators are suspendable functions that collect the values received from the stream. Terminal operators also a trigger for the flow to start its value emission.

See: [collect](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/collect.html), [single](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/single.html), [reduce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/reduce.html), [toList](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-list.html), [asLiveData](https://developer.android.com/reference/kotlin/androidx/lifecycle/package-summary#(kotlinx.coroutines.flow.Flow).asLiveData(kotlin.coroutines.CoroutineContext,%20kotlin.Long))*

### Intermediates operators

Intermediates operators are functions that applied to the upstream flow and return a downstream flow. Intermediates operators don’t always return a value. This is known as a _cold flow_ property.

See: [map](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/map.html), [filter](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/filter.html), [take](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/take.html), [zip](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/zip.html)

# StateFlow and SharedFlow

## StateFlow

`StateFlow` is a state-holder observable flow that emits the current and new state update to its collectors. It represents a read-only state with a single updateable data value. It is designed to make a stateful stream that can be updateable over time. A state flow is a _hot_ flow because its active instance exists independently of the presence of collectors. You can’t create a `StateFlow` directly since it’s an interface. A mutable state flow is created using MutableStateFlow(initialValue) constructor. Note that we must define the initial value of the state, guaranteeing that the flow has an initial state.

### StateFlow over ConflatedBroadcastChannel

`StateFlow` is designed to completely replace ConflatedBroadcastChannel. `StateFlow` is designed to better cover typical use-cases of keeping track of state changes in time.

* `StateFlow` always has a value that can be read at any time using value property.
* `StateFlow` has a clear separation into a read-only `StateFlow` interface and a `MutableStateFlow`.
* `StateFlow` conflation is based on equality like distinctUntilChanged operator, unlike conflation in `ConflatedBroadcastChannel` that is based on reference identity
* `StateFlow` cannot be closed like `ConflatedBroadcastChannel` and can never represent a failure. 

### StateFlow over Flow

There are some notable differences between StateFlow and Flow.

* `StateFlow` can have more than a collector.
* `StateFlow` doesn’t have an execution context by itself.
* `StateFlow` is a non-reactive flow, which means you can get the value using value property. It is useful if you need to know what’s the current value of the state without waiting for the stream to emit a new value.

### StateFlow over LiveData

`StateFlow` and `LiveData` have similarities. Both are observable data holder classes. But, this two do behave differently:

* `StateFlow` requires an initial state to be passed into the constructor, while `LiveData` does not.
* `LiveData.observe()` automatically unregisters the consumer when the view goes to the `STOPPED` state, whereas collecting from a `StateFlow` or any other flow does not.

You can’t create `StateFlow` directly. Instead, we can use `MutableStateFlow` to create the flow.

```Kotlin
private val mNumber = MutableStateFlow(0)
val number = mNumber.asStateFlow()

fun increment() {
    mNumber.value++
}
```


You can also convert a flow to StateFlow using stateIn operator.
```Kotlin
val state: StateFlow<Int> =
  flow {
    emit(1)
    emit(2)
  }.stateIn(scope, SharingStarted.Eagerly, initialState)
```


`stateIn` converts a _cold_ flow into a _hot_ `StateFlow `that is started in the given coroutine scope, sharing the most recently emitted value from a single running instance of the upstream flow with multiple downstream subscribers. The `stateIn` operator is useful in situations when there is a _cold_ flow that provides updates to the value of some state and is expensive to create and/or to maintain, but there are multiple subscribers that need to collect the most recent state value. And as you can see on the second param, there’s something called started param. There are options for started param:

* Eagerly Sharing is started immediately and never stops.
* Lazily Sharing is started when the first subscriber appears and never stops.
* WhileSubscribed Sharing is started when the first subscriber appears, immediately stops when the last subscriber disappears (by default), keeping the replay cache forever (by default).

## SharedFlow

`SharedFlow` is a _hot_ flow that emits values to all consumers that collect from it. A `SharedFlow` is a highly configurable generalization of `StateFlow`. Just like state flow, a shared flow is a _hot_ flow because its active instance exists independently of the presence of collectors. You can’t create a `SharedFlow` directly since it’s an interface. A mutable state flow is created using `MutableSharedFlow()` constructor.

### Replay cache and buffer

A shared flow keeps a specific number of the most recent values in its _replay cache_. Every new subscriber first gets the values from the replay cache and then gets new emitted values. The maximum size of the replay cache is specified when the shared flow is created by the replay parameter. A snapshot of the current replay cache is available via the replayCache property and it can be reset with the resetReplayCache function.

A replay cache also provides buffer for emissions to the shared flow, allowing slow subscribers to get values from the buffer without suspending emitters. The buffer space determines how much slow subscribers can lag from the fast ones.

### SharedFlow over ConflatedBroadcastChannel

`SharedFlow` is designed to completely replace `ConflatedBroadcastChannel`, just like `StateFlow`.

* `SharedFlow` is simpler because it does not have to implement all the `Channel` APIs
* `SharedFlow` supports configurable replay and buffer overflow strategy.
* `SharedFlow` cannot be closed like `BroadcastChannel` and can never represent a failure.

You can’t create `SharedFlow` directly. Instead, we can use `MutableSharedFlow` to create the flow.

```Kotlin
private val mAction = MutableSharedFlow<Action>()
val action = mAction.asSharedFlow()

fun onAction(action: Action) {
    mAction.emit(action)
}
```


You can also convert a flow to SharedFlow using shareIn operator.

```Kotlin
val state: SharedFlow<Action> =
  flow {
    emit(ActionSomething)
  }.shareIn(
      externalScope,
      replay = 1,
      started = SharingStarted.Eagerly
  )
```


`shareIn` converts a _cold_ flow into a _hot_ flow that is started in the given coroutine scope, sharing emissions from a single running instance of the upstream flow with multiple downstream subscribers, and replaying a specified number of replay values to new subscribers. Just like `stateIn`, the shareIn operator is useful in situations when there is a _cold_ flow that is expensive to create and/or to maintain, but there are multiple subscribers that need to collect its values. 

## ChannelFlow

`ChannelFlow` creates an instance of a _cold_ flow with elements that are sent to a `SendChannel` provided to the builder’s block of code via `ProducerScope`. It allows elements to be produced by code that is running in a different context or concurrently. The resulting flow is _cold_, which means that block is called every time a terminal operator is applied to the resulting flow. `ChannelFlow` ensures thread-safety and context preservation, thus the provided `ProducerScope` can be used concurrently from different contexts. 

`ChannelFlow` is experimental and you should add `@ExperimentalCoroutinesApi` annotation. (at least on Kotlin `1.4.32`)

# Channel

`Channel` is a non-blocking primitives for communication between a sender and a receiver. Conceptually, a channel is similar to Java’s `BlockingQueue`, but it has suspending operations instead of blocking and can be closed. In general, the concept of channel is pretty much the same with pub-sub, so any cases that can be represented as a pub-sub can use channel.

Unlike flow, you can emit and receive values from channel wherever you want via reference.

```Kotlin
val channel = Channel<Int>()
launch {
    for (x in 1..5) channel.send(x * x)
}

repeat(5) { println(channel.receive()) }
println("Done!")
```


Note that Kotlin `1.5.0` brings some refined API changes into Channel so some of the mentioned APIs here probably deprecated or become a stable API.

# Case study: Virgo

Let’s dive into our app as a case study.

## ViewModel

We are using an MVI pattern and have two components that we need to take care of, ViewState and ViewAction. Currently, we exposed ViewState using a LiveData that transforms ViewAction's MutableLiveData into ViewState. Let’s take a look at each component.

* ViewAction - We want it to be updated via onAction method that called on the main thread (UI layer). We can’t use a flow since you can’t emit something outside the building block using flow. Instead, using SharedFlow is a good practice here.

** ViewAction is not a state and there’s no guarantee an initial value always present on each ViewModel so StateFlow is not good.
** We want it to be updated in a non-reactive way using onAction method so we can’t useflow
* ViewState - It’s a perfect candidate for the StateFlow for some reason:

** It is a state holder.
** StateFlow has an initial value.
** StateFlow guarantees at least emitting a single value to the subscriber.

We also want to implement a side effect, an action that emitted as a side result of the user intent. I’m personally not quite grasp the full concept of the side effect, it is clear that side effect have some basic concepts, like it always have a single observer or subscriber to the side effect (mostly UI), and it should be something that recurring, a perfect example of SingleLiveEvent in the LiveData world. In this case, flow seems like good stuff to use because we can treat it as a ViewState and let the UI observe it. But we also want it to be updated in a non-reactive way, just like ViewAction. SharedFlow and Channel is good stuff to use here, but if want to make it behave like the SingleLiveEvent, Channel is a good stuff to use for some reason:

* Channel's event is delivered to a single subscriber. An attempt to post an event without subscribers will suspend as soon as the channel buffer becomes full, waiting for a subscriber to appear.
* Channel is _hot_, and this is good because its active instance exists independently of the presence of collectors, in this case a UI layer such as a Fragment.

Now let’s take a look at the BaseSideEffectViewModel:

```Kotlin
abstract class BaseSideEffectViewModel<ViewState : id.capitalx.mobile.core.model.ViewState, ViewAction : id.capitalx.mobile.core.model.ViewAction, SideEffect : id.capitalx.mobile.core.model.SideEffect>(
    initialState: ViewState
) : ViewModel() {

    private val viewAction: MutableSharedFlow<ViewAction> = MutableSharedFlow()

    private val mSideEffect: Channel<SideEffect> = Channel()
    val sideEffect = mSideEffect.receiveAsFlow()

    val viewState: StateFlow<ViewState> =
        viewAction.asSharedFlow().flatMapLatest(::handleAction).map { actionResult: ActionResult ->
            renderViewState(actionResult)
        }.stateIn(viewModelScope, SharingStarted.Eagerly, initialState)

    protected abstract fun renderViewState(result: ActionResult?): ViewState

    protected abstract fun handleAction(action: ViewAction): Flow<ActionResult>

    protected fun getCurrentViewState(): ViewState = viewState.value

    protected fun onSideEffect(sideEffectBuilder: suspend () -> SideEffect) {
        viewModelScope.launch {
            mSideEffect.send(sideEffectBuilder.invoke())
        }
    }

    @MainThread
    fun onAction(action: ViewAction) {
        viewModelScope.launch {
            viewAction.emit(action)
        }
    }
}
```


Note that the change is very small if you compared to the our current BaseViewModel. In the nutshell, we only change what data we’re exposing to the UI (from LiveData to Flow), we change the viewAction to a SharedFlow instead of just a LiveData and we implement SideEffect as a Channel and we update the value via onSideEffect method. In this implementation, we construct our viewState directly from the viewAction stream. This behavior is an attempt to match up with our BaseUseCase. This have some notable stuff that we need to take care of:

* We don’t construct the viewState using a MutableStateFlow.
* SharingStarted.Eagerly is not a good started option in term of resource because they will not closed even there is no subscriber.
* We construct the StateFlow and directly expose it. Initially I want to construct it outside the exposed value (via MutableStateFlow) but then it become a huge issue since we also rely on viewAction value changes.

Let’s take a look at the implementation of this new base ViewModel class:

```Kotlin
class HomeV2ViewModel(
  ///
) : BaseSideEffectViewModel<HomeV2ViewState, HomeV2ViewAction, HomeV2SideEffect>(initialState = HomeV2ViewState()) {

    private var balanceJob: Job? = null
    private var recentTransactionJob: Job? = null

    override fun renderViewState(result: ActionResult?): HomeV2ViewState = when (result) {
        ///
    }

    override fun handleAction(action: HomeV2ViewAction): Flow<ActionResult> =
        channelFlow {
            when (action) {
                HomeV2ViewAction.LoadHomeData -> {
                    if (shouldFetchHomeSections()) {
                        send(getHomeSectionsUseCase.getResult())
                    }
                    // fetch user data from BE if connection available, else load from local
                    val userActionResult =
                        getUserUseCase.getResult(param = connectivityChecker.isConnectedToInternet())
                    send(userActionResult)
                    checkVerificationStatus(userActionResult)
                    if (connectivityChecker.isConnectedToInternet()) {
                        refreshBalance()
                        refreshRecentTransaction()
                        send(hasUnreadNotificationUseCase.getResult())
                    } else {
                        send(GoToNoInternetActionResult)
                    }
                }
                HomeV2ViewAction.RefreshBalance -> {
                    refreshBalance()
                }
                HomeV2ViewAction.RefreshRecentTransaction -> {
                    refreshRecentTransaction()
                }

            awaitClose()
        }

    private fun shouldFetchHomeSections(): Boolean =
        getCurrentViewState().sections.peekValue().isEmpty()

    private suspend fun ProducerScope<ActionResult>.refreshBalance() {
        val user = (getUserUseCase.getResult() as? GetUserActionResult.Success)?.user
        val balanceTypeParam = GetBalancesByTypeUseCase.Param(
            customerId = user?.customerId.toString(),
            phoneNumber = user?.mobileNumber.orEmpty()
        )

        balanceJob?.cancelIfActive()
        balanceJob = viewModelScope.launch {
            getBalancesByTypeUseCase.getResult(balanceTypeParam).collectLatest {
                onSideEffect { HomeV2SideEffect.UpdateBalance }
                channel.send(it)
            }
        }
    }

    private suspend fun ProducerScope<ActionResult>.refreshRecentTransaction() {
        val user = (getUserUseCase.getResult() as? GetUserActionResult.Success)?.user

        recentTransactionJob?.cancelIfActive()
        recentTransactionJob = viewModelScope.launch {
            getRecentHistoryTransactionUseCase.getResult(user?.mobileNumber).collectLatest {
                channel.send(it)
            }
        }
    }

    private suspend fun ProducerScope<ActionResult>.checkVerificationStatus(user: GetUserActionResult) {
        val verificationStatus =
            (user as? GetUserActionResult.Success)?.user?.verificationStatus ?: 0
        val showVerificationStatus =
            (checkStatusKycHomeNotificationUseCase.getResult() as? CheckStatusKycHomeNotificationUseCaseActionResult.Success)?.isShow
                ?: false
        send(HomeV2ActionResult.SetVerificationStatus(verificationStatus, showVerificationStatus))
    }
}
```


As you can see, we use channelFlow instead of flow for our handleAction method. This is done because we need to collect our stream from observable UseCase from different scope because if it’s on the same coroutine scope with the other one-shot UseCase, the observable UseCase will suspend the other UseCase below since it never stopped, which mean the other UseCase will not be able to emit a data to our ViewState. Hence, we use channelFlow to make sure we can send a value from different coroutine scope, in this case from getRecentHistoryTransactionUseCase and getBalancesByTypeUseCase. But we need to make sure we close the scope because launch block creates a new coroutines, which mean if we try to navigate to other screen and go back to home screen, it’ll create another scope which is something we don’t want to expect.

The changes in the UI layer is down to a very minimum, because basically we only change our perception from “observing” a viewState to “collecting” value from a viewState stream.

```Kotlin
class HomeV2Fragment : BaseFragment(R.layout.fragment_home_v2) {
  ///
  
  override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        ///
        lifecycleScope.launchWhenStarted {
            homeViewModel.viewState.collect {
                it.sections.onValueChanged(homeSectionAdapter::updateHomeSections)
                it.balanceViewState.onValueChanged(homeSectionAdapter::updateBalanceViewState)
                it.profileViewState.onValueChanged(homeSectionAdapter::updateProfileViewState)
                it.recentTransactionViewState.onValueChanged(homeSectionAdapter::updateRecentTransactionViewState)
                it.showUnreadNotificationIcon.onValueChanged(homeSectionAdapter::updateNotificationIconState)
                it.kycVerificationStatusViewState.onValueChanged(homeSectionAdapter::updateVerificationStatusState)
                it.navigateAction.onValueChanged { action ->
                    action?.invoke(findNavController())
                }
            }
        }

        lifecycleScope.launchWhenStarted {
            homeViewModel.sideEffect.collect {
                when (it) {
                    HomeV2SideEffect.UpdateBalance -> {
                        Timber.d("Balance updated!")
                    }
                }
            }
        }
    }
    
    ///
}
```


We wrap our collect block inside launchWhenStarted block. This is because, unlike LiveData, Flow is not designed to lifecycle aware, so we need to “manually” handle the lifecycle. We also now not collecting viewState but also the sideEffect that emitted from the ViewModel. Again, same like the viewState's collect, we need to wrap it inside the launchWhenStarted block, making sure that the scope is launched when the lifecycle at least in Lifecycle.State.STARTED state.

## UseCase and data source

Currently, we have suspended functions for every UseCase we have. This is good for a one-shot operation like fetching user detail, getting the inquiry response, etc. But of course, our app UseCase is not always a one-shot operation. Let’s take a look again at home. We have several actions going on at home, getting user detail, fetching the home section, getting user balance, getting recent transactions, etc. If you take a look, getting user balance and recent transactions are actually a perfect fit for an observable operation since they should be updated over time to make sure the displayed result on the screen is representing the actual balance and recent transactions that users have.

To support it we need to change our UseCase to support emitting stream of data instead of one single operation. We have two options here, using flow or channel. Looking at how [Google’s adssched|https://github.com/google/iosched/tree/adssched]implementation, they prefer flow over channel for some reasons:

* Prefer exposing flow since it gives you more flexibility, more explicit contracts, and operators thanchannel
* flow automatically close the stream of data due to the nature of the terminal operators which trigger the execution of the stream of data and complete successfully or exceptionally depending on all the flow operations in the producer side. On the other hand, the producer might not clean up heavy resources if the Channel is not closed properly which possibly can leaks resources.

Why not using StateFlow? StateFlow is also a good candidate for exposing stream of data from our UseCase since we wrap the data with DataResult which is also considered a state. But due to the natural behavior of StateFlow which is a _hot_ flow and we can’t directly create the StateFlow, it is clear that is not a perfect candidate for our UseCase and data sources.

Now let’s take a look at the implementation of FlowUseCase:

```Kotlin
abstract class FlowUseCase<RequestParam, ResponseObject, Result : ActionResult>
    (private val dispatcher: CoroutineDispatcher) {

    protected abstract fun execute(param: RequestParam): Flow<DataResult<ResponseObject>>

    protected abstract fun transformToFlowUseCaseResult(response: DataResult<ResponseObject>): Result

    fun getResult(param: RequestParam): Flow<Result> = execute(param)
        .map { responseObject -> transformToFlowUseCaseResult(responseObject) }
        .flowOn(dispatcher)

}
```


As you can see, it almost looks the same as our current BaseUseCase. In fact, it only changes the execute return type and the getResult operations. So what happened here is execute will return a flow of DataResult<ResponseObject> from the data source. You can do some mapping or some transform before emitting the data here which we will look on the example. And then just like our current BaseUseCase operations, we transform the ResponseObject to our actual ActionResult. Here, we heavily use map block and transformToFLowUseCaseResult to map our ResponseObject. Lastly, we specify where the flow is executed to the given coroutine dispatcher using flowOn.

And that’s it, now you’re ready to observe your stuff using FlowUseCase. Let’s take a look at how we use it to get balance usecase starts with the repository!

```Kotlin
fun fetchAccountInfoAsFlow(
    customerId: String,
    accountType: AccountType
): Flow<DataResult<List<AccountInfo>>>
```


So if you look at the AccountNetworkDataSource we specify a function to return a flow of data result of  list of account info. This is what FlowUseCase will look into. We expose the stream from the bottom layer of our architecture, the data source, and then into the repository. Nothing too fancy here, now let’s move into the GetBalancesByTypeFlowUseCase :

```Kotlin
class GetBalancesByTypeFlowUseCase(
    private val accountRepository: AccountRepository,
    coroutineDispatcher: CoroutineDispatcher
) : FlowUseCase<GetBalancesByTypeUseCase.Param, List<AccountInfo>, GetAccountInfosActionResult>(
    coroutineDispatcher
) {

    override fun execute(param: GetBalancesByTypeUseCase.Param): Flow<DataResult<List<AccountInfo>>> {
        return flow {
            if (param.phoneNumber.isBlank()) emit(DataResult.Exception(IllegalArgumentException("phoneNumber cannot be blank")))
            if (param.customerId.isBlank()) emit(DataResult.Exception(IllegalArgumentException("customerId cannot be blank")))
            if (param.accountType == AccountType.Undefined) emit(
                DataResult.Exception(
                    IllegalArgumentException("Undefined account type is not supported")
                )
            )

            accountRepository.fetchAccountInfoAsFlow(
                customerId = param.customerId,
                accountType = param.accountType
            ).collect {
                emit(it.map { networkAccounts: List<AccountInfo> ->
                    accountRepository.replaceAccounts(param.phoneNumber, networkAccounts, false)
                    networkAccounts.firstOrNull()?.let { accountInfo ->
                        accountRepository.saveDefaultEmoneyAccountId(accountInfo.id)
                    }

                    DataResult.Success(networkAccounts)
                })
            }
        }
    }

    override fun transformToFlowUseCaseResult(response: DataResult<List<AccountInfo>>): GetAccountInfosActionResult {
        return when (response) {
            is DataResult.Success -> {
                if (response.value.isEmpty()) {
                    GetAccountInfosActionResult.Empty
                } else {
                    GetAccountInfosActionResult.Success(response.value)
                }
            }
            is DataResult.Exception -> {
                GetAccountInfosActionResult.Failed(
                    response.throwable.message.orEmpty()
                )
            }
        }
    }
}
```

As you can see there, we’re not returning anything from the execute block, instead, we emit the data. Before we emit the data, we have some stuff going on in the map block. To be honest, I’m not quite sure whether we need to do it every time we fetch the balance (especially calling replaceAccounts every time we fetch the balance) but I will leave it as is for now. And that’s it, you can now exposing your UseCase stream to our ViewModel that we already modified to support collecting the data from the stream!

# Extra: Jetpack Compose and Flow

Jetpack Compose concepts are heavily based on the “State” of the UI. By default, Jetpack Compose already supports Flow and StateFlow to handle UI recomposing when the state is changing. Every composable value that emitted to the tree is having the State. State is a value holder where reads to the value property during the execution of a Composable function, the current RecomposeScope will be subscribed to changes of that value.

Take a look at this code snippet:

```Kotlin
val projectList by remember(homeViewModel) { homeViewModel.loadProjectList() }.collectAsState(
    initial = Result.Loading
)

val logList by remember(homeViewModel) {
    homeViewModel.loadLog()
}.collectAsState(initial = Result.Empty)
@Composable
fun <T> NanakuraResultContainer(
    result: Result<T>?,
    loadingContent: (@Composable () -> Unit)? = null,
    failedContent: (@Composable () -> Unit)? = null,
    emptyContent: (@Composable () -> Unit)? = null,
    successContent: @Composable (T) -> Unit
) {
    when (result) {
        is Result.Success -> {
            successContent(result.data)
        }
        is Result.Loading -> {
            if (loadingContent != null)
                loadingContent()
            else
                Text(
                    text = "fetching your data ...",
                    textAlign = TextAlign.Center,
                    modifier = Modifier.fillMaxWidth()
                )
        }
        is Result.Empty -> {
            if (emptyContent != null)
                emptyContent()
            else
                Text(
                    text = "nothing here, so quite :(",
                    textAlign = TextAlign.Center,
                    modifier = Modifier.fillMaxWidth()
                )
        }
        else -> {
            if (failedContent != null)
                failedContent()
            else
                Text(
                    text = "something went wrong :(",
                    textAlign = TextAlign.Center,
                    modifier = Modifier.fillMaxWidth()
                )
        }
    }
}
```


As you can see, we can collect Flow and StateFlow and then remember it as a composable UI’s state via collectAsState method.

----

## References:

[Flow - kotlinx-coroutines-core](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/)

[Sequences | Kotlin (kotlinlang.org)](https://kotlinlang.org/docs/sequences.html)

[Releases · Kotlin/kotlinx.coroutines (github.com)](https://github.com/Kotlin/kotlinx.coroutines/releases)

[Kotlin flows on Android | Android Developers](https://developer.android.com/kotlin/flow)

[StateFlow - kotlinx-coroutines-core](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)

[stateIn - kotlinx-coroutines-core](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/state-in.html)

[shareIn - kotlinx-coroutines-core](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/share-in.html)

[SharedFlow - kotlinx-coroutines-core](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/index.html)

[Channel - kotlinx-coroutines-core](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html)

[StateFlow and SharedFlow | Android Developers](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)

[Additional resources for Kotlin coroutines and flow (android.com)](https://developer.android.com/kotlin/coroutines/additional-resources)

[google/iosched at adssched (github.com)](https://github.com/google/iosched/tree/adssched)

[androidx.compose.runtime | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary)

Good to read:

[Lessons learnt using Coroutines Flow in the Android Dev Summit 2019 app | by Manuel Vivo | Android Developers | Medium](https://medium.com/androiddevelopers/lessons-learnt-using-coroutines-flow-4a6b285c0d06)

[Migrating from LiveData to Kotlin’s Flow | by Jose Alcérreca | Android Developers | May, 2021 | Medium](https://medium.com/androiddevelopers/migrating-from-livedata-to-kotlins-flow-379292f419fb)

[MVI — another member of the MV* band | by Iveta Jurčíková | ProAndroidDev](https://proandroiddev.com/mvi-a-new-member-of-the-mv-band-6f7f0d23bc8a)

[Going deep on Flows & Channels — Part 2: Flows | ProAndroidDev](https://proandroiddev.com/going-deep-on-flows-channels-part-2-flows-c30fee00c2a4)