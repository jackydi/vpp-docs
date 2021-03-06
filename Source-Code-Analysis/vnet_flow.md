## Flow

### DPDK流分类

**RSS关键字**
| 数据包类型 | 哈希计算输入 |
| -------- | -------- |
| ipv4-udp | s-ip,d-ip,s-port,d-port |
| ipv4-tcp | s-ip,d-ip,s-port,d-port |
| ipv4-sctp | s-ip,d-ip,s-port,d-port,verification-tag |
| ipv4-other | s-ip,d-ip |
| ipv6-udp | s-ip,d-ip,s-port,d-port |
| ipv6-tcp | s-ip,d-ip,s-port,d-port |
| ipv6-sctp | s-ip,d-ip,s-port,d-port,verification-tag |
| ipv6-other | s-ip,d-ip |

**Flow Director关键字**
| 数据包类型 | 哈希计算输入 |
| -------- | -------- |
| ipv4-udp | s-ip,d-ip,s-port,d-port |
| ipv4-tcp | s-ip,d-ip,s-port,d-port |
| ipv4-sctp | s-ip,d-ip,s-port,d-port,verification-tag |
| ipv4-other | s-ip,d-ip |
| ipv6-udp | s-ip,d-ip,s-port,d-port |
| ipv6-tcp | s-ip,d-ip,s-port,d-port |
| ipv6-sctp | s-ip,d-ip,s-port,d-port,verification-tag |
| ipv6-other | s-ip,d-ip |

### Flow模块支持的流

| 流类型 | 哈希计算输入 |
| -------- | -------- |
| ethernet | s-mac,d-mac,type |
| ipv4-n-tuple | s-ip,d-ip,s-port,d-port,proto |
| ipv4-vxlan | s-sip,d-ip,d-port,vni |
| ipv4-gtpc | s-ip,d-ip,s-port,d-port,proto,teid |
| ipv4-gtpu | s-ip,d-ip,s-port,d-port,proto,teid |
| ipv4-gtpu-ipv4 | s-ip,d-ip,s-port,d-port,proto,teid,in-s-ip,in-d-ip |
| ipv4-gtpu-ipv6 | s-ip,d-ip,s-port,d-port,proto,teid,in-s-ip,in-d-ip |
| ipv6-n-tuple | s-ip,d-ip,s-port,d-port,proto |
| ipv6-vxlan | s-sip,d-ip,d-port,vni |
| ipv6-gtpc | s-ip,d-ip,s-port,d-port,proto,teid |
| ipv6-gtpu | s-ip,d-ip,s-port,d-port,proto,teid |
| ipv6-gtpu-ipv4 | s-ip,d-ip,s-port,d-port,proto,teid,in-s-ip,in-d-ip |
| ipv6-gtpu-ipv6 | s-ip,d-ip,s-port,d-port,proto,teid,in-s-ip,in-d-ip |

### 配置

源文件位于`vnet/flow/flow_cli.c`文件中。

```
VLIB_CLI_COMMAND (test_flow_command, static) = {
    .path = "test flow",
    .short_help = "test flow add [src-ip <ip-addr/mask>] [dst-ip "
      "<ip-addr/mask>] [src-port <port/mask>] [dst-port <port/mask>] "
      "[proto <ip-proto>",
    .function = test_flow,
};
```

