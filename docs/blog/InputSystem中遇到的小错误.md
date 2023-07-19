最近研究 [*Unity InputSystem*](https://docs.unity.cn/cn/2022.1/Manual/com.unity.inputsystem.html) 时犯了一个超级低级的错误,记录一下以后别犯了(真的很低级,但第一次遇到这样的,所以记录一下).<br>

## 脚本关系

脚本关系大致是这样的

1. *`PlayerInput.cs`* 继承 `ScriptableObject`用于接收InputSystem的事件并调用其中包含的几个事件委托.
2. *`PlayerController.cs`* 继承一个 `SingletonMonoBehaviour`类(单例MonoBehaviour),其中的一些事件会被赋值到`PlayerInput`中的委托中,由`PlayerInput`调用.

这两个脚本十分的特殊阿,一个`ScriptableObject`也就是会对其中的变量进行持久化储存.
一个 `单例模式脚本` 这个就不多解释了.

## 问题

一个十分神奇的问题,打破了我一直认为的 **程序能跑一遍,就可以跑第二遍** 的思想.<br>
其实一般在*Unity*中没毛病,但一旦牵扯到持久化储存就不一样了,就像我这一次一样.
程序编译后运行第一次没问题,第二次就开始报错了.

!> MissingReferenceException: The object of type `'PlayerController'` has been destroyed but you are still trying to acce

大意就是在`PlayerInput`调用`PlayerController`委托时`PlayerController`已经被删除了.

## 分析

以下是部分代码

```csharp
    //PlayerController(节选的部分基本方法)
    private void OnEnable()
    {
        playerInput.onMove += Move;
        playerInput.onStopMove += StopMove;
    }
    private void OnDisable()
    {
        //空的
    }


    //PlayerInput(节选)
    public event UnityAction<Vector2> onMove = delegate { };
    public event UnityAction onStopMove = delegate { };

    public void OnMove(InputAction.CallbackContext context)
    {
        if(context.phase == InputActionPhase.Performed)
        {
            onMove.Invoke(context.ReadValue<Vector2>());
        }
        if (context.phase == InputActionPhase.Canceled)
        {
            onStopMove.Invoke();
        }
    }
```

问题十分明显了,`OnDisable`中是空的,也就意味着`PlayerInput`中的委托事件在`OnEnable`增加,却在之后没被减少.

意思就是:<br>
在第一次运行时`PlayerInput`中的委托,如`onMove(Action)` 会有一个值`OnMove(1)`.<br>
原本也没什么,可时`PlayerInput`是持久化的,也就是说在下次编译时`onMove(Action)`中的值不会被还原为null.<br>
所以在第二次运行时`onMove(Action)`中就会有`OnMove(1)`和`OnMove(2)`两个值.当调用时程序就会发现这个`OnMove(1)`是上一次运行时`PlayerController`的值,在这次运行时上次运行时的`PlayerController`已经被删了.所以自然会报`MissingReferenceException`的错误.

## 解决

知道了问题,剩下就好办了.<br>
在`OnDisable`中将`onMove(Action)`中的值减去就行了.<br>
如下:

```csharp
    //PlayerController(节选的部分基本方法)
    private void OnEnable()
    {
        playerInput.onMove += Move;
        playerInput.onStopMove += StopMove;
    }
    private void OnDisable()
    {
        playerInput.onMove -= Move;
        playerInput.onStopMove -= StopMove;
    }
```

## 总结

总结一下,在使用委托时有加就要有减.
