---
layout: post
title:  "python ironic client使用"
date:   2017-03-11 16:05:14 +0800
categories: openstack
tags: [ironic]
location: Nanjing, China
description: 介绍了ironic client的基本用法.
---

## 使用cli
```python
from ironicclient import client
kwargs = {'os_username': 'ironic',
          'os_password': 'IRONIC_PASSWORD',
          'os_auth_url': 'http://192.168.1.72:5000/',
          'os_tenant_name': 'services'}

ironic = client.get_client(1, **kwargs)

print ironic.node.list()                                                                                                                                   
print ironic.driver.list()
```
上面的os_username和os_password是ironic的账号，而不是keystone的账号。

ironicclient是一个cli工具，用来和用户交互的。首先写一个简单的例子，获取ironic所有的node节点：

```python
from ironicclient import client


if __name__ == '__main__':
    kwargs = {'os_username': 'ironic',
              'os_password': 'IRONIC_PASSWORD',
              'os_auth_url': 'http://192.168.1.72:5000/',
              'os_tenant_name': 'services'}
    ironic = client.get_client(1, **kwargs)
    print ironic.node.list()
```

这里我们创建了一个client对象，这个对象是通过client类的get_client方法返回的，这是一个工厂模式，下面看下get_client方法：

```python
def get_client(api_version, os_auth_token=None, ironic_url=None,
               os_username=None, os_password=None, os_auth_url=None,
               os_project_id=None, os_project_name=None, os_tenant_id=None,
               os_tenant_name=None, os_region_name=None,
               os_user_domain_id=None, os_user_domain_name=None,
               os_project_domain_id=None, os_project_domain_name=None,
               os_service_type=None, os_endpoint_type=None,
               insecure=None, timeout=None, os_cacert=None, ca_file=None,
               os_cert=None, cert_file=None, os_key=None, key_file=None,
               os_ironic_api_version=None, max_retries=None,
               retry_interval=None, session=None, **ignored_kwargs):
    os_service_type = os_service_type or 'baremetal'
    os_endpoint_type = os_endpoint_type or 'publicURL'
    project_id = (os_project_id or os_tenant_id)
    project_name = (os_project_name or os_tenant_name)
    kwargs = {
        'os_ironic_api_version': os_ironic_api_version,
        'max_retries': max_retries,
        'retry_interval': retry_interval,
    }
    endpoint = ironic_url
    cacert = os_cacert or ca_file
    cert = os_cert or cert_file
    key = os_key or key_file
    if os_auth_token and endpoint:
        kwargs.update({
            'token': os_auth_token,
            'insecure': insecure,
            'ca_file': cacert,
            'cert_file': cert,
            'key_file': key,
            'timeout': timeout,
        })
    elif os_auth_url:
        auth_type = 'password'
        auth_kwargs = {
            'auth_url': os_auth_url,
            'project_id': project_id,
            'project_name': project_name,
            'user_domain_id': os_user_domain_id,
            'user_domain_name': os_user_domain_name,
            'project_domain_id': os_project_domain_id,
            'project_domain_name': os_project_domain_name,
        }
        if os_username and os_password:
            auth_kwargs.update({
                'username': os_username,
                'password': os_password,
            })
        elif os_auth_token:
            auth_type = 'token'
            auth_kwargs.update({
                'token': os_auth_token,
            })
        # Create new session only if it was not passed in
        if not session:
            loader = kaloading.get_plugin_loader(auth_type)
            auth_plugin = loader.load_from_options(**auth_kwargs)
            # Let keystoneauth do the necessary parameter conversions
            session = kaloading.session.Session().load_from_options(
                auth=auth_plugin, insecure=insecure, cacert=cacert,
                cert=cert, key=key, timeout=timeout,
            )

    exception_msg = _('Must provide Keystone credentials or user-defined '
                      'endpoint and token')
    if not endpoint:
        if session:
            try:
                # Pass the endpoint, it will be used to get hostname
                # and port that will be used for API version caching. It will
                # be also set as endpoint_override.
                endpoint = session.get_endpoint(
                    service_type=os_service_type,
                    interface=os_endpoint_type,
                    region_name=os_region_name
                )
            except Exception as e:
                raise exc.AmbiguousAuthSystem(
                    exception_msg + _(', error was: %s') % e)
        else:
            # Neither session, nor valid auth parameters provided
            raise exc.AmbiguousAuthSystem(exception_msg)

    # Always pass the session
    kwargs['session'] = session

    return Client(api_version, endpoint, **kwargs)
```


