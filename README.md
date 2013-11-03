Student: Carlos Sotelo
UID: 303891983

Student: Tania DePasquale
UID: 704018998

Ok didn't see you say "don't do too much" till it was too late...
anyways
Got this consistently passing 14+ test cases
Biggest thing we need to figure out I think is a way to test dead lock (and fix those flaky test cases)
we can't have more than 1 write lock right? or am I wrong?


Here be the deal:

This project consists of relatively easy things exercises with really obscure functions that you
would otherwise have to hunt down. I have implemented the read/write funtionality (which isnt much), and it passes the first eight test cases as a result. Heres what you need to know:

linux/blkdev.h is your friend. It defines 'struct request' and a lot of other stuff used throughout the project.

rq_data_dir() takes in a struct request and returns READ or WRITE depending on whether the request is a read or write request.

req->sector gives you the starting sector number the request is dealing with. req->current_nr_sectors tells you how many sectors your request will be dealing with.

ticket_tail, local_ticket, and ticket_head are used to guarnatee a queuing, FIFO model. 
ticket_tail and ticket_head are shared, while each thread has a local_ticket. 

We should do the following: Initially set the ticket_tail and ticket_head to 0. Whenever a process enters the queue, set its ticket number to the current ticket_head, then increment the ticket_head. Whenever a ticket number is equal to the ticket_tail, it means we currently have or can have the lock. Whenever a process releases the lock, increment ticket_tail so that the next process's ticket number is equal to ticket tail.

wait_event_interruptible() takes in a waitqueue and a condition. returns 0 when successful, otherwise it returns ERESTARTSYS if interrupted by a signal. When called by a thread, it makes that thread wait in the queue until the given condition is true.

Code written in discussion:

Possible implementation for Acquring a lock (blocks/hangs)

bool condition = d->wlock_count == 0
		&& (!filp_writeable || d->rlock_count == 0)
		&& d->ticket_tail == local_ticket;

if(!wait_event_interruptible(waitqueue,condition)){
	//grant the lock here
	if( flip_writeable )
		d->wlock_count++;
	else
		d->rlock_count++;
		d->ticket_tail++;
	}
}



Possible implementation for trying to Acquire a lock (no hang/returns immedately)

if(d->wlock_count != 0
|| (d->ticket_tail != local_ticket)
|| (filp_writeable && d->rlock_count != 0))
	r = -EBUSY;
else
//obtain lock here
//...
//...



Checking for deadlock:

//..
//..
os_spin_lock(&d->mutex);
	d->writing=0;
	d->reading=0;
	for_each_open_file(current,check_locks,d);
	if(current->pid == d->lockpid)
	{
		if((d->writing && !filp_writeable)
		|| (d->reading && filp_writeable))
		{
			osp_spin_unlock(&d->mutex);
			return -EDEADLOCK;
		}
	}
//..
//..

void checklocks( struct file * filp, osprd_info_t *d )
{
	if(filp->f_mode & FMODE_WRITE)
		d->writing =1;
	else
		d->reading =1;
}



