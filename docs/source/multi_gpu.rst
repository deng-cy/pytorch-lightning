.. testsetup:: *

    import torch
    from pytorch_lightning.trainer.trainer import Trainer
    from pytorch_lightning.core.lightning import LightningModule

.. _multi-gpu-training:

Multi-GPU training
==================
Lightning supports multiple ways of doing distributed training.

Preparing your code
-------------------
To train on CPU/GPU/TPU without changing your code, we need to build a few good habits :)

Delete .cuda() or .to() calls
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Delete any calls to .cuda() or .to(device).

.. testcode::

    # before lightning
    def forward(self, x):
        x = x.cuda(0)
        layer_1.cuda(0)
        x_hat = layer_1(x)

    # after lightning
    def forward(self, x):
        x_hat = layer_1(x)

Init using type_as
^^^^^^^^^^^^^^^^^^
When you need to create a new tensor, use `type_as`.
This will make your code scale to any arbitrary number of GPUs or TPUs with Lightning

.. testcode::

    # before lightning
    def forward(self, x):
        z = torch.Tensor(2, 3)
        z = z.cuda(0)

    # with lightning
    def forward(self, x):
        z = torch.Tensor(2, 3)
        z = z.type_as(x, device=self.device)

Every LightningModule knows what device it is on. You can access that reference via `self.device`.

Remove samplers
^^^^^^^^^^^^^^^
For multi-node or TPU training, in PyTorch we must use `torch.nn.DistributedSampler`. The
sampler makes sure each GPU sees the appropriate part of your data.

.. testcode::

    # without lightning
    def train_dataloader(self):
        dataset = MNIST(...)
        sampler = None

        if self.on_tpu:
            sampler = DistributedSampler(dataset)

        return DataLoader(dataset, sampler=sampler)

With Lightning, you don't need to do this because it takes care of adding the correct samplers
when needed.

.. testcode::

    # with lightning
    def train_dataloader(self):
        dataset = MNIST(...)
        return DataLoader(dataset)

.. note:: If you don't want this behavior, disable it with `Trainer(replace_sampler_ddp=False)`

.. note:: For iterable datasets, we don't do this automatically.

Make model picklable
^^^^^^^^^^^^^^^^^^^^
It's very likely your code is already `picklable <https://docs.python.org/3/library/pickle.html>`_,
so you don't have to do anything to make this change.
However, if you run distributed and see an error like this:

.. code-block::

    self._launch(process_obj)
    File "/net/software/local/python/3.6.5/lib/python3.6/multiprocessing/popen_spawn_posix.py", line 47,
    in _launch reduction.dump(process_obj, fp)
    File "/net/software/local/python/3.6.5/lib/python3.6/multiprocessing/reduction.py", line 60, in dump
    ForkingPickler(file, protocol).dump(obj)
    _pickle.PicklingError: Can't pickle <function <lambda> at 0x2b599e088ae8>:
    attribute lookup <lambda> on __main__ failed

This means you have something in your model definition, transforms, optimizer, dataloader or callbacks
that is cannot be pickled. By pickled we mean the following would fail.

.. code-block:: python

    import pickle
    pickle.dump(some_object)

This is a limitation of using multiple processes for distributed training within PyTorch.
To fix this issue, find your piece of code that cannot be pickled. The end of the stacktrace
is usually helpful.

.. code-block::

    self._launch(process_obj)
    File "/net/software/local/python/3.6.5/lib/python3.6/multiprocessing/popen_spawn_posix.py", line 47,
    in _launch reduction.dump(process_obj, fp)
    File "/net/software/local/python/3.6.5/lib/python3.6/multiprocessing/reduction.py", line 60, in dump
    ForkingPickler(file, protocol).dump(obj)
    _pickle.PicklingError: Can't pickle [THIS IS THE THING TO FIND AND DELETE]:
    attribute lookup <lambda> on __main__ failed

ie: in the stacktrace example here, there seems to be a lambda function somewhere in the user code
which cannot be pickled.

GPU device selection
--------------------

You can select the GPU devices with ranges, a list of indices or a string containing
a comma separated list of GPU ids:

.. testsetup::

    k = 1

.. testcode::

    # DEFAULT (int) specifies how many GPUs to use.
    Trainer(gpus=k)

    # Above is equivalent to
    Trainer(gpus=list(range(k)))

    # You specify which GPUs (don't use if running on cluster)
    Trainer(gpus=[0, 1])

    # can also be a string
    Trainer(gpus='0, 1')

    # can also be -1 or '-1', this uses all available GPUs
    # this is equivalent to list(range(torch.cuda.available_devices()))
    Trainer(gpus=-1)

