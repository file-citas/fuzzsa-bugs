# Overview
In this document we describe the modifications, that we applied to driver code in order to aid the analysis.
The modifications fall into two categories:
* Patches to fix bugs that were identified during research
* Fuzzing optimizations (such as removing checks on checksum comparisons, that are difficult to  solve for a fuzzer)

## virtio_blk
### Patches
1. [low severity] Avoid reachable assertion:
```
@@ -228,7 +230,13 @@ static blk_status_t virtio_queue_rq(struct blk_mq_hw_ctx *hctx,
   bool unmap = false;
   u32 type;
 
-  BUG_ON(req->nr_phys_segments + 2 > vblk->sg_elems);
+   if(lkl_ops->fuzz_ops->apply_patch()) {                                                                                              
+      if(req->nr_phys_segments + 2 > vblk->sg_elems) {
+         return BLK_STS_IOERR;
+      }
+   } else {
+      BUG_ON(req->nr_phys_segments + 2 > vblk->sg_elems);
+   }

```
2. [high severity] Handle invalid number of virtqueues to avoid null pointer dereference:
```
@@ -442,6 +450,7 @@ static void virtblk_update_capacity(struct virtio_blk *vblk, bool resize)
   struct request_queue *q = vblk->disk->queue;
   char cap_str_2[10], cap_str_10[10];
   unsigned long long nblocks;
   u64 capacity;
 
   /* Host must always specify the capacity. */
@@ -506,6 +515,11 @@ static int init_vq(struct virtio_blk *vblk)
      num_vqs = 1;
 
   num_vqs = min_t(unsigned int, nr_cpu_ids, num_vqs);
+   if(lkl_ops->fuzz_ops->apply_patch_2()) {
+      if(num_vqs<1) {
+         num_vqs=1;
+      }
+   }
 

```
