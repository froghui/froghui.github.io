---
layout: post
title:  "docker背后的存储之device mapper(三)：dm-thin设计实现"
date:   2016-02-12 20:33:25
categories: linux
tags: linux docker
---

我们知道，docker在centOS上背后使用的存储技术是device mapper。本文将从device mapper实现的细节来讨论device mapper在docker中是如何使用的。

虽然C并不是直接支持面向对象的编程，但是通过C的struct以及复杂的函数指针，还是可以“曲折”的实现面向对象的编程。下图显示了device mapper中persistent data中的主要结构(类)和成员。
![](/assets/2016-01-21-device-mapper/persistent_data.svg)

#1.dm_space_map
基础类dm_space_map定义了一系列接口，用来对metadata和data分区的space map相应管理。例如new_block就是向metadata或者data分区申请一块可以使用的block；get_count统计某个block被引用的个数；inc_block和dec_block用来通知某个block的引用计数增加和减少；extend告诉space map需要扩展管理的blocks的个数是多少。

```
/*
 * struct dm_space_map keeps a record of how many times each block in a device
 * is referenced.  It needs to be fixed on disk as part of the transaction.
 */
struct dm_space_map {
    void (*destroy)(struct dm_space_map *sm);

    /*
     * You must commit before allocating the newly added space.
     */
    int (*extend)(struct dm_space_map *sm, dm_block_t extra_blocks);

    /*
     * Extensions do not appear in this count until after commit has
     * been called.
     */
    int (*get_nr_blocks)(struct dm_space_map *sm, dm_block_t *count);

    /*
     * Space maps must never allocate a block from the previous
     * transaction, in case we need to rollback.  This complicates the
     * semantics of get_nr_free(), it should return the number of blocks
     * that are available for allocation _now_.  For instance you may
     * have blocks with a zero reference count that will not be
     * available for allocation until after the next commit.
     */
    int (*get_nr_free)(struct dm_space_map *sm, dm_block_t *count);

    int (*get_count)(struct dm_space_map *sm, dm_block_t b, uint32_t *result);
    int (*count_is_more_than_one)(struct dm_space_map *sm, dm_block_t b,
                      int *result);
    int (*set_count)(struct dm_space_map *sm, dm_block_t b, uint32_t count);

    int (*commit)(struct dm_space_map *sm);

    int (*inc_block)(struct dm_space_map *sm, dm_block_t b);
    int (*dec_block)(struct dm_space_map *sm, dm_block_t b);

    /*
     * new_block will increment the returned block.
     */
    int (*new_block)(struct dm_space_map *sm, dm_block_t *b);

    /*
     * The root contains all the information needed to fix the space map.
     * Generally this info is small, so squirrel it away in a disk block
     * along with other info.
     */
    int (*root_size)(struct dm_space_map *sm, size_t *result);
    int (*copy_root)(struct dm_space_map *sm, void *copy_to_here_le, size_t len);

    /*
     * You can register one threshold callback which is edge-triggered
     * when the free space in the space map drops below the threshold.
     */
    int (*register_threshold_callback)(struct dm_space_map *sm,
                       dm_block_t threshold,
                       dm_sm_threshold_fn fn,
                       void *context);
};
```

通过实现以上的函数，并且使用指针，在C代码中实现了C++面向对象的技术。
sm_bootstrap_ops主要用来对metadata block进行初始化分配，并不需要对index bitmap进行查找。 