The table below lists examples of possible input formats and how they are interpreted by Lightning.
Note in particular the difference between `gpus=0`, `gpus=[0]` and `gpus="0"`.

+---------------+-----------+---------------------+---------------------------------+
| `gpus`        | Type      | Parsed              | Meaning                         |
+===============+===========+=====================+=================================+
| None          | NoneType  | None                | CPU                             |
+---------------+-----------+---------------------+---------------------------------+
| 0             | int       | None                | CPU                             |
+---------------+-----------+---------------------+---------------------------------+
| 3             | int       | [0, 1, 2]           | first 3 GPUs                    |
+---------------+-----------+---------------------+---------------------------------+
| -1            | int       | [0, 1, 2, ...]      | all available GPUs              |
+---------------+-----------+---------------------+---------------------------------+
| [0]           | list      | [0]                 | GPU 0                           |
+---------------+-----------+---------------------+---------------------------------+
| [1, 3]        | list      | [1, 3]              | GPUs 1 and 3                    |
+---------------+-----------+---------------------+---------------------------------+
| "0"           | str       | [0]                 | GPU 0                           |
+---------------+-----------+---------------------+---------------------------------+
| "3"           | str       | [3]                 | GPU 3                           |
+---------------+-----------+---------------------+---------------------------------+
| "1, 3"        | str       | [1, 3]              | GPUs 1 and 3                    |
+---------------+-----------+---------------------+---------------------------------+
| "-1"          | str       | [0, 1, 2, ...]      | all available GPUs              |
+---------------+-----------+---------------------+---------------------------------+

CUDA flags
^^^^^^^^^^

CUDA flags make certain GPUs visible to your script.
Lightning sets these for you automatically, there's NO NEED to do this yourself.

.. testcode::

    # lightning will set according to what you give the trainer
    os.environ["CUDA_DEVICE_ORDER"] = "PCI_BUS_ID"
    os.environ["CUDA_VISIBLE_DEVICES"] = "0"

However, when using a cluster, Lightning will NOT set these flags (and you should not either).
SLURM will set these for you.
For more details see the `SLURM cluster guide <slurm.rst>`_.


Distributed modes
-----------------
Lightning allows multiple ways of training

- Data Parallel (`distributed_backend='dp'`) (multiple-gpus, 1 machine)
- DistributedDataParallel (`distributed_backend='ddp'`) (multiple-gpus across many machines).
- DistributedDataParallel 2 (`distributed_backend='ddp2'`) (dp in a machine, ddp across machines).
- Horovod (`distributed_backend='horovod'`) (multi-machine, multi-gpu, configured at runtime)
- TPUs (`tpu_cores=8|x`) (tpu or TPU pod)

.. note::
    If you request multiple GPUs or nodes without setting a mode, ddp will be automatically used.

For a deeper understanding of what Lightning is doing, feel free to read this
`guide <https://medium.com/@_willfalcon/9-tips-for-training-lightning-fast-neural-networks-in-pytorch-8e63a502f565>`_.



Data Parallel
^^^^^^^^^^^^^
`DataParallel <https://pytorch.org/docs/stable/nn.html#torch.nn.DataParallel>`_ splits a batch across k GPUs.
That is, if you have a batch of 32 and use dp with 2 gpus, each GPU will process 16 samples,
after which the root node will aggregate the results.

.. warning:: DP use is discouraged by PyTorch and Lightning. Use ddp which is more stable and at least 3x faster

.. testcode::
    :skipif: torch.cuda.device_count() < 2

    # train on 2 GPUs (using dp mode)
    trainer = Trainer(gpus=2, distributed_backend='dp')

Distributed Data Parallel
^^^^^^^^^^^^^^^^^^^^^^^^^
`DistributedDataParallel <https://pytorch.org/docs/stable/nn.html#distributeddataparallel>`_ works as follows.

1. Each GPU across every node gets its own process.

2. Each GPU gets visibility into a subset of the overall dataset. It will only ever see that subset.

3. Each process inits the model.

.. note:: Make sure to set the random seed so that each model initializes with the same weights.

4. Each process performs a full forward and backward pass in parallel.

5. The gradients are synced and averaged across all processes.

6. Each process updates its optimizer.

.. code-block:: python

    # train on 8 GPUs (same machine (ie: node))
    trainer = Trainer(gpus=8, distributed_backend='ddp')

    # train on 32 GPUs (4 nodes)
    trainer = Trainer(gpus=8, distributed_backend='ddp', num_nodes=4)