```
static clib_error_t *
test_flow (vlib_main_t * vm, unformat_input_t * input,
	   vlib_cli_command_t * cmd_arg)
{
  vnet_flow_t flow;
  vnet_main_t *vnm = vnet_get_main ();
  unformat_input_t _line_input, *line_input = &_line_input;
  // 动作
  enum
  {
    FLOW_UNKNOWN_ACTION,
    FLOW_ADD,
    FLOW_DEL,
    FLOW_ENABLE,
    FLOW_DISABLE
  } action = FLOW_UNKNOWN_ACTION;
  u32 hw_if_index = ~0, flow_index = ~0;
  int rv;
  u32 prot = 0, teid = 0;
  vnet_flow_type_t type = VNET_FLOW_TYPE_IP4_N_TUPLE;
  bool is_gtpc_set = false;
  bool is_gtpu_set = false;
  vnet_flow_type_t outer_type = VNET_FLOW_TYPE_UNKNOWN;
  vnet_flow_type_t inner_type = VNET_FLOW_TYPE_UNKNOWN;
  bool outer_ip4_set = false, inner_ip4_set = false;
  bool outer_ip6_set = false, inner_ip6_set = false;
  ip4_address_and_mask_t ip4s = { };
  ip4_address_and_mask_t ip4d = { };
  ip4_address_and_mask_t inner_ip4s = { };
  ip4_address_and_mask_t inner_ip4d = { };
  ip6_address_and_mask_t ip6s = { };
  ip6_address_and_mask_t ip6d = { };
  ip6_address_and_mask_t inner_ip6s = { };
  ip6_address_and_mask_t inner_ip6d = { };
  ip_port_and_mask_t sport = { };
  ip_port_and_mask_t dport = { };
  u16 eth_type;
  bool ethernet_set = false;

  // 初始化flow内存
  clib_memset (&flow, 0, sizeof (vnet_flow_t));
  // 流索引
  flow.index = ~0;
  // 流动作：count/mark/buffer-advance/redirect-to-node/redirect-to-queue/drop
  flow.actions = 0;
  // 初始化流ip4-n-tuple协议
  flow.ip4_n_tuple.protocol = ~0;
  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
    // 解析参数
    if (unformat (line_input, "add"))
	  action = FLOW_ADD;
    else if (unformat (line_input, "del"))
	  action = FLOW_DEL;
    else if (unformat (line_input, "enable"))
	  action = FLOW_ENABLE;
    else if (unformat (line_input, "disable"))
	  action = FLOW_DISABLE;
    else if (unformat (line_input, "eth-type %U",
			 unformat_ethernet_type_host_byte_order, &eth_type))
	  ethernet_set = true;
    else if (unformat (line_input, "src-ip %U",
			 unformat_ip4_address_and_mask, &ip4s))
	  outer_ip4_set = true;
    else if (unformat (line_input, "dst-ip %U",
			 unformat_ip4_address_and_mask, &ip4d))
	  outer_ip4_set = true;
    else if (unformat (line_input, "ip6-src-ip %U",
			 unformat_ip6_address_and_mask, &ip6s))
	  outer_ip6_set = true;
    else if (unformat (line_input, "ip6-dst-ip %U",
			 unformat_ip6_address_and_mask, &ip6d))
	  outer_ip6_set = true;
    else if (unformat (line_input, "inner-src-ip %U",
			 unformat_ip4_address_and_mask, &inner_ip4s))
	  inner_ip4_set = true;
    else if (unformat (line_input, "inner-dst-ip %U",
			 unformat_ip4_address_and_mask, &inner_ip4d))
	  inner_ip4_set = true;
    else if (unformat (line_input, "inner-ip6-src-ip %U",
			 unformat_ip6_address_and_mask, &inner_ip6s))
	  inner_ip6_set = true;
    else if (unformat (line_input, "inner-ip6-dst-ip %U",
			 unformat_ip6_address_and_mask, &inner_ip6d))
	  inner_ip6_set = true;
    else if (unformat (line_input, "src-port %U", unformat_ip_port_and_mask,
			 &sport))
	;
    else if (unformat (line_input, "dst-port %U", unformat_ip_port_and_mask,
			 &dport))
	;
    else if (unformat (line_input, "proto %U", unformat_ip_protocol, &prot))
	;
    else if (unformat (line_input, "proto %u", &prot))
	;
    else if (unformat (line_input, "gtpc teid %u", &teid))
	  is_gtpc_set = true;
    else if (unformat (line_input, "gtpu teid %u", &teid))
	  is_gtpu_set = true;
    else if (unformat (line_input, "index %u", &flow_index))
	;
    else if (unformat (line_input, "next-node %U", unformat_vlib_node, vm,
			 &flow.redirect_node_index))
	  flow.actions |= VNET_FLOW_ACTION_REDIRECT_TO_NODE;
    else if (unformat (line_input, "mark %d", &flow.mark_flow_id))
	  flow.actions |= VNET_FLOW_ACTION_MARK;
    else if (unformat (line_input, "buffer-advance %d",
			 &flow.buffer_advance))
	  flow.actions |= VNET_FLOW_ACTION_BUFFER_ADVANCE;
    else if (unformat (line_input, "redirect-to-queue %d",
			 &flow.redirect_queue))
	  flow.actions |= VNET_FLOW_ACTION_REDIRECT_TO_QUEUE;
    else if (unformat (line_input, "drop"))
	  flow.actions |= VNET_FLOW_ACTION_DROP;
    else if (unformat (line_input, "%U", unformat_vnet_hw_interface, vnm,
			 &hw_if_index))
	;
    else
	  return clib_error_return (0, "parse error: '%U'",
				  format_unformat_error, line_input);
  }

  unformat_free (line_input);

  // 启用/禁用流必须指定接口
  if (hw_if_index == ~0 && (action == FLOW_ENABLE || action == FLOW_DISABLE))
    return clib_error_return (0, "Please specify interface name");

  // 启用/禁用/删除流必须指定流索引
  if (flow_index == ~0 && (action == FLOW_ENABLE || action == FLOW_DISABLE ||
			   action == FLOW_DEL))
    return clib_error_return (0, "Please specify flow index");

  switch (action)
  {
    // 添加
    case FLOW_ADD:
      if (flow.actions == 0)
        return clib_error_return (0, "Please specify at least one action");

      /* Adjust the flow type */
      // outer流类型为ethernet流
      if (ethernet_set == true)
        outer_type = VNET_FLOW_TYPE_ETHERNET;
      // outer流类型为ipv[4|6]流
      if (outer_ip4_set == true)
        outer_type = VNET_FLOW_TYPE_IP4_N_TUPLE;
      else if (outer_ip6_set == true)
        outer_type = VNET_FLOW_TYPE_IP6_N_TUPLE;
      // inner流类型为ipv[4|6]流
      if (inner_ip4_set == true)
        inner_type = VNET_FLOW_TYPE_IP4_N_TUPLE;
      else if (inner_ip6_set == true)
        inner_type = VNET_FLOW_TYPE_IP6_N_TUPLE;

      if (outer_type == VNET_FLOW_TYPE_UNKNOWN)
        return clib_error_return (0, "Please specify a supported flow type");

      // outer流类型为以太网流
      if (outer_type == VNET_FLOW_TYPE_ETHERNET)
        type = VNET_FLOW_TYPE_ETHERNET;
      // outer流类型为ipv4流
      else if (outer_type == VNET_FLOW_TYPE_IP4_N_TUPLE)
      {
        // 根据inner流类型确定具体类型type
        type = VNET_FLOW_TYPE_IP4_N_TUPLE;

        // ipv4-gtpc ipv4-gtpu
	    if (inner_type == VNET_FLOW_TYPE_UNKNOWN)
	    {
	      if (is_gtpc_set)
            type = VNET_FLOW_TYPE_IP4_GTPC;
	      else if (is_gtpu_set)
            type = VNET_FLOW_TYPE_IP4_GTPU;
	    }
	    // ipv4-gtpu-ipv4
	    else if (inner_type == VNET_FLOW_TYPE_IP4_N_TUPLE)
	    {
	      if (is_gtpu_set)
            type = VNET_FLOW_TYPE_IP4_GTPU_IP4;
	    }
	    // ipv4-gtpu-ipv6
	    else if (inner_type == VNET_FLOW_TYPE_IP6_N_TUPLE)
	    {
	      if (is_gtpu_set)
            type = VNET_FLOW_TYPE_IP4_GTPU_IP6;
	    }
      }
      // outer流类型为ipv6流
      else if (outer_type == VNET_FLOW_TYPE_IP6_N_TUPLE)
      {
        // 根据inner流类型确定具体类型type
	    type = VNET_FLOW_TYPE_IP6_N_TUPLE;

        // ipv6-gtpc ipv6-gtpu
	    if (inner_type == VNET_FLOW_TYPE_UNKNOWN)
	    {
	      if (is_gtpc_set)
            type = VNET_FLOW_TYPE_IP6_GTPC;
	      else if (is_gtpu_set)
            type = VNET_FLOW_TYPE_IP6_GTPU;
	    }
	    // ipv6-gtpu-ipv4
	    else if (inner_type == VNET_FLOW_TYPE_IP4_N_TUPLE)
	    {
	      if (is_gtpu_set)
            type = VNET_FLOW_TYPE_IP6_GTPU_IP4;
	    }
	    // ipv6-gtpu-ipv6
	    else if (inner_type == VNET_FLOW_TYPE_IP6_N_TUPLE)
	    {
	      if (is_gtpu_set)
            type = VNET_FLOW_TYPE_IP6_GTPU_IP6;
	    }
      }

      //assign specific field values per flow type
      switch (type)
      {
	  case VNET_FLOW_TYPE_ETHERNET:
        // 赋s-mac, d-mac, type
	    memset (&flow.ethernet, 0, sizeof (flow.ethernet));
	    flow.ethernet.eth_hdr.type = eth_type;
	    break;

	  case VNET_FLOW_TYPE_IP4_N_TUPLE:
	  case VNET_FLOW_TYPE_IP4_GTPC:
	  case VNET_FLOW_TYPE_IP4_GTPU:
	  case VNET_FLOW_TYPE_IP4_GTPU_IP4:
	  case VNET_FLOW_TYPE_IP4_GTPU_IP6:
        // 赋s-ip, d-ip, s-port, d-port, proto
	    clib_memcpy (&flow.ip4_n_tuple.src_addr, &ip4s,
		       sizeof (ip4_address_and_mask_t));
	    clib_memcpy (&flow.ip4_n_tuple.dst_addr, &ip4d,
		       sizeof (ip4_address_and_mask_t));
	    clib_memcpy (&flow.ip4_n_tuple.src_port, &sport,
		       sizeof (ip_port_and_mask_t));
	    clib_memcpy (&flow.ip4_n_tuple.dst_port, &dport,
		       sizeof (ip_port_and_mask_t));
	    flow.ip4_n_tuple.protocol = prot;

        // ipv4-gtpc, ipv4-gtpu赋teid
	    if (type == VNET_FLOW_TYPE_IP4_GTPC)
	      flow.ip4_gtpc.teid = teid;
	    else if (type == VNET_FLOW_TYPE_IP4_GTPU)
	      flow.ip4_gtpu.teid = teid;
        // ipv4-gtpu-ipv4赋teid, in-s-ip, in-d-ip
	    else if (type == VNET_FLOW_TYPE_IP4_GTPU_IP4)
	    {
	      flow.ip4_gtpu_ip4.teid = teid;
	      clib_memcpy (&flow.ip4_gtpu_ip4.inner_src_addr, &inner_ip4s,
			   sizeof (ip4_address_and_mask_t));
	      clib_memcpy (&flow.ip4_gtpu_ip4.inner_dst_addr, &inner_ip4d,
			   sizeof (ip4_address_and_mask_t));
	    }
        // ipv4-gtpu-ipv6赋teid, in-s-ip, in-d-ip
	    else if (type == VNET_FLOW_TYPE_IP4_GTPU_IP6)
	    {
	      flow.ip4_gtpu_ip6.teid = teid;
	      clib_memcpy (&flow.ip4_gtpu_ip6.inner_src_addr, &inner_ip6s,
			   sizeof (ip6_address_and_mask_t));
	      clib_memcpy (&flow.ip4_gtpu_ip6.inner_dst_addr, &inner_ip6d,
			   sizeof (ip6_address_and_mask_t));
	    }

        // 校验
	    if (flow.ip4_n_tuple.protocol == (ip_protocol_t) ~ 0)
	      return clib_error_return (0, "Please specify ip protocol");
	    if ((type != VNET_FLOW_TYPE_IP4_N_TUPLE) && (flow.ip4_n_tuple.protocol != IP_PROTOCOL_UDP))
	      return clib_error_return (0, "For GTP related flow, ip protocol must be UDP");
	    break;

	  case VNET_FLOW_TYPE_IP6_N_TUPLE:
	  case VNET_FLOW_TYPE_IP6_GTPC:
	  case VNET_FLOW_TYPE_IP6_GTPU:
	  case VNET_FLOW_TYPE_IP6_GTPU_IP4:
	  case VNET_FLOW_TYPE_IP6_GTPU_IP6:
        // 赋s-ip, d-ip, s-port, d-port, proto
	    clib_memcpy (&flow.ip6_n_tuple.src_addr, &ip6s,
		       sizeof (ip6_address_and_mask_t));
	    clib_memcpy (&flow.ip6_n_tuple.dst_addr, &ip6d,
		       sizeof (ip6_address_and_mask_t));
	    clib_memcpy (&flow.ip6_n_tuple.src_port, &sport,
		       sizeof (ip_port_and_mask_t));
	    clib_memcpy (&flow.ip6_n_tuple.dst_port, &dport,
		       sizeof (ip_port_and_mask_t));
	    flow.ip6_n_tuple.protocol = prot;

        // ipv6-gtpc, ipv6-gtpu赋teid
	    if (type == VNET_FLOW_TYPE_IP6_GTPC)
	      flow.ip6_gtpc.teid = teid;
	    else if (type == VNET_FLOW_TYPE_IP6_GTPU)
	      flow.ip6_gtpu.teid = teid;
        // ipv6-gtpu-ipv4赋teid, in-s-ip, in-d-ip
	    else if (type == VNET_FLOW_TYPE_IP6_GTPU_IP4)
	    {
	      flow.ip6_gtpu_ip4.teid = teid;
	      clib_memcpy (&flow.ip6_gtpu_ip4.inner_src_addr, &inner_ip4s,
			   sizeof (ip4_address_and_mask_t));
	      clib_memcpy (&flow.ip6_gtpu_ip4.inner_dst_addr, &inner_ip4d,
			   sizeof (ip4_address_and_mask_t));
	    }
        // ipv6-gtpu-ipv6赋teid, in-s-ip, in-d-ip
	    else if (type == VNET_FLOW_TYPE_IP6_GTPU_IP6)
	    {
	      flow.ip6_gtpu_ip6.teid = teid;
	      clib_memcpy (&flow.ip6_gtpu_ip6.inner_src_addr, &inner_ip6s,
			   sizeof (ip6_address_and_mask_t));
	      clib_memcpy (&flow.ip6_gtpu_ip6.inner_dst_addr, &inner_ip6d,
			   sizeof (ip6_address_and_mask_t));
	    }

        // 校验
	    if (flow.ip6_n_tuple.protocol == (ip_protocol_t) ~ 0)
	      return clib_error_return (0, "Please specify ip protocol");
	    if ((type != VNET_FLOW_TYPE_IP4_N_TUPLE) &&
	      (flow.ip6_n_tuple.protocol != IP_PROTOCOL_UDP))
	      return clib_error_return (0,
				      "For GTP related flow, ip protocol must be UDP");
	    break;

	  default:
	    break;
      }

      // 赋流类型
      flow.type = type;
      // 添加流
      rv = vnet_flow_add (vnm, &flow, &flow_index);
      if (!rv)
        printf ("flow %u added\n", flow_index);

      break;
    // 删除
    case FLOW_DEL:
      // 删除流
      rv = vnet_flow_del (vnm, flow_index);
      break;
    // 启用
    case FLOW_ENABLE:
      // 启用接口流功能
      rv = vnet_flow_enable (vnm, flow_index, hw_if_index);
      break;
    // 禁用
    case FLOW_DISABLE:
      // 禁用接口流功能
      rv = vnet_flow_disable (vnm, flow_index, hw_if_index);
      break;
    default:
      return clib_error_return (0, "please specify action (add, del, enable,"
				" disable)");
  }

  if (rv < 0)
    return clib_error_return (0, "flow error: %U", format_flow_error, rv);
  return 0;
}
```

