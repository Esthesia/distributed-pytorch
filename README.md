# Distributed training with PyTorch
## Based on the Publication of HAL
Kindratenko, Volodymyr, Dawei Mu, Yan Zhan, John Maloney, Sayed Hadi Hashemi, Benjamin Rabe, Ke Xu, Roy Campbell, Jian Peng, and William Gropp. 
"HAL: Computer System for Scalable Deep Learning." In Practice and Experience in Advanced Research Computing, pp. 41-48. 2020. https://doi.org/10.1145/3311790.3396649.

## Requirements

- Python 3
- CUDA >= 9.0
- Install PyTorch ([pytorch.org](http://pytorch.org)) with GPU, version >= 1.0
- Download the ImageNet dataset and move validation images to labeled subfolders
    - To do this, you can use the following script: https://raw.githubusercontent.com/soumith/imagenetloader.torch/master/valprep.sh
    - on HAL cluster, use `/home/shared/imagenet/raw/`
- Install NVIDIA `Apex` (not required for PyTorch > 1.6.0)
    ```
    $ git clone https://github.com/NVIDIA/apex
    $ cd apex
    $ pip install -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" ./
    ```

## Non-distributed (ND) training

Use cases:
- Single node single GPU training

This is the most basic training method, no data parallel at all

```bash
python imagnet_nd.py -a resnet50 /home/shared/imagenet/raw/
```

The default learning rate schedule starts at 0.1 and decays by a factor of 10 every 30 epochs. This is appropriate for ResNet and models with batch normalization, but too high for AlexNet and VGG. Use 0.01 as the initial learning rate for AlexNet or VGG:

```bash
python imagnet_nd.py -a alexnet --lr 0.01 /home/shared/imagenet/raw/
```

## Single-processing Data Parallel (DP)

Use cases:
- Single node multi-GPU training

References:

- https://pytorch.org/tutorials/beginner/blitz/data_parallel_tutorial.html

`torch.nn.DataParallel` is easier to use (just wrap the model and run your training script). However, because it uses one process to compute the model weights and then distribute them to each GPU on the current node during each batch, networking quickly becomes a bottle-neck and GPU utilization is often very low. Furthermore, it requires that all the GPUs be on the same node and doesn’t work with `Apex` for mixed-precision training.

## Multi-processing Distributed Data Parallel (DDP)

Use cases:
- Single node multi-GPU training
- Multi-node multi-GPU training

`torch.nn.DistributedDataParallel` is the recommeded way of doing distributed training in PyTorch. It is proven to be significantly faster than `torch.nn.DataParallel` for single-node multi-GPU data parallel training. `nccl` backend is currently the fastest and highly recommended backend to be used with distributed training and this applies to both single-node and multi-node distributed training.

Multiprocessing with DistributedDataParallel duplicates the model on each GPU on each compute node. The GPUs can all be on the same node or spread across multiple nodes. If you have 2 computer nodes with 4 GPUs each, you have a total of 8 model replicas. Each replica is controlled by one process and handles a portion of the input data.  Every process does identical tasks, and each process communicates with all the others. During the backwards pass, gradients from each node are averaged. Only gradients are passed between the processes/GPUs so that network communication is less of a bottleneck.

During training, each process loads its own minibatches from disk and passes them to its GPU. Each GPU does its own forward pass, and then the gradients are all-reduced across the GPUs. Gradients for each layer do not depend on previous layers, so the gradient all-reduce is calculated concurrently with the backwards pass to futher alleviate the networking bottleneck. At the end of the backwards pass, every node has the averaged gradients, ensuring that the model weights stay synchronized.

All this requires that the multiple processes, possibly on multiple nodes, are synchronized and communicate. Pytorch does this through its `distributed.init_process_group` function. This function needs to know where to find process 0 so that all the processes can sync up and the total number of processes to expect. Each individual process also needs to know the total number of processes as well as its rank within the processes and which GPU to use. It’s common to call the total number of processes the `world size`. Finally, each process needs to know which slice of the data to work on so that the batches are non-overlapping. Pytorch provides `nn.utils.data.DistributedSampler` to accomplish this.


### Single node, multiple GPUs:

```bash
python  imagenet_ddp.py -a resnet50 --dist-url 'tcp://MASTER_IP:MASTER_PORT' --dist-backend 'nccl' --world-size 1 --rank 0 --desired-acc 0.75 /home/shared/imagenet/raw/
```

### Multiple nodes, multiple GPUs:

To run your programe on 4 nodes with 4 GPU each, you will need to open 4 terminals and run slightly different command on each node.

Node 0:
```bash
python  imagenet_ddp.py -a resnet50 --dist-url 'tcp://MASTER_IP:MASTER_PORT' --dist-backend 'nccl' --world-size 4 --rank 0 --desired-acc 0.75 /home/shared/imagenet/raw/
```

- `MASTER_IP`: IP address for the master node of your choice
- `MASTER_PORT`: open port number on the master node. if you don't know, use `8888`
- `--world-size`: equals the # of compute nodes you are using
- `--rank`: rank of the current node, should be an int between `0` and `--world-size - 1`
- `--desired-acc`: desired accuracy to stop training
- `--workers`: # of data loading workers for the current node. this is different from the processes that run the programe on each GPU. the total # of processes = # of data loading workers + # of GPUs (one process to run each GPU)

Node 1:
```bash
python  imagenet_ddp.py -a resnet50 --dist-url 'tcp://MASTER_IP:MASTER_PORT' --dist-backend 'nccl' --world-size 4 --rank 1 --desired-acc 0.75 /home/shared/imagenet/raw/
```

Node 2:
```bash
python  imagenet_ddp.py -a resnet50 --dist-url 'tcp://MASTER_IP:MASTER_PORT' --dist-backend 'nccl' --world-size 4 --rank 2 --desired-acc 0.75 /home/shared/imagenet/raw/
```

Node 3:
```bash
python  imagenet_ddp.py -a resnet50 --dist-url 'tcp://MASTER_IP:MASTER_PORT' --dist-backend 'nccl' --world-size 4 --rank 3 --desired-acc 0.75 /home/shared/imagenet/raw/
```

