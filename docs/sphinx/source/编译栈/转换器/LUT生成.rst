========================================================================
LUT生成
========================================================================

LUT生成的基本原理
########################################

Look-up Table，一般用来在硬件上高效处理非线性函数，基本原理是：获取一定范围内的有限个数的输入，将这些输入经过一个非线性函数的输出计算出来，这些计算结果就被存成LUT，使用时可通过输入对LUT进行索引。由于LUT需要保证每个输入对应一个输出，所以输入需要是一定范围内的离散值，确保LUT本身的存储不会代价过大。

LUT的地址有效位宽和输出数据类型决定了LUT的内存占用量，TianjicX支持的LUT输入输出数据类型为INT32/INT8，当输入数据位宽超过地址有效位宽时，需要先选择输入数据的一部分作为地址，对LUT进行索引。这部分操作在量化中实现，我们可以认为在LUT生成部分，输入数据位宽和地址有效位宽是一样的。

在不同的地址有效位宽和输出数据类型下LUT占用的内存如下表所示： 

========================= ======== ====== ======= ========
输出数据类型/地址有效位宽   4bit     8bit   12bit   16bit 
========================= ======== ====== ======= ========
INT8                       16B      256B   4KB     64KB  
INT32                      64B      1KB    16KB    256KB
========================= ======== ====== ======= ========

LUT的生成和量化密切相关，我们以下面的例子进行说明：假设输入数据类型为INT8，LUT地址有效位宽是8bit，输出数据类型为INT8，LUT想要存储的非线性函数为Sigmoid，即在浮点数形式下：

.. math::

   y = {\rm sigmoid}(x)

输入数据满足如下的浮点数\ :math:`x`\ 到整数\ :math:`X`\ 的映射：

.. math::

   x = S_X X

输出数据满足如下的浮点数\ :math:`y`\ 到整数\ :math:`Y`\ 的映射：

.. math::

   y = S_Y Y

则有

.. math::

   S_Y Y = {\rm sigmoid}(S_X X)

即

.. math::

   Y = \frac{1}{S_Y}{\rm sigmoid}(S_X X)

在一个量化模型中，非线性函数的实际输入是\ :math:`X`\ ，希望得到的输出为\ :math:`Y`\ ，所以实际上LUT对应的非线性函数为\ :math:`f(X) = 1 / S_Y {\rm sigmoid}(S_X X)`\ 。即我们生成LUT的流程如下：

1. 对模型进行参数统计，获得\ :math:`S_X`\ 和\ :math:`S_Y`\ 的值(详见量化文档)
2. 我们将-128到127之间每个整数\ :math:`X`\ 作为输入，计算\ :math:`1 / S_Y {\rm sigmoid}(S_X X)`
3. 以无符号数的值为索引将计算得到的结果存入列表，即列表中的元素为\ ``[f(0), f(1), ..., f(127), f(-128), f(-127), ..., f(-1)]``

LUT框架的实现及使用
########################################

1. 创建一个\ ``LUT``\ 对象，创建对象时可配置的参数包括：

-  ``function``:
   非线性函数，可以是一个自定义的python函数或一个\ ``torch.nn.Module``\ 类，例如

.. code:: python

   import math
   import torch

   def sigmoid(x):
       return 1 / (1 + math.exp(-x))

   lut = LUT(function=sigmoid)
   lut = LUT(function=torch.nn.Sigmoid)

-  ``input_width``: 输入数据位宽，默认为8bit
-  ``output_width``: 输出数据位宽，默认为8bit
-  ``fp_input_absmax``:
   原浮点数输入数据的绝对值最大值，即被映射到整数最大值的浮点数值，默认值为1
-  ``fp_output_absmax``:
   原浮点数输出数据的绝对值最大值，即被映射到整数最大值的浮点数值，默认值为1

2. 调用\ ``generate()``\ 方法生成LUT，也可以直接获得某个整数对应的LUT值。例如

.. code:: python

   import math

   def sigmoid(x):
       return 1 / (1 + math.exp(-x))

   lut = LUT(function=sigmoid)
   lut_list = lut.generate()
   y = lut(1)  # 可直接获得整数1对应的LUT值

生成的LUT为：输入数据类型为INT8，即-128到127的整数\ :math:`X`\ ，\ :math:`S_Y = 1 / 127`\ ，\ :math:`S_X = 1 / 127`\ ，输出数据为\ :math:`f(X) = 127 {\rm sigmoid}(1/127 X)`\ 的值，存储顺序为\ ``[f(0), f(1), ..., f(127), f(-128), f(-127), ..., f(-1)]``\ 。
