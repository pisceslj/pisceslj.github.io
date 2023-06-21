---
layout: post
title: Schema Registry ACL
tags:
- Kafka
- Schema Registry ACL
categories: Kafka
description: Schema Registry 认证
---

# SchemaRegistrySecurityResourceExtension

## registry 函数

获取配置

```Java
confluentSecureSchemaRegistryConfig = new SecureSchemaRegistryConfig(schemaRegistryConfig.originalProperties());
```

获取范围

```Java
scope = determineScope(kafkaSchemaRegistry);
```

注册资源

```Java
registerResources(configurable, confluentSecureSchemaRegistryConfig, kafkaSchemaRegistry, scope);
```

下面具体从这三个方面展开：

### 获取配置

首先将 schema registry 的配置传入，作为基础配置集合；

再通过静态代码块，执行初始化认证相关类的配置：

```Java
confluentConfigDef = initConfigDef();

private static ConfigDef initConfigDef() {
        return baseSchemaRegistryConfigDef()
                .define(
                        "schema.registry.auth.mechanism",
                        ConfigDef.Type.STRING,
                        "SSL",
                        ConfigDef.ValidString.in((String[])RestAuthType.NAMES.toArray(new String[RestAuthType.NAMES.size()])),
                        ConfigDef.Importance.LOW,
                        "The mechanism used to authenticate SchemaRegistry requests.The principal from the authentication mechanism is then used to optionally authorize using a configured authorizer."
                ......省略相关配置
                );
}
```

### 获取范围

首先根据配置的 brokerList 创建了一个 adminClient，获取 clusterId

最终返回 Scope 对象，包含：clusterId、schemaRegistryGroupId

```Java
KafkaStore.addSchemaRegistryConfigsToClientProperties(config, adminClientProps);
adminClientProps.put("bootstrap.servers", config.bootstrapBrokers());

try {
    AdminClient adminClient = AdminClient.create(adminClientProps);

    try {
        kafkaClusterId = adminClient.describeCluster().clusterId().get(CLUSTER_ID_TIMEOUT_MS, TimeUnit.MILLISECONDS);
    } catch (Throwable var15) {            
    } finally {
        
    }
} catch (ExecutionException | TimeoutException | InterruptedException var17) {
}

String schemaRegistryGroupId = config.getString("schema.registry.group.id");
return (new Builder(new String[0])).withKafkaCluster(kafkaClusterId).withCluster(SCHEMA_REGISTRY_CLUSTER_TYPE, schemaRegistryGroupId).build();    
```

### 注册资源

首先注册 AuthenticationCleanupFilter.class，创建 AuthenticationCleanupFilter.class 实例；

最后再创建一个 PermissionsResource

```Java
void registerResources(Configurable<?> configurable,
                           SecureSchemaRegistryConfig confluentSecureSchemaRegistryConfig,
                           SchemaRegistry kafkaSchemaRegistry,
                           Scope scope) {
        String restAuthTypeConfig = confluentSecureSchemaRegistryConfig.getString("schema.registry.auth.mechanism");
        String sslMappingRules = confluentSecureSchemaRegistryConfig.getString("schema.registry.auth.ssl.principal.mapping.rules");
        Optional<SslPrincipalMapper> sslPrincipalMapperOpt = Optional.empty();
        if (sslMappingRules != null) {
            sslPrincipalMapperOpt = Optional.of(SslPrincipalMapper.fromRules(sslMappingRules));
        }

        configurable.register(AuthenticationCleanupFilter.class);
        configurable.register(new AuthenticationFilter(restAuthTypeConfig, sslPrincipalMapperOpt, confluentSecureSchemaRegistryConfig.getBoolean("schema.registry.anonymous.principal")));
        this.authorizationFilter = new AuthorizationFilter(confluentSecureSchemaRegistryConfig, kafkaSchemaRegistry);
        configurable.register(this.authorizationFilter);
        configurable.register(new PermissionsResource(scope, kafkaSchemaRegistry, this.authorizationFilter.authorizer()));
}
```

# AuthorizationFilter


## initializeSchemaRegistryResourceActionMap

