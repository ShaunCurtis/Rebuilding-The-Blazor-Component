# Rebuilding the Blazor Base Component

*ComponentBase* is the out-of-the-box base component for Blazor.  This article takes a personal look at it's deficiencies, and describes how to build an alternative.

The Blazor UI is built by a renderer which holds a tree structure of classes implementing *IComponent* - commonly called the render tree.  The only requirement for a component to be added to the render tree is it implements *IComponent*.  The  renderer communicates with components in the tree through *IComponent* interface methods.  *ComponentBase* is the out-of-the-box *IComponent* implementation shipped with Blazor.  You're components don't have to use *ComponentBase*, they need to implement *IComponent*.

## Why Change

This may be a personal thing, but I have a few issues with *ComponentBase*:

1. The nomenclature is misleading: it leads to a lot of miconceptions.  When I started using Blazor for real, I had to look at *ComponentBase* code to get my head around what was really going on. Some examples:  
   * *OnInitialized* sounds like it gets run when the class is initialised, maybe replacing new: misconception, it has nothing to do with class initialization.
   * *OnParametersSet*, what does it do other than set parameters. 
   * *StateHasChanged* doesn't convey that it's real purpose is to render the component.
2. The sequencing of first render and subsequent render isn't logical.  Using *OnInitializedAsync* for the first rendering sort of works, but *OnParametersSetAsync* for the first and any subsequent rendering doesn't.  It just isn't intiutive.
3. I want my applications to implement "Async all the Way", no non-async methods for bottling out. 

## Component

