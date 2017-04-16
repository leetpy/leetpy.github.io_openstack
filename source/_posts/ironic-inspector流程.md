---
layout: post
title:  "ironic inspector流程"
date:   2017-03-11 16:05:14 +0800
categories: openstack
tags: [ironic]
location: Nanjing, China
description: 介绍了ironic inspector的实现过程，包括ironic api的调用，ipa数据收集，inspector数据解析，上报.
---
# inspector流程

## ironic 处理阶段

首先我们通过ironic的api，将node节点状态设置为manage，然后再设置为inspect。

```python
# ironic/api/controllers/v1/node.py
def provision():
    ...
    elif target == ir_state.VERBS['inspect']:
        pecan.request.rpcapi.inspect_hardware()
```

这里 provision是一个修改node状态的api，我们在request请求中，将target设置为inspect。接着会通过rpc调`inspect_hardware(rpc.node_uuid, topic=topic)`方法。最终会调用manager.py的`inspect_hardware`方法。

inspect的具体实现是跟driver有关的，在driver.inspect.inspect_hardware中。

```python
# ironic/drivers/modules/inspector.py

def _start_inspection(node_uuid, context):
    try:
        _call_inspector(client.introspect, node_uuid, context)
        ...
        
# ironic_inspector_client/client.py
def introspect(uuid, base_url=None, auth_token=None,
               new_ipmi_password=None, new_ipmi_username=None,
               api_version=DEFAULT_API_VERSION, session=None, **kwargs):

    c = v1.ClientV1(api_version=api_version, auth_token=auth_token,
                    inspector_url=base_url, session=session, **kwargs)
    return c.introspect(uuid, new_ipmi_username=new_ipmi_username,
                        new_ipmi_password=new_ipmi_password)

# ironic_inspector_client/v1.py
def introspect(self, uuid, new_ipmi_password=None, new_ipmi_username=None):
    ...
    params = {'new_ipmi_username': new_ipmi_username,
              'new_ipmi_password': new_ipmi_password}
    self.request('post', '/introspection/%s' % uuid, params=params)
```

然后创建一个inspector的client，并调用intropect函数。这个最终会发送post请求到/v1/introspection/<uuid>，并传入参数新的ipmi信息（如果需要更新ipmi用户名或者密码）

## inspector处理阶段

inspector的api是用flask实现的，这里我们根据url找到对应的代码：

```python
# main.py
@app.route('/v1/introspection/<uuid>')
@convert_exceptions
def api_introspection(uuid):
    ...
    introspect.introspect(uuid,
                          new_ipmi_credentials=new_ipmi_credentails,
                          token=flask.request.headers.get('X-Auth-Token'))
    return '', 202
```

再看看introspect的具体实现：

```python
# introspect.py
def introspect():
    node_info = node_cache.add_node(node.uuid,
                                    bmc_address=bmc_address,
                                    ironic=ironic)
    future = utils.executor().submit(_background_introspect, ironic, node_info)
    ...
```

introspect函数先是更新了ipmi信息，然后在inspector的node表里添加一条记录，另外在attributes表里添加bmc_address信息。最终后台调用_background_introspect做主机发现。

接着调用 _background_introspect_locked设置裸机从pxe启动，并重启裸机。

```python
# introspect.py
def _background_introspect_locked(ironic, node_info):
    try:
        ironic.node.set_boot_device(node_info.uuid, 'pxe',
                                    persistent=False)
    try:
        ironic.node.set_power_state(node_info.uuid, 'reboot')
```

我们在已经配置好了裸机，并在`/tftpboot/pxelinux.cfg/default`设置了如下信息：

```shell
default introspect                                                                                                                                           
label introspect
kernel ironic-agent.vmlinuz
append initrd=ironic-agent.initramfs ipa-inspection-callback-url=http://192.168.2.40:5050/v1/continue ipa-inspection-collectors=default,logs systemd.journald.forward_to_console=no

ipappend 3

```

## ipa阶段

裸机从小系统启动之后，会启动ironic-python-agent服务。该服务会收集裸机的硬件信息，并发送到`ipa-inspection-callback-url`指定的url。

这里ipa发送的post数据内容包括：

