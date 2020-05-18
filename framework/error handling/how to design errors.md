# 如何制定错误

错误处理是一个程序健壮的基本保障，`go`是以错误码的方式提供错误，因此学习如何制定错误更为重要。

`go`的标准库提供了很好的范式，总结后就是：立体化，扁平化和追根溯源。

在继续下面的内容前需要先了解错误类型，`go`中的`error`是一个接口，而`go`的接口是非侵入式，高度灵活的类型。接口是分为静态类型和动态类型的，对于一个接口的实例值来说，静态类型是不变的，但是动态类型是根据绑定的实例值确定的，因此可以借助这个特性来创建立体化的错误体系。

比如当发起一个网络连接的操作时发生了一个网络错误，网络错误可能有`n`个孩子节点（或者说有那个类型实现了网络错误的接口），对于每个孩子节点又有对应的孙子节点，因此最开始拿到错误的时候，可以理解为拿到了一个树的根节点，不断的对接口进行断言，不断的将错误值在树上进行下沉，就会得到更精确的结果。

对于立体化的错误类型，最终会定位到一个叶子节点上，每个叶子节点对应了一种具体类型的错误，根据类型已经无法满足进一步获取详细的错误信息，此时需要一种扁平化的错误体系，通常来说是提供一些包级别的错误变量，通过等值的对比来获取更详细的错误信息。