**添加流**

```
int
vnet_flow_add (vnet_main_t * vnm, vnet_flow_t * flow, u32 * flow_index)
{
  vnet_flow_main_t *fm = &flow_main;
  vnet_flow_t *f;

  // 从flow池中申请flow
  pool_get (fm->global_flow_pool, f);
  // 计算flow索引
  *flow_index = f - fm->global_flow_pool;
  // 拷贝flow内容
  clib_memcpy_fast (f, flow, sizeof (vnet_flow_t));
  // 赋私有数据
  f->private_data = 0;
  // 赋flow索引
  f->index = *flow_index;
  return 0;
}
```

**删除流**

```
int
vnet_flow_del (vnet_main_t * vnm, u32 flow_index)
{
  vnet_flow_main_t *fm = &flow_main;
  vnet_flow_t *f = vnet_get_flow (flow_index);
  uword hw_if_index;
  uword private_data;

  if (f == 0)
    return VNET_FLOW_ERROR_NO_SUCH_ENTRY;

  /* *INDENT-OFF* */
  // 遍历流hash，删除指定接口上的流
  hash_foreach (hw_if_index, private_data, f->private_data,
    ({
     vnet_flow_disable (vnm, flow_index, hw_if_index);
    }));
  /* *INDENT-ON* */

  // 释放私有数据内存
  hash_free (f->private_data);
  // 初始化flow内存，清空
  clib_memset (f, 0, sizeof (*f));
  // 把删除的flow放到flow池中重复利用
  pool_put (fm->global_flow_pool, f);
  return 0;
}
```