The full class code is [here](https://github.com/ShaunCurtis/CEC.Blazor/blob/Experimental/CEC.Blazor/Components/Base/Component.cs) on GitHub in CEC.Blazor/Components/Base on the Experimental Branch.

The class is abstract so can't be used directly.  In addition to *IComponent*, it implements *IHandleEvent* and *IHandleAfterRender* for Render Event notification by the Renderer. 

```c#
public abstract class Component : IComponent, IHandleEvent, IHandleAfterRender
```

### Properties

The full property list is shown below.  These are almost the same as *ComponentBase*.
```c#
/// Property to check if the component is loading -  set internally
/// I use this in UI components with significant lag to show a "working" rotator when Loading is in process
public bool Loading { get; protected set; } = true;

/// Holds the render fragment for the component - called by the Renderer whenever a queued render event is executed.
private readonly RenderFragment _renderFragment;
/// Holds the RenderHandle for the Renderer, passed to the component in when the Render calls Attach.
private RenderHandle _renderHandle;
/// Boolean set during Attach or Reset
private bool _firstRender = true;
/// Boolean set once the component has rendered for the first time
private bool _hasNeverRendered = true;
// Boolean set when a render event is queued but not yet executed.
private bool _hasPendingQueuedRender;
// Boolean used for internal control of After Render process
private bool _hasCalledOnAfterRender;
```

### Methods

The public methods we want to implement are:

1. *OnRenderAsync(bool firstRender)* - this replaces *OnInitializedAsync* and *OnParametersSetAsync*.  *firstRender* is only set true when the component is rendered for the first time (as part of the *Attach* process).  A virtual method, normally overridden.
2. *Render()* - this replaces *StateHasChanged*.  It kicks off a render of the component.  It's not declared virtual so can't be overridden.
3. *OnAfterRenderAsync(bool firstRender)* - the same as *ComponentBase*.  It's called after the component has rendered.   A virtual method, normally overridden.
4. *ResetAsync()* - a method to manually re-render the component for the first time with the current parameters.


The component initialization method is straignt for *ComponentBase*.  It builds the component render fragment.  To be clear, if you are a little confused, we build the render fragment here, using the builder to load code into the render fragment.  We don't actually execute it, a *RenderFragment* is a block of code run by the Renderer to render the component.
```c#
public Component()
{
    _renderFragment = builder =>
    {
        _hasPendingQueuedRender = false;
        _hasNeverRendered = false;
        BuildRenderTree(builder);
    };
}
```
*SetParametersAsync* is defined by *IComponent*.  It's called by the Renderer whenever any of the component's registered parameters or cascaded parameters are changed externally.

```c#
public async Task SetParametersAsync(ParameterView parameters)
{
    /// Applies the supplied parameters to the component's properties.
    parameters.SetParameterProperties(this);
    /// Kick off the render process
    await this._StartRenderAsync();
}
```

*_StartRenderAsync* is the internal method that runs the render process.  It's code is separated from *SetParametersAsync* so it's also called by *ResetAsync*. 

```c#
private async Task _StartRenderAsync()
{
    /// sets Loading
    this.Loading = true;
    /// call the public (normally overridden) OnRenderAsync
    await this.OnRenderAsync(this._firstRender);
    /// first render complete
    this._firstRender = false;
    /// Queue a Render
    await InvokeAsync(Render);
}
```

*OnRenderAsync* is a prototype method - it's normally overridden.

```c#
protected virtual Task OnRenderAsync(bool firstRender) => Task.CompletedTask;
```
*Render* contains the same code as *StateHasChanged* in *ComponentBose*.  *Render* does what it says on the tin, places the component's render fragment on the Renderer's render queue.

```c#
protected void Render()
{
    /// check if we already have a render queued - if so stop, any changes will be captured by the queued render
    if (_hasPendingQueuedRender) return;
    /// check if we've never rendered or should render (ShouldRender unless overridden returns true)
    if (_hasNeverRendered || ShouldRender())
    {
        _hasPendingQueuedRender = true;
        /// Queues the component render fragment on the Renderer's render queue.
        try
        {
            _renderHandle.Render(_renderFragment);
        }
        catch
        {
            _hasPendingQueuedRender = false;
            throw;
        }
    }
}

/// can be overridden for external control of rendering
protected virtual bool ShouldRender() => true;
```

*ResetAsync* resets the component to new and runs a complete re-render.

```c#
public virtual async Task ResetAsync()
{
    this._firstRender = true;
    await this._StartRenderAsync();
}
```

*BuildRenderTree* does nothing.  It's overridden either directly in derived classes with code or the Razor compiler builds it from the Razor markup during the build process.

```c#
protected virtual void BuildRenderTree(RenderTreeBuilder builder) {}
```

*Attach* implements *IComponent.Attach*. It captures the *RenderHandle* for the renderer and sets *_firstRender*.

```c#
/// IComponent Attach implementation
void IComponent.Attach(RenderHandle renderHandle)
{
    if (_renderHandle.IsInitialized)
    {
        throw new InvalidOperationException($"The render handle is already set. Cannot initialize a {nameof(ComponentBase)} more than once.");
    }
    _firstRender = true;
    _renderHandle = renderHandle;
}
```
The two *InvokeAsync* methods provide thread safe methods for running functions or actions on the renderer's synchronisation thread using the RenderHandle's Dispatcher.
```c#
/// Executes the supplied work item on the associated renderer's synchronization context.
protected Task InvokeAsync(Action workItem) => _renderHandle.Dispatcher.InvokeAsync(workItem);

/// Executes the supplied work item on the associated renderer's synchronization context.
protected Task InvokeAsync(Func<Task> workItem) => _renderHandle.Dispatcher.InvokeAsync(workItem);
```

In general, call *Render* like this to make sure it's run on the correct thread:

```c#
InvokeAsync(Render);
// or
await InvokeAsync(Render);
// rather than 
Render();
```

The final methods are straight from *ComponentBase* and implement the various interfaces. 

```c#
/// Internal method to track the render event
Task IHandleEvent.HandleEventAsync(EventCallbackWorkItem callback, object arg)
{
    var task = callback.InvokeAsync(arg);
    var shouldAwaitTask = task.Status != TaskStatus.RanToCompletion && task.Status != TaskStatus.Canceled;
    InvokeAsync(Render);
    return shouldAwaitTask ? CallRenderOnAsyncCompletion(task) : Task.CompletedTask;
}

/// Internal Method triggered after the component has rendered calling OnAfterRenderAsync
Task IHandleAfterRender.OnAfterRenderAsync()
{
    var firstRender = !_hasCalledOnAfterRender;
    _hasCalledOnAfterRender |= true;
    return OnAfterRenderAsync(firstRender);
}

/// Internal method to handle render completion
private async Task CallRenderOnAsyncCompletion(Task task)
{
    try
    {
        await task;
    }
    catch // avoiding exception filters for AOT runtime support
    {
        // Ignore exceptions from task cancellations, but don't bother issuing a state change.
        if (task.IsCanceled) return;
        else throw;
    }
    await InvokeAsync(Render);
}
```
# Using the Component 

You need to declare inheritance in the Razor file - by default Razor uses *ComponentBase*.

```c#
@inherits Component
```

The basic pattern you need to follow for *OnRenderAsync* and *OnAfterRenderAsync* is shown below.

```c#
// Depending on whether your running any async functions declare as async
protected async override Task OnRenderAsync(bool firstRender)
{
    // Do anything you need to do before you call down the inheritance tree
    if (firstRender) 
    {
        // Do anything you need to do only on the first render
    }
    else
    {
        // Do anything you need to do on any subsequent render (but not the first)
    }
    // Do anything you need to do on any render
    
    await base.OnRenderAsync(firstRender);
    
    // Do anything you need to do after you've called down the inheritance tree
    if (firstRender) 
    {
        // Do anything you need to do only on the first render
    }
    else
    {
        // Do anything you need to do on any subsequent render (but not the first)
    }
    // Do anything you need to do on any render

    // Return completed task if you haven't declared the method as async
    //return Task.CompletedTask;
}
```

Be careful about calling *Render*.  It's very easy to "over" render.  Check in debug mode with a breakpoint on Render and see how often it's called.

```c#
InvokeAsync(Render);
// or
await InvokeAsync(Render);
```

## Code Repository and Example Sites

You can see the component in action in the standard Blazor Weather Application at these sites:

[WASM Weather Version](https://cec-blazor-wasm.azurewebsites.net/)

[Blazor Server Version](https://cec-blazor-server.azurewebsites.net/)

There's a GitHub repository [here](https://github.com/ShaunCurtis/CEC.Blazor/tree/Experimental).  Note you need to be on the Experimental Branch.  *Component* is in CEC.Blazor/Components/Base.

## Wrap Up

My decision to dump *ComponentBase* wasn't taken lightly.  Don't get me wrong, I love Blazor, but I don't think we've got SPA development right in this first iteration.  I've been working with Blazor for 9 months now with live Blazor commercial applications running on Azure.  As a "grey" developer I have more time than most, so during my COVID restricted travels, I've started thinking outside-the-box on my development framework.  I think SPA's are still glued in the Web paradigm, more articles to come on some radical thinking on developing with Blazor.

Hopefully this article shows you there's life outside *ComponentBase*.  If you're relatively new to Blazor this article should give you a good eye-opener into components.  If you agree with me, use my *Component* code in any way, sshape or form - no permission required.  If you disagree, no problem, I understand your position - diversity is the spice of life! 
