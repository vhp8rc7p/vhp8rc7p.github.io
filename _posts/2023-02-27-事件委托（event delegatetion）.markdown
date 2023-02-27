# 什么是事件委托（event delegation）？
***
事件捕获和事件冒泡使我们能够实现最强大的事件处理模式之一事件委托。

如果有很多元素以类似的方式处理，我们只需要在他们的共同的祖先上设置事件监听器，而不是在每一个元素都设置一个。

假如有如下html代码，在ul节点上绑定了事件监听器：
```
<ul id="parent" onclick="alert(event.type + '!')">
    <li id="1">One</li>
    <li id="2">Two</li>
    <li id="3">Three</li>
</ul>
```

![](/vhp8rc7p.github.io/assets/2023-02-27-事件委托（event%20delegatetion）/list.png "list")


当你点击任意的&lt;li>子节点时,都会触发点击事件。

![](/vhp8rc7p.github.io/assets/2023-02-27-事件委托（event%20delegatetion）/alert.png "alert")

但这样我们怎么知道是哪个子节点被点击了呢？我们只需要在事件传播到父节点的时候，检查event对象的属性就可以了。

如下代码：
```
document.getElementById("parent").addEventListener("click", function(e) {
	
	if(e.target && e.target.nodeName == "LI") {
		console.log("List item ", e.target.id.replace("post-", ""), " was clicked!");
	}
});

```

![](/vhp8rc7p.github.io/assets/2023-02-27-事件委托（event%20delegatetion）/which.png "which")

那么这样做有什么好处呢？

首先，在我们需要向大量元素上绑定事件监听器时，元素不断地在DOM树上新增或者删除，事件委托能使改变元素的操作与绑定事件的操作解耦，为我们省去很多繁琐操作。

其二则是可以减少使用的内存，对于小型页面来说可能效果微乎其微，但对于大型应用来说效果可能很显著，有了事件委托，你就无需担心在删除节点时没有解绑事件监听器,减少了可能造成内存泄漏的风险。


reference：

1.https://stackoverflow.com/questions/1687296/what-is-dom-event-delegation

2.https://davidwalsh.name/event-delegate