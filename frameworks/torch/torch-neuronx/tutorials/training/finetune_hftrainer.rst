.. _torch-hf-bert-finetune:

PyTorch Neuron for Trainium Hugging Face BERT MRPC task finetuning using Hugging Face Trainer API
=================================================================================================

In this tutorial, we show how to run a Hugging Face script that uses Hugging Face Trainer API
to do fine-tuning on Trainium. The example follows the `text-classification
example <https://github.com/huggingface/transformers/tree/master/examples/pytorch/text-classification>`__
which fine-tunes BERT-base model for sequence classification on the GLUE
benchmark.


.. contents:: Table of Contents
   :local:
   :depth: 2

.. include:: ../note-performance.txt

Setup and compilation
---------------------

Before running the tutorial please follow the installation instructions at:

* :ref:`Install PyTorch Neuron on Trn1 <pytorch-neuronx-install>`

Please set the storage of instance to *512GB* or more if you also want to run through the BERT pretraining and GPT pretraining tutorials.

For all the commands below, make sure you are in the virtual environment that you have created above before you run the commands:

.. code:: shell

   source ~/aws_neuron_venv_pytorch/bin/activate

First we install recent version of HF transformers, scikit-learn and evaluate packages in our environment. In this
example, we chose version 4.15.0:

.. code:: bash

    pip install -U transformers==4.15.0 datasets evaluate scikit-learn

Next we download the transformers source, checking out the tag that
matches the installed version:

.. code:: bash

    cd ~/
    git clone https://github.com/huggingface/transformers --branch v4.15.0

Go into the examples BERT training directory:

.. code:: bash

    cd ~/transformers/examples/pytorch/text-classification

Edit the python script run_glue.py and add the following lines after the Python
imports. They set the compiler flag for transformer model type and enable data parallel training using torchrun:

.. code:: python

    # Set compiler flag to compile for transformer model type
    os.environ["NEURON_CC_FLAGS"] = os.environ.get('NEURON_CC_FLAGS', '') + " --model-type=transformer"

    # Enable torchrun
    import torch
    import torch_xla.distributed.xla_backend
    if os.environ.get("WORLD_SIZE"):
        torch.distributed.init_process_group('xla')

    # Fixup to enable distributed training with XLA
    from packaging import version
    from transformers import __version__
    if version.parse(__version__) < version.parse("4.20.0"):
        Trainer._wrap_model = lambda self, model, training=True: model
    else:
        Trainer._wrap_model = lambda self, model, training=True, dataloader=None: model
        
    # Workaround for NaNs seen with transformers version >= 4.21.0
    # https://github.com/aws-neuron/aws-neuron-sdk/issues/593
    if os.environ.get("XLA_USE_BF16") or os.environ.get("XLA_DOWNCAST_BF16"):
        transformers.modeling_utils.get_parameter_dtype = lambda x: torch.bfloat16

We will run MRPC task fine-tuning following the example in README.md. In this part of the tutorial we will use the Hugging Face model hub's pretrained ``bert-large-uncased`` model.

We precompile the model and training script to warm up the persistent
graph cache (Neuron Cache) such that the actual run has fewer compilations (faster run
time). We use full BF16 casting using XLA_USE_BF16=1 to enable best performance.
Create a “compile” file with the following content, change it to
executable, and run it:

.. code:: bash

    export TASK_NAME=mrpc
    neuron_parallel_compile XLA_USE_BF16=1 python3 ./run_glue.py \
    --model_name_or_path bert-large-uncased \
    --task_name $TASK_NAME \
    --do_train \
    --max_seq_length 128 \
    --per_device_train_batch_size 8 \
    --learning_rate 2e-5 \
    --num_train_epochs 2 \
    --overwrite_output_dir \
    --output_dir /tmp/$TASK_NAME/ |& tee log_compile_train

