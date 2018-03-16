#   Activity的启动模式
在AndroidManifest.xml文件中,对Activity可以设置的LunchMode有4种值.
```
1.  standard模式: 在不给Activity特别指定的情况下,standard启是默认启动默认.
    对于使用该启动模式的Activity,只要发送一个打开该Activity的Intent.都会创建一个新的Activity实例.

2.  singleTop模式:
    singleTop和standar只有一个区别,那就是当singleTop的Activity任务栈顶的时候,不会打开一个新的Activity.而是直接使用任务栈顶的Activity实例
    接受Intent.(系统会调用onNewIntent方法).其他情况下,standard和singleTop是一样的.

3.  singleTask模式:
    当前是否存在需要打开Activity的任务栈.不存在创建对应任务栈,并创建Activity实例入栈.存在任务栈,则检查当前栈中是否存在需要的Activity实例,如果存在,
    则将当前Activity上面的其他Activity出栈.目标Activity移动到栈顶.系统调用其onNewIntent接受intent.如果不存在,则创建目标Activity实例放在栈顶.

4. singleInstance模式:
   对于该模式的Actiivty.其任务栈始终都只有目标Activity一个.且始终打开的都是同一个实例.在该Activity上打开别的Activity,也会将Activity放入别的任务栈.
```

同样的,Intent中也有几个flag是可以配合启动模式来打开activity的.
```
1.  FLAG_ACTIVITY_NEW_TASK:
    在新任务中启动 Activity,如果已为正在启动的 Activity 运行任务，则该任务会转到前台并恢复其最后状态，
    同时 Activity 会在 onNewIntent() 中收到新 Intent.

2.  FLAG_ACTIVITY_SINGLE_TOP:
    如果正在启动的 Activity 是当前 Activity（位于返回栈的顶部），则 现有实例会接收对 onNewIntent() 的调用，而不是创建 Activity 的新实例.

3.  FLAG_ACTIVITY_CLEAR_TOP:
    如果正在启动的 Activity 已在当前任务中运行，则会销毁当前任务顶部的所有 Activity，
    并通过 onNewIntent() 将此 Intent 传递给 Activity 已恢复的实例（现在位于顶部），而不是启动该 Activity 的新实例.
```

Activity在Manifest还有几个可以配置的属性比较重要:
```
1.  android:taskAffinity:
    与 Activity 有着亲和关系的任务。从概念上讲，具有相同亲和关系的 Activity 归属同一任务（从用户的角度来看，则是归属同一“应用”）*[]:
    任务的亲和关系由其根 Activity 的亲和关系确定。亲和关系确定两件事 - Activity 更改到的父项任务（请参阅 allowTaskReparenting 属性）和通过 FLAG_ACTIVITY_NEW_TASK
    标志启动 Activity 时将用来容纳它的任务。 默认情况下，应用中的所有 Activity 都具有相同的亲和关系。您可以设置该属性来以不同方式组合它们，甚至可以将在不同应用中定义的 Activity 置于同一任务内。
    要指定 Activity 与任何任务均无亲和关系，请将其设置为空字符串。如果未设置该属性，则 Activity 继承为应用设置的亲和关系（请参阅 <application> 元素的 taskAffinity 属性)
    应用默认亲和关系的名称是 <manifest> 元素设置的软件包名称。

2.` android:allowTaskReparenting:
    当启动 Activity 的任务接下来转至前台时，Activity 是否能从该任务转移至与其有亲和关系的任务 —“true”表示它可以转移，“false”表示它仍须留在启动它的任务处。
    如果未设置该属性，则对 Activity 应用由 <application> 元素的相应 allowTaskReparenting 属性设置的值。 默认值为“false”。

3.  android:alwaysRetainTaskState:  只可配置在任务的根Activity上.
  系统是否始终保持 Activity 所在任务的状态 —“true”表示保持，“false”表示允许系统在特定情况下将任务重置到其初始状态。
  默认值为“false”。该属性只对任务的根 Activity 有意义；对于所有其他 Activity，均忽略该属性。
  正常情况下，当用户从主屏幕重新选择某个任务时，系统会在特定情况下清除该任务（从根 Activity 之上的堆栈中移除所有 Activity）。
  系统通常会在用户一段时间（如 30 分钟）内未访问任务时执行此操作。不过，如果该属性的值是“true”，则无论用户如何到达任务，
  将始终返回到最后状态的任务。 例如，在网络浏览器这类存在大量用户不愿失去的状态（如多个打开的标签）的应用中，该属性会很有用。

4.  android:clearTaskOnLaunch: 只能配置在任务的根Activity上其作用.
    如果根Activty在打开下一个Activity时销毁了.该属性还是会起作用,会转到任务当前的根Activity身上.

    是否每当从主屏幕重新启动任务时都从中移除根 Activity 之外的所有 Activity —“true”表示始终将任务清除到只剩其根 Activity.
    “false”表示不做清除。 默认值为“false”。该属性只对启动新任务的 Activity（根 Activity）有意义；对于任务中的所有其他 Activity，均忽略该属性。
    当值为“true”时，每次用户再次启动任务时，无论用户最后在任务中正在执行哪个 Activity，也无论用户是使用返回还是主屏幕按钮离开，
    都会将用户转至任务的根 Activity。 当值为“false”时，可在某些情况下清除任务中的 Activity（请参阅 alwaysRetainTaskState 属性），但并非一律可以。
    例如，假定有人从主屏幕启动了 Activity P，然后从那里转到 Activity Q。该用户接着按了主屏幕按钮，然后返回到 Activity P。正常情况下，
    用户将看到 Activity Q，因为那是其最后在 P 的任务中执行的 Activity。 不过，如果 P 将此标志设置为“true”，则当用户按下主屏幕将任务转入后台时，
    其上的所有 Activity（在本例中为 Q）都会被移除。 因此用户返回任务时只会看到 P。
     如果该属性和 allowTaskReparenting 的值均为“true”，则如上所述，任何可以更改父项的 Activity 都将转移到与其有亲和关系的任务；
     其余 Activity 随即被移除。

5. android:finishOnTaskLaunch: 可配置在任何Activity上.
    每当用户再次启动其任务（在主屏幕上选择任务）时，是否应关闭（完成）现有 Activity 实例 —“true”表示应关闭，“false”表示不应关闭。
    默认值为“false”。
    如果该属性和 allowTaskReparenting 均为“true”，则优先使用该属性。 Activity 的亲和关系会被忽略。
    系统不是更改 Activity 的父项，而是将其销毁。

6.  android:excludeFromRecents
    是否应将该 Activity 启动的任务排除在最近使用的应用列表（即概览屏幕）之外。 也就是说，当该 Activity 是新任务的根 Activity 时，
    此属性确定任务是否应出现在最近使用的应用列表中。
    如果应将任务排除在列表之外，请设置“true”；如果应将其包括在内，则设置“false”。 默认值为“false”。
```