# 三、.2 属性

我们首先描述 DSN 体系所必须的两个属性，然后描述 Filecoin DSN 体系需要的另外的一些其他属性。

#### 3.2.1 数据完整性

该属性要求不存在一个敌对节点 A 能够说服客户在执行 Get 命令后去接受那些被更改了的或被伪造的数据。

定义 2.2

一个 DSN 方案(Π)提供了数据完整性：如果在使用密钥 k 的情况下，对数据 d 任意成功的 Put 操作，那就不存在敌手 A 能使得客户接受 d'，因为使用密钥 k 执行 Get 命令后得到的 d 和 d'不相同。

#### 3.2.2 可恢复性

该属性满足了以下要求：给定我们Π中的容错假设，如果有些数据已经成功存储在Π中并且存储提供者继续遵循协议，那么客户最终能够检索到数据。

定义 2.3

一个 DSN 体系(Π)提供可恢复性：如果利用密钥成功执行 Put 命令，将数据得以保存，那么也会存在用户使用相同的密钥成功执行 Get 命令，得到想要的副本数据。（这个定义并不保证每次 Get 操作都能成功，如果每次 Get 操作最终都能返回数据，那这个方案是公平的）。