Distributed Data Parallel 2
^^^^^^^^^^^^^^^^^^^^^^^^^^^
In certain cases, it's advantageous to use all batches on the same machine instead of a subset.
For instance you might want to compute a NCE loss where it pays to have more negative samples.

In  this case, we can use ddp2 which behaves like dp in a machine and ddp across nodes. DDP2 does the following:

1. Copies a subset of the data to each node.

2. Inits a model on each node.

3. Runs a forward and backward pass using DP.

4. Syncs gradients across nodes.

5. Applies the optimizer updates.

.. code-block:: python

    # train on 32 GPUs (4 nodes)
    trainer = Trainer(gpus=8, distributed_backend='ddp2', num_nodes=4)

Horovod
^^^^^^^
`Horovod <http://horovod.ai>`_ allows the same training script to be used for single-GPU,
multi-GPU, and multi-node training.

Like Distributed Data Parallel, every process in Horovod operates on a single GPU with a fixed
subset of the data.  Gradients are averaged across all GPUs in parallel during the backward pass,
then synchronously applied before beginning the next step.

The number of worker processes is configured by a driver application (`horovodrun` or `mpirun`). In
the training script, Horovod will detect the number of workers from the environment, and automatically
scale the learning rate to compensate for the increased total batch size.

Horovod can be configured in the training script to run with any number of GPUs / processes as follows:

.. code-block:: python

    # train Horovod on GPU (number of GPUs / machines provided on command-line)
    trainer = Trainer(distributed_backend='horovod', gpus=1)

    # train Horovod on CPU (number of processes / machines provided on command-line)
    trainer = Trainer(distributed_backend='horovod')

When starting the training job, the driver application will then be used to specify the total
number of worker processes:

.. code-block:: bash

    # run training with 4 GPUs on a single machine
    horovodrun -np 4 python train.py

    # run training with 8 GPUs on two machines (4 GPUs each)
    horovodrun -np 8 -H hostname1:4,hostname2:4 python train.py

See the official `Horovod documentation <https://horovod.readthedocs.io/en/stable>`_ for details
on installation and performance tuning.

DP/DDP2 caveats
^^^^^^^^^^^^^^^
In DP and DDP2 each GPU within a machine sees a portion of a batch.
DP and ddp2 roughly do the following:

.. testcode::

    def distributed_forward(batch, model):
        batch = torch.Tensor(32, 8)
        gpu_0_batch = batch[:8]
        gpu_1_batch = batch[8:16]
        gpu_2_batch = batch[16:24]
        gpu_3_batch = batch[24:]

        y_0 = model_copy_gpu_0(gpu_0_batch)
        y_1 = model_copy_gpu_1(gpu_1_batch)
        y_2 = model_copy_gpu_2(gpu_2_batch)
        y_3 = model_copy_gpu_3(gpu_3_batch)

        return [y_0, y_1, y_2, y_3]

So, when Lightning calls any of the `training_step`, `validation_step`, `test_step`
you will only be operating on one of those pieces.

.. testcode::

    # the batch here is a portion of the FULL batch
    def training_step(self, batch, batch_idx):
        y_0 = batch

For most metrics, this doesn't really matter. However, if you want
to add something to your computational graph (like softmax)
using all batch parts you can use the `training_step_end` step.

.. testcode::

    def training_step_end(self, outputs):
        # only use when  on dp
        outputs = torch.cat(outputs, dim=1)
        softmax = softmax(outputs, dim=1)
        out = softmax.mean()
        return out

In pseudocode, the full sequence is:

.. code-block:: python

    # get data
    batch = next(dataloader)

    # copy model and data to each gpu
    batch_splits = split_batch(batch, num_gpus)
    models = copy_model_to_gpus(model)

    # in parallel, operate on each batch chunk
    all_results = []
    for gpu_num in gpus:
        batch_split = batch_splits[gpu_num]
        gpu_model = models[gpu_num]
        out = gpu_model(batch_split)
        all_results.append(out)

    # use the full batch for something like softmax
    full out = model.training_step_end(all_results)

to illustrate why this is needed, let's look at DataParallel

.. testcode::

    def training_step(self, batch, batch_idx):
        x, y = batch
        y_hat = self(batch)

        # on dp or ddp2 if we did softmax now it would be wrong
        # because batch is actually a piece of the full batch
        return y_hat

    def training_step_end(self, batch_parts_outputs):
        # batch_parts_outputs has outputs of each part of the batch

        # do softmax here
        outputs = torch.cat(outputs, dim=1)
        softmax = softmax(outputs, dim=1)
        out = softmax.mean()

        return out

