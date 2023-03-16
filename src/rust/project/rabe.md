## Rabe封装

由于学习需要用到基于属性的加密算法（ABE），但是需要在Go或Java中调用，github搜了一圈想要的算法要不就是只有Go的或者只有Java的，并且很多都是学生自己做的，API难用，而且只有几种算法里的一种。最后找到一个[Rust实现的项目](https://github.com/Fraunhofer-AISEC/rabe)，于是就想到将这个Rust库导出为C库来用。

