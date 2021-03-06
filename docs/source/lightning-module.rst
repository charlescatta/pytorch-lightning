.. role:: hidden
    :class: hidden-section
.. _lightning-module:

LightningModule
===============
A :class:`~LightningModule` organizes your PyTorch code into 5 sections

- Computations (init).
- Train loop (training_step)
- Validation loop (validation_step)
- Test loop (test_step)
- Optimizers (configure_optimizers)

|

.. raw:: html

    <video width="100%" controls autoplay src="https://pl-bolts-doc-images.s3.us-east-2.amazonaws.com/pl_docs/pl_mod_vid.m4v"></video>

|

Notice a few things.

1.  It's the SAME code.
2.  The PyTorch code IS NOT abstracted - just organized.
3.  All the other code that's not in the :class:`~LightningModule`
    has been automated for you by the trainer.

|

    .. code-block:: python

        net = Net()
        trainer = Trainer()
        trainer.fit(net)

4.  There are no .cuda() or .to() calls... Lightning does these for you.

|

    .. code-block:: python

        # don't do in lightning
        x = torch.Tensor(2, 3)
        x = x.cuda()
        x = x.to(device)

        # do this instead
        x = x  # leave it alone!

        # or to init a new tensor
        new_x = torch.Tensor(2, 3)
        new_x = new_x.type_as(x)

5.  There are no samplers for distributed, Lightning also does this for you.

|

    .. code-block:: python

        # Don't do in Lightning...
        data = MNIST(...)
        sampler = DistributedSampler(data)
        DataLoader(data, sampler=sampler)

        # do this instead
        data = MNIST(...)
        DataLoader(data)

6.  A :class:`~LightningModule` is a :class:`torch.nn.Module` but with added functionality. Use it as such!

|

    .. code-block:: python

        net = Net.load_from_checkpoint(PATH)
        net.freeze()
        out = net(x)

