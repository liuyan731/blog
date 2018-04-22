---
title: Tensorflow模型的保存与恢复加载
date: 2017-11-25 14:06:04
categories: Machine Learning
tags: [machine learning,tensorflow]

---
*Blog的Github地址：[https://github.com/liuyan731/blog](https://github.com/liuyan731/blog)*

***

近期做了一些反垃圾的工作，除了使用常用的规则匹配过滤等手段，也采用了一些机器学习方法进行分类预测。我们使用TensorFlow进行模型的训练，训练好的模型需要保存，预测阶段我们需要将模型进行加载还原使用，这就涉及TensorFlow模型的保存与恢复加载。

总结一下Tensorflow常用的模型保存方式。

## 保存checkpoint模型文件（.ckpt）
首先，TensorFlow提供了一个非常方便的api，```tf.train.Saver()```来保存和还原一个机器学习模型。

#### 模型保存
使用```tf.train.Saver()```来保存模型文件非常方便，下面是一个简单的例子：
```python
import tensorflow as tf
import os

def save_model_ckpt(ckpt_file_path):
    x = tf.placeholder(tf.int32, name='x')
    y = tf.placeholder(tf.int32, name='y')
    b = tf.Variable(1, name='b')
    xy = tf.multiply(x, y)
    op = tf.add(xy, b, name='op_to_store')

    sess = tf.Session()
    sess.run(tf.global_variables_initializer())

    path = os.path.dirname(os.path.abspath(ckpt_file_path))
    if os.path.isdir(path) is False:
        os.makedirs(path)

    tf.train.Saver().save(sess, ckpt_file_path)
    
    # test
    feed_dict = {x: 2, y: 3}
    print(sess.run(op, feed_dict))
```

程序生成并保存四个文件（在版本0.11之前只会生成三个文件：checkpoint, model.ckpt, model.ckpt.meta）

- checkpoint 文本文件，记录了模型文件的路径信息列表
- model.ckpt.data-00000-of-00001 网络权重信息
- model.ckpt.index .data和.index这两个文件是二进制文件，保存了模型中的变量参数（权重）信息
- model.ckpt.meta 二进制文件，保存了模型的计算图结构信息（模型的网络结构）protobuf

以上是```tf.train.Saver().save()```的基本用法，```save()```方法还有很多可配置的参数：

```python
tf.train.Saver().save(sess, ckpt_file_path, global_step=1000)
```
加上global_step参数代表在每1000次迭代后保存模型，会在模型文件后加上"-1000"，model.ckpt-1000.index, model.ckpt-1000.meta, model.ckpt.data-1000-00000-of-00001

每1000次迭代保存一次模型，但是模型的结构信息文件不会变，就只用1000次迭代时保存一下，不用相应的每1000次保存一次，所以当我们不需要保存meta文件时，可以加上```write_meta_graph=False```参数，如下：
```python
tf.train.Saver().save(sess, ckpt_file_path, global_step=1000, write_meta_graph=False)
```

如果想每两小时保存一次模型，并且只保存最新的4个模型，可以加上使用```max_to_keep```（默认值为5，如果想每训练一个epoch就保存一次，可以将其设置为None或0，但是没啥用不推荐）, ```keep_checkpoint_every_n_hours```参数，如下：
```python
tf.train.Saver().save(sess, ckpt_file_path, max_to_keep=4, keep_checkpoint_every_n_hours=2)
```

同时在```tf.train.Saver()```类中，如果我们不指定任何信息，则会保存所有的参数信息，我们也可以指定部分想要保存的内容，例如只保存x, y参数（可传入参数list或dict）：
```python
tf.train.Saver([x, y]).save(sess, ckpt_file_path)
```

ps. 在模型训练过程中需要在保存后拿到的变量或参数名属性name不能丢，不然模型还原后不能通过```get_tensor_by_name()```获取。

#### 模型加载还原
针对上面的模型保存例子，还原模型的过程如下：
```python
import tensorflow as tf

def restore_model_ckpt(ckpt_file_path):
    sess = tf.Session()
    saver = tf.train.import_meta_graph('./ckpt/model.ckpt.meta')  # 加载模型结构
    saver.restore(sess, tf.train.latest_checkpoint('./ckpt'))  # 只需要指定目录就可以恢复所有变量信息

    # 直接获取保存的变量
    print(sess.run('b:0'))

    # 获取placeholder变量
    input_x = sess.graph.get_tensor_by_name('x:0')
    input_y = sess.graph.get_tensor_by_name('y:0')
    # 获取需要进行计算的operator
    op = sess.graph.get_tensor_by_name('op_to_store:0')

    # 加入新的操作
    add_on_op = tf.multiply(op, 2)

    ret = sess.run(add_on_op, {input_x: 5, input_y: 5})
    print(ret)
```
首先还原模型结构，然后还原变量（参数）信息，最后我们就可以获得已训练的模型中的各种信息了（保存的变量、placeholder变量、operator等），同时可以对获取的变量添加各种新的操作（见以上代码注释）。

并且，我们也可以加载部分模型，在此基础上加入其它操作，具体可以参考官方文档和demo。

> 针对ckpt模型文件的保存与还原，stackoverflow上有一个[回答](https://stackoverflow.com/questions/33759623/tensorflow-how-to-save-restore-a-model)解释比较清晰，可以参考。

> 同时cv-tricks.com上面的TensorFlow模型保存与恢复的[教程](http://cv-tricks.com/tensorflow-tutorial/save-restore-tensorflow-models-quick-complete-tutorial/)也非常好，可以参考。

> [《tensorflow 1.0 学习：模型的保存与恢复(Saver)》](https://www.cnblogs.com/denny402/p/6940134.html)有一些Saver使用技巧。

## 保存单个模型文件（.pb）
我自己运行过Tensorflow的inception-v3的demo，发现运行结束后会生成一个.pb的模型文件，这个文件是作为后续预测或迁移学习使用的，就一个文件，非常炫酷，也十分方便。

这个过程的主要思路是graph_def文件中没有包含网络中的Variable值（通常情况存储了权重），但是却包含了constant值，所以如果我们能把Variable转换为constant（使用```graph_util.convert_variables_to_constants()```函数），即可达到使用一个文件同时存储网络架构与权重的目标。

ps：这里.pb是模型文件的后缀名，当然我们也可以用其它的后缀（使用.pb与google保持一致 ╮(╯▽╰)╭）

#### 模型保存
同样根据上面的例子，一个简单的demo：
```python
import tensorflow as tf
import os
from tensorflow.python.framework import graph_util

def save_mode_pb(pb_file_path):
    x = tf.placeholder(tf.int32, name='x')
    y = tf.placeholder(tf.int32, name='y')
    b = tf.Variable(1, name='b')
    xy = tf.multiply(x, y)
    # 这里的输出需要加上name属性
    op = tf.add(xy, b, name='op_to_store')

    sess = tf.Session()
    sess.run(tf.global_variables_initializer())

    path = os.path.dirname(os.path.abspath(pb_file_path))
    if os.path.isdir(path) is False:
        os.makedirs(path)

    # convert_variables_to_constants 需要指定output_node_names，list()，可以多个
    constant_graph = graph_util.convert_variables_to_constants(sess, sess.graph_def, ['op_to_store'])
    with tf.gfile.FastGFile(pb_file_path, mode='wb') as f:
        f.write(constant_graph.SerializeToString())

    # test
    feed_dict = {x: 2, y: 3}
    print(sess.run(op, feed_dict))
```
程序生成并保存一个文件
- model.pb 二进制文件，同时保存了模型网络结构和参数（权重）信息

#### 模型加载还原
针对上面的模型保存例子，还原模型的过程如下：
```python
import tensorflow as tf
from tensorflow.python.platform import gfile

def restore_mode_pb(pb_file_path):
    sess = tf.Session()
    with gfile.FastGFile(pb_file_path, 'rb') as f:
        graph_def = tf.GraphDef()
        graph_def.ParseFromString(f.read())
        sess.graph.as_default()
        tf.import_graph_def(graph_def, name='')

    print(sess.run('b:0'))

    input_x = sess.graph.get_tensor_by_name('x:0')
    input_y = sess.graph.get_tensor_by_name('y:0')

    op = sess.graph.get_tensor_by_name('op_to_store:0')

    ret = sess.run(op, {input_x: 5, input_y: 5})
    print(ret)
```
模型的还原过程与checkpoint差不多一样。

> CSDN[《将TensorFlow的网络导出为单个文件》](http://blog.csdn.net/encodets/article/details/54428456)上介绍了TensorFlow保存单个模型文件的方式，大同小异，可以看看。

## 思考
模型的保存与加载只是TensorFlow中最基础的部分之一，虽然简单但是也必不可少，在实际运用中还需要注意模型何时保存，哪些变量需要保存，如何设计加载实现迁移学习等等问题。

同时TensorFlow的函数和类都在一直变化更新，以后也有可能出现更丰富的模型保存和还原的方法。

选择保存为checkpoint或单个pb文件视业务情况而定，没有特别大的差别。checkpoint保存感觉会更加灵活一些，pb文件更适合线上部署吧（个人看法）。

**以上完整代码：[github](https://github.com/liuyan731/tf_demo.git)**

***
*2017/11/25 done*

*此文章也同步至[个人Github博客](https://liuyan731.github.io/)*