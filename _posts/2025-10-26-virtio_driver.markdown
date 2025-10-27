---
layout: post
title:  "virtio driver"
date:   2025-10-26 01:39:55 +0100
categories: jekyll update
---


This post is a short sketch about a virtIO disk driver. All the relevant code is in the file virtio_disk.c. Virtio is a virtual disk (not super precise wording but i'll stick with it) used by qemu. In memory it's mapped starting from the address 0x10001000 (VIRTIO0 in memlayout.h). Virtio is divided into sectors of 512 bytes each. In xv6 we use blocks which are twice as long (BSIZE 1024). So for example a block indexed 0 will map to 2 sectors (with indices 0 and 1) etc etc etc.

The driver code has these important functions
- init (check and init the device, negotiate features, initialize the descriptors queue)
- virtio_disk_rw (for write/read to the disk by means of writing descriptors in the queue)
- virtio_disk_intr (handles an interrupt from virtio once a request to write/read has been processed)

below some code (from virtio_disk.c and virtio.h) to serve as a reference


{% highlight c %}

////////////////////////////////////////////////////////////////////////////////////////////////////
//struct buf {
//  int valid;   // has data been read from disk?
//  int disk;    // does disk "own" buf?
//  uint dev;
//  uint blockno;
//  struct sleeplock lock;
//  uint refcnt;
//  struct buf *prev; // LRU cache list
//  struct buf *next;
//  uchar data[BSIZE];
//};


////////////////////////////////////////////////////////////////////////////////////////////////////
//struct virtq_used_elem {
//  uint32 id;   // index of start of completed descriptor chain
//  uint32 len;
//};


static struct disk {
////////////////////////////////////////////////////////////////////////////////////////////////////
  // a set (not a ring) of DMA descriptors, with which the
  // driver tells the device where to read and write individual
  // disk operations. there are NUM descriptors.
  // most commands consist of a "chain" (a linked list) of a couple of
  // these descriptors.

//struct virtq_desc {
//  uint64 addr;
//  uint32 len;
//  uint16 flags;
//  uint16 next;
//};
  struct virtq_desc *desc;


////////////////////////////////////////////////////////////////////////////////////////////////////
  // a ring in which the driver writes descriptor numbers
  // that the driver would like the device to process. it only
  // includes the head descriptor of each chain. the ring has
  // NUM elements.

//struct virtq_avail {
//  uint16 flags; // always zero
//  uint16 idx;   // driver will write ring[idx] next
//  uint16 ring[NUM]; // descriptor numbers of chain heads
//  uint16 unused;
//};
  struct virtq_avail *avail;


////////////////////////////////////////////////////////////////////////////////////////////////////
  // a ring in which the device writes descriptor numbers that
  // the device has finished processing (just the head of each chain).
  // there are NUM used ring entries.

//struct virtq_used {
//  uint16 flags; // always zero
//  uint16 idx;   // device increments when it adds a ring[] entry
//  struct virtq_used_elem ring[NUM];
//};
  struct virtq_used *used;


  // our own book-keeping.
  char free[NUM];  // is a descriptor free?
  uint16 used_idx; // we've looked this far in used[2..NUM].

  // track info about in-flight operations,
  // for use when completion interrupt arrives.
  // indexed by first descriptor index of chain.
  struct {
    struct buf *b;
    char status;
  } info[NUM];


////////////////////////////////////////////////////////////////////////////////////////////////////
  // disk command headers.
  // one-for-one with descriptors, for convenience.

//struct virtio_blk_req {
//  uint32 type; // VIRTIO_BLK_T_IN or ..._OUT
//  uint32 reserved;
//  uint64 sector;
//};
  struct virtio_blk_req ops[NUM];
 

  struct spinlock vdisk_lock;
} disk;
{% endhighlight %}


Let's have a look at the functions rw (for read/write) and intr (called when interrupt from the device is raised when the device finished processing a request). Below some things (e.g. __sync_synchronize) from the original code have been removed so it's clear and easy to read (hope there's enough comments and self-descriptive names).


{% highlight c %}

void virtio_disk_rw(struct buf *b, int write)
{
  // xv6 operates on blocks (1024 bytes long) but virtio understands sectors (512 bytes long)
  // it's OK as long as block size is a multiple of sector size
  uint64 sector = b->blockno * (BSIZE / 512);

  // 3 descriptors: 1 for type/reserved/sector, 1 for the data, 1 for a 1-byte status result.
  int idx[3];
  alloc3_desc(idx); // first alloc and then below fill descriptors


  // --- idx 0 -----------------------------------------------------------------------------------------------------------------
  // disk.ops
  if(write) disk.ops[idx[0]].type = VIRTIO_BLK_T_OUT;     // write the disk
  else      disk.ops[idx[0]].type = VIRTIO_BLK_T_IN;      // read the disk
  disk.ops[idx[0]].reserved = 0;
  disk.ops[idx[0]]sector = sector;                        // sector number where to write/read

  // disk.desc
  disk.desc[idx[0]].addr = (uint64) (&disk.ops[idx[0]]);  // point to the stuff populated above 
  disk.desc[idx[0]].len = sizeof(struct virtio_blk_req);  // size of that stuff
  disk.desc[idx[0]].flags = VRING_DESC_F_NEXT;            // there's next descriptor 
  disk.desc[idx[0]].next = idx[1];                        // here's the next descriptor index


  // --- idx 1 -----------------------------------------------------------------------------------------------------------------
  // disk.desc
  disk.desc[idx[1]].addr = (uint64) b->data;              // buffer with data to write to disk or for the data from disk  
  disk.desc[idx[1]].len = BSIZE;                          // size of the buffer
  if(write) disk.desc[idx[1]].flags = 0;                  // device reads b->data
  else      disk.desc[idx[1]].flags = VRING_DESC_F_WRITE; // device writes b->data
  disk.desc[idx[1]].flags |= VRING_DESC_F_NEXT;           // there's next descriptor
  disk.desc[idx[1]].next = idx[2];                        // here's the next descriptor index


  // --- idx 2 -----------------------------------------------------------------------------------------------------------------
  // disk.info
  disk.info[idx[0]].status = 0xff;                        // device writes 0 on success. this is actually never checked later. guess ideally (in a real os) should be
  b->disk = 1;                                            // virtio_disk_intr() is expected to change this to 0
  disk.info[idx[0]].b = b;

  // disk.desc
  disk.desc[idx[2]].addr = (uint64) &disk.info[idx[0]].status; // status set above. on success it becomes 0.
  disk.desc[idx[2]].len = 1;                                   // status is a char so size is 1 byte
  disk.desc[idx[2]].flags = VRING_DESC_F_WRITE;                // device writes the status
  disk.desc[idx[2]].next = 0;                                  // no more descriptors



  // tell the device the first index in our chain of descriptors.
  disk.avail->ring[disk.avail->idx % NUM] = idx[0];

  // tell the device another avail ring entry is available.
  disk.avail->idx += 1; // not % NUM ...

  *R(VIRTIO_MMIO_QUEUE_NOTIFY) = 0; // value is queue number

  // Wait for virtio_disk_intr() to say request has finished.
  while(b->disk == 1) {
    sleep(b, &disk.vdisk_lock);
  }

  disk.info[idx[0]].b = 0;
  free_chain(idx[0]);

  release(&disk.vdisk_lock);
}


void virtio_disk_intr()
{
  acquire(&disk.vdisk_lock);

  *R(VIRTIO_MMIO_INTERRUPT_ACK) = *R(VIRTIO_MMIO_INTERRUPT_STATUS) & 0x3; // tell the device we've seen this interrupt

  // we might have more than one processed request, hence we need a loop used->idx is maintained by device (incremented when new entry
  // added to the used ring), used_idx is managed by us. so loop till they're equal 
  while(disk.used_idx != disk.used->idx){
    int id = disk.used->ring[disk.used_idx % NUM].id;

    struct buf *b = disk.info[id].b;
    b->disk = 0;   // disk is done with buf
    wakeup(b);

    disk.used_idx += 1;
  }

  release(&disk.vdisk_lock);
}

{% endhighlight %}


just to explain the sleep/wakeup mechanism used (could dedicate a post to it) - basically we put a process to sleep (set SLEEPING state) and remember that it's interested in chan (so in our case it's the buf used to write/read). Then later when wakeup is called on that chan, we go through all processes and set RUNNABLE state for all the processes that are interested in the chan. (lock is also involved so there's no surprise).

soooo to summarise the flow is - to read/write to disk populate the descriptors, then notify the device, then go to sleep and then when the device is done with the request, it fires an interrupt which
we handle in virtio_disk_intr which wakes up the process.

some useful links
[https://brennan.io/2020/03/22/sos-block-device/](https://brennan.io/2020/03/22/sos-block-device/)
[https://wiki.osdev.org/Virtio](https://wiki.osdev.org/Virtio)