Note that we separating out the train/eval steps during compilation.
Also, we reduce batch size to 8 which is currently optimal for Trainium.
Also, to reduce time to extract graph, we limit the trial run to 2
epochs. Please ignore the results from this trial run as it is only for
extracting and compiling the XLA graphs.

Single-worker training
----------------------

After precompilation, the actual run will be faster with minimal
additional compilations. NOTE: currently the evaluation step would incur
full compilation (see Known Issues below).

To run the actual training, create a “run” file with the following
content and run it, again using full BF16 casting (XLA_USE_BF16=1) to enable best performance.:

.. code:: bash

    export TASK_NAME=mrpc
    XLA_USE_BF16=1 python3 ./run_glue.py \
    --model_name_or_path bert-large-uncased \
    --task_name $TASK_NAME \
    --do_train \
    --do_eval \
    --max_seq_length 128 \
    --per_device_train_batch_size 8 \
    --learning_rate 2e-5 \
    --num_train_epochs 5 \
    --overwrite_output_dir \
    --output_dir /tmp/$TASK_NAME/ |& tee log_run

Rerunning the same script a second time would show quicker execution as the compiled graphs are already cached in persistent cache.

Multi-worker training
---------------------

The above script would run one worker on one NeuronCore. To run on
multiple cores, run the run_glue.py script with torchrun using ``--nproc_per_node=N`` option to specify the number of workers
(N=2 for trn1.2xlarge, and N=2, 8, or 32 for trn1.32xlarge).
The following example runs 2 workers:

.. code:: bash

    export TASK_NAME=mrpc
    XLA_USE_BF16=1 torchrun --nproc_per_node=2 ./run_glue.py \
    --model_name_or_path bert-large-uncased \
    --task_name $TASK_NAME \
    --do_train \
    --do_eval \
    --max_seq_length 128 \
    --per_device_train_batch_size 8 \
    --learning_rate 2e-5 \
    --num_train_epochs 5 \
    --overwrite_output_dir \
    --output_dir /tmp/$TASK_NAME/ |& tee log_run_2w

During run, you will notice that the "Total train batch size" is now 16 and the "Total optimization steps" is now half the number for one worker training.

Converting BERT pretrained checkpoint to Hugging Face pretrained model format
-----------------------------------------------------------------------------
If you have a pretrained checkpoint (i.e., from the BERT phase 2 pretraining tutorial), you can run the script below (saved as "convert.py") to convert BERT pretrained saved checkpoint to Hugging Face pretrained model format. An example phase 2 pretrained checkpoint can be downloaded from ``s3://neuron-s3/training_checkpoints/pytorch/dp_bert_large_hf_pretrain/ckpt_29688.pt``. Note that here we also use the ``bert-large-uncased`` model configuration to match the BERT-Large model trained following BERT phase 2 pretraining tutorial.

.. code:: python

    import os
    import sys
    import argparse
    import torch
    import transformers
    from transformers import (
        BertForPreTraining,
    )
    import torch_xla.core.xla_model as xm
    from transformers.utils import check_min_version
    from transformers.utils.versions import require_version
    
    if __name__ == '__main__':
        parser = argparse.ArgumentParser()
        parser.add_argument('--model_name', type=str, default='bert-large-uncased',  help="Path to model identifier from huggingface.co/models")
        parser.add_argument('--output_saved_model_path', type=str, default='./hf_saved_model', help="Directory to save the HF pretrained model format.")
        parser.add_argument('--checkpoint_path', type=str, required=True, help="Path to pretrained checkpoint which needs to be converted to a HF pretrained model format")
        args = parser.parse_args(sys.argv[1:])
        
        model = BertForPreTraining.from_pretrained(args.model_name)
        check_point = torch.load(args.checkpoint_path, map_location='cpu')
        model.load_state_dict(check_point['model'], strict=False)
        model.save_pretrained(args.output_saved_model_path, save_config=True, save_function=xm.save)

Run the conversion script as:

