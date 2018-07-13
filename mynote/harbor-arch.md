# Architecture of Harbor

![](https://camo.githubusercontent.com/e0de62fb4f08efedd2c5abd44786410d3af06c7b/687474703a2f2f7777772e7468696e6b2d666f756e6472792e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031362f30392f61727469636c65315f696d616765322e706e67)

## Components in kubernetes
- harbor-adminserver
- harbor-jobservice
- harbor-db: Stores the meta data of projects, users, roles, replication policies and images.
- registry: Responsible for storing Docker images and processing Docker push/pull commands
- harbor-ui: 
- ingress: Like proxy, such as registry, UI and token services, are all behind a reversed proxy

Ref: https://github.com/vmware/harbor/wiki/Architecture-Overview-of-Harbor  
