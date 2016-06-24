---
published: true
title: Linux Networking: an internal insight
layout: post
---
## Intro

In this post I'll have an insight of the flow of a network packet in current linux OS, from wire to socket.
Tools will be my preferred editor and a pretty recent kernel source code, a 4.6 branch will be fine.
Docs from the net will help alot, and I'll reporting refs at the end of post.

## NAPI theory

[todo]

## Look @ source

In this post I'll use as sample the net card:

* **Realtek rtl8169** (in `drivers/net/ethernet/realtek/r8169.c`)

Look at the `new module_pci_driver()` macro that wraps `module_init()` and `module_exit()` calls, in `drivers/net/ethernet/realtek/r8169.c`:

{% highlight c %}
module_pci_driver(rtl8169_pci_driver);
{% endhighlight %}

The `rtl8169_pci_driver` struct defines functions used by kernel to init/destroy the pci device:

{% highlight c %}
static struct pci_driver rtl8169_pci_driver = {
...
	.probe		= rtl_init_one,
	.remove		= rtl_remove_one,
	.shutdown	= rtl_shutdown,
...
};
{% endhighlight %}

I'm interested in `.probe` function, used by kernel to initialize the device.<br>
For realtek, the `.probe` function is `rtl_init_one()`.<br>

### rtl_init_one()
This function make most of the work of device initialization.<br>
Let start evidencing the function `netif_napi_add()`:

