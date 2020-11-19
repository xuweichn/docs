# Running Management

<!-- TOC -->

- [Running Management](#running-management)
    - [Overview](#overview)
    - [Execution Mode Management](#execution-mode-management)
        - [Mode Selection](#mode-selection)
        - [Mode Switching](#mode-switching)
    - [Hardware Management](#hardware-management)
    - [Distributed Management](#distributed-management)
    - [Maintenance and Test Management](#maintenance-and-test-management)
        - [Profiling Data Collection](#profiling-data-collection)
        - [Saving MindIR](#saving mindir)
        - [Print Operator Disk Flushing](#print-operator-disk-flushing)

<!-- /TOC -->

<a href="https://gitee.com/mindspore/docs/blob/master/docs/programming_guide/source_en/context.md" target="_blank"><img src="./_static/logo_source.png"></a>

## Overview

Before initializing the network, configure the context parameter to control the policy executed by the program. For example, you can select an execution mode and backend, and configure distributed parameters. Different context parameter configurations implement different functions, including execution mode management, hardware management, distributed management, and maintenance and test management.

## Execution Mode Management

MindSpore supports two running modes: PyNative and Graph.

- `PYNATIVE_MODE`: dynamic graph mode. In this mode, operators in the neural network are delivered and executed one by one, facilitating the compilation and debugging of the neural network model.

- `GRAPH_MODE`: static graph mode or graph mode. In this mode, the neural network model is compiled into an entire graph, and then the graph is delivered for execution. This mode uses graph optimization to improve the running performance and facilitates large-scale deployment and cross-platform running.

### Mode Selection

You can set and control the running mode of the program. By default, MindSpore is in PyNative mode.

A code example is as follows:

```python
from mindspore import context
context.set_context(mode=context.GRAPH_MODE)
```

### Mode Switching

You can switch between the two modes.

When MindSpore is in PyNative mode, you can switch it to the graph mode using `context.set_context(mode=context.GRAPH_MODE)`. Similarly, when MindSpore is in graph mode, you can switch it to PyNative mode using `context.set_context(mode=context.PYNATIVE_MODE)`.

A code example is as follows:

```python
import numpy as np
import mindspore.nn as nn
from mindspore import context, Tensor

context.set_context(mode=context.GRAPH_MODE, device_target="GPU")

conv = nn.Conv2d(3, 4, 3, bias_init='zeros')
input_data = Tensor(np.ones([1, 3, 5, 5]).astype(np.float32))
conv(input_data)
context.set_context(mode=context.PYNATIVE_MODE)
conv(input_data)
```

In the preceding example, the running mode is set to `GRAPH_MODE` and then switched to `PYNATIVE_MODE`.

## Hardware Management

Hardware management involves the `device_target` and `device_id` parameters.

- `device_target`: sets the target device. Ascend, GPU, and CPU are supported. Set this parameter based on the actual requirements.

- `device_id`: specifies the physical sequence number of a device, that is, the actual sequence number of the device on the corresponding host. If the target device is Ascend and the specification is N*Ascend (N > 1, for example, 8*Ascend), in non-distributed mode, you can set `device_id` to determine the device ID for program execution to avoid device usage conflicts. The value ranges from 0 to the total number of devices minus 1. The total number of devices cannot exceed 4096. The default value is 0.

> On the GPU and CPU, the `device_id` parameter setting is invalid.

A code example is as follows:

```python
from mindspore import context
context.set_context(device_target="Ascend", device_id=6)
```

## Distributed Management

The context contains the context.set_auto_parallel_context API that is used to configure parallel training parameters. This API must be called before the network is initialized.

- `parallel_mode`: parallel distributed mode. The default value is `ParallelMode.STAND_ALONE`. The options are `ParallelMode.DATA_PARALLEL` and `ParallelMode.AUTO_PARALLEL`.

- `gradients_mean`: During backward computation, the framework collects gradients of parameters in data parallel mode across multiple hosts, obtains the global gradient value, and transfers the global gradient value to the optimizer for update. The default value is `False`, which indicates that the `allreduce_sum` operation is applied. The value `True` indicates that the `allreduce_mean` operation is applied.

- `enable_parallel_optimizer`: This feature is being developed. It enables the model parallelism of an optimizer and splits the weight to each device for update and synchronization to improve performance. This parameter is valid only in data parallel mode and when the number of parameters is greater than the number of hosts. The `Lamb` and `Adam` optimizers are supported.

- `device_num`: indicates the number of available device. Its value is int type and must be in the range of 1~4096.

- `global_rank`: indicates the logical sequence number of the current device, its value is int type and must be in the range of 0~4095.

> You are advised to set `device_num` and `global_rank` to their default values. The framework calls the HCCL API to obtain the values.

A code example is as follows:

```python
from mindspore import context
from mindspore.context import ParallelMode
context.set_auto_parallel_context(parallel_mode=ParallelMode.AUTO_PARALLEL, gradients_mean=True)
```

For details about distributed parallel training, see [Distributed Training](https://www.mindspore.cn/tutorial/training/en/master/advanced_use/distributed_training_tutorials.html).

## Maintenance and Test Management

To facilitate maintenance and fault locating, the context provides a large number of maintenance and test parameter configurations, such as profiling data collection, asynchronous data dump function, and print operator disk flushing.

### Profiling Data Collection

The system can collect profiling data during training and use the profiling tool for performance analysis. Currently, the following profiling data can be collected:

- `enable_profiling`: indicates whether to enable the profiling function. If this parameter is set to True, the profiling function is enabled, and profiling options are read from enable_options. If this parameter is set to False, the profiling function is disabled and only training_trace is collected.

- `profiling_options`: profiling collection options. The values are as follows. Multiple data items can be collected. training_trace: collects step trace data, that is, software information about training tasks and AI software stacks, to analyze the performance of training tasks. It focuses on data argumentation, forward and backward computation, and gradient aggregation update. task_trace: collects task trace data, that is, hardware information of the Ascend 910 processor HWTS/AICore and analysis of task start and end information. op_trace: collects performance data of a single operator. Format: ['op_trace','task_trace','training_trace']

A code example is as follows:

```python
from mindspore import context
context.set_context(enable_profiling=True, profiling_options="training_trace")
```

### Saving MindIR

Saving the intermediate code of each compilation stage through context.set_context(save_graphs=True).

The saved intermediate code has two formats: one is a text format with a suffix of `.ir`, and the other is a graphical format with a suffix of `.dot`.

When the network is large, it is recommended to use a more efficient text format for viewing. When the network is not large, it is recommended to use a more intuitive graphical format for viewing.

A code example is as follows:

```python
from mindspore import context
context.set_context(save_graphs=True)
```

> For details about the debugging method, see [Asynchronous Dump](https://www.mindspore.cn/tutorial/training/en/master/advanced_use/custom_debugging_info.html#asynchronous-dump).

### Print Operator Disk Flushing

By default, the MindSpore self-developed print operator can output the tensor or character string information entered by users. Multiple character string inputs, multiple tensor inputs, and hybrid inputs of character strings and tensors are supported. The input parameters are separated by commas (,).

> For details about the print function, see [MindSpore Print Operator](https://www.mindspore.cn/tutorial/training/en/master/advanced_use/custom_debugging_info.html#mindspore-print-operator).

- `print_file_path`: saves the print operator data to a file and disables the screen printing function. If the file to be saved exists, a timestamp suffix is added to the file. Saving data to a file can solve the problem that the data displayed on the screen is lost when the data volume is large.

A code example is as follows:

```python
from mindspore import context
context.set_context(print_file_path="print.pb")
```

> For details about the context API, see [mindspore.context](https://www.mindspore.cn/doc/api_python/en/master/mindspore/mindspore.context.html).