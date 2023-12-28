# iostat

```
root@yingnzhang-dev-vm-8jv9d:/# strace iostat -x
...
openat(AT_FDCWD, "/sys/block/vdb/stat", O_RDONLY) = 4
...
```
Document in kernel
https://github.corp.ebay.com/tess-contrib/jammy/blob/Ubuntu-5.15.0-26.26-ebay/Documentation/admin-guide/iostats.rst
```
the same information is found in two places: one is in the file /proc/diskstats, and the other is within the sysfs file system, which must be mounted in order to obtain the information. Throughout this document we'll assume that sysfs is mounted on /sys, although of course it may be mounted anywhere. Both /proc/diskstats and sysfs use the same source for the information and so should not differ.
```

The stat in `/proc/diskstats`
```
root@yingnzhang-dev-vm-8jv9d:/sys/block/vda# cat /proc/diskstats
   7       0 loop0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       1 loop1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       2 loop2 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       3 loop3 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       4 loop4 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       5 loop5 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       6 loop6 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   7       7 loop7 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 252       0 vda 30178 763 1045081 5976 83413647 2295326 794897363 12553541 0 9255144 12990640 0 0 0 0 2330370 431122
 252       1 vda1 117 0 936 9 0 0 0 0 0 32 9 0 0 0 0 0 0
 252       2 vda2 1993 0 16968 116 44 0 4646 9 0 224 125 0 0 0 0 0 0
 252       3 vda3 27967 763 1022769 5835 83413603 2295326 794892717 12553531 0 9255000 12559367 0 0 0 0 0 0
 252      16 vdb 540085 2255 27219110 131609 4507052552 50703686 30964668507 935307761 0 4157488264 1146075965 0 0 0 0 1834355632 210636594
 252      32 vdc 343 134 13742 25 0 0 0 0 0 68 25 0 0 0 0 0 0
 253       0 dm-0 130447 0 7528122 29364 87006818 0 878964784 9019692 0 9377600 9049056 0 0 0 0 0 0
 253       1 dm-1 439908 0 20696944 107120 4554660939 0 32711447668 905938356 0 4162378980 906045476 0 0 0 0 0 0
```

In block/genhd.c, it register show function `diskstats_show` to proc entry "diskstats". So the columns in `/proc/diskstats` are corresponding to the fields printed by this show function.

```
/*
 * aggregate disk stat collector.  Uses the same stats that the sysfs
 * entries do, above, but makes them available through one seq_file.
 *
 * The output looks suspiciously like /proc/partitions with a bunch of
 * extra fields.
 */
static int diskstats_show(struct seq_file *seqf, void *v)
{
	struct gendisk *gp = v;
	struct block_device *hd;
	unsigned int inflight;
	struct disk_stats stat;
	unsigned long idx;

	/*
	if (&disk_to_dev(gp)->kobj.entry == block_class.devices.next)
		seq_puts(seqf,	"major minor name"
				"     rio rmerge rsect ruse wio wmerge "
				"wsect wuse running use aveq"
				"\n\n");
	*/

	rcu_read_lock();
	xa_for_each(&gp->part_tbl, idx, hd) {
		if (bdev_is_partition(hd) && !bdev_nr_sectors(hd))
			continue;
		part_stat_read_all(hd, &stat);
		if (queue_is_mq(gp->queue))
			inflight = blk_mq_in_flight(gp->queue, hd);
		else
			inflight = part_in_flight(hd);

		seq_printf(seqf, "%4d %7d %pg "
			   "%lu %lu %lu %u "
			   "%lu %lu %lu %u "
			   "%u %u %u "
			   "%lu %lu %lu %u "
			   "%lu %u"
			   "\n",
			   MAJOR(hd->bd_dev), MINOR(hd->bd_dev), hd,
			   stat.ios[STAT_READ],
			   stat.merges[STAT_READ],
			   stat.sectors[STAT_READ],
			   (unsigned int)div_u64(stat.nsecs[STAT_READ],
							NSEC_PER_MSEC),
			   stat.ios[STAT_WRITE],
			   stat.merges[STAT_WRITE],
			   stat.sectors[STAT_WRITE],
			   (unsigned int)div_u64(stat.nsecs[STAT_WRITE],
							NSEC_PER_MSEC),
			   inflight,
			   jiffies_to_msecs(stat.io_ticks),
			   (unsigned int)div_u64(stat.nsecs[STAT_READ] +
						 stat.nsecs[STAT_WRITE] +
						 stat.nsecs[STAT_DISCARD] +
						 stat.nsecs[STAT_FLUSH],
							NSEC_PER_MSEC),
			   stat.ios[STAT_DISCARD],
			   stat.merges[STAT_DISCARD],
			   stat.sectors[STAT_DISCARD],
			   (unsigned int)div_u64(stat.nsecs[STAT_DISCARD],
						 NSEC_PER_MSEC),
			   stat.ios[STAT_FLUSH],
			   (unsigned int)div_u64(stat.nsecs[STAT_FLUSH],
						 NSEC_PER_MSEC)
			);
	}
	rcu_read_unlock();

	return 0;
}

static const struct seq_operations diskstats_op = {
	.start	= disk_seqf_start,
	.next	= disk_seqf_next,
	.stop	= disk_seqf_stop,
	.show	= diskstats_show
};

static int __init proc_genhd_init(void)
{
	proc_create_seq("diskstats", 0, NULL, &diskstats_op);
	proc_create_seq("partitions", 0, NULL, &partitions_op);
	return 0;
}
module_init(proc_genhd_init);
#endif /* CONFIG_PROC_FS */
```

