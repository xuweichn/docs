# 推理模型

 `Linux` `Ascend` `GPU` `CPU` `推理应用` `初级` `中级` `高级`

<!-- TOC -->

- [多平台推理](#多平台推理)
    - [概述](#概述)
    - [Ascend 910 AI处理器上推理](#ascend-910-ai处理器上推理)
        - [使用checkpoint格式文件推理](#使用checkpoint格式文件推理)
    - [Ascend 310 AI处理器上推理](#ascend-310-ai处理器上推理)
        - [使用ONNX与AIR格式文件推理](#使用onnx与air格式文件推理)
    - [GPU上推理](#gpu上推理)
        - [使用checkpoint格式文件推理](#使用checkpoint格式文件推理-1)
        - [使用ONNX格式文件推理](#使用onnx格式文件推理)
    - [CPU上推理](#cpu上推理)
        - [使用checkpoint格式文件推理](#使用checkpoint格式文件推理-2)
        - [使用ONNX格式文件推理](#使用onnx格式文件推理-1)
    - [端侧推理](#端侧推理)

<!-- /TOC -->

<a href="https://gitee.com/mindspore/docs/blob/r1.0/tutorials/inference/source_zh_cn/multi_platform_inference.md" target="_blank"><img src="../_static/logo_source.png"></a>

## 概述

基于MindSpore训练后的模型，支持在不同的硬件平台上执行推理。本文介绍各平台上的推理流程。

按照原理不同，推理可以有两种方式：
- 直接使用checkpiont文件进行推理，即在MindSpore训练环境下，使用推理接口加载数据及checkpoint文件进行推理。
- 将checkpiont文件转化为通用的模型格式，如ONNX、AIR格式模型文件进行推理，推理环境不需要依赖MindSpore。这样的好处是可以跨硬件平台，只要支持ONNX/AIR推理的硬件平台即可进行推理。譬如在Ascend 910 AI处理器上训练的模型，可以在GPU/CPU上进行推理。

MindSpore支持的推理场景，按照硬件平台维度可以分为下面几种：

硬件平台 | 模型文件格式 | 说明
--|--|--
Ascend 910 AI处理器 | checkpoint格式 | 与MindSpore训练环境依赖一致
Ascend 310 AI处理器 | ONNX、AIR格式 | 搭载了ACL框架，支持OM格式模型，需要使用工具转化模型为OM格式模型。
GPU | checkpoint格式 | 与MindSpore训练环境依赖一致。
GPU | ONNX格式 | 支持ONNX推理的runtime/SDK，如TensorRT。
CPU | checkpoint格式 | 与MindSpore训练环境依赖一致。
CPU | ONNX格式 | 支持ONNX推理的runtime/SDK，如TensorRT。

> ONNX，全称Open Neural Network Exchange，是一种针对机器学习所设计的开放式的文件格式，用于存储训练好的模型。它使得不同的人工智能框架（如PyTorch, MXNet）可以采用相同格式存储模型数据并交互。详细了解，请参见ONNX官网<https://onnx.ai/>。

> AIR，全称Ascend Intermediate Representation，类似ONNX，是华为定义的针对机器学习所设计的开放式的文件格式，能更好地适配Ascend AI处理器。

> ACL，全称Ascend Computer Language，提供Device管理、Context管理、Stream管理、内存管理、模型加载与执行、算子加载与执行、媒体数据处理等C++ API库，供用户开发深度神经网络应用。它匹配Ascend AI处理器，使能硬件的运行管理、资源管理能力。

> OM，全称Offline Model，华为Ascend AI处理器支持的离线模型，实现算子调度的优化，权值数据重排、压缩，内存使用优化等可以脱离设备完成的预处理功能。

> TensorRT，NVIDIA 推出的高性能深度学习推理的SDK，包括深度推理优化器和runtime，提高深度学习模型在边缘设备上的推断速度。详细请参见<https://developer.nvidia.com/tensorrt>。

## Ascend 910 AI处理器上推理

### 使用checkpoint格式文件推理

1. 使用`model.eval`接口来进行模型验证。                                  

   1.1 模型已保存在本地  

   首先构建模型，然后使用`mindspore.train.serialization`模块的`load_checkpoint`和`load_param_into_net`从本地加载模型与参数，传入验证数据集后即可进行模型推理，验证数据集的处理方式与训练数据集相同。
    ```python
    network = LeNet5(cfg.num_classes)
    net_loss = nn.SoftmaxCrossEntropyWithLogits(sparse=True, reduction="mean")
    net_opt = nn.Momentum(network.trainable_params(), cfg.lr, cfg.momentum)
    model = Model(network, net_loss, net_opt, metrics={"Accuracy": Accuracy()})

    print("============== Starting Testing ==============")
    param_dict = load_checkpoint(args.ckpt_path)
    load_param_into_net(network, param_dict)
    dataset = create_dataset(os.path.join(args.data_path, "test"),
                             cfg.batch_size,
                             1)
    acc = model.eval(dataset, dataset_sink_mode=args.dataset_sink_mode)
    print("============== {} ==============".format(acc))
    ```
    其中，  
    `model.eval`为模型验证接口，对应接口说明：<https://www.mindspore.cn/doc/api_python/zh-CN/r1.0/mindspore/mindspore.html#mindspore.Model.eval>。
    > 推理样例代码：<https://gitee.com/mindspore/mindspore/blob/r1.0/model_zoo/official/cv/lenet/eval.py>。

   1.2 使用MindSpore Hub从华为云加载模型
    
   首先构建模型，然后使用`mindspore_hub.load`从云端加载模型参数，传入验证数据集后即可进行推理，验证数据集的处理方式与训练数据集相同。
    ```python
    model_uid = "mindspore/ascend/0.7/googlenet_v1_cifar10"  # using GoogleNet as an example.
    network = mindspore_hub.load(model_uid, num_classes=10)
    net_loss = nn.SoftmaxCrossEntropyWithLogits(sparse=True, reduction="mean")
    net_opt = nn.Momentum(network.trainable_params(), cfg.lr, cfg.momentum)
    model = Model(network, net_loss, net_opt, metrics={"Accuracy": Accuracy()})

    print("============== Starting Testing ==============")
    dataset = create_dataset(os.path.join(args.data_path, "test"),
                             cfg.batch_size,
                             1)
    acc = model.eval(dataset, dataset_sink_mode=args.dataset_sink_mode)
    print("============== {} ==============".format(acc))
    ``` 
    其中，  
    `mindspore_hub.load`为加载模型参数接口，对应接口说明：<https://www.mindspore.cn/doc/api_python/zh-CN/r1.0/mindspore_hub/mindspore_hub.html#module-mindspore_hub>。

2. 使用`model.predict`接口来进行推理操作。
   ```python
   model.predict(input_data)
   ```
   其中，  
   `model.predict`为推理接口，对应接口说明：<https://www.mindspore.cn/doc/api_python/zh-CN/r1.0/mindspore/mindspore.html#mindspore.Model.predict>。

## Ascend 310 AI处理器上推理

### 使用ONNX与AIR格式文件推理

Ascend 310 AI处理器上搭载了ACL框架，他支持OM格式，而OM格式需要从ONNX或者AIR模型进行转换。所以在Ascend 310 AI处理器上推理，需要下述两个步骤：

1. 在训练平台上生成ONNX或AIR格式模型，具体步骤请参考[导出AIR格式文件](https://www.mindspore.cn/tutorial/training/zh-CN/r1.0/advanced_use/save_load_model_hybrid_parallel.html#air)和[导出ONNX格式文件](https://www.mindspore.cn/tutorial/training/zh-CN/r1.0/advanced_use/save_load_model_hybrid_parallel.html#onnx)。

2. 将ONNX/AIR格式模型文件，转化为OM格式模型，并进行推理。
   - 云上（ModelArt环境），请参考[Ascend910训练和Ascend310推理的样例](https://support.huaweicloud.com/bestpractice-modelarts/modelarts_10_0026.html)完成推理操作。
   - 本地的裸机环境（对比云上环境，即本地有Ascend 310 AI 处理器），请参考Ascend 310 AI处理器配套软件包的说明文档。

## GPU上推理

### 使用checkpoint格式文件推理

与在Ascend 910 AI处理器上推理一样。

### 使用ONNX格式文件推理

1. 在训练平台上生成ONNX格式模型，具体步骤请参考[导出ONNX格式文件](https://www.mindspore.cn/tutorial/training/zh-CN/r1.0/advanced_use/save_load_model_hybrid_parallel.html#onnx)。

2. 在GPU上进行推理，具体可以参考推理使用runtime/SDK的文档。如在Nvidia GPU上进行推理，使用常用的TensorRT，可参考[TensorRT backend for ONNX](https://github.com/onnx/onnx-tensorrt)。

## CPU上推理

### 使用checkpoint格式文件推理
与在Ascend 910 AI处理器上推理一样。

### 使用ONNX格式文件推理
与在GPU上进行推理类似，需要以下几个步骤：

1. 在训练平台上生成ONNX格式模型，具体步骤请参考[导出ONNX格式文件](https://www.mindspore.cn/tutorial/training/zh-CN/r1.0/advanced_use/save_load_model_hybrid_parallel.html#onnx)。

2. 在CPU上进行推理，具体可以参考推理使用runtime/SDK的文档。如使用ONNX Runtime，可以参考[ONNX Runtime说明文档](https://github.com/microsoft/onnxruntime)。

## 端侧推理

端侧推理需使用MindSpore Lite推理引擎，详细操作请参考[导出MINDIR格式文件](https://www.mindspore.cn/tutorial/training/zh-CN/r1.0/advanced_use/save_load_model_hybrid_parallel.html#mindir)和[端侧推理教程](https://www.mindspore.cn/lite)。