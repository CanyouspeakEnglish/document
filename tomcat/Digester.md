# Digester

| 方法             | 参数                                                         |                                                 |                          |                    规则                     |
| ---------------- | :----------------------------------------------------------- | :---------------------------------------------: | ------------------------ | :-----------------------------------------: |
| addObjectCreate  | Server                                                       |                 StandardServer                  | className                |              ObjectCreateRule               |
| addSetProperties | Server                                                       |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server                                                       |                    setServer                    | Server                   |                 SetNextRule                 |
| addObjectCreate  | Server/GlobalNamingResources                                 |                                                 |                          |              ObjectCreateRule               |
| addSetProperties | Server/GlobalNamingResources                                 |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/GlobalNamingResources                                 |            setGlobalNamingResources             | NamingResourcesImpl      |              ObjectCreateRule               |
| addRule          | Server/Listener                                              |                                                 | className                |             ListenerCreateRule              |
| addSetProperties | Server/Listener                                              |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/Listener                                              |              addLifecycleListener               | LifecycleListener        |                 SetNextRule                 |
| addObjectCreate  | Server/Service                                               |                 StandardService                 | className                |              ObjectCreateRule               |
| addSetProperties | Server/Service                                               |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service                                               |                   addService                    | Service                  |                 SetNextRule                 |
| addObjectCreate  | Server/Service/Listener                                      |                                                 | className                |              ObjectCreateRule               |
| addSetProperties | Server/Service/Listener                                      |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service/Listener                                      |              addLifecycleListener               | LifecycleListener        |                 SetNextRule                 |
| addObjectCreate  | Server/Service/Executor                                      |             StandardThreadExecutor              | className                |              ObjectCreateRule               |
| addSetProperties | Server/Service/Executor                                      |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service/Executor                                      |                   addExecutor                   | Executor                 |                 SetNextRule                 |
| addRule          | Server/Service/Connector                                     |                                                 |                          |             ConnectorCreateRule             |
| addSetProperties | Server/Service/Connector                                     | "executor", "sslImplementationName", "protocol" |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service/Connector                                     |                  addConnector                   | Connector                |                 SetNextRule                 |
| addRule          | Server/Service/Connector                                     |                                                 |                          |              AddPortOffsetRule              |
| addObjectCreate  | Server/Service/Connector/SSLHostConfig                       |                  SSLHostConfig                  |                          |              ObjectCreateRule               |
| addSetProperties | Server/Service/Connector/SSLHostConfig                       |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service/Connector/SSLHostConfig                       |                addSslHostConfig                 | SSLHostConfig            |                 SetNextRule                 |
| addRule          | Server/Service/Connector/SSLHostConfig/Certificate           |                                                 |                          |            CertificateCreateRule            |
| addSetProperties | Server/Service/Connector/SSLHostConfig/Certificate           |                      type                       |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service/Connector/SSLHostConfig/Certificate           |                 addCertificate                  | SSLHostConfigCertificate |                 SetNextRule                 |
| addObjectCreate  | Server/Service/Connector/SSLHostConfig/OpenSSLConf           |                   OpenSSLConf                   |                          |              ObjectCreateRule               |
| addSetProperties | Server/Service/Connector/SSLHostConfig/OpenSSLConf           |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service/Connector/SSLHostConfig/OpenSSLConf           |                 setOpenSslConf                  | OpenSSLConf              |           SetNextRuleOpenSSLConf            |
| addObjectCreate  | Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd |                 OpenSSLConfCmd                  |                          |              ObjectCreateRule               |
| addSetProperties | Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd |                     addCmd                      | OpenSSLConfCmd           |                 SetNextRule                 |
| addObjectCreate  | Server/Service/Connector/Listener                            |                    className                    |                          |              ObjectCreateRule               |
| addSetProperties | Server/Service/Connector/Listener                            |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service/Connector/Listener                            |              addLifecycleListener               | LifecycleListener        |                 SetNextRule                 |
| addObjectCreate  | Server/Service/Connector/UpgradeProtocol                     |                    className                    |                          |              ObjectCreateRule               |
| addSetProperties | Server/Service/Connector/UpgradeProtocol                     |                                                 |                          |              SetPropertiesRule              |
| addSetNext       | Server/Service/Connector/UpgradeProtocol                     |               addUpgradeProtocol                | UpgradeProtocol          |                 SetNextRule                 |
| addRule          | Server/Service/Engine                                        |                                                 |                          | SetParentClassLoaderRule(parentClassLoader) |

## RuleSet

**这个是为了解决子标签的创建赋值等关系**

```java
NamingRuleSet("Server/GlobalNamingResources/")
EngineRuleSet("Server/Service/")
HostRuleSet("Server/Service/Engine/")
ContextRuleSet("Server/Service/Engine/Host/")    
ClusterRuleSet("Server/Service/Engine/Host/Cluster/")
NamingRuleSet("Server/Service/Engine/Host/Context/")
ClusterRuleSet("Server/Service/Engine/Cluster/")
//统一调用 addRuleInstances() 设置规则
```