```json
{
    "boot_interface": "01-98-40-bb-81-37-5b",
    "ipmi_address": "10.43.200.161",
    "error": null,
    "root_disk": {
        "name": "/dev/sda",
        "model": "PERC H730 Mini",
        "hctl": "0:2:0:0",
        "wwn_with_extension": "0x61866da06bc6e7001f920081106a8012",
        "wwn": "0x61866da06bc6e700",
        "vendor": "DELL",
        "size": 299439751168,
        "rotational": true,
        "wwn_vendor_extension": "0x1f920081106a8012",
        "serial": "61866da06bc6e7001f920081106a8012"
    },
    "inventory": {
        "bmc_address": "10.43.200.161",
        "interfaces": [
            {
                "name": "eno1",
                "vendor": "0x8086",
                "lldp": null,
                "ipv4_address": null,
                "mac_address": "98:40:bb:81:37:59",
                "has_carrier": false,
                "client_id": null,
                "product": "0x10f8"
            },
            {
                "name": "eno2",
                "vendor": "0x8086",
                "lldp": null,
                "ipv4_address": "192.168.2.50",
                "mac_address": "98:40:bb:81:37:5b",
                "has_carrier": true,
                "client_id": null,
                "product": "0x10f8"
            }
        ],
        "cpu": {
            "count": 48,
            "frequency": "3100.0000",
            "model_name": "Intel(R) Xeon(R) CPU E5-2670 v3 @ 2.30GHz",
            "flags": [
                "acpi",
                "pts"
            ],
            "architecture": "x86_64"
        },
        "disks": [
            {
                "name": "/dev/sda",
                "model": "PERC H730 Mini",
                "hctl": "0:2:0:0",
                "wwn_with_extension": "0x61866da06bc6e7001f920081106a8012",
                "wwn": "0x61866da06bc6e700",
                "vendor": "DELL",
                "size": 299439751168,
                "rotational": true,
                "wwn_vendor_extension": "0x1f920081106a8012",
                "serial": "61866da06bc6e7001f920081106a8012"
            }
        ],
        "boot": {
            "current_boot_mode": "bios",
            "pxe_interface": "01-98-40-bb-81-37-5b"
        },
        "system_vendor": {
            "serial_number": "G032KG2",
            "product_name": "PowerEdge M630",
            "manufacturer": "Dell Inc."
        },
        "memory": {
            "physical_mb": 131072,
            "total": 135082627072
        }
    }
}
```

## inspector主机上报阶段

先看看inspector怎么处理ipa上报的数据：

```python
# main.py
@app.route('/v1/continue', methods=[post])
@convert_exceptions
def api_continue():
    data = flask.request.get_json(force=True)
    return flask.jsonify(process.process(data))


# process.py
def process(introspection_data):
    """Process data from the ramdisk.
    
    This function heavily relies on the hooks to do the actual data processing.
    """
    hooks = plugins_base.processing_hooks_manager()
    failures = []
    for hook_ext in hooks:
        try:
            hook_ext.obj.before_processing(introspection_data)
        except utils.Error as exc:
            ...
    # 根据ipmi_address和macs获取inpsector node
    node_info = _finde_node_info(introspection_data, failures)
    try:
        node = node_info.node()
        ...
        
    try:
        return _process_node(node, introspection_data, node_info)
```

我们可以看到这里数据是交由process出来，而process函数又调用各种钩子来出来ipa数据。接着根据ipmi_address查找对应的inspector node，再根据获取到的uuid来得到ironic node，交由_process_node()函数处理。

```python
# process.py
def _process_node(node, introspection_data, node_info):
    ir_utils.check_provision_state(node)
    node_info.create_ports(introspection_data.get('macs') or ())
    _run_post_hooks(node_inof, introspection_data)
    
    if CONF.processing.store_data == 'switf':
        stored_data = {k: v for k, v in introspection_data.items()
                       if k not in _STORAGE_EXCLUDE_KEYS}
        swift_object_name = switf.store.store_introspection_data(stored_data,
                                                                 node_info.uuid)
    ironic = ir_utils.get_client()
    firewall.update_filters(ironic)
    
    node_info.invalidate_cache()
    rules.apply(node_info, introspection_data)
    ...
    utils.executor().submit(_finish, ironic, node_info, introspection_data)

    
def _finish(ironic, node_info, introspection_data):
    try:
        ironic.node.set_power_state(node_info.uuid, 'off')
    node_info.finished()
   
```

我们可以看到，如果配置了`store_data=swift`，inspector会把ipa上报的数据存储到swift中。最后的 node_info.finished()是删除inspector数据库中已完成的数据。
