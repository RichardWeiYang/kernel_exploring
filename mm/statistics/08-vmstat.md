文件/proc/vmstat统计了内存子系统使用的详细信息。

其中大部分单位是页面数，少数是次数。

# 代码

读取的代码在mm/vmstat.c，主要函数是vmstat_start。这个函数有意思，所有要查的信息可以通过vmstat_text[]这个数组看到。

这个数组长度是NR_VMSTAT_ITEMS。

```
#define NR_VMSTAT_ITEMS (NR_VM_ZONE_STAT_ITEMS + \
			 NR_VM_NUMA_EVENT_ITEMS + \
			 NR_VM_NODE_STAT_ITEMS + \
			 NR_VM_STAT_ITEMS + \
			 (IS_ENABLED(CONFIG_VM_EVENT_COUNTERS) ? \
			  NR_VM_EVENT_ITEMS : 0))
```

所以可以看出，这里面要展示的有zone, numa, node, event等信息。

因为填写的时候也是按照这个顺序填写的，所以查看的时候也就知道是按照这个顺序去看。

# 格式

```
nr_free_pages 862440
nr_zone_inactive_anon 0
nr_zone_active_anon 388096
nr_zone_inactive_file 916401
nr_zone_active_file 462578
nr_zone_unevictable 0
nr_zone_write_pending 91
nr_mlock 0
nr_bounce 0
nr_zspages 0
nr_free_cma 0
nr_unaccepted 0
numa_hit 5052791
numa_miss 0
numa_foreign 0
numa_interleave 33
numa_local 5052791
numa_other 0
nr_inactive_anon 0
nr_active_anon 388096
nr_inactive_file 916401
nr_active_file 462578
nr_unevictable 0
nr_slab_reclaimable 69229
nr_slab_unreclaimable 40412
nr_isolated_anon 0
nr_isolated_file 0
workingset_nodes 0
workingset_refault_anon 0
workingset_refault_file 0
workingset_activate_anon 0
workingset_activate_file 0
workingset_restore_anon 0
workingset_restore_file 0
workingset_nodereclaim 0
nr_anon_pages 399281
nr_mapped 162117
nr_file_pages 1367810
nr_dirty 91
nr_writeback 0
nr_writeback_temp 0
nr_shmem 24573
nr_shmem_hugepages 0
nr_shmem_pmdmapped 0
nr_file_hugepages 0
nr_file_pmdmapped 0
nr_anon_transparent_hugepages 0
nr_vmscan_write 0
nr_vmscan_immediate_reclaim 0
nr_dirtied 430596
nr_written 277543
nr_throttled_written 0
nr_kernel_misc_reclaimable 0
nr_foll_pin_acquired 0
nr_foll_pin_released 0
nr_kernel_stack 14560
nr_page_table_pages 7349
nr_sec_page_table_pages 0
nr_swapcached 0
pgpromote_success 0
pgpromote_candidate 0
pgdemote_kswapd 0
pgdemote_direct 0
pgdemote_khugepaged 0
nr_dirty_threshold 440872
nr_dirty_background_threshold 220166
pgpgin 4739895
pgpgout 1138773
pswpin 0
pswpout 0
pgalloc_dma 512
pgalloc_dma32 147553
pgalloc_normal 5054184
pgalloc_movable 0
pgalloc_device 0
allocstall_dma 0
allocstall_dma32 0
allocstall_normal 0
allocstall_movable 0
allocstall_device 0
pgskip_dma 0
pgskip_dma32 0
pgskip_normal 0
pgskip_movable 0
pgskip_device 0
pgfree 6082476
pgactivate 0
pgdeactivate 0
pglazyfree 39709
pgfault 4875818
pgmajfault 18102
pglazyfreed 0
pgrefill 0
pgreuse 362153
pgsteal_kswapd 0
pgsteal_direct 0
pgsteal_khugepaged 0
pgscan_kswapd 0
pgscan_direct 0
pgscan_khugepaged 0
pgscan_direct_throttle 0
pgscan_anon 0
pgscan_file 0
pgsteal_anon 0
pgsteal_file 0
zone_reclaim_failed 0
pginodesteal 0
slabs_scanned 0
kswapd_inodesteal 0
kswapd_low_wmark_hit_quickly 0
kswapd_high_wmark_hit_quickly 0
pageoutrun 0
pgrotated 0
drop_pagecache 0
drop_slab 0
oom_kill 0
numa_pte_updates 0
numa_huge_pte_updates 0
numa_hint_faults 0
numa_hint_faults_local 0
numa_pages_migrated 0
pgmigrate_success 0
pgmigrate_fail 0
thp_migration_success 0
thp_migration_fail 0
thp_migration_split 0
compact_migrate_scanned 0
compact_free_scanned 0
compact_isolated 0
compact_stall 0
compact_fail 0
compact_success 0
compact_daemon_wake 0
compact_daemon_migrate_scanned 0
compact_daemon_free_scanned 0
htlb_buddy_alloc_success 0
htlb_buddy_alloc_fail 0
unevictable_pgs_culled 60995
unevictable_pgs_scanned 0
unevictable_pgs_rescued 28
unevictable_pgs_mlocked 28
unevictable_pgs_munlocked 28
unevictable_pgs_cleared 0
unevictable_pgs_stranded 0
thp_fault_alloc 0
thp_fault_fallback 0
thp_fault_fallback_charge 0
thp_collapse_alloc 0
thp_collapse_alloc_failed 0
thp_file_alloc 0
thp_file_fallback 0
thp_file_fallback_charge 0
thp_file_mapped 0
thp_split_page 0
thp_split_page_failed 0
thp_deferred_split_page 0
thp_split_pmd 0
thp_scan_exceed_none_pte 0
thp_scan_exceed_swap_pte 0
thp_scan_exceed_share_pte 0
thp_split_pud 0
thp_zero_page_alloc 0
thp_zero_page_alloc_failed 0
thp_swpout 0
thp_swpout_fallback 0
balloon_inflate 0
balloon_deflate 0
balloon_migrate 0
swap_ra 0
swap_ra_hit 0
ksm_swpin_copy 0
cow_ksm 0
zswpin 0
zswpout 0
zswpwb 0
direct_map_level2_splits 51
direct_map_level3_splits 0
nr_unstable 0
```

# 使用
