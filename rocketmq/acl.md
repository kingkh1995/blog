# [RocketMQ](/blog/component/rocketmq)

> ACL & ACL 2.0
>> Reference: <https://developer.aliyun.com/article/1554227>

***

## ACL 1.0

## AclClientRPCHook implements RPCHook

客户端使用，基于RPCHook实现，用于发送请求前使用AK/SK生成签名。

```java
@Override
public void doBeforeRequest(String remoteAddr, RemotingCommand request) {
    // Add AccessKey and SecurityToken into signature calculating.
    request.addExtField(ACCESS_KEY, sessionCredentials.getAccessKey());
    ...
    // 获取getExtFields并排序，作为被签名的内容
    byte[] total = AclUtils.combineRequestContent(request, parseRequestContent(request));
    // 使用SK生成签名，SK只参与签名。
    String signature = AclUtils.calSignature(total, sessionCredentials.getSecretKey());
    request.addExtField(SIGNATURE, signature);
}
```

## initialAcl

BrokerController启动时初始化ACL，使用SPI加载AccessValidator，也是基于RPCHook实现。

```java
private void initialAcl() {
    if (!this.brokerConfig.isAclEnable()) {
        return;
    }

    List<AccessValidator> accessValidators = ServiceProvider.load(AccessValidator.class);
    if (accessValidators.isEmpty()) {
        accessValidators.add(new PlainAccessValidator());
    }

    for (AccessValidator accessValidator : accessValidators) {
        final AccessValidator validator = accessValidator;
        accessValidatorMap.put(validator.getClass(), validator);
        this.registerServerRPCHook(new RPCHook() {

            @Override
            public void doBeforeRequest(String remoteAddr, RemotingCommand request) {
                //Do not catch the exception
                validator.validate(validator.parse(request, remoteAddr));
            }

            @Override
            public void doAfterResponse(String remoteAddr, RemotingCommand request, RemotingCommand response) {
            }

        });
    }
}
```

### AccessValidator

两个核心接口：parse用于从请求中构造需要进行验证的资源，validate用于验证是否有权限操作该资源。

### PlainAccessValidator implements AccessValidator

使用yaml格式的acl配置文件

### PlainPermissionManager

管理yaml格式的acl配置文件，支持配置多个文件及默认文件，并支持监听文件内容变化。

```java
public void validate(PlainAccessResource plainAccessResource) {
    // 全局级白名单配置验证，支持多种策略。
    for (RemoteAddressStrategy remoteAddressStrategy : globalWhiteRemoteAddressStrategy) {
        if (remoteAddressStrategy.match(plainAccessResource)) {
            return;
        }
    }

    // 校验客户端及服务端AK是否存在
    if (plainAccessResource.getAccessKey() == null) {
        throw new AclException("No accessKey is configured");
    }
    if (!accessKeyTable.containsKey(plainAccessResource.getAccessKey())) {
        throw new AclException(String.format("No acl config for %s", plainAccessResource.getAccessKey()));
    }

    // 用户级白名单配置校验
    String aclFileName = accessKeyTable.get(plainAccessResource.getAccessKey());
    // Q: 此处可优化，不需要每次new一个HashMap。
    PlainAccessResource ownedAccess = aclPlainAccessResourceMap.getOrDefault(aclFileName, new HashMap<>()).get(plainAccessResource.getAccessKey());
    if (ownedAccess == null) {
        throw new AclException(String.format("No PlainAccessResource for accessKey=%s", plainAccessResource.getAccessKey()));
    }
    if (ownedAccess.getRemoteAddressStrategy().match(plainAccessResource)) {
        return;
    }

    // 校验签名是否正确，同AclClientRPCHook，使用服务端的SK重新签名，并与客户端计算的签名进行比对。
    String signature = AclUtils.calSignature(plainAccessResource.getContent(), ownedAccess.getSecretKey());
    if (plainAccessResource.getSignature() == null
        || !MessageDigest.isEqual(signature.getBytes(AclSigner.DEFAULT_CHARSET), plainAccessResource.getSignature().getBytes(AclSigner.DEFAULT_CHARSET))) {
        throw new AclException(String.format("Check signature failed for accessKey=%s", plainAccessResource.getAccessKey()));
    }

    ...

    // 签名认证通过后，使用PermissionChecker验证是否有权限，验证失败抛出AclException。
    checkPerm(plainAccessResource, ownedAccess);
}
```

### AclFileWatchService extends ServiceThread

最新实现是单线程定时每5秒检查acl文件更新时间是否变化，如果发生了变化则触发监听器执行，即重新加载acl配置。

***

## ACL 2.0

细粒度ACL，分为 Authentication（认证）和 Authorization（鉴权），支持集群内ACL。

## AuthenticationProvider

## AuthorizationProvider