If `training_step_end` is defined it will be called regardless of tpu, dp, ddp, etc... which means
it will behave the same no matter the backend.

Validation and test step also have the same option when using dp

.. testcode::

    def validation_step_end(self, batch_parts_outputs):
        ...

    def test_step_end(self, batch_parts_outputs):
        ...


Distributed and 16-bit precision
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Due to an issue with apex and DistributedDataParallel (PyTorch and NVIDIA issue), Lightning does
not allow 16-bit and DP training. We tried to get this to work, but it's an issue on their end.

Below are the possible configurations we support.

+-------+---------+----+-----+---------+------------------------------------------------------------+
| 1 GPU | 1+ GPUs | DP | DDP | 16-bit  | command                                                    |
+=======+=========+====+=====+=========+============================================================+
| Y     |         |    |     |         | `Trainer(gpus=1)`                                          |
+-------+---------+----+-----+---------+------------------------------------------------------------+
| Y     |         |    |     | Y       | `Trainer(gpus=1, use_amp=True)`                            |
+-------+---------+----+-----+---------+------------------------------------------------------------+
|       | Y       | Y  |     |         | `Trainer(gpus=k, distributed_backend='dp')`                |
+-------+---------+----+-----+---------+------------------------------------------------------------+
|       | Y       |    | Y   |         | `Trainer(gpus=k, distributed_backend='ddp')`               |
+-------+---------+----+-----+---------+------------------------------------------------------------+
|       | Y       |    | Y   | Y       | `Trainer(gpus=k, distributed_backend='ddp', use_amp=True)` |
+-------+---------+----+-----+---------+------------------------------------------------------------+


Implement Your Own Distributed (DDP) training
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If you need your own way to init PyTorch DDP you can override :meth:`pytorch_lightning.core.LightningModule.`.

If you also need to use your own DDP implementation, override:  :meth:`pytorch_lightning.core.LightningModule.configure_ddp`.


Batch size
----------
When using distributed training make sure to modify your learning rate according to your effective
batch size.

Let's say you have a batch size of 7 in your dataloader.

.. testcode::

    class LitModel(LightningModule):

        def train_dataloader(self):
            return Dataset(..., batch_size=7)

In (DDP, Horovod) your effective batch size will be 7 * gpus * num_nodes.

.. code-block:: python

    # effective batch size = 7 * 8
    Trainer(gpus=8, distributed_backend='ddp|horovod')

    # effective batch size = 7 * 8 * 10
    Trainer(gpus=8, num_nodes=10, distributed_backend='ddp|horovod')


In DDP2, your effective batch size will be 7 * num_nodes.
The reason is that the full batch is visible to all GPUs on the node when using DDP2.

.. code-block:: python

    # effective batch size = 7
    Trainer(gpus=8, distributed_backend='ddp2')

    # effective batch size = 7 * 10
    Trainer(gpus=8, num_nodes=10, distributed_backend='ddp2')


.. note:: Huge batch sizes are actually really bad for convergence. Check out:
        `Accurate, Large Minibatch SGD: Training ImageNet in 1 Hour <https://arxiv.org/abs/1706.02677>`_
                
PytorchElastic
--------------
Lightning supports the use of PytorchElastic to enable fault-tolerent and elastic distributed job scheduling. To use it, specify the 'ddp' or 'ddp2' backend and the number of gpus you want to use in the trainer.

.. code-block:: python

    Trainer(gpus=8, distributed_backend='ddp')
    
    
Following the `PytorchElastic Quickstart documentation <https://pytorch.org/elastic/latest/quickstart.html>`_, you then need to start a single-node etcd server on one of the hosts:

.. code-block:: bash

    etcd --enable-v2
         --listen-client-urls http://0.0.0.0:2379,http://127.0.0.1:4001
         --advertise-client-urls PUBLIC_HOSTNAME:2379
         
     
And then launch the elastic job with:

.. code-block:: bash

    python -m torchelastic.distributed.launch
            --nnodes=MIN_SIZE:MAX_SIZE
            --nproc_per_node=TRAINERS_PER_NODE
            --rdzv_id=JOB_ID
            --rdzv_backend=etcd
            --rdzv_endpoint=ETCD_HOST:ETCD_PORT
            YOUR_LIGHTNING_TRAINING_SCRIPT.py (--arg1 ... train script args...)
            

See the official `PytorchElastic documentation <https://pytorch.org/elastic>`_ for details
on installation and more use cases.