{% highlight c %}
static int rtl_init_one(struct pci_dev *pdev, const struct pci_device_id *ent)
{
...
netif_napi_add(dev, &tp->napi,rtl8169_poll,R8169_NAPI_WEIGHT);
...
{% endhighlight %}

this func initializes a core struct of NAPI system:<br>
 `struct napi_struct` (`tp->napi`);<br>

Here I show the struct:

{% highlight c %}
struct napi_struct {
	/* The poll_list must only be managed by the entity which
	 * changes the state of the NAPI_STATE_SCHED bit.  This means
	 * whoever atomically sets that bit can add this napi_struct
	 * to the per-CPU poll_list, and whoever clears that bit
	 * can remove from the list right before clearing the bit.
	 */
	struct list_head	poll_list;

	unsigned long		state;
	int			weight;
	unsigned int		gro_count;
	int			(*poll)(struct napi_struct *, int);
#ifdef CONFIG_NETPOLL
	spinlock_t		poll_lock;
	int			poll_owner;
#endif
	struct net_device	*dev;
	struct sk_buff		*gro_list;
	struct sk_buff		*skb;
	struct hrtimer		timer;
	struct list_head	dev_list;
	struct hlist_node	napi_hash_node;
	unsigned int		napi_id;
};
{% endhighlight %}

this struct contains device specific parameters, fundamental afterwards to consume packet:<br>
there is an instance of `struct napi_struct` for each device ring queue (and so one for each interrupt),
and each instance contains a *poll* function (`rtl8169_poll`) that will be responsible to process
incoming packets.<br>
`netif_napi_add()` register a *poll* function inside the `struct napi_struct`, initialize
the `poll_list` and a *weight* value (remember those stuffs, we'll be important later).

Continuing in analysis of `rtl_init_one()`, I find the `struct net_device_ops`:

{% highlight c %}
static const struct net_device_ops rtl_netdev_ops = {
	.ndo_open		= rtl_open,
	.ndo_stop		= rtl8169_close,
	.ndo_get_stats64	= rtl8169_get_stats64,
	.ndo_start_xmit		= rtl8169_start_xmit,
	.ndo_tx_timeout		= rtl8169_tx_timeout,
	.ndo_validate_addr	= eth_validate_addr,
	.ndo_change_mtu		= rtl8169_change_mtu,
	.ndo_fix_features	= rtl8169_fix_features,
	.ndo_set_features	= rtl8169_set_features,
	.ndo_set_mac_address	= rtl_set_mac_address,
	.ndo_do_ioctl		= rtl8169_ioctl,
	.ndo_set_rx_mode	= rtl_set_rx_mode,
#ifdef CONFIG_NET_POLL_CONTROLLER
	.ndo_poll_controller	= rtl8169_netpoll,
#endif
};
{% endhighlight %}

This struct contains device operations, callback functions called after some action on the device (like setup, change mac, etc...).<br>
In `rtl_init_one` it is initialized and all device operations registered:

{% highlight c %}
static int rtl_init_one(struct pci_dev *pdev, const struct pci_device_id *ent)
{
...
struct net_device *dev;
dev = alloc_etherdev(sizeof (*tp));
SET_NETDEV_DEV(dev, &pdev->dev);
dev->netdev_ops = &rtl_netdev_ops;
...
rc = register_netdev(dev);
{% endhighlight %}

Now, when the device is activated (using *ifconfig dev up*), the `.ndo_open` callback (`rtl_open()`) is called.

### rtl_open()

The `.ndo_open` callback function is interesting:

* Create Rx ring buffer

The Rx ring buffer is the queue that store network packets received from wire.
They are written directly from NIC using DMA.
If packets from network arrives faster than they are processed, the queue will be filled,
and when full, new packets will be dropped.

{% highlight c %}
#define NUM_RX_DESC	256U  /* Number of Rx descriptor registers */
#define R8169_RX_RING_BYTES	(NUM_RX_DESC * sizeof(struct RxDesc))

struct RxDesc {
  __le32 opts1;   // 4-byte
  __le32 opts2;   // 4-byte
  __le64 addr;    // 8-byte (contains network packet buffer address)
};

struct rtl8169_private {
  struct RxDesc *RxDescArray;  /* 256-aligned Rx descriptor ring */
  dma_addr_t RxPhyAddr;
  void *Rx_databuff[NUM_RX_DESC];  /* Rx data buffers */
...
}

static int rtl_open(struct net_device *dev)
{
...
  // alloc 256 x 16byte rx desc  ==> Rx-Descriptor ring-buffer
  tp->RxDescArray = dma_alloc_coherent(&pdev->dev, R8169_RX_RING_BYTES, &tp->RxPhyAddr, GFP_KERNEL);

  retval = rtl8169_init_ring(dev);  // it calls rtl8169_rx_fill()
...
}

static int rtl8169_rx_fill(struct rtl8169_private *tp)
{
  for (i = 0; i < NUM_RX_DESC; i++) {
...
    data = rtl8169_alloc_rx_data(tp, tp->RxDescArray + i);
...
    // save packet buffer address
    tp->Rx_databuff[i] = data;
  }
...
}

static int rx_buf_sz = 16383;
static struct sk_buff *rtl8169_alloc_rx_data(struct rtl8169_private *tp, struct RxDesc *desc)
{
  void *data;
  dma_addr_t mapping;
...
  // create network packet buffer
  data = kmalloc_node(rx_buf_sz, GFP_KERNEL, node);
...
  // get packet buffer physic addr for driver DMA access
  mapping = dma_map_single(d, rtl8169_align(data), rx_buf_sz, DMA_FROM_DEVICE);
...
  // save packet buffer to Rx-Descriptor slot (done calling rtl8169_map_to_asic())
  desc->addr = cpu_to_le64(mapping);
...
  return data;
}
{% endhighlight %}

So after this init we have:

* 256 network packet buffer (each 16383 byte)
* an Rx ring buffer (*RxDescArray*) with 256 slot, each slot pointing to the physic address of a packet buffer
* an array of 256 pointer to packet buffer (virtual address)

| RxDescArray | | | | Rx_databuff|
|-----------------| |--------------| |-------------|
| slot 0:   |--- PhyAddr --> | 16383 byte | <-- VirtAddr ---| array 0   |
|-----------------| |--------------| |-------------|
| slot 1:   |--- PhyAddr --> | 16383 byte | <-- VirtAddr ---| array 1   |
|-----------------| |--------------| |-------------|
|-----------------| |--------------| |-------------|
| slot 255: |--- PhyAddr --> | 16383 byte | <-- VirtAddr ---| array 255 |
|-----------------| |--------------| |-------------|


* Register interrupt handler

The `request_irq()` function register the irq handler `rtl8169_interrupt` using the irq number obtained earlier from the system.
If MSI interrupts are available they are used, failing back to legacy one if not availables (IRQF_SHARED).
MSI interrupts are better, specially in multi ring queue device,
where each ring queue can have its own IRQ assigned and can be handled by a specific cpu (using irqbalance or smp_affinity).<br>
Here the code for interrupt handler registration:

{% highlight c %}
static int rtl_open(struct net_device *dev)
{
...
  retval = request_irq(pdev->irq, rtl8169_interrupt,
			(tp->features & RTL_FEATURE_MSI) ? 0 : IRQF_SHARED,
			dev->name, dev);
...
}
{% endhighlight %}


* Enable NAPI

Enable NAPI subsystem: it simply clear a bit in the `state` member of `struct napi_struct`.

{% highlight c %}
static int rtl_open(struct net_device *dev)
{
...
  napi_enable(&tp->napi);
...
}
{% endhighlight %}


* Enable interrupts

Finally enable interrupts on the device.
From now incoming packets start to be received.
Here the code:

{% highlight c %}
static int rtl_open(struct net_device *dev)
{
...
  rtl_hw_start(dev);
...
}

static void rtl_hw_start(struct net_device *dev)
{
...
	rtl_irq_enable_all(tp);
}
{% endhighlight %}


* Start NAPI queue

This code starts the NAPI queue:

{% highlight c %}
static int rtl_open(struct net_device *dev)
{
...
  netif_start_queue(dev);
...
}
{% endhighlight %}


### Incoming packets

#### IRQ handler
When a packet arrives, if the receive ring buffer is not full, it is written to
ring buffer using DMA, then IRQ is fired.<br>
The IRQ handler is called:

{% highlight c %}
static irqreturn_t rtl8169_interrupt(int irq, void *dev_instance)
{
...
  rtl_irq_disable(tp);
  napi_schedule(&tp->napi);
...
}
{% endhighlight %}

The IRQ handler is very simple and fast, because when it runs other interrupts are blocked.
The packet processing is not executed in this context but, as we see, it is scheduled to be executed by softirq.
IRQ handler simply disable further NAPI irq, and schedule execution (`napi_schedule`, in `net/core/dev.c`):

{% highlight c %}
void __napi_schedule(struct napi_struct *n)
{
	unsigned long flags;

	local_irq_save(flags);
	____napi_schedule(this_cpu_ptr(&softnet_data), n);
	local_irq_restore(flags);
}

/* Called with irq disabled */
static inline void ____napi_schedule(struct softnet_data *sd,
				     struct napi_struct *napi)
{
	list_add_tail(&napi->poll_list, &sd->poll_list);
	__raise_softirq_irqoff(NET_RX_SOFTIRQ);
}

// from net/core/dev.c
struct softnet_data {
	struct list_head	poll_list;
...
}

{% endhighlight %}

<b>`napi_schedule()` retrive the `struct softnet_data` associated to the current cpu,
add the `napi_struct` associated to irq to the `softnet_data` linked list and raise softirq `NET_RX_SOFTIRQ`;</b><br>
those actions are the core of the NAPI system: the softirq will cycle on `struct softnet_data` and will grab
all `napi_struct` it will find on that list.<br>
It is important to notice the IRQ handler wakes up the NAPI softirq process on the same CPU as the IRQ handler.<br>


#### Softirq handler