Thus, to use Lightning, you just need to organize your code which takes about 30 minutes,
(and let's be real, you probably should do anyhow).

------------

Minimal Example
---------------

Here are the only required methods.

.. code-block:: python

    >>> import pytorch_lightning as pl
    >>> class LitModel(pl.LightningModule):
    ...
    ...     def __init__(self):
    ...         super().__init__()
    ...         self.l1 = torch.nn.Linear(28 * 28, 10)
    ...
    ...     def forward(self, x):
    ...         return torch.relu(self.l1(x.view(x.size(0), -1)))
    ...
    ...     def training_step(self, batch, batch_idx):
    ...         x, y = batch
    ...         y_hat = self(x)
    ...         loss = F.cross_entropy(y_hat, y)
    ...         return pl.TrainResult(loss)
    ...
    ...     def configure_optimizers(self):
    ...         return torch.optim.Adam(self.parameters(), lr=0.02)

Which you can train by doing:

.. code-block:: python

    train_loader = DataLoader(MNIST(os.getcwd(), download=True, transform=transforms.ToTensor()))
    trainer = pl.Trainer()
    model = LitModel()

    trainer.fit(model, train_loader)

----------

LightningModule for research
----------------------------
For research, LightningModules are best structured as systems.

A model (colloquially) refers to something like a resnet or RNN. A system, may be a collection of models. Here
are examples of systems:

- GAN (generator, discriminator)
- RL (policy, actor, critic)
- Autoencoders (encoder, decoder)
- Seq2Seq (encoder, attention, decoder)
- etc...

A LightningModule is best used to define a complex system:

.. code-block:: python

    import pytorch_lightning as pl
    import torch
    from torch import nn

    class Autoencoder(pl.LightningModule):

         def __init__(self, latent_dim=2):
            super().__init__()
            self.encoder = nn.Sequential(nn.Linear(28 * 28, 256), nn.ReLU(), nn.Linear(256, latent_dim))
            self.decoder = nn.Sequential(nn.Linear(latent_dim, 256), nn.ReLU(), nn.Linear(256, 28 * 28))

         def training_step(self, batch, batch_idx):
            x, _ = batch

            # encode
            x = x.view(x.size(0), -1)
            z = self.encoder(x)

            # decode
            recons = self.decoder(z)

            # reconstruction
            reconstruction_loss = nn.functional.mse_loss(recons, x)
            return pl.TrainResult(reconstruction_loss)

         def validation_step(self, batch, batch_idx):
            x, _ = batch
            x = x.view(x.size(0), -1)
            z = self.encoder(x)
            recons = self.decoder(z)
            reconstruction_loss = nn.functional.mse_loss(recons, x)

            result = pl.EvalResult(checkpoint_on=reconstruction_loss)
            return result

         def configure_optimizers(self):
            return torch.optim.Adam(self.parameters(), lr=0.0002)

Which can be trained like this:

.. code-block:: python

    autoencoder = Autoencoder()
    trainer = pl.Trainer(gpus=1)
    trainer.fit(autoencoder, train_dataloader, val_dataloader)

This simple model generates examples that look like this (the encoders and decoders are too weak)

.. figure:: https://pl-bolts-doc-images.s3.us-east-2.amazonaws.com/pl_docs/ae_docs.png
    :width: 300

The methods above are part of the lightning interface:

- training_step
- validation_step
- test_step
- configure_optimizers

Note that in this case, the train loop and val loop are exactly the same. We can of course reuse this code.

.. code-block:: python

    class Autoencoder(pl.LightningModule):

         def __init__(self, latent_dim=2):
            super().__init__()
            self.encoder = nn.Sequential(nn.Linear(28 * 28, 256), nn.ReLU(), nn.Linear(256, latent_dim))
            self.decoder = nn.Sequential(nn.Linear(latent_dim, 256), nn.ReLU(), nn.Linear(256, 28 * 28))

         def training_step(self, batch, batch_idx):
            loss = self.shared_step(batch)
            return pl.TrainResult(loss)

         def validation_step(self, batch, batch_idx):
            loss = self.shared_step(batch)
            result = pl.EvalResult(checkpoint_on=loss)
            return result

         def shared_step(self, batch):
            x, _ = batch

            # encode
            x = x.view(x.size(0), -1)
            z = self.encoder(x)

            # decode
            recons = self.decoder(z)

            # loss
            return nn.functional.mse_loss(recons, x)

         def configure_optimizers(self):
            return torch.optim.Adam(self.parameters(), lr=0.0002)

We create a new method called `shared_step` that all loops can use. This method name is arbitrary and NOT reserved.

Inference in Research
^^^^^^^^^^^^^^^^^^^^^
In the case where we want to perform inference with the system we can add a `forward` method to the LightningModule.

.. code-block:: python

    class Autoencoder(pl.LightningModule):
        def forward(self, x):
            return self.decoder(x)

The advantage of adding a forward is that in complex systems, you can do a much more involved inference procedure,
such as text generation:

.. code-block:: python

    class Seq2Seq(pl.LightningModule):

        def forward(self, x):
            embeddings = self(x)
            hidden_states = self.encoder(embeddings)
            for h in hidden_states:
                # decode
                ...
            return decoded

---------------------

LightningModule for production
------------------------------
For cases like production, you might want to iterate different models inside a LightningModule.

.. code-block:: python

    import pytorch_lightning as pl
    from pytorch_lightning.metrics import functional as FM

    class ClassificationTask(pl.LightningModule):

         def __init__(self, model):
             super().__init__()
             self.model = model

         def training_step(self, batch, batch_idx):
             x, y = batch
             y_hat = self.model(x)
             loss = F.cross_entropy(y_hat, y)
             return pl.TrainResult(loss)

         def validation_step(self, batch, batch_idx):
            x, y = batch
            y_hat = self.model(x)
            loss = F.cross_entropy(y_hat, y)
            acc = FM.accuracy(y_hat, y)
            result = pl.EvalResult(checkpoint_on=loss)
            result.log_dict({'val_acc': acc, 'val_loss': loss})
            return result

         def test_step(self, batch, batch_idx):
            result = self.validation_step(batch, batch_idx)
            result.rename_keys({'val_acc': 'test_acc', 'val_loss': 'test_loss'})
            return result

         def configure_optimizers(self):
             return torch.optim.Adam(self.model.parameters(), lr=0.02)

Then pass in any arbitrary model to be fit with this task

.. code-block:: python

    for model in [resnet50(), vgg16(), BidirectionalRNN()]:
        task = ClassificationTask(model)

        trainer = Trainer(gpus=2)
        trainer.fit(task, train_dataloader, val_dataloader)

Tasks can be arbitrarily complex such as implementing GAN training, self-supervised or even RL.

.. code-block:: python

    class GANTask(pl.LightningModule):

         def __init__(self, generator, discriminator):
             super().__init__()
             self.generator = generator
             self.discriminator = discriminator
         ...

Inference in production
^^^^^^^^^^^^^^^^^^^^^^^
When used like this, the model can be separated from the Task and thus used in production without needing to keep it in
a `LightningModule`.

- You can export to onnx.
- Or trace using Jit.
- or run in the python runtime.

.. code-block:: python

        task = ClassificationTask(model)

        trainer = Trainer(gpus=2)
        trainer.fit(task, train_dataloader, val_dataloader)

        # use model after training or load weights and drop into the production system
        model.eval()
        y_hat = model(x)


Training loop
-------------
To add a training loop use the `training_step` method

.. code-block:: python

    class LitClassifier(pl.LightningModule):

         def __init__(self, model):
             super().__init__()
             self.model = model

         def training_step(self, batch, batch_idx):
             x, y = batch
             y_hat = self.model(x)
             loss = F.cross_entropy(y_hat, y)
             return pl.TrainResult(loss)

Under the hood, Lightning does the following (pseudocode):

.. code-block:: python

    # put model in train mode
    model.train()
    torch.set_grad_enabled(True)

    outs = []
    for batch in train_dataloader:
        # forward
        out = training_step(val_batch)

        # backward
        loss.backward()

        # apply and clear grads
        optimizer.step()
        optimizer.zero_grad()

Training epoch-level metrics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If you want to calculate epoch-level metrics and log them, use the `TrainResult.log` method

.. code-block:: python

     def training_step(self, batch, batch_idx):
         x, y = batch
         y_hat = self.model(x)
         loss = F.cross_entropy(y_hat, y)
         result = pl.TrainResult(loss)

         # logs metrics for each training_step, and the average across the epoch, to the progress bar and logger
         result.log('train_loss', loss, on_step=True, on_epoch=True, prog_bar=True, logger=True)
         return result

The `TrainResult.log` object automatically reduces the requested metrics across the full epoch.
Here's the pseudocode of what it does under the hood:

.. code-block:: python

    outs = []
    for batch in train_dataloader:
        # forward
        out = training_step(val_batch)

        # backward
        loss.backward()

        # apply and clear grads
        optimizer.step()
        optimizer.zero_grad()

    epoch_metric = torch.mean(torch.stack([x['train_loss'] for x in outs]))

Train epoch-level operations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If you need to do something with all the outputs of each `training_step`, override `training_epoch_end` yourself.

.. code-block:: python

     def training_step(self, batch, batch_idx):
         x, y = batch
         y_hat = self.model(x)
         loss = F.cross_entropy(y_hat, y)
         result = pl.TrainResult(loss)
         result.prediction = some_prediction

     def training_epoch_end(self, training_step_outputs):
        all_predictions = training_step_outputs.prediction
        ...
        return result

The matching pseudocode is:

.. code-block:: python

    outs = []
    for batch in train_dataloader:
        # forward
        out = training_step(val_batch)

        # backward
        loss.backward()

        # apply and clear grads
        optimizer.step()
        optimizer.zero_grad()

    epoch_out = training_epoch_end(outs)

Training with DataParallel
^^^^^^^^^^^^^^^^^^^^^^^^^^
When training using a `distributed_backend` that splits data from each batch across GPUs, sometimes you might
need to aggregate them on the master GPU for processing (dp, or ddp2).

In this case, implement the `training_step_end` method

.. code-block:: python

     def training_step(self, batch, batch_idx):
         x, y = batch
         y_hat = self.model(x)
         loss = F.cross_entropy(y_hat, y)
         result = pl.TrainResult(loss)
         result.prediction = some_prediction

     def training_step_end(self, batch_parts):
         gpu_0_prediction = batch_parts.prediction[0]
         gpu_1_prediction = batch_parts.prediction[1]

         # do something with both outputs
         return result

     def training_epoch_end(self, training_step_outputs):
        all_predictions = training_step_outputs.prediction
        ...
        return result

The full pseudocode that lighting does under the hood is:

.. code-block:: python

    outs = []
    for train_batch in train_dataloader:
        batches = split_batch(train_batch)
        dp_outs = []
        for sub_batch in batches:
            # 1
            dp_out = training_step(sub_batch)
            dp_outs.append(dp_out)

        # 2
        out = training_step_end(dp_outs)
        outs.append(out)

    # do something with the outputs for all batches
    # 3
    training_epoch_end(outs)

------------------

Validation loop
---------------
To add a validation loop, override the `validation_step` method of the :class:`~LightningModule`:

.. code-block:: python

    class LitModel(pl.LightningModule):
        def validation_step(self, batch, batch_idx):
            x, y = batch
            y_hat = self.model(x)
            loss = F.cross_entropy(y_hat, y)
            result = pl.EvalResult(checkpoint_on=loss)
            return result

Under the hood, Lightning does the following:

.. code-block:: python

    # ...
    for batch in train_dataloader:
        loss = model.training_step()
        loss.backward()
        # ...

        if validate_at_some_point:
            # disable grads + batchnorm + dropout
            torch.set_grad_enabled(False)
            model.eval()

            # ----------------- VAL LOOP ---------------
            for val_batch in model.val_dataloader:
                val_out = model.validation_step(val_batch)
            # ----------------- VAL LOOP ---------------

            # enable grads + batchnorm + dropout
            torch.set_grad_enabled(True)
            model.train()

Validation epoch-level metrics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If you need to do something with all the outputs of each `validation_step`, override `validation_epoch_end`.

.. code-block:: python

     def validation_step(self, batch, batch_idx):
         x, y = batch
         y_hat = self.model(x)
         loss = F.cross_entropy(y_hat, y)
         result = pl.EvalResult(loss)
         result.prediction = some_prediction

     def validation_epoch_end(self, validation_step_outputs):
        all_predictions = validation_step_outputs.prediction
        ...
        return result

Validating with DataParallel
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
When training using a `distributed_backend` that splits data from each batch across GPUs, sometimes you might
need to aggregate them on the master GPU for processing (dp, or ddp2).

In this case, implement the `validation_step_end` method

.. code-block:: python

     def validation_step(self, batch, batch_idx):
         x, y = batch
         y_hat = self.model(x)
         loss = F.cross_entropy(y_hat, y)
         result = pl.EvalResult(loss)
         result.prediction = some_prediction

     def validation_step_end(self, batch_parts):
         gpu_0_prediction = batch_parts.prediction[0]
         gpu_1_prediction = batch_parts.prediction[1]

         # do something with both outputs
         return result

     def validation_epoch_end(self, validation_step_outputs):
        all_predictions = validation_step_outputs.prediction
        ...
        return result

The full pseudocode that lighting does under the hood is:

.. code-block:: python

    outs = []
    for batch in dataloader:
        batches = split_batch(batch)
        dp_outs = []
        for sub_batch in batches:
            # 1
            dp_out = validation_step(sub_batch)
            dp_outs.append(dp_out)

        # 2
        out = validation_step_end(dp_outs)
        outs.append(out)

    # do something with the outputs for all batches
    # 3
    validation_epoch_end(outs)

----------------

Test loop
---------
The process for adding a test loop is the same as the process for adding a validation loop. Please refer to
the section above for details.

The only difference is that the test loop is only called when `.test()` is used:

.. code-block:: python

    model = Model()
    trainer = Trainer()
    trainer.fit()

    # automatically loads the best weights for you
    trainer.test(model)

There are two ways to call `test()`:

.. code-block:: python

    # call after training
    trainer = Trainer()
    trainer.fit(model)

    # automatically auto-loads the best weights
    trainer.test(test_dataloaders=test_dataloader)

    # or call with pretrained model
    model = MyLightningModule.load_from_checkpoint(PATH)
    trainer = Trainer()
    trainer.test(model, test_dataloaders=test_dataloader)

----------

Live demo
---------
Check out this
`COLAB <https://colab.research.google.com/drive/1F_RNcHzTfFuQf-LeKvSlud6x7jXYkG31#scrollTo=HOk9c4_35FKg>`_
for a live demo.

-----------

LightningModule API
-------------------

Training loop methods
^^^^^^^^^^^^^^^^^^^^^

training_step
~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.training_step
    :noindex:

training_step_end
~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.training_step_end
    :noindex:

training_epoch_end
~~~~~~~~~~~~~~~~~~
.. autofunction:: pytorch_lightning.core.lightning.LightningModule.training_epoch_end
    :noindex:

---------------

Validation loop methods
^^^^^^^^^^^^^^^^^^^^^^^

validation_step
~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.validation_step
    :noindex:

validation_step_end
~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.validation_step_end
    :noindex:

validation_epoch_end
~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.validation_epoch_end
    :noindex:

----------------

test loop methods
^^^^^^^^^^^^^^^^^

test_step
~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.test_step
    :noindex:

test_step_end
~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.test_step_end
    :noindex:

test_epoch_end
~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.test_epoch_end
    :noindex:

--------------

configure_optimizers
^^^^^^^^^^^^^^^^^^^^

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.configure_optimizers
    :noindex:

--------------

Convenience methods
^^^^^^^^^^^^^^^^^^^
Use these methods for convenience

print
~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.print
    :noindex:

save_hyperparameters
~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.save_hyperparameters
    :noindex:

------------

Inference methods
^^^^^^^^^^^^^^^^^
Use these hooks for inference with a lightning module

forward
~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.forward
    :noindex:

freeze
~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.freeze
    :noindex:

to_onnx
~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.to_onnx
    :noindex:

to_torchscript
~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.to_torchscript
    :noindex:

unfreeze
~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.unfreeze
    :noindex:

------------

Properties
^^^^^^^^^^
These are properties available in a LightningModule.

-----------

current_epoch
~~~~~~~~~~~~~
The current epoch

.. code-block:: python

    def training_step(...):
        if self.current_epoch == 0:

-------------

device
~~~~~~
The device the module is on. Use it to keep your code device agnostic

.. code-block:: python

    def training_step(...):
        z = torch.rand(2, 3, device=self.device)

-------------

global_rank
~~~~~~~~~~~
The global_rank of this LightningModule. Lightning saves logs, weights etc only from global_rank = 0. You
normally do not need to use this property

Global rank refers to the index of that GPU across ALL GPUs. For example, if using 10 machines, each with 4 GPUs,
the 4th GPU on the 10th machine has global_rank = 39

-------------

global_step
~~~~~~~~~~~
The current step (does not reset each epoch)

.. code-block:: python

    def training_step(...):
        self.logger.experiment.log_image(..., step=self.global_step)

-------------

hparams
~~~~~~~
After calling `save_hyperparameters` anything passed to init() is available via hparams.

.. code-block:: python

    def __init__(self, learning_rate):
        self.save_hyperparameters()

    def configure_optimizers(self):
        return Adam(self.parameters(), lr=self.hparams.learning_rate)

--------------

logger
~~~~~~
The current logger being used (tensorboard or other supported logger)

.. code-block:: python

    def training_step(...):
        # the generic logger (same no matter if tensorboard or other supported logger)
        self.logger

        # the particular logger
        tensorboard_logger = self.logger.experiment

--------------

local_rank
~~~~~~~~~~~
The local_rank of this LightningModule. Lightning saves logs, weights etc only from global_rank = 0. You
normally do not need to use this property

Local rank refers to the rank on that machine. For example, if using 10 machines, the GPU at index 0 on each machine
has local_rank = 0.


-----------

precision
~~~~~~~~~
The type of precision used:

.. code-block:: python

    def training_step(...):
        if self.precision == 16:

------------

trainer
~~~~~~~
Pointer to the trainer

.. code-block:: python

    def training_step(...):
        max_steps = self.trainer.max_steps
        any_flag = self.trainer.any_flag

------------

use_ddp
~~~~~~~
True if using ddp

------------

use_ddp2
~~~~~~~~
True if using ddp2

------------

use_dp
~~~~~~
True if using dp

------------

use_tpu
~~~~~~~
True if using TPUs

--------------

Hooks
-----

Hook lifecycle pseudocode
^^^^^^^^^^^^^^^^^^^^^^^^^
This is the pseudocode to describe how all the hooks are called during a call to `.fit()`

.. code-block:: python

    def fit(...):
        on_fit_start()

        if global_rank == 0:
            # prepare data is called on GLOBAL_ZERO only
            prepare_data()

        for gpu/tpu in gpu/tpus:
            train_on_device(model.copy())

        on_fit_end()

    def train_on_device(model):
        # setup is called PER DEVICE
        setup()
        configure_optimizers()
        on_pretrain_routine_start()

        for epoch in epochs:
            train_loop()

        teardown()

    def train_loop():
        on_train_epoch_start()
        train_outs = []
        for train_batch in train_dataloader():
            on_train_batch_start()

            # ----- train_step methods -------
            out = training_step(batch)
            train_outs.append(out)

            loss = out.loss

            backward()
            on_after_backward()
            optimizer_step()
            on_before_zero_grad()
            optimizer_zero_grad()

            on_train_batch_end()

            if should_check_val:
                val_loop()

        # end training epoch
        logs = training_epoch_end(outs)

    def val_loop():
        model.eval()
        torch.set_grad_enabled(False)

        on_validation_epoch_start()
        val_outs = []
        for val_batch in val_dataloader():
            on_validation_batch_start()

            # -------- val step methods -------
            out = validation_step(val_batch)
            val_outs.append(out)

            on_validation_batch_end()

        validation_epoch_end(val_outs)
        on_validation_epoch_end()

        # set up for train
        model.train()
        torch.set_grad_enabled(True)


Advanced hooks
^^^^^^^^^^^^^^
Use these hooks to modify advanced functionality

configure_apex
~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.configure_apex
    :noindex:

configure_ddp
~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.configure_ddp
    :noindex:

configure_sync_batchnorm
~~~~~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.configure_ddp
    :noindex:

get_progress_bar_dict
~~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.get_progress_bar_dict
    :noindex:

init_ddp_connection
~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.init_ddp_connection
    :noindex:

tbptt_split_batch
~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.tbptt_split_batch
    :noindex:

Checkpoint hooks
^^^^^^^^^^^^^^^^
These hooks allow you to modify checkpoints

on_load_checkpoint
~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.on_load_checkpoint
    :noindex:

on_save_checkpoint
~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.on_save_checkpoint
    :noindex:

-------------

Data hooks
^^^^^^^^^^
Use these hooks if you want to couple a LightningModule to a dataset.

.. note:: The same collection of hooks is available in a DataModule class to decouple the data from the model.

train_dataloader
~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.train_dataloader
    :noindex:

val_dataloader
~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.val_dataloader
    :noindex:

test_dataloader
~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.test_dataloader
    :noindex:

prepare_data
~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.prepare_data
    :noindex:

------------

Optimization hooks
^^^^^^^^^^^^^^^^^^
These are hooks related to the optimization procedure.

backward
~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.backward
    :noindex:

on_after_backward
~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.on_after_backward
    :noindex:

on_before_zero_grad
~~~~~~~~~~~~~~~~~~~
.. autofunction:: pytorch_lightning.core.lightning.LightningModule.on_before_zero_grad
    :noindex:

optimizer_step
~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.optimizer_step
    :noindex:

optimizer_zero_grad
~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.lightning.LightningModule.optimizer_zero_grad
    :noindex:

Training lifecycle hooks
^^^^^^^^^^^^^^^^^^^^^^^^^
These hooks are called during training

on_fit_start
~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_fit_start
    :noindex:

on_fit_end
~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_fit_end
    :noindex:

on_pretrain_routine_start
~~~~~~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_pretrain_routine_start
    :noindex:

on_pretrain_routine_end
~~~~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_pretrain_routine_end
    :noindex:

on_test_epoch_start
~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_test_epoch_start
    :noindex:

on_test_epoch_end
~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_test_epoch_end
    :noindex:

on_test_batch_start
~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_test_batch_start
    :noindex:

on_test_batch_end
~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_test_batch_end
    :noindex:

on_train_batch_start
~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_train_batch_start
    :noindex:

on_train_batch_end
~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_train_batch_end
    :noindex:

on_train_epoch_start
~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_train_epoch_start
    :noindex:

on_train_epoch_end
~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_train_epoch_end
    :noindex:

on_validation_batch_start
~~~~~~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_validation_batch_start
    :noindex:

on_validation_batch_end
~~~~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_validation_batch_end
    :noindex:

on_validation_epoch_start
~~~~~~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_validation_epoch_start
    :noindex:

on_validation_epoch_end
~~~~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.on_validation_epoch_end
    :noindex:

setup
~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.setup
    :noindex:

teardown
~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.teardown
    :noindex:

transfer_batch_to_device
~~~~~~~~~~~~~~~~~~~~~~~~

.. autofunction:: pytorch_lightning.core.hooks.ModelHooks.transfer_batch_to_device
    :noindex:
