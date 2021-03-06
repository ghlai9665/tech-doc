# Train & Upload AI Models to Cortex

# Introduction

Cortex is currently the _only_ public blockchain that allows on-chain execution of machine learning models. Every step of the inference is transparent to the public, verified by blockchain consensus.

**Why should I upload my model to Cortex?**

You will get paid a small amount in CTXC each time your model is called in a smart contract. This award comes from part of the miner's reward and its aim is to incentivize machine learning developers to upload their models to the blockchain for the world to use.

A few essential vocabulary for the rest of the tutorial:

**CVM**:

CVM, short for Cortex Virtual Machine, is the core of the Cortex blockchain. It is responsible for executing code on-chain. It is backward compatible with Ethereum's [EVM](https://ethdocs.org/en/latest/introduction/what-is-ethereum.html) with added support for on-chain machine learning inference. While it is virtually impossible to execute nontrivial ML models on the EVM, CVM is specially designed for on-chain ML inference with innovations such as utilization of node machines' GPUs and deterministic quantization of ML models.

**CVM-Runtime**:

An open-source deterministic machine learning framework written in C++. During runtime, the CVM (Cortex Virtual Machine) executes your models via CVM-Runtime. It is comparable to other frameworks such as MXNet, PyTorch, TensorFlow etc. with the important distinction that it is deterministic across different computing environments.

**MRT**:

MRT (written in Python), short for Model Representation Tool, is one of the most important parts of CVM-Runtime. It quantizes and prepares your model for on-chain execution by CVM-Runtime. At the risk of oversimplification, MRT basically compresses and converts your model from floating point to integer-only so that its execution is efficient and deterministic across different devices. The original research was endorsed by the official team of Amazon MXNet and the details can be read [here](https://medium.com/apache-mxnet/quantizing-neural-network-models-in-mxnet-for-strict-consistency-on-blockchain-b5c950674866).

In this tutorial, we will write a simple handwritten digit recognition model, convert it via MRT, and upload it to the Cortex blockchain so that it can be executed on-chain.

We will walk through 4 main stages:

(1) Install CVM-Runtime and other dependencies.

(2) Train a model using the MXNet framework.

(3) Quantize the model trained in stage 2 using MRT.

(4) Upload the model

**Prerequisites:**

- A machine with the Linux Operating System <sup>[1]</sup>. The official implementation is currently only in Linux, but if you're passionate about implementing CVM-Runtime in a different operating system, pull requests are more than welcomed!

- A machine with GPU and CUDA installed properly for your Linux version.

- Working knowledge of Linux programming.

- Willingness to debug for yourself. While we seek to be as comprehensive as possible in this tutorial, we cannot cover all the idiosyncrasies of different machines and environments. If you notice an issue in the documentation, feel free to open Github issues or pull requests.

If you encounter any problems during the course, feel free to reach out to our core dev team via our [Telegram](https://t.me/CortexOfficialEN) or [Twitter](https://twitter.com/CTXCBlockchain/). We seek to constantly improve our documentation.

Let's get started!

# Tutorial

## Stage I: Install CVM-runtime and Other Dependencies

### **Install CVM-Runtime**

### 1. Git clone the CVM-runtime repository

```bash
git clone -b wlt https://github.com/CortexFoundation/cvm-runtime.git

cd cvm-runtime
```

### 2. Configure for compilation

In `config.cmake`, set the `ENABLE_CUDA` variable to `ON` on line 6. This allows us to utilize the GPU.

![config](imgs/config.png)

### 3. Compile

You may need to install `make` if you have not already.

```bash
make -j8 lib
```

If you encounter this error below

![cmake](imgs/cmake.png)

you should install `cmake`. Try the following commands:

```bash
sudo apt-get update
sudo apt-get install cmake
```

Now type `g++` in your shell to see if you have `g++` installed. If not, run `sudo apt-get install g++`

You might need to switch to a machine with GPU and CUDA installed if you don't have one.

### **Install MXNET & MRT dependencies**

Now go to [MXNET's website](https://mxnet.apache.org/get_started/?platform=linux&language=python&processor=gpu&environ=pip&) to
install the GPU version of MXNET suited for your CUDA. The install command should look something like

```bash
pip3 install mxnet-cu102
```

Run `$nvcc --version` to find out your CUDA version; make sure that it matches the number after "cu", in this case it is general version 10.

Now run the commands below to install other dependencies needed for our model training and quantization later.

```bash
pip3 install gluoncv

make dep
```

# Stage II: Train Your Model

Execute the following command:

```bash
python3 tests/mrt/train_mnist.py
```

This Python program will train a handwritten digit model using MXNet. It will take a few minutes to run. Upon completion, the trained model is stored under `~/mrt_model` (you can alter this path in the `python/mrt/conf.py` file).

If you don't care about the training process, proceed to the next stage. Otherwise, let's dig a bit into the model training process here. We only sketch the main ideas here - always refer back to the source code `tests/mrt/train_mnist.py` when you're confused.

Now go ahead and open the `train_mnist.py` file. There are mostly 5 steps: (1) load the data (2) define the model architecture (3) randomly initialize the parameters & set the hyperparameters (4) start training (5) export the model

### Step 1: Load the Data

```python
# load training and validation data
train_data = mx.gluon.data.vision.MNIST(
        train=True).transform_first(data_xform)
val_data = mx.gluon.data.vision.MNIST(
        train=False).transform_first(data_xform)

batch_size = 4
train_loader = mx.gluon.data.DataLoader(train_data, shuffle=True, batch_size=batch_size)
val_loader = mx.gluon.data.DataLoader(val_data, shuffle=False, batch_size=batch_size)
```

### Step 2: Define the Model

Our main interest here is the `train_mnist()` function. Notice by default, our code provides 3 architectures that you can choose from (`dapp`, `lenet`, `mlp`). If you do not specify the architecture, the default is `dapp`.

```python
if version == 'dapp':
    net = nn.HybridSequential(prefix='DApp_')
    with net.name_scope():
        net.add(
                nn.Conv2D(channels=16, kernel_size=(3, 3), activation='relu'),
                nn.MaxPool2D(pool_size=(2, 2), strides=(1, 1)),
                nn.Conv2D(channels=32, kernel_size=(3, 3), activation='relu'),
                nn.MaxPool2D(pool_size=(2, 2), strides=(1, 1)),
                nn.Conv2D(channels=64, kernel_size=(3, 3), activation='relu'),
                nn.MaxPool2D(pool_size=(2, 2), strides=(2, 2)),
                nn.Conv2D(channels=128, kernel_size=(1, 1), activation='relu'),
                nn.MaxPool2D(pool_size=(2, 2), strides=(2, 2)),
                nn.Flatten(),
                nn.Dense(10, activation=None),
        )
```

### Step 3: Randomly Initialize the Parameters & Set the Hyperparameters

```python
# Random initialize all the mnist model parameters
net.initialize(mx.init.Xavier(), ctx=ctx)
net.summary(nd.zeros((1, 1, 28, 28), ctx=ctx))
# Set hyperparameters
trainer = gluon.Trainer(
        params=net.collect_params(),
	optimizer='adam',
	optimizer_params={'learning_rate': 1e-3},
        )
metric = mx.metric.Accuracy()
loss_function = gluon.loss.SoftmaxCrossEntropyLoss()
num_epochs = 5
```

### Step 4: Start Training

```python
for epoch in range(num_epochs):
        for inputs, labels in train_loader:
            inputs = inputs.as_in_context(ctx)
            labels = labels.as_in_context(ctx)

            with autograd.record():
                outputs = net(inputs)
                loss = loss_function(outputs, labels)

            loss.backward()
            metric.update(labels, outputs)
            trainer.step(batch_size=inputs.shape[0])

        name, acc = metric.get()
        print('After epoch {}: {} = {:5.2%}'.format(epoch + 1, name, acc))
        metric.reset()

for inputs, labels in val_loader:
        inputs = inputs.as_in_context(ctx)
        labels = labels.as_in_context(ctx)
        metric.update(labels, net(inputs))
print('Validaton: {} = {}'.format(*metric.get()))
assert metric.get()[1] > 0.96
```

### Step 5: Export the Model

```python
sym = net(mx.sym.var('data'))
sym_file, param_file = load_fname(version)
print ("Dump model into ", sym_file, " & ", param_file)

# dump the mxnet model
with open(sym_file, "w") as fout:
        fout.write(sym.tojson())
net.collect_params().save(param_file)
```

Again, here we only show the general structure of writing a program that trains AI models - refer to the original file under `tests/mrt/train_mnist.py` for the complete code and modify as you see fit!

# Stage III: Quantize Your Model

To prepare the model for the Cortex blockchain, we need to quantize it with MRT. Recall from **Introduction** that MRT is a tool originating from Cortex's research in on-chain inference of ML models - it helps us quantize ML models for deterministic inference on the blockchain.

Execute the following command:

```bash
python3 python/mrt/main2.py python/mrt/model_zoo/mnist.ini
```

All the pre-quantized model configuration files are stored in `python/mrt/model_zoo`, and the file `config.example.ini` contains the model configuration for our case.

**Very Important Note:**

If you're training a custom model with your custom dataset, keep in mind that you must alter the source code to allow MRT to read your dataset properly. This is because MRT needs to see your dataset (during calibration) in order to get an idea of the range of activation values for each neural layer - only then can it know how to quantize your model for optimal performance. We're working on a more friendly interface regarding this. Meanwhile, feel free to directly alter the source code yourself for workarounds.

# Stage IV: Upload Your Model

Now the model is fully quantized. Specifically, `mnist_.json` and `mnist_.params` are your quantized models, stored under `~/mrt_model` (assuming you have not altered the path in the `python/mrt/conf.py` file). Create a separate folder named `data` and rename your `mnist_.json` to `symbol` and your `mnist_.params` to `params`. Run `ls -l` to check your operating system is not hiding the file name extension, because the full file names need to be correct for successful upload.

We're now ready to upload them! Like most PoW blockchains, transaction data are broadcast out, received by different full nodes and relayed to propagate the entire network; miners then include these transactions in a block later mined into the blockchain. In our case, model file (which is mostly matrices of weights) is the transaction data; to upload this model file, we broadcast it out by seeding it with a torrent file (which is a file including metadata about the model file that enables its reception and relay by other full nodes).

In this final stage, we will have three main steps:
(1) Notify the Cortex network about the model and generate a torrent file on Cerebro.
(2) Seed/broadcast this torrent
(3) Push the upload progress on Cerebro.

### Step 1: Notify the Network & Generate Torrent

Let's first go the [TestNet Cerebro Explorer](https://cerebro.test.cortexlabs.ai/) to generate a torrent file for our model file, which we will later use to broadcast (upload) the model file to the network. (When you deploy to MainNet, you need to go to the MainNet [Cerebro Explorer](https://cerebro.cortexlabs.ai/))

In the menu bar at the top, find "upload" under "AI Contract"

![cerebroMenu](imgs/cerebroMenu.png)

Then choose `Fixed Point Model Data` - recall that our MRT has converted our ML model from float-point to (integer-only) fixed-point for deterministic on-chain inference.

![fixedPoint](imgs/fixedPoint.png)

Now select the two model files simultaneously.

![select](imgs/select.png)

![upload](imgs/upload.png)

Change the input shape in the grey box `Shape`. (Note: even if your input is of shape 1,28,28 you need to re-type it anyway, and that's what you should do here if you've been following this tutorial.)

Now make sure that you have signed into your TestNet wallet and don't have any other transaction confirmation windows open (otherwise the new transaction confirmation window will not pop out).

Type in the model description (any texts you wish), click `save` and then hit `upload`. A new transaction confirmation window will pop out. Click `confirm`.

We're not done yet - recall that we're only generating a torrent file here, which includes metadata about the model files that enable their broadcast to other full nodes.

### Step 2: Seed/Broadcast This Torrent

To finish the upload, we now proceed to seed (upload/broadcast) the model files along with the torrent file (the model files' metadata). In this example, we will use [torrentf](https://github.com/CortexFoundation/torrentfs), a client based on the Anacrolix BitTorrent client.

After confirming the upload transaction on Cerebro above, a torrent file is generated and downloaded. The file name should be something like `3145ad19228c1cd2d051314e72f26c1ce77b7f02.torrent`. The alphanumeric string is called `infohash` - make sure you save this `infohash` and then rename the file to just `torrent` (make sure your operating system is not hiding the file name extension; i.e. make sure the file name is not `torrent.torrent`)

Next, we create a new folder named `3145ad19228c1cd2d051314e72f26c1ce77b7f02` (same as the infohash) in which we put the `torrent` file alongside the `data` folder containing the two model files. Once that is done, create a new directory called `test` and put everything into it.

The resulting directory should have this structure:

![seed](imgs/seed.png)

Now, if you don't have Go installed already, install Go first. We recommend this [guide](https://www.linode.com/docs/development/go/install-go-on-ubuntu/), but make sure to install the latest stable version.

Now run

```bash
git clone https://github.com/CortexFoundation/torrentfs.git
cd torrentfs
make
```

Once you finish running the commands above, torrentfs should be compiled. Now we will go back to our root directory and run

```bash
torrentfs/build/bin/seeding -dataDir=test
```

Note that this command above is assuming that you have both `torrentfs` and `test` folders in the root directory. If you have a different directory structure, make sure to adjust the command accordingly.

Once you run this command, the seeding should start and there will be a message telling you that it is seeding.

Let it seed/broadcast for ~6 hours to ensure the file propagate to the entire network.

### Step 3: Push the Upload Progress on Cerebro.

To prevent spams, the Cortex protocol requires uploaders of large model files to manually push the upload progress by consuming Endorphin (similar to Gas on Ethereum). We need to go to the transaction page to manually send a few more transactions - otherwise, the full nodes will ignore our seed as spam instead of receiving or relaying it.

To do this, let's go to the transaction page of the uploaded model. Open your wallet and find the transaction in which you uploaded the model.

![wallet-transaction](imgs/wallet-transaction.png)

Click on it to go to the transaction page.

![model-transaction](imgs/model-transaction.png)

On the row "To", you will see "Model XXXXXXXX Created". Click on the XXXXXXXX, which will take you to a new page where you can see:

![push-progress](imgs/push-progress.png)

Click on `send` to push the progress. Keep sending transactions until the upload is complete, at which point you should see:

![push-progress-success](imgs/push-progress-success.png)

Only then, will the full nodes stop ignoring your model files as spam and start receiving or relaying your files.

Now we're done! It's strongly recommended that you keep seeding for ~6 hours for the model to successfully propagate throughout the entire network.

# Conclusion

Voila! You have successfully trained a handwritten digit recognition model and uploaded it to the Cortex blockchain. Now anyone, anywhere in the world can see and use your model. Each time your model is called, you will get a small reward in CTXC (this amount comes from a share of the miner's award). Note that this amount will be added to the balance of the address from which you deployed the model, but usually the reward for one single call to your model is small, so you will need to adjust the unit to Turing to be able to see it.

If you want to learn how to call your model from a smart contract, we have a [tutorial](https://github.com/CortexFoundation/tech-doc/blob/master/hello-world-tutorial-contract.md) for that too.

Happy building!
<br />
<br/>

# FAQ

1. Relationship between CVM-Runtime, CVM and MRT?

CVM is the virtual machine that executes code on the Cortex blockchain. CVM-Runtime, if you recall from earlier, is a machine learning framework responsible for on-chain execution of _ML models_. You can consider CVM-Runtime part of the CVM; in fact, it is perhaps the biggest innovation of CVM - before CVM, there was no known way to execute ML models on-chain. MRT is the part of CVM-Runtime that prepares regular float-point ML models for on-chain execution via quantization. You can learn more details from this tree graph [here](https://github.com/CortexFoundation/tech-doc/blob/master/cortex-guide.md).

# Footnotes

[1] If you don't have access to a Linux system locally, you can open a Linux EC2 Instance on AWS and connect it to your editor via ssh. There will be a more detailed bonus tutorial on how to set this up - for now, here are the general steps to setting up a Linux system on AWS. Play around with it and use Google if you get stuck. You're always welcomed to ask questions in our Telegram group too!

> (1) Go to AWS console, set up an EC20 Ubuntu (Ubuntu is one of the most user-friendly Linux systems) Instance.

> (2) Start a folder named `.ssh`, put in your key pair and start a text file named `config`

> (3) Open the command palette in Visual Studio code (`command + P` in Mac), type in
> `> Remote-SSH: Connect to Host`.
> Then choose `Add New SSH Host`.

> Type in your address `ubuntu@your-aws-instance-public-ip`

> Substitute in your own public ip here. Note that each time you restart your instance, the public ip changes, so you need to reconfigure upon each restart.

> Type in the absolute path to your configuration file. Your path should end with something like `/.ssh/config`