.. code:: bash

    python convert.py --checkpoint_path ckpt_29688.pt

After conversion, the new Hugging Face pretrained model is stored in the output directory specified by the ``--output_saved_model_path`` option which is ``hf_saved_model`` by default. You will use this directory in the next step.

To run the actual fine-tuning, create a “run_converted" file with the following
script and run it:

.. code:: bash

    export TASK_NAME=mrpc
    XLA_USE_BF16=1 torchrun --nproc_per_node=2 ./run_glue.py \
    --model_name_or_path hf_saved_model \
    --tokenizer_name bert-large-uncased \
    --task_name $TASK_NAME \
    --do_train \
    --do_eval \
    --max_seq_length 128 \
    --per_device_train_batch_size 8 \
    --learning_rate 2e-5 \
    --num_train_epochs 5 \
    --overwrite_output_dir \
    --output_dir /tmp/$TASK_NAME/ |& tee log_run_converted

Known issues and limitations
----------------------------

The following are currently known issues:

-  Long compilation times: this can be alleviated with
   ``neuron_parallel_compile`` tool to extract graphs from a short trial run and
   compile them in parallel ahead of the actual run, as shown above.
-  When precompiling using batch size of 16 on trn1.2xlarge, you will see ``ERROR ||PARALLEL_COMPILE||: parallel compilation with neuronx-cc exited with error.Received error code: -9``. To workaround this error, please set NEURON_PARALLEL_COMPILE_MAX_RETRIES=1 in the environment.
-  When running HuggingFace BERT (any size) fine-tuning tutorial or pretraining tutorial with transformers version >= 4.21.0 and using XLA_USE_BF16=1 or XLA_DOWNCAST_BF16=1, you will see NaNs in the loss immediately at the first step. More details on the issue can be found at `pytorch/xla#4152 <https://github.com/pytorch/xla/issues/4152>`_. The workaround is to use 4.20.0 or earlier or add ``transformers.modeling_utils.get_parameter_dtype = lambda x: torch.bfloat16`` to your Python script (i.e. run_glue.py).
-  Using ``neuron_parallel_compile`` tool to run ``run_glue.py`` script
   with both train and evaluation options (``--do_train`` and ``--do_eval``), you will
   encounter INVALID_ARGUMENT error. To avoid this, only enable train for parallel
   compile (``--do_train``). This will cause compilations during evaluation step.
   The INVALID_ARGUMENT error will fixed in release 2.6 together with latest transformers package version 4.25.1.
-  With release 2.6 and transformers==4.25.1,
   using ``neuron_parallel_compile`` tool to run ``run_glue.py`` script
   with both train and evaluation option (``--do_train`` and ``--do_eval``), you will encounter harmless error
   ``ValueError: Target is multiclass but average='binary'``
-  Reduced accuracy for RoBerta-Large is seen with Neuron PyTorch 1.12 (release 2.6) in FP32 mode with compiler BF16 autocast.
   The workaround is to set NEURON_CC_FLAGS="--auto-cast none" or set NEURON_RT_STOCHASTIC_ROUNDING_EN=1.
-  Some recompilation is seen at the epoch boundary even after ``neuron_parallel_compile`` is used.
-  When running multi-worker training, you may see the process getting killed at the time of model saving.
   This happens because the transformers ``trainer.save_model`` api uses ``xm.save`` for saving models.
   This api is known to cause high host memory usage in multi-worker setting `see Saving and Loading XLA Tensors in  <https://github.com/pytorch/xla/blob/master/API_GUIDE.md>`__ . Coupled with a compilation 
   at the same time results in a host OOM. To avoid this issue, we can: Precompile all the graphs in multi-worker 
   training. This can be done by running the multi-worker training first with ``neuron_parallel_compile`` 
   followed by the actual training. This would avoid the compilation at model save during actual training.



The following are currently known limitations:

-  Only MRPC task fine-tuning is supported. Other tasks will be
   supported in the future.