中间这一大串可以先不看，主要是认证的一些信息。在get_cleint函数结尾，调用了Client() 方法作为返回值。这里传入了三个参数：

- api_version
- endpoint
- **kwargs

其中api_version是1，endpoint是http://192.168.1.72:6385，kwargs保存了session的信息：

`{'os_ironic_api_version': None, 'session': <keystoneauth1.session.Session object at 0x2441690>, 'retry_interval': None, 'max_retries': None}`

我们再来看看Client()方法的具体实现：

```python
def Client(version, *args, **kwargs):
    module = utils.import_versioned_module(version, 'client')
    client_class = getattr(module, 'Client')
    return client_class(*args, **kwargs)
```

这里module内容如下：
`<module 'ironicclient.v1.client' from '/usr/lib/python2.7/site-packages/ironicclient/v1/client.pyc'>`

后面的client_class使用反射的机制来获取Client类。简单的说就是调用文件
`/usr/lib/python2.7/site-packages/ironicclient/v1/client.py`中的Client类，并返回其对象。再来看下Client类的具体实现：

```python
class Client(object):
    def __init__(self, *args, **kwargs):
        """Initialize a new client for the Ironic v1 API."""
        if kwargs.get('os_ironic_api_version'):
            kwargs['api_version_select_state'] = "user"
        else:
            # If the user didn't specify a version, use a cached version if
            # one has been stored
            host, netport = http.get_server(args[0])
            saved_version = filecache.retrieve_data(host=host, port=netport)
            if saved_version:
                kwargs['api_version_select_state'] = "cached"
                kwargs['os_ironic_api_version'] = saved_version
            else:
                kwargs['api_version_select_state'] = "default"
                kwargs['os_ironic_api_version'] = DEFAULT_VER

        self.http_client = http._construct_http_client(*args, **kwargs)

        self.chassis = chassis.ChassisManager(self.http_client)
        self.node = node.NodeManager(self.http_client)
        self.port = port.PortManager(self.http_client)
        self.driver = driver.DriverManager(self.http_client)
```

可以看出Client类的chassis，node，port，driver属性都是对应类的manager。先看看NodeManager的实现：

```python
class NodeManager(base.CreateManager):
    resource_class = Node
    _creation_attributes = ['chassis_uuid', 'driver', 'driver_info',
                            'extra', 'uuid', 'properties', 'name']
    _resource_name = 'nodes'

    def list(self, associated=None, maintenance=None, marker=None, limit=None,
             detail=False, sort_key=None, sort_dir=None, fields=None,
             provision_state=None, driver=None):
        if limit is not None:
            limit = int(limit)

        if detail and fields:
            raise exc.InvalidAttribute(_("Can't fetch a subset of fields "
                                         "with 'detail' set"))

        filters = utils.common_filters(marker, limit, sort_key, sort_dir,
                                       fields)
        if associated is not None:
            filters.append('associated=%s' % associated)
        if maintenance is not None:
            filters.append('maintenance=%s' % maintenance)
        if provision_state is not None:
            filters.append('provision_state=%s' % provision_state)
        if driver is not None:
            filters.append('driver=%s' % driver)

        path = ''
        if detail:
            path += 'detail'
        if filters:
            path += '?' + '&'.join(filters)

        if limit is None:
            return self._list(self._path(path), "nodes")
        else:
            return self._list_pagination(self._path(path), "nodes",
                                         limit=limit)
...
```

我们看list方法，前面是一些过滤条件，因为我们没有穿limit参数，所有limit是None,然后调用了_list()方法。这个是在父类里定义的。实现代码在：`/usr/lib/python2.7/site-packages/ironicclient/common/base.py`

```python
    def _list(self, url, response_key=None, obj_class=None, body=None):
        resp, body = self.api.json_request('GET', url)

        if obj_class is None:
            obj_class = self.resource_class

        data = self._format_body_data(body, response_key)
        return [obj_class(self, res, loaded=True) for res in data if res]
```

我们可以看到，这里是发送了HTTP GET请求，然就将收到的数据格式化并返回。这里的api是我们在前面传入的，使用http创建的一个client对象。

```python
self.http_client = http._construct_http_client(*args, **kwargs)
...

self.node = node.NodeManager(self.http_client)
```