It use the disk_stats of block_device to collect the diskstats.
```
struct block_device {
	sector_t		bd_start_sect;
==>>>	struct disk_stats __percpu *bd_stats;
	unsigned long		bd_stamp;
...
} __randomize_layout;

struct disk_stats {
	u64 nsecs[NR_STAT_GROUPS];
	unsigned long sectors[NR_STAT_GROUPS];
	unsigned long ios[NR_STAT_GROUPS];
	unsigned long merges[NR_STAT_GROUPS];
	unsigned long io_ticks;
	local_t in_flight[2];
};

enum stat_group {
	STAT_READ,
	STAT_WRITE,
	STAT_DISCARD,
	STAT_FLUSH,

	NR_STAT_GROUPS
};
```
**stat.ios** will be add 1 for each io request completion.

**stat.io_ticks** record the io processing ticks.

**stat.nsecs** record the io requests overall ticks, including queued time.
```code
static void update_io_ticks(struct block_device *part, unsigned long now,
		bool end)
{
	unsigned long stamp;
again:
	stamp = READ_ONCE(part->bd_stamp);
	if (unlikely(time_after(now, stamp))) {
		if (likely(cmpxchg(&part->bd_stamp, stamp, now) == stamp))
			__part_stat_add(part, io_ticks, end ? now - stamp : 1);
	}
	if (part->bd_partno) {
		part = bdev_whole(part);
		goto again;
	}
}

void blk_account_io_done(struct request *req, u64 now)
{
	/*
	 * Account IO completion.  flush_rq isn't accounted as a
	 * normal IO on queueing nor completion.  Accounting the
	 * containing request is enough.
	 */
	if (req->part && blk_do_io_stat(req) &&
	    !(req->rq_flags & RQF_FLUSH_SEQ)) {
		const int sgrp = op_stat_group(req_op(req));

		part_stat_lock();
		update_io_ticks(req->part, jiffies, true);
		part_stat_inc(req->part, ios[sgrp]);
		part_stat_add(req->part, nsecs[sgrp], now - req->start_time_ns);
		part_stat_unlock();
	}
}

void blk_account_io_start(struct request *rq)
{
	if (!blk_do_io_stat(rq))
		return;

	/* passthrough requests can hold bios that do not have ->bi_bdev set */
	if (rq->bio && rq->bio->bi_bdev)
		rq->part = rq->bio->bi_bdev;
	else
		rq->part = rq->rq_disk->part0;

	part_stat_lock();
	update_io_ticks(rq->part, jiffies, false);
	part_stat_unlock();
}
```

Now, back to iostat
```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          10.75    0.00   17.09    0.04    0.04   72.07

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
dm-0             0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
dm-1             0.00      0.00     0.00   0.00    0.00     0.00  324.00    993.00     0.00   0.00    0.21     3.06    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.07  30.00
vda              0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
vdb              0.00      0.00     0.00   0.00    0.00     0.00  324.00    944.50     0.00   0.00    0.16     2.92    0.00      0.00     0.00   0.00    0.00     0.00  131.00    0.11    0.07  30.00
vdc              0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
```


|w/s     |wkB/s   |wrqm/s  |%wrqm |w_await |wareq-sz|f/s |f_await  |aqu-sz  |%util|
|-|-|-|-|-|-|-|-|-|-|
|delta(stat.ios[STAT_WRITE])|delta(512*stat.sectors[STAT_WRITE]/1024)|delta(stat.merges[STAT_WRITE])|stat.merges/stat.ios|div_u64(stat.nsecs[STAT_WRITE],NSEC_PER_MSEC)|(wkB/s) / (w/s)|delta(stat.ios[STAT_FLUSH])|div_u64(stat.nsecs[STAT_FLUSH],NSEC_PER_MSEC)|overall wait time to handle all the IO requests in this second.<dlv> (w_await(ms) * w/s +  f_await * f/s) / 1000ms|delta(stat.io_ticks) / delta(t)|




# iotop
strace iotop
```
sendto(5, [{nlmsg_len=28, nlmsg_type=TASKSTATS, nlmsg_flags=NLM_F_REQUEST, nlmsg_seq=11106, nlmsg_pid=-840669896}, "\x01\x00\x00\x00\x08\x00\x01\x00\x3f\xaa\x39\x00"], 28, 0, NULL, 0) = 28
recvfrom(5, [{nlmsg_len=388, nlmsg_type=TASKSTATS, nlmsg_flags=0, nlmsg_seq=11106, nlmsg_pid=-840669896}, "\x02\x01\x00\x00\x70\x01\x04\x00\x08\x00\x01\x00\x3f\xaa\x39\x00\x64\x01\x03\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"...], 16384, 0, {sa_family=AF_NETLINK, nl_pid=0, nl_groups=00000000}, [12]) = 388
```
It use netlink socket to get "TASKSTATS" from kernel.