```C
/*
 * When a new space map is created that manages its own space.  We use
 * this tiny bootstrap allocator.
 */
static void sm_bootstrap_destroy(struct dm_space_map *sm)
{
}

static int sm_bootstrap_extend(struct dm_space_map *sm, dm_block_t extra_blocks)
{
    DMERR("bootstrap doesn't support extend");

    return -EINVAL;
}

static int sm_bootstrap_get_nr_blocks(struct dm_space_map *sm, dm_block_t *count)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    return smm->ll.nr_blocks;
}

static int sm_bootstrap_get_nr_free(struct dm_space_map *sm, dm_block_t *count)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    *count = smm->ll.nr_blocks - smm->begin;

    return 0;
}

static int sm_bootstrap_get_count(struct dm_space_map *sm, dm_block_t b,
                  uint32_t *result)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    return b < smm->begin ? 1 : 0;
}

static int sm_bootstrap_count_is_more_than_one(struct dm_space_map *sm,
                           dm_block_t b, int *result)
{
    *result = 0;

    return 0;
}

static int sm_bootstrap_set_count(struct dm_space_map *sm, dm_block_t b,
                  uint32_t count)
{
    DMERR("bootstrap doesn't support set_count");

    return -EINVAL;
}

static int sm_bootstrap_new_block(struct dm_space_map *sm, dm_block_t *b)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    /*
     * We know the entire device is unused.
     */
    if (smm->begin == smm->ll.nr_blocks)
        return -ENOSPC;

    *b = smm->begin++;

    return 0;
}

static int sm_bootstrap_inc_block(struct dm_space_map *sm, dm_block_t b)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    return add_bop(smm, BOP_INC, b);
}

static int sm_bootstrap_dec_block(struct dm_space_map *sm, dm_block_t b)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    return add_bop(smm, BOP_DEC, b);
}

static int sm_bootstrap_commit(struct dm_space_map *sm)
{
    return 0;
}

static int sm_bootstrap_root_size(struct dm_space_map *sm, size_t *result)
{
    DMERR("bootstrap doesn't support root_size");

    return -EINVAL;
}

static int sm_bootstrap_copy_root(struct dm_space_map *sm, void *where,
                  size_t max)
{
    DMERR("bootstrap doesn't support copy_root");

    return -EINVAL;
}
```

sm_metadata_ops对metadata block进行管理，需要用到index bitmap。