首先是对资源的actions进行初始化定义，主要定义了以下权限：

    SUBJECT_READ,
    SUBJECT_WRITE,
    SUBJECT_DELETE,
    SCHEMA_READ,
    SUBJECT_COMPATIBILITY_READ,
    SUBJECT_COMPATIBILITY_WRITE,
    GLOBAL_COMPATIBILITY_READ,
    GLOBAL_COMPATIBILITY_WRITE,
    GLOBAL_SUBJECTS_READ,
    AUTHORIZATION_NOT_REQUIRED;

## AuthorizationFilter

实例化函数里进行类实例化：

```Java
public AuthorizationFilter(SecureSchemaRegistryConfig secureSchemaRegistryConfig,
                               SchemaRegistry kafkaSchemaRegistry) {
        String authorizerClassName = secureSchemaRegistryConfig.getString("schema.registry.authorizer.class");
        if (StringUtil.isNotBlank(authorizerClassName)) {
            try {
                Class<SchemaRegistryAuthorizer> schemaRegistryAuthorizerClass = (Class<SchemaRegistryAuthorizer>) Class.forName(authorizerClassName);
                this.authorizer = schemaRegistryAuthorizerClass.newInstance();
                this.authorizer.configure(secureSchemaRegistryConfig, kafkaSchemaRegistry);
            }
        }
    }
```

## filter

首先从 requestContext，获取 schemaRegistryResourceOperation，根据 AUTHORIZATION_NOT_REQUIRED 字段判断是否需要认证；

如果需要认证，则进一步判断；

如果 schemaRegistryResourceOperation 为 null，则请求终止，用户无法访问 resource；

反之，获取请求中的 subject，调用 authorize 方法进行授权验证，其中传入了：username、subject、resourceOperation、requestContext

如果权限不够，则终止请求，得到访问拒绝的信息。

```Java
public void filter(ContainerRequestContext requestContext) {
        SchemaRegistryResourceOperation schemaRegistryResourceOperation = this.operation(requestContext);
        if (!SchemaRegistryResourceOperation.AUTHORIZATION_NOT_REQUIRED.equals(schemaRegistryResourceOperation)) {
            if (schemaRegistryResourceOperation == null) {
                requestContext.abortWith(this.accessDenied("User cannot access the resource."));
            } else {
                try {
                    String subject = this.currentSubject();
                    if (!this.authorizer.authorize(new AuthorizeRequest(requestContext.getSecurityContext().getUserPrincipal(), subject, schemaRegistryResourceOperation, requestContext, this.httpServletRequest))) {
                        requestContext.abortWith(this.accessDenied(this.operationDeniedMessage(schemaRegistryResourceOperation, subject)));
                    }
                }
            }
        }
    }
```

## filter

如果响应code是ok，并且请求的操作是GLOBAL_SUBJECTS_READ，则从响应中读取subjects，返回认证过的authorizedSubjects

```Java
public void filter(ContainerRequestContext requestContext, ContainerResponseContext responseContext) {
        if (responseContext.getStatus() == Status.OK.getStatusCode() && SchemaRegistryResourceOperation.GLOBAL_SUBJECTS_READ.equals(this.operation(requestContext))) {
            Set<String> subjects = (Set)readEntity(responseContext);
            if (subjects != null) {
                responseContext.setEntity(this.authorizedSubjects(requestContext, subjects));
            }
        }

    }
```

## authorizedSubjects

遍历subjects，对其resourceOperation进行验证；

验证时，先构造了authorizeRequests：传入参数是：密码、subject、operation、requestContext；

再调用 bulkAuthorize，传入上述的请求，然后返回认证结果；

最后返回校验授权情况后的subjects；

```Java
Set<String> authorizedSubjects(ContainerRequestContext requestContext, Set<String> allSubjects) {
        if (allSubjects.isEmpty()) {
            return allSubjects;
        } else {
            Principal principal = requestContext.getSecurityContext().getUserPrincipal();
            
            allSubjects.forEach((subject) -> {
                SchemaRegistryResourceOperation.SUBJECT_RESOURCE_OPERATIONS.forEach((operation) -> {
                    authorizeRequests.add(new AuthorizeRequest(principal, subject, operation, requestContext, this.httpServletRequest));
                });
            });

            List<Boolean> authorizations;
            try {
                authorizations = this.authorizer.bulkAuthorize(principal, authorizeRequests);
            }

            return (Set) StreamUtils.zip(authorizations, authorizeRequests).filter(Pair::getLeft).map((p) -> {
                return ((AuthorizeRequest)p.getRight()).getSubject();
            }).collect(Collectors.toSet());
        }
    }
```


