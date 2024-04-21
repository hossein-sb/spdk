## IO Request Transmission CallBack
This document describes how to use callback function for IO Requests that are being transmitted out through the TCP. 
### Problem
In general cases, a BDEV module accepts IO write requests via any of the spdk_bdev_writev_blocks_ext, spdk_bdev_writev_blocks_with_md, \
spdk_bdev_writev_blocks, spdk_bdev_writev, spdk_bdev_write_blocks_with_md, spdk_bdev_write_blocks and spdk_bdev_write functions. \
The mentioned functions accept a callback function that will be called in case the IO request is completed. In a module like BDEV NVME TCP (in client mode), \
the IO request will be completed whenever PDU success is being received by the module. This means each IO waits TCP RTT(Round Trip Time) to get completed. \
In some specific scenario, in order to reduce the latency, they can rely on sending their IO request out through the TCP protocol and no need \
for waiting a PDU success response message. 
### Solution
A new function spdk_bdev_augment_write_cb_by_ontransmit_cb allows the BDEV Module users to pass additional callback function to the BDEV Modules. \
In order to be called once the BDEV module sent out the request. This function is only beneficial on BDEV Modules that supports this feature \
like NVME module, otherwise the transmission callback will be called at the same time when IO request completion callback will be called.
### Example
The following is an example of a normal case when sending an IO request to a BDEV module while passing a write request to a it in which \
nvmf_bdev_ctrlr_complete_cmd is a callback function and req is its argument.
```cpp
rc = spdk_bdev_writev_blocks_ext(desc, ch, req->iov, req->iovcnt, start_lba, num_blocks,
					 nvmf_bdev_ctrlr_complete_cmd, req, &opts);
```

The following exmple shows how to set a callback function named io_transmit_done and its argument alongside the IO Request Completion callback \
that had been used in the previous example. In this example, nvmf_bdev_ctrlr_complete_cmd is the IO completion callback and req is its argument. \
io_transmit_done is the transmission callback and transmit_arg is its argument. The spdk_bdev_augment_write_cb_by_ontransmit_cb function accepts \
both of the callback functions as its arguments and prepare another callback function and argument which need to be pass to the \
spdk_bdev_writev_blocks_ext function.
```cpp
spdk_bdev_io_completion_cb augmented_cb;
void *augmented_cb_arg;
spdk_bdev_augment_write_cb_by_ontransmit_cb(nvmf_bdev_ctrlr_complete_cmd, req, io_transmit_done, transmit_arg, &augmented_cb, &augmented_cb_arg);
rc = spdk_bdev_writev_blocks_ext(desc, ch, req->iov, req->iovcnt, start_lba, num_blocks,
				 augmented_cb, augmented_cb_arg, &opts);
```