```C
static void sm_metadata_destroy(struct dm_space_map *sm)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    kfree(smm);
}

static int sm_metadata_get_nr_blocks(struct dm_space_map *sm, dm_block_t *count)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    *count = smm->ll.nr_blocks;

    return 0;
}

static int sm_metadata_get_nr_free(struct dm_space_map *sm, dm_block_t *count)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    *count = smm->old_ll.nr_blocks - smm->old_ll.nr_allocated -
         smm->allocated_this_transaction;

    return 0;
}

static int sm_metadata_get_count(struct dm_space_map *sm, dm_block_t b,
                 uint32_t *result)
{
    int r;
    unsigned i;
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);
    unsigned adjustment = 0;

    /*
     * We may have some uncommitted adjustments to add.  This list
     * should always be really short.
     */
    for (i = smm->uncommitted.begin;
         i != smm->uncommitted.end;
         i = brb_next(&smm->uncommitted, i)) {
        struct block_op *op = smm->uncommitted.bops + i;

        if (op->block != b)
            continue;

        switch (op->type) {
        case BOP_INC:
            adjustment++;
            break;

        case BOP_DEC:
            adjustment--;
            break;
        }
    }

    r = sm_ll_lookup(&smm->ll, b, result);
    if (r)
        return r;

    *result += adjustment;

    return 0;
}

// b is the block no, for example, bitmap_root no 1
static int sm_metadata_count_is_more_than_one(struct dm_space_map *sm,
                          dm_block_t b, int *result)
{
    int r, adjustment = 0;
    unsigned i;
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);
    uint32_t rc;

    /*
     * We may have some uncommitted adjustments to add.  This list
     * should always be really short.
     */
    for (i = smm->uncommitted.begin;
         i != smm->uncommitted.end;
         i = brb_next(&smm->uncommitted, i)) {

        struct block_op *op = smm->uncommitted.bops + i;

        if (op->block != b)
            continue;

        switch (op->type) {
        case BOP_INC:
            adjustment++;
            break;

        case BOP_DEC:
            adjustment--;
            break;
        }
    }

    if (adjustment > 1) {
        *result = 1;
        return 0;
    }

    r = sm_ll_lookup_bitmap(&smm->ll, b, &rc);
    if (r)
        return r;

    if (rc == 3)
        /*
         * We err on the side of caution, and always return true.
         */
        *result = 1;
    else
        *result = rc + adjustment > 1;

    return 0;
}

static int sm_metadata_set_count(struct dm_space_map *sm, dm_block_t b,
                 uint32_t count)
{
    int r, r2;
    enum allocation_event ev;
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    if (smm->recursion_count) {
        DMERR("cannot recurse set_count()");
        return -EINVAL;
    }

    in(smm);
    r = sm_ll_insert(&smm->ll, b, count, &ev);
    r2 = out(smm);

    return combine_errors(r, r2);
}

static int sm_metadata_inc_block(struct dm_space_map *sm, dm_block_t b)
{
    int r, r2 = 0;
    enum allocation_event ev;
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    if (recursing(smm))
        r = add_bop(smm, BOP_INC, b);
    else {
        in(smm);
        r = sm_ll_inc(&smm->ll, b, &ev);
        r2 = out(smm);
    }

    return combine_errors(r, r2);
}

static int sm_metadata_dec_block(struct dm_space_map *sm, dm_block_t b)
{
    int r, r2 = 0;
    enum allocation_event ev;
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    if (recursing(smm))
        r = add_bop(smm, BOP_DEC, b);
    else {
        in(smm);
        r = sm_ll_dec(&smm->ll, b, &ev);
        r2 = out(smm);
    }

    return combine_errors(r, r2);
}

static int sm_metadata_new_block_(struct dm_space_map *sm, dm_block_t *b)
{
    int r, r2 = 0;
    enum allocation_event ev;
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    // find one block who is not used
    r = sm_ll_find_free_block(&smm->old_ll, smm->begin, smm->old_ll.nr_blocks, b);
    if (r)
        return r;

    smm->begin = *b + 1;


    if (recursing(smm))
        r = add_bop(smm, BOP_INC, *b);
    else {
        in(smm);
        r = sm_ll_inc(&smm->ll, *b, &ev);
        r2 = out(smm);
    }

    if (!r)
        smm->allocated_this_transaction++;

    return combine_errors(r, r2);
}

static int sm_metadata_new_block(struct dm_space_map *sm, dm_block_t *b)
{
    dm_block_t count;
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    int r = sm_metadata_new_block_(sm, b);
    if (r) {
        DMERR_LIMIT("unable to allocate new metadata block");
        return r;
    }

    r = sm_metadata_get_nr_free(sm, &count);
    if (r) {
        DMERR_LIMIT("couldn't get free block count");
        return r;
    }

    check_threshold(&smm->threshold, count);

    return r;
}

// call metadata_commit to refresh the bitmap root block
static int sm_metadata_commit(struct dm_space_map *sm)
{
    int r;
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    r = sm_ll_commit(&smm->ll);
    if (r)
        return r;

    // why make the begin from 0 ??
    memcpy(&smm->old_ll, &smm->ll, sizeof(smm->old_ll));
    smm->begin = 0;
    smm->allocated_this_transaction = 0;

    return 0;
}

static int sm_metadata_register_threshold_callback(struct dm_space_map *sm,
                           dm_block_t threshold,
                           dm_sm_threshold_fn fn,
                           void *context)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    set_threshold(&smm->threshold, threshold, fn, context);

    return 0;
}

static int sm_metadata_root_size(struct dm_space_map *sm, size_t *result)
{
    *result = sizeof(struct disk_sm_root);

    return 0;
}

static int sm_metadata_copy_root(struct dm_space_map *sm, void *where_le, size_t max)
{
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);
    struct disk_sm_root root_le;

    root_le.nr_blocks = cpu_to_le64(smm->ll.nr_blocks);
    root_le.nr_allocated = cpu_to_le64(smm->ll.nr_allocated);
    root_le.bitmap_root = cpu_to_le64(smm->ll.bitmap_root);
    root_le.ref_count_root = cpu_to_le64(smm->ll.ref_count_root);

    if (max < sizeof(root_le))
        return -ENOSPC;

    memcpy(where_le, &root_le, sizeof(root_le));

    return 0;
}

static int sm_metadata_extend(struct dm_space_map *sm, dm_block_t extra_blocks);
```

sm_disk_ops对data space map进行管理。使用了btree来进行bitmap index block的管理，而不是sm_metadata_ops里面的静态bitmap index block数组。

