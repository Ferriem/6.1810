## Lab net

- **Target**: complete `e1000_transmit()` and `e1000_recv()`, both in `kernel/e1000.c`, so that the driver can transmit and receive packets.

Just follow the hints can easily complete the lab. You should look at *kernel/e1000_dev.h* and *kernel/net.h* to roughly get the idea of functions.

`e1000_tramsmit`

​		`regs[E1000_TDT]` stores the place of next packet.

```
int
e1000_transmit(struct mbuf *m)
{
  acquire(&e1000_lock);
  
  //check if the the ring is overflowing.
  if(!(tx_ring[regs[E1000_TDT]].status & E1000_TXD_STAT_DD)){
    release(&e1000_lock);
    return -1;
  }
  
	//use `mbuffree()` to free the last mbuf
  if(tx_mbufs[regs[E1000_TDT]]){
    mbuffree(tx_mbufs[regs[E1000_TDT]]);
    tx_mbufs[regs[E1000_TDT]] = 0;
  }
  
  //fill in the descriptor: m->head, m->len, set the necessary cmd flags.
  tx_ring[regs[E1000_TDT]].addr = (uint64) m->head;
  tx_ring[regs[E1000_TDT]].length = m->len;
  tx_ring[regs[E1000_TDT]].cmd = E1000_TXD_CMD_RS | E1000_TXD_CMD_EOP;
  tx_mbufs[regs[E1000_TDT]] = m;
  
  //update the ring position by adding one to E1000_TDT modulo TX_RING_SIZE.
  regs[E1000_TDT] = (regs[E1000_TDT] + 1) % TX_RING_SIZE;
  release(&e1000_lock);
  return 0;
}
```

`e1000_recv`

```
static void
e1000_recv(void)
{
  while(1){
  	//get the index of next received packet.
    int index = regs[E1000_RDT];
    index = (index + 1) % RX_RING_SIZE;
    
    //check if a new packet is avaliable.
    if(!(rx_ring[index].status & E1000_RXD_STAT_DD))
      break;
      
    //update the mbuf's m->len
    //deliver the mbuf to the network stack using net_rx().
    struct mbuf *m = rx_mbufs[index];
    m->len = rx_ring[index].length;
    net_rx(m);
    
    //allocate a new mbuf using mbufalloc() to replace
    //program its data pointer (m->head) into the descriptor
    //clear the descriptor's status bits to zero
    m = mbufalloc(0);
    rx_ring[index].addr = (uint64) m->head;
    rx_ring[index].status = 0x00;
    rx_mbufs[index] = m;
    
    //update the E1000_RDT register
    regs[E1000_RDT] = index;
  }
}
```

```sh
prompt > make grade
...
== Test   nettest: ping ==
  nettest: ping: OK
== Test   nettest: single process ==
  nettest: single process: OK
== Test   nettest: multi-process ==
  nettest: multi-process: OK
== Test   nettest: DNS ==
  nettest: DNS: FAIL
    ...
         testing multi-process pings: OK
         testing DNS
         DNS arecord for pdos.csail.mit.edu. is 128.52.129.126
         invalid name for EDNS
         $ qemu-system-riscv64: terminating on signal 15 from pid 21929 (<unknown process>)
    MISSING '^DNS OK$'
```