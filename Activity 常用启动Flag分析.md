# Activity常用启动Flag和启动模式分析  


## 1. Tasks和Back Stack  
  
  关于Stack和Tasks的官方有这样的一段描述：
  > A task is a collection of activities that users interact with when performing a certain job. The activities are arranged in a stack (the "back stack"), in the order in which each activity is opened.  
  
  个人对这段话的理解是：Task是一些完成特定工作的一些Activity的集合，这些Activity被这个Task的Back Stack所管理。  
  Activity中可以在manifest中定义四种启动模式，这四种模式会影响到Activity的Task或者对应的Back Stack。除此之外在启动Activity的Intent中也可以通过设置Flag来影响Task和Back Stack，Flag一共有20种，而且不同的Flag之间还可以相互组合，相对于manifest中定义的启动模式要复杂很多。  
  
  
  开始以下内容前先理解下官方文档对于taskAffinity属性的描述：
  
  > The task that the activity has an affinity for. Activities with the same affinity conceptually belong to the same task (to the same "application" from the user's perspective). The affinity of a task is determined by the affinity of its root activity.
The affinity determines two things — the task that the activity is re-parented to (see the allowTaskReparenting attribute) and the task that will house the activity when it is launched with the FLAG\_ACTIVITY\_NEW\_TASK flag.  
By default, all activities in an application have the same affinity. You can set this attribute to group them differently, and even place activities defined in different applications within the same task. To specify that the activity does not have an affinity for any task, set it to an empty string.  
If this attribute is not set, the activity inherits the affinity set for the application (see the <application> element's taskAffinity attribute). The name of the default affinity for an application is the package name set by the <manifest> element.  


## 2. Activity的四种启动模式  
  
  首先看看官方的描述：  
  
|  Use Cases | Launch Mode |  Multiple Instances?| Comments |
|:-----|:-----|:-----|:-----|
|Normal launches for most activities |  "standard"  |   Yes  | 	Default. The system always creates a new instance of the activity in the target task and routes the intent to it. |
|Normal launches for most activities |  "standard"  |   Yes  | 	Default. The system always creates a new instance of the activity in the target task and routes the intent to it. |
|Normal launches for most activities  |  "singleTop"  |  Conditionally | 	If an instance of the activity already exists at the top of the target task, the system routes the intent to that instance through a call to its onNewIntent() method, rather than creating a new instance of the activity. |
|Specialized launches (not recommended for general use) | "singleTask"	 |  No  | The system creates the activity at the root of a new task and routes the intent to it. However, if an instance of the activity already exists, the system routes the intent to existing instance through a call to its onNewIntent() method, rather than creating a new one. |
|Specialized launches (not recommended for general use) |  "singleInstance"  |   No  | 	Same as "singleTask", except that the system doesn't launch any other activities into the task holding the instance. The activity is always the single and only member of its task. |
  
### standard
  
  这是Activity默认的启动模式，允许存在多个实例，将新的Activity实例放在目标task的Stack顶部。 


### singleTop
  
  将创建的Activity放置于目标task的栈顶，如果栈顶存在当前activity就不会创建实例，同时会调用已经存在实例的onNewIntent()方法。反之就和standard的表现一样。  
  
### singleTask
  
  该模式不允许Activity存在多个实例，系统会为该Activity创建一个新的task同时将该Activity设置为Stack中的Root Activity。（注意：当该Activity没有指定taskAffinity时，是不会创建新的task的，而是放置在启动该Activity的task上。） 如果该Activity已经启动，则会使在该Activity之上的其他Activity全部出栈，同时调用该Activity的onNewIntent()方法。  
  
### singleInstance
  
  该模式大部分情况和singleTask的表现一样。不一样的地方：就算没有指定taskAffinity系统也会为该Activity创建一个task，这个task中不能容纳其他的Activity。
  
  
## 3. Activity启动Flag分析
  
### FLAG\_ACTIVITY\_CLEAR\_TASK

  指定该Flag的intent会在执行前将该task栈中的Activity全部finish掉，前提是该Flag必须和FLAG\_ACTIVITY\_NEW\_TASK结合使用。（注意：只有当新启动的Activity是和原来task一样的时候才会有效。）  
  
### FLAG\_ACTIVITY\_CLEAR\_TOP
  
  如果启动的目标Activity在栈顶的话，会杀死该Activity重新创建，如果不是栈顶，在他之上的Activity都会被杀死。如果该和FLAG\_ACTIVITY\_NEW\_TASK一起使用还可以达到切换栈的效果。
  
### FLAG\_ACTIVITY\_MULTIPLE\_TASK
  
  该Flag通常和FLAG\_ACTIVITY\_NEW\_DOCUMENT或者FLAG\_ACTIVITY\_NEW\_TASK一起使用。该flag能够强制创建一个task，默认情况下如果一个task已经存在是不会再创建的。
  
### FLAG\_ACTIVITY\_NEW\_DOCUMENT
  
  关于document的含义我也没太明白，可能就是在点击recent tasks键（原生安卓三大金刚键中最右边的那个键）后显示的卡片。  
  下面是比较官方的描述(没太懂)：
  
  > We call multiple tasks “concurrent documents” and concurrent documents is what you see when you see two or more cards in the recents menu, all from one app. Since Android 5.0 allows more than one task from each app to appear in the Overview screen, it’s useful to know how to leverage concurrent documents in your own app. Especially because doing so can make it easier for users to do things and that will make users love you.  
We call it “concurrent” because your app is multi-tasking here, as the user interacts with multiple parts of your app at once. And we refer to these separate instances as “documents” because we didn’t have a better way to say “anything that you create that lives separately from the rest of your app.”  
That “document” could actually be a document, in a text editor. But it could also be that draft email we discussed earlier, or a tab in a browser, or even a specific conversation view within a messaging app. Your imagination is your only limit here.
  
  
  现在我们继续回到这个Flag。这个Flag也可以在Mainfest中的documentLaunchMode属性设置，直接用这个flag等于intoExisting属性，这个Flag和FLAG\_ACTIVITY\_MULTIPLE\_TASK等于always。直接添加这个flag，如果这个task不存在，会创建一个新的task中的启动这个Activity同时为Root Activity，如果这个task存在就会切换到对应的task，同时使对应的activity栈上面的activity全部出栈；如果和FLAG\_ACTIVITY\_MULTIPLE\_TASK一起使用会无脑创建新的task。  
  
### FLAG\_ACTIVITY\_NEW\_TASK
  
  这个Flag和FLAG\_ACTIVITY\_NEW\_DOCUMENT非常类似和FLAG\_ACTIVITY\_MULTIPLE\_TASK Flag的组合效果也一样。不同点是：如果该Flag单独使用同时启动的目标Activity没有指定taskAffinity属性时（用的是默认值，为当前App的ApplicationId），则不会有效果，如果有指定taskAffinity值则效果和FLAG\_ACTIVITY\_NEW\_DOCUMENT一样。  
  
### FLAG\_ACTIVITY\_NO\_ANIMATION
  
  
  这个Flag没有太多讲的，禁止使用Activity启动的动画。有一点要注意：当有的时候没有创建新的Activity的时候可能会失效，例如使用FLAG\_ACTIVITY\_REORDER\_TO\_FRONT时就会失效。  
  
### FLAG\_ACTIVITY\_NO\_HISTORY

  
  设置该Flag，不会让目标Activity保存在task栈中，同时旧Activity的onActivityRresult方法永远不会被触发。
  

### FLAG\_ACTIVITY\_REORDER\_TO\_FRONT

  如果当前task中没有目标Activity就会重新创建一个Activity并加入到task栈中，如果有就会把目标Activity移动到栈顶。如果和FLAG\_ACTIVITY\_CLEAR\_TOP一起使用则该Flag失效。  
  
### FLAG\_ACTIVITY\_RETAIN\_IN\_RECENTS
  
  当一个task中的activity全部finish后，就会在recent tasks移除该task（也就是任务卡片中没法看到该task了）。但是如果设置了该flag会使该task中的activity全部finish后依然可以出现在recent tasks中，当选中该卡片会重新创建该task的root activity。当在manifest中设置autoRemoveFromRecents属性或者调用Activity的finishAndRemoveTask()方法也会使该flag失效。  
  

### FLAG\_ACTIVITY\_SINGLE\_TOP

  和启动模式中的singleTop是一样的。  
  
### FLAG\_ACTIVITY\_FORWARD\_RESULT

  假设有这样一个Activity调用顺序A->B->C，A通过startActivityForResult启动B，B然后启动C同时加入FLAG\_ACTIVITY\_FORWARD\_RESULT flag，这时C调用setResult方法时B是收不到result的，而是直接通知A。  
  
### FLAG\_ACTIVITY\_EXCLUDE\_FROM\_RECENTS

  如果启动的Activity是task的Root Activity，那么该task不会保存在recent tasks中。  
  
  
### FLAG\_ACTIVITY\_RESET\_TASK\_IF\_NEEDED
  
  该Flag单独使用没有效果，需要和FLAG\_ACTIVITY\_NEW\_TASK结合使用。结合使用时，如果没有task有对应的目标Activity的实例，就会创建一个新的task来容纳该Activity；如果目标Activity在当前的task有目标Activity实例则没有效果；如果在其他的task有目标Activity的实例，就会切换到目标task。（注意：仅仅是切换task，而不会有Activity的创建和finish）
  
  
### FLAG\_ACTIVITY\_TASK\_ON\_HOME

  该Flag单独使用没有效果，需要结合FLAG\_ACTIVITY\_NEW\_TASK或者FLAG\_ACTIVITY\_NEW\_DOCUMENT一起使用。使用该Flag移动到另外一个task后，在那个task中的Activity全部结束后直接返回HOME。例如TaskA中的栈为A->B->C，当在C中使用该Flag在TaskB中启动D。 这时D finish后不会返回TaskA，而是直接返回HOME。  
  
  
### 小结
  
  还有一些Flag没有分析，有些是已经弃用了，有新的Flag可以代替；有些在官方文档中明确大部分时候是系统使用的Flag；还有一些我也没搞清楚怎么用。但是以上的Flag可以满足绝大部分的使用场景。而且这些Flag还是可以相互组合使用的，你需要做的就是发挥你的想象力，完成一些奇特的效果。  