```C
static void sm_disk_destroy(struct dm_space_map *sm)
{
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);

    kfree(smd);
}

static int sm_disk_extend(struct dm_space_map *sm, dm_block_t extra_blocks)
{
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);

    return sm_ll_extend(&smd->ll, extra_blocks);
}

static int sm_disk_get_nr_blocks(struct dm_space_map *sm, dm_block_t *count)
{
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);
    *count = smd->old_ll.nr_blocks;

    return 0;
}

static int sm_disk_get_nr_free(struct dm_space_map *sm, dm_block_t *count)
{
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);
    *count = (smd->old_ll.nr_blocks - smd->old_ll.nr_allocated) - smd->nr_allocated_this_transaction;

    return 0;
}

static int sm_disk_get_count(struct dm_space_map *sm, dm_block_t b,
                 uint32_t *result)
{
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);
    return sm_ll_lookup(&smd->ll, b, result);
}

static int sm_disk_count_is_more_than_one(struct dm_space_map *sm, dm_block_t b,
                      int *result)
{
    int r;
    uint32_t count;

    r = sm_disk_get_count(sm, b, &count);
    if (r)
        return r;

    return count > 1;
}

static int sm_disk_set_count(struct dm_space_map *sm, dm_block_t b,
                 uint32_t count)
{
    int r;
    uint32_t old_count;
    enum allocation_event ev;
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);

    r = sm_ll_insert(&smd->ll, b, count, &ev);
    if (!r) {
        switch (ev) {
        case SM_NONE:
            break;

        case SM_ALLOC:
            /*
             * This _must_ be free in the prior transaction
             * otherwise we've lost atomicity.
             */
            smd->nr_allocated_this_transaction++;
            break;

        case SM_FREE:
            /*
             * It's only free if it's also free in the last
             * transaction.
             */
            r = sm_ll_lookup(&smd->old_ll, b, &old_count);
            if (r)
                return r;

            if (!old_count)
                smd->nr_allocated_this_transaction--;
            break;
        }
    }

    return r;
}

static int sm_disk_inc_block(struct dm_space_map *sm, dm_block_t b)
{
    int r;
    enum allocation_event ev;
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);

    r = sm_ll_inc(&smd->ll, b, &ev);
    if (!r && (ev == SM_ALLOC))
        /*
         * This _must_ be free in the prior transaction
         * otherwise we've lost atomicity.
         */
        smd->nr_allocated_this_transaction++;

    return r;
}

static int sm_disk_dec_block(struct dm_space_map *sm, dm_block_t b)
{
    enum allocation_event ev;
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);

    return sm_ll_dec(&smd->ll, b, &ev);
}

static int sm_disk_new_block(struct dm_space_map *sm, dm_block_t *b)
{
    int r;
    enum allocation_event ev;
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);

    /* FIXME: we should loop round a couple of times */
    r = sm_ll_find_free_block(&smd->old_ll, smd->begin, smd->old_ll.nr_blocks, b);
    if (r)
        return r;

    smd->begin = *b + 1;
    r = sm_ll_inc(&smd->ll, *b, &ev);
    if (!r) {
        BUG_ON(ev != SM_ALLOC);
        smd->nr_allocated_this_transaction++;
    }

    return r;
}

static int sm_disk_commit(struct dm_space_map *sm)
{
    int r;
    dm_block_t nr_free;
    struct sm_disk *smd = container_of(sm, struct sm_disk, sm);

    // nr_free shoule be 0
    r = sm_disk_get_nr_free(sm, &nr_free);
    if (r)
        return r;

    // call ll_commit to refresh the tow btree nodes
    r = sm_ll_commit(&smd->ll);
    if (r)
        return r;

    // now we can copy ll into old_ll, but why ??
    memcpy(&smd->old_ll, &smd->ll, sizeof(smd->old_ll));
    smd->begin = 0;
    smd->nr_allocated_this_transaction = 0;

    // nr_free shoule be 0 again
    r = sm_disk_get_nr_free(sm, &nr_free);
    if (r)
        return r;

    return 0;
}
```

#2.disk_ll

基础接口disk_ll定义了一系列函数，用来对metadata和data分区的bitmap index block相应管理。例如load_ie_fn就是从metadata磁盘加载一个顺序号为index的bitmap index block, save_ie_fn就是将顺序号为index的bitmap index block存储到metadata磁盘上。
```
typedef int (*load_ie_fn)(struct ll_disk *ll, dm_block_t index, struct disk_index_entry *result);
typedef int (*save_ie_fn)(struct ll_disk *ll, dm_block_t index, struct disk_index_entry *ie);
typedef int (*init_index_fn)(struct ll_disk *ll);
typedef int (*open_index_fn)(struct ll_disk *ll);
typedef dm_block_t (*max_index_entries_fn)(struct ll_disk *ll);
typedef int (*commit_fn)(struct ll_disk *ll);
```

metadata_ll是对metadata space以bitmap index数组为管理目标的实现。