**启用接口流功能**

```
int
vnet_flow_enable (vnet_main_t * vnm, u32 flow_index, u32 hw_if_index)
{
  vnet_flow_t *f = vnet_get_flow (flow_index);
  vnet_hw_interface_t *hi;
  vnet_device_class_t *dev_class;
  uword private_data;
  int rv;

  if (f == 0)
    return VNET_FLOW_ERROR_NO_SUCH_ENTRY;

  if (!vnet_hw_interface_is_valid (vnm, hw_if_index))
    return VNET_FLOW_ERROR_NO_SUCH_INTERFACE;

  /* don't enable flow twice */
  if (hash_get (f->private_data, hw_if_index) != 0)
    return VNET_FLOW_ERROR_ALREADY_DONE;
  
  // 根据硬件接口索引，获取硬件接口
  hi = vnet_get_hw_interface (vnm, hw_if_index);
  // 获取硬件接口设备类
  dev_class = vnet_get_device_class (vnm, hi->dev_class_index);

  // 检查硬件接口flow offload操作函数，对于dpdk，flow_ops_function为dpdk_flow_ops_fn(源文件位于`plugins/dpdk/device/flow.c`)
  if (dev_class->flow_ops_function == 0)
    return VNET_FLOW_ERROR_NOT_SUPPORTED;

  // flow动作为redirect-to-node
  if (f->actions & VNET_FLOW_ACTION_REDIRECT_TO_NODE)
  {
    vnet_hw_interface_t *hw = vnet_get_hw_interface (vnm, hw_if_index);
    // 设置hw->input_node_index的next node为f->redirect_node_index
    f->redirect_device_input_next_index =
	    vlib_node_add_next (vnm->vlib_main, hw->input_node_index, f->redirect_node_index);
  }

  // 添加流到硬件接口
  rv = dev_class->flow_ops_function (vnm, VNET_FLOW_DEV_OP_ADD_FLOW,
				     hi->dev_instance, flow_index,
				     &private_data);

  if (rv)
    return rv;

  // 设置私有数据
  hash_set (f->private_data, hw_if_index, private_data);
  return 0;
}
```

