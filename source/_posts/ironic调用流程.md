---
layout: post
title: ironic代码走读
date: 2017-04-04
categories: openstack
tags: [ironic]
description: ironic调用流程
---

## ironic调用流程

在openstack中，各组件之间是通过api调用的，组件内部是通过rpc调用的。例如我们通过nova来部署裸机。你首先在命令行敲下nova boot命令，nova经过schedule只有，通过ironic api来通知ironic进行部署。

ironic的api入口在`ironic/api/controllers/v1/nodes.py`

```python
class NodeStatesController(rest.RestController):
    @METRICS.timer('NodeStatesController.provision')
    @expose.expose(None, types.uuid_or_name, wtypes.text,
                   wtypes.text, types.jsontype,
                   status_code=http_client.ACCEPTED)
    def provision(self, node_ident, target, configdrive=None,
                  clean_steps=None):
        # 设置节点状态
    	if target == ir_states.ACTIVE:
            pecan.request.rpcapi.do_node_deploy(pecan.request.context,
                                                rpc_node.uuid, False,
                                                configdrive, topic)
        ...
        url_args = '/'.join([node_ident, 'states'])
        pecan.response.location = link.build_url('nodes', url_args)
```

我们可以看到，这里通过RPC调用`do_node_deploy`方法，需要注意的是上面的调用是异步的，HTTP只是返回202，然后在后台执行实际的代码。RPC API代码如下：

```python
# file: ironic/conductor/rpcapi.py

    def do_node_deploy(self, context, node_id, rebuild, configdrive,
                       topic=None):
        
        cctxt = self.client.prepare(topic=topic or self.topic, version='1.22')
        return cctxt.call(context, 'do_node_deploy', node_id=node_id,
                          rebuild=rebuild, configdrive=configdrive)
```

实现代码：

```python
 # file: manager.py   
    def do_node_deploy(self, context, node_id, rebuild=False,
                       configdrive=None):
		
        # context包括http请求的内容
        with task_manager.acquire(context, node_id, shared=False,
                                  purpose='node deployment') as task:
            node = task.node
            if node.maintenance:
                raise exception.NodeInMaintenance(op=_('provisioning'),
                                                  node=node.uuid)

            if rebuild:
                event = 'rebuild'

                # Note(gilliard) Clear these to force the driver to
                # check whether they have been changed in glance
                # NOTE(vdrok): If image_source is not from Glance we should
                # not clear kernel and ramdisk as they're input manually
                if glance_utils.is_glance_image(
                        node.instance_info.get('image_source')):
                    instance_info = node.instance_info
                    instance_info.pop('kernel', None)
                    instance_info.pop('ramdisk', None)
                    node.instance_info = instance_info
            else:
                event = 'deploy'

            driver_internal_info = node.driver_internal_info
            # Infer the image type to make sure the deploy driver
            # validates only the necessary variables for different
            # image types.
            # NOTE(sirushtim): The iwdi variable can be None. It's up to
            # the deploy driver to validate this.
            iwdi = images.is_whole_disk_image(context, node.instance_info)
            driver_internal_info['is_whole_disk_image'] = iwdi
            node.driver_internal_info = driver_internal_info
            node.save()

            try:
                task.driver.power.validate(task)
                task.driver.deploy.validate(task)
            except exception.InvalidParameterValue as e:
                raise exception.InstanceDeployFailure(
                    _("Failed to validate deploy or power info for node "
                      "%(node_uuid)s. Error: %(msg)s") %
                    {'node_uuid': node.uuid, 'msg': e})

            try:
                task.process_event(
                    event,
                    callback=self._spawn_worker,
                    call_args=(do_node_deploy, task, self.conductor.id,
                               configdrive),
                    err_handler=utils.provisioning_error_handler)
            except exception.InvalidState:
                raise exception.InvalidStateRequested(
                    action=event, node=task.node.uuid,
                    state=task.node.provision_state)

                
@METRICS.timer('do_node_deploy')
@task_manager.require_exclusive_lock
def do_node_deploy(task, conductor_id, configdrive=None):
    """Prepare the environment and deploy a node."""
    node = task.node

    def handle_failure(e, task, logmsg, errmsg):
        # NOTE(deva): there is no need to clear conductor_affinity
        task.process_event('fail')
        args = {'node': task.node.uuid, 'err': e}
        LOG.error(logmsg, args)
        node.last_error = errmsg % e

    try:
        try:
            if configdrive:
                _store_configdrive(node, configdrive)
        except exception.SwiftOperationError as e:
            with excutils.save_and_reraise_exception():
                handle_failure(
                    e, task,
                    _LE('Error while uploading the configdrive for '
                        '%(node)s to Swift'),
                    _('Failed to upload the configdrive to Swift. '
                      'Error: %s'))

        try:
            task.driver.deploy.prepare(task)
        except Exception as e:
            with excutils.save_and_reraise_exception():
                handle_failure(
                    e, task,
                    _LE('Error while preparing to deploy to node %(node)s: '
                        '%(err)s'),
                    _("Failed to prepare to deploy. Error: %s"))

        try:
            new_state = task.driver.deploy.deploy(task)
        except Exception as e:
            with excutils.save_and_reraise_exception():
                handle_failure(
                    e, task,
                    _LE('Error in deploy of node %(node)s: %(err)s'),
                    _("Failed to deploy. Error: %s"))

        # Update conductor_affinity to reference this conductor's ID
        # since there may be local persistent state
        node.conductor_affinity = conductor_id

        # NOTE(deva): Some drivers may return states.DEPLOYWAIT
        #             eg. if they are waiting for a callback
        if new_state == states.DEPLOYDONE:
            task.process_event('done')
            LOG.info(_LI('Successfully deployed node %(node)s with '
                         'instance %(instance)s.'),
                     {'node': node.uuid, 'instance': node.instance_uuid})
        elif new_state == states.DEPLOYWAIT:
            task.process_event('wait')
        else:
            LOG.error(_LE('Unexpected state %(state)s returned while '
                          'deploying node %(node)s.'),
                      {'state': new_state, 'node': node.uuid})
    finally:
        node.save()
```