# AbstractSchemaRegistryAuthorizer

这个类主要完成授权校验的过程，主要有以下一些方法：

## authorize

首先从授权请求中解析用户名、密码、subject、resourceOperation，

如果resourceOperation等于SCHEMA_READ，则调用authorizeSchemaIdLookup，寻找schema，找到则返回true；

如果resourceOperation等于SUBJECT_RESOURCE_OPERATIONS，则调用authorizeSubjectOperation，如果可以操作，能则返回true；

如果resourceOperation等于GLOBAL_RESOURCE_OPERATIONS，则调用authorizeGlobalOperation，如果可以操作，能则返回true；


```Java
public final boolean authorize(AuthorizeRequest authorizeRequest) throws AuthorizerException {
        SchemaRegistryResourceOperation schemaRegistryResourceOperation = authorizeRequest.getSchemaRegistryResourceOperation();
        Principal principal = authorizeRequest.getUser();
        String user = principal.getName();
        String subject = authorizeRequest.getSubject();
        if (SchemaRegistryResourceOperation.SCHEMA_READ.equals(schemaRegistryResourceOperation)) {
            boolean result = this.authorizeSchemaIdLookup(principal, schemaRegistryResourceOperation, authorizeRequest);
            log.info("Authorization of schema ID lookup {} for user {} and subject {}", new Object[]{result ? "SUCCESSFUL" : "FAILED", user != null ? user : "N/A", subject != null ? subject : "N/A"});
            return result;
        } else if (SchemaRegistryResourceOperation.SUBJECT_RESOURCE_OPERATIONS.contains(schemaRegistryResourceOperation)) {
            return this.authorizeSubjectOperation(user, subject, schemaRegistryResourceOperation, authorizeRequest);
        } else {
            return SchemaRegistryResourceOperation.GLOBAL_RESOURCE_OPERATIONS.contains(schemaRegistryResourceOperation) ? this.authorizeGlobalOperation(user, schemaRegistryResourceOperation, authorizeRequest) : false;
        }
    }
```

## authorizeSchemaIdLookup


```Java
public final boolean authorizeSchemaIdLookup(Principal principal,
                                                 SchemaRegistryResourceOperation schemaRegistryResourceOperation,
                                                 AuthorizeRequest authorizeRequest) throws AuthorizerException {
        UriInfo uriInfo = authorizeRequest.getContainerRequestContext().getUriInfo();
        if ("schemas/types".equals(uriInfo.getPath())) {
            return true;
        } else {
            String schemaId = uriInfo.getPathParameters().getFirst("id");

            Set<String> subjects;
            SchemaString schemaString;
            try {
                schemaString = this.schemaRegistry.get(Integer.parseInt(schemaId));
                if (schemaString == null) {
                    return false;
                }
                subjects = this.schemaRegistry.listSubjects();
            } catch (SchemaRegistryException e) {
                throw new AuthorizerException("Couldn't lookup schema ids ", e);
            }

            List<AuthorizeRequest> authorizeRequests = subjects.stream().filter((subject) -> {
                Schema schema = new Schema(subject, 0, 0, schemaString.getSchemaType(), schemaString.getReferences(), schemaString.getSchemaString());

                try {
                    return this.schemaRegistry.lookUpSchemaUnderSubject(subject, schema, true) != null;
                } catch (SchemaRegistryException var5) {
                    log.warn("Failed to lookup up schema {} under subject {}", schema, subject);
                    return false;
                }
            }).map((subject) -> subjectReadRequest(subject, authorizeRequest)).collect(Collectors.toList());
            return this.bulkAuthorize(principal, authorizeRequests).stream().anyMatch((b) -> b);
        }
    }
```
