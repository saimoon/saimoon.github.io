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

* **realtek rtl8169** (in *drivers/net/ethernet/realtek/r8169.c*)

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

We are interested in `.probe` function, used by kernel to initialize the device.<br>
For RTL, the *.probe* function is *rtl_init_one*.
This func make alot of work to initialize the device.
It's interesting to evidence the function *netif_napi_add*:

{% highlight c %}
static int rtl_init_one(struct pci_dev *pdev, const struct pci_device_id *ent)
{
...
netif_napi_add(dev, &tp->napi, rtl8169_poll, R8169_NAPI_WEIGHT);
...
{% endhighlight %}

here it is initialized a core struct of NAPI system: *struct napi_struct* (*tp->napi*);
this struct contains device specific parameters, fundamental afterwards to consume packet;
there is an instance of *struct napi_struct* for each device ring queue (and so one for each irq),
and each instance contains a *poll* function (*rtl8169_poll*) that will be responsible to process
incoming packets.
*netif_napi_add()* register *poll* function inside the *struct napi_struct* linking also a weight (more about it soon).

Always in our previous *.probe* function, *struct net_device* is initialized,
and all device operation *struct net_device_ops* are registered:

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
...
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


Now, when the device is activated (*ifconfig <dev> up*), the *.ndo_open* callback (*rtl_open*) is called.
The *.ndo_open* callback function do the following:

* Create Rx ring buffer

{% highlight c %}
#define NUM_RX_DESC	256U						/* Number of Rx descriptor registers */
#define R8169_RX_RING_BYTES	(NUM_RX_DESC * sizeof(struct RxDesc))

struct RxDesc {
	__le32 opts1;		// 4-byte
	__le32 opts2;		// 4-byte
	__le64 addr;		// 8-byte (contains network packet buffer address)
};

struct rtl8169_private {
	struct RxDesc *RxDescArray;				/* 256-aligned Rx descriptor ring */
	dma_addr_t RxPhyAddr;
	void *Rx_databuff[NUM_RX_DESC];		/* Rx data buffers */
	...
}

static int rtl_open(struct net_device *dev)
{
...
	// alloc 256 x 16byte rx desc (==> 256 x rx desc slot) = Rx-Descriptor ring-buffer
	tp->RxDescArray = dma_alloc_coherent(&pdev->dev, R8169_RX_RING_BYTES, &tp->RxPhyAddr, GFP_KERNEL);

	retval = rtl8169_init_ring(dev); // which call rtl8169_rx_fill()
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
- 256 network packet buffer (each 16383 byte)
- an Rx ring buffer (*RxDescArray*) with 256 slot
		each slot point to the physic address of a packet buffer
- an array of 256 pointer to packet buffer (virtual address)

  RxDescArray                                              Rx_databuff
-----------------             --------------              -------------
| slot 0:  addr |---PhyAddr-> | 16383 byte | <--VirtAddr--| array 0   |
-----------------             --------------              -------------
| slot 1:  addr |---PhyAddr-> | 16383 byte | <--VirtAddr--| array 1   |
-----------------             --------------              -------------
.......................................................................
-----------------             --------------              -------------
| slot 255:addr |---PhyAddr-> | 16383 byte | <--VirtAddr--| array 255 |
-----------------             --------------              -------------