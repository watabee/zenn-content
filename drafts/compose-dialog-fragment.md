
これは BottomSheetDialogFragment を使って Compose の画面を表示することを紹介する記事です。

## この記事を書く理由

- 弊社で提供しているモバイルアプリ KANNA は React Native で作られているが、一部の機能や画面を Kotlin、Compose を使って開発している
- その際に BottomSheetDialogFragment を使うことで、React Native の世界から脱却して開発することができる

## この記事で伝えたいこと

- BottomSheetDialogFragment を全画面で表示し、スワイプを無効にする方法
- BottomSheetDialogFragment で Compose を表示するための設定
- navigation-compose を使う場合のバックキーの制御方法

## BottomSheetDialogFragment を全画面で表示し、スワイプを無効にする方法

ここでは BottomSheetDialogFragment を継承した ComposeDialogFragment を作成している。
StackOverflow を参考に、以下のようにすることで全画面表示し、スワイプ操作を無効にできる。

```kotlin
abstract class ComposeDialogFragment : BottomSheetDialogFragment() {

    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog {
        val dialog = BottomSheetDialog(requireContext(), theme)

        fun setupFullHeight(bottomSheet: View) {
            val layoutParams = bottomSheet.layoutParams
            layoutParams.height = WindowManager.LayoutParams.MATCH_PARENT
            bottomSheet.layoutParams = layoutParams
        }

        // ボトムシートの初期表示をフルスクリーンにする.
        // https://stackoverflow.com/a/62958074
        dialog.setOnShowListener {
            val bottomSheet: View = dialog.findViewById(
                com.google.android.material.R.id.design_bottom_sheet
            ) ?: return@setOnShowListener

            val behavior = BottomSheetBehavior.from(bottomSheet)
            setupFullHeight(bottomSheet)
            behavior.state = BottomSheetBehavior.STATE_EXPANDED

            // ボトムシートのスワイプ操作を無効にする.
            // https://stackoverflow.com/a/35794743
            behavior.addBottomSheetCallback(object : BottomSheetBehavior.BottomSheetCallback() {
                override fun onStateChanged(bottomSheet: View, newState: Int) {
                    if (newState == BottomSheetBehavior.STATE_DRAGGING) {
                        behavior.state = BottomSheetBehavior.STATE_EXPANDED
                    }
                }

                override fun onSlide(bottomSheet: View, slideOffset: Float) {
                    // do nothing.
                }
            })
        }
        return dialog
    }
}
```

## BottomSheetDialogFragment で Compose を表示するための設定

`onCreateView()` で `ComposeView` のインスタンスを作り、Compose の UI を表示できるようにする。
その際に `setViewCompositionStrategy()` メソッドに `ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed` を設定する必要がある。
参考: https://developer.android.com/develop/ui/compose/migrate/interoperability-apis/compose-in-views#compose-in-fragments

```kotlin
abstract class ComposeDialogFragment : BottomSheetDialogFragment() {

    ...

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return ComposeView(requireContext()).apply {
            setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
            setContent {
                AppTheme {
                    Surface(modifier = Modifier.fillMaxSize()) {
                        this@ComposeDialogFragment.Content()
                    }
                }
            }
        }
    }

    @Composable
    abstract fun Content()
}
```

## navigation-compose を使う場合のバックキーの制御方法

バックキーを押下すると Fragment が閉じられてしまう。
もし navigation-compose を使う場合、バックキーが押された時の処理を別途実装する必要がある。

以下の方針で実装する。

- NavHost をラップする関数を作る
- `BackHandler` を使ってバックキーで Navigation で行った画面遷移を戻れるようにする。 Navigation での画面遷移で戻れなかった場合、Fragment が閉じられるようにする
- そのために `CompositionLocalProvider()` を使って `LocalOnBackPressedDispatcherOwner` を `BottomSheetDialog` に上書きする


```kotlin
abstract class ComposeDialogFragment : BottomSheetDialogFragment() {

    ...

    @Composable
    protected fun ComposeDialogFragmentNavHost(
        navController: NavHostController,
        startDestination: Any,
        modifier: Modifier = Modifier,
        contentAlignment: Alignment = Alignment.TopStart,
        route: KClass<*>? = null,
        typeMap: Map<KType, @JvmSuppressWildcards NavType<*>> = emptyMap(),
        enterTransition:
        (@JvmSuppressWildcards
        AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition) =
            {
                fadeIn(animationSpec = tween(700))
            },
        exitTransition:
        (@JvmSuppressWildcards
        AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition) =
            {
                fadeOut(animationSpec = tween(700))
            },
        popEnterTransition:
        (@JvmSuppressWildcards
        AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition) =
            enterTransition,
        popExitTransition:
        (@JvmSuppressWildcards
        AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition) =
            exitTransition,
        sizeTransform:
        (@JvmSuppressWildcards
        AnimatedContentTransitionScope<NavBackStackEntry>.() -> SizeTransform?)? =
            null,
        builder: NavGraphBuilder.() -> Unit,
    ) {
        CompositionLocalProvider(LocalOnBackPressedDispatcherOwner provides dialog as BottomSheetDialog) {
            val canGoBack by produceState(false) {
                @SuppressLint("RestrictedApi")
                navController.currentBackStack
                    .map { backStack -> backStack.filter { it.destination !is NavGraph }.size > 1 }
                    .collect { value = it }
            }
            BackHandler(enabled = canGoBack) {
                navController.popBackStack()
            }

            NavHost(
                navController = navController,
                startDestination = startDestination,
                modifier = modifier,
                contentAlignment = contentAlignment,
                route = route,
                typeMap = typeMap,
                enterTransition = enterTransition,
                exitTransition = exitTransition,
                popEnterTransition = popEnterTransition,
                popExitTransition = popExitTransition,
                sizeTransform = sizeTransform,
                builder = builder,
            )
        }
    }
}
```

上記で定義した `ComposeDialogFragmentNavHost` を使うサンプルは以下。
`ComposeDialogFragment` を継承したクラスを作成し、`Content` 関数をオーバーライドして `ComposeDialogFragmentNavHost` を呼び出している。

```kotlin
class SampleDialogFragment : ComposeDialogFragment() {
    @Composable
    override fun Content() {
        val onBackPressedDispatcher = LocalOnBackPressedDispatcherOwner.current?.onBackPressedDispatcher
        val navController = rememberNavController()
        ComposeDialogFragmentNavHost(navController = navController, startDestination = FirstScreen) {
            composable<FirstScreen> {
                Screen(
                    title = "First Screen",
                    onBackButtonClicked = { onBackPressedDispatcher?.onBackPressed() },
                ) {
                    Button(onClick = { navController.navigate(SecondScreen) }) {
                        Text(text = "Go to Second Screen")
                    }
                }
            }

            composable<SecondScreen> {
                Screen(
                    title = "Second Screen",
                    onBackButtonClicked = { onBackPressedDispatcher?.onBackPressed() },
                ) {
                    Button(onClick = { navController.navigate(ThirdScreen) }) {
                        Text(text = "Go to Third Screen")
                    }
                }
            }
        }
    }
}
```