**禁用接口流功能**

```
int
vnet_flow_disable (vnet_main_t * vnm, u32 flow_index, u32 hw_if_index)
{
  vnet_flow_t *f = vnet_get_flow (flow_index);
  vnet_hw_interface_t *hi;
  vnet_device_class_t *dev_class;
  uword *p;
  int rv;

  if (f == 0)
    return VNET_FLOW_ERROR_NO_SUCH_ENTRY;

  if (!vnet_hw_interface_is_valid (vnm, hw_if_index))
    return VNET_FLOW_ERROR_NO_SUCH_INTERFACE;

  /* don't disable if not enabled */
  if ((p = hash_get (f->private_data, hw_if_index)) == 0)
    return VNET_FLOW_ERROR_ALREADY_DONE;

  // 根据硬件接口索引，获取硬件接口
  hi = vnet_get_hw_interface (vnm, hw_if_index);
  // 获取硬件接口设备类
  dev_class = vnet_get_device_class (vnm, hi->dev_class_index);

  // 从硬件接口删除流
  rv = dev_class->flow_ops_function (vnm, VNET_FLOW_DEV_OP_DEL_FLOW,
				     hi->dev_instance, flow_index, p);

  if (rv)
    return rv;

  // 删除私有数据
  hash_unset (f->private_data, hw_if_index);
  return 0;
}
```