```C
// get the disk_metadata_index from memory
static int metadata_ll_load_ie(struct ll_disk *ll, dm_block_t index,
                   struct disk_index_entry *ie)
{
    memcpy(ie, ll->mi_le.index + index, sizeof(*ie));
    return 0;
}

// save the disk_metadata_index into memory
static int metadata_ll_save_ie(struct ll_disk *ll, dm_block_t index,
                   struct disk_index_entry *ie)
{
    ll->bitmap_index_changed = true;
    memcpy(ll->mi_le.index + index, ie, sizeof(*ie));
    return 0;
}

// this is used to create the bitmap_root
static int metadata_ll_init_index(struct ll_disk *ll)
{
    int r;
    struct dm_block *b;
    
    // call sm_bootstrap_ops to get the block number (it's 1) and begin++
    r = dm_tm_new_block(ll->tm, &index_validator, &b);
    if (r < 0)
        return r;

    // copy disk_metadata_index into block data (4K)
    memcpy(dm_block_data(b), &ll->mi_le, sizeof(ll->mi_le));
    // now bitmap_root points to the block_nr 1
    ll->bitmap_root = dm_block_location(b);

    return dm_tm_unlock(ll->tm, b);
}

static int metadata_ll_open(struct ll_disk *ll)
{
    int r;
    struct dm_block *block;

    r = dm_tm_read_lock(ll->tm, ll->bitmap_root,
                &index_validator, &block);
    if (r)
        return r;

    memcpy(&ll->mi_le, dm_block_data(block), sizeof(ll->mi_le));
    return dm_tm_unlock(ll->tm, block);
}

static dm_block_t metadata_ll_max_entries(struct ll_disk *ll)
{
    return MAX_METADATA_BITMAPS;
}

// refresh the bitmap root block, the data of bitmap root block comes from metadata_ll disck_metadata_index
static int metadata_ll_commit(struct ll_disk *ll)
{
    int r, inc;
    struct dm_block *b;

    r = dm_tm_shadow_block(ll->tm, ll->bitmap_root, &index_validator, &b, &inc);
    if (r)
        return r;

    // copy the metadata_index_data into the block (4K)
    memcpy(dm_block_data(b), &ll->mi_le, sizeof(ll->mi_le));
    // now bitmap_root points to the new block
    ll->bitmap_root = dm_block_location(b);

    return dm_tm_unlock(ll->tm, b);
}
```


disk_ll是对data space以btree为管理目标的实现。

```C
static int disk_ll_load_ie(struct ll_disk *ll, dm_block_t index,
               struct disk_index_entry *ie)
{
    return dm_btree_lookup(&ll->bitmap_info, ll->bitmap_root, &index, ie);
}

static int disk_ll_save_ie(struct ll_disk *ll, dm_block_t index,
               struct disk_index_entry *ie)
{
    __dm_bless_for_disk(ie);
    return dm_btree_insert(&ll->bitmap_info, ll->bitmap_root,
                   &index, ie, &ll->bitmap_root);
}

static int disk_ll_init_index(struct ll_disk *ll)
{
    return dm_btree_empty(&ll->bitmap_info, &ll->bitmap_root);
}

static int disk_ll_open(struct ll_disk *ll)
{
    /* nothing to do */
    return 0;
}

static dm_block_t disk_ll_max_entries(struct ll_disk *ll)
{
    return -1ULL;
}

static int disk_ll_commit(struct ll_disk *ll)
{
    return 0;
}

````


#3.metadata space map的初始化

```C
// superblock is 0, nr_blocks is the total block numbers (block_size is 4096)
int dm_sm_metadata_create(struct dm_space_map *sm,
              struct dm_transaction_manager *tm,
              dm_block_t nr_blocks,
              dm_block_t superblock)
{
    int r;
    dm_block_t i;
    enum allocation_event ev;
    struct sm_metadata *smm = container_of(sm, struct sm_metadata, sm);

    // begin is from 1
    smm->begin = superblock + 1;
    smm->recursion_count = 0;
    smm->allocated_this_transaction = 0;
    brb_init(&smm->uncommitted);
    threshold_init(&smm->threshold);

    memcpy(&smm->sm, &bootstrap_ops, sizeof(smm->sm));

    // below is using sm_bootstrap_opts
    r = sm_ll_new_metadata(&smm->ll, tm);
    if (r)
        return r;

    if (nr_blocks > DM_SM_METADATA_MAX_BLOCKS)
        nr_blocks = DM_SM_METADATA_MAX_BLOCKS;
    r = sm_ll_extend(&smm->ll, nr_blocks);
    if (r)
        return r;

    // from now on will use sm_metadata_opts
   
    memcpy(&smm->sm, &ops, sizeof(smm->sm));

    /*
     * Now we need to update the newly created data structures with the
     * allocated blocks that they were built from.
     */
    for (i = superblock; !r && i < smm->begin; i++)
        r = sm_ll_inc(&smm->ll, i, &ev);

    if (r)
        return r;

    return sm_metadata_commit(sm);
}
```
