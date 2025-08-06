# Burrow 单实例 Notifier 功能

## 问题背景

Burrow 的 notifier 功能默认使用 Zookeeper 分布式锁来确保在多个 Burrow 实例运行时，只有一个实例发送通知。这对于单实例部署来说是不必要的。

## 解决方案

我们添加了一个配置选项 `skip-zookeeper-lock` 来跳过 Zookeeper 分布式锁验证，允许在单实例部署中使用 notifier 功能。

## 配置方法

在你的 Burrow 配置文件中，为 notifier 配置添加 `skip-zookeeper-lock=true`：

```toml
[notifier.default]
class-name="http"
url-open="http://your-api-endpoint"
interval=60
timeout=5
keepalive=30
extras={ api_key="your-api-key", app="burrow", tier="PROD" }
template-open="config/your-template.tmpl"
method-close="DELETE"
send-close=false
threshold=1
skip-zookeeper-lock=true  # 添加这一行
```

## 代码修改

### 1. core/internal/notifier/coordinator.go

修改了 `manageEvalLoop()` 方法，添加了对 `skip-zookeeper-lock` 配置的检查：

```go
func (nc *Coordinator) manageEvalLoop() {
    // Check if we should skip zookeeper lock (for single instance deployments)
    skipZkLock := false
    for name := range viper.GetStringMap("notifier") {
        if viper.GetBool("notifier." + name + ".skip-zookeeper-lock") {
            skipZkLock = true
            break
        }
    }
    
    if skipZkLock {
        // Skip zookeeper lock for single instance deployments
        nc.Log.Info("skipping zookeeper lock for single instance deployment")
        nc.doEvaluations = true
        nc.running.Add(1)
        go nc.sendEvaluatorRequests()
        
        // Wait for shutdown signal
        <-nc.quitChannel
        return
    }
    
    // Original distributed lock logic for multi-instance deployments
    // ...
}
```

### 2. core/burrow.go

修改了 `newCoordinators()` 方法，当检测到 `skip-zookeeper-lock=true` 时，不加载 Zookeeper coordinator：

```go
func newCoordinators(app *protocol.ApplicationContext) []protocol.Coordinator {
    // ...
    haveNotifiers := viper.IsSet("notifier")

    // Only include zookeeper if we have dependant coordinators and not skipping zookeeper lock
    if haveNotifiers {
        skipZkLock := false
        for name := range viper.GetStringMap("notifier") {
            if viper.GetBool("notifier." + name + ".skip-zookeeper-lock") {
                skipZkLock = true
                break
            }
        }
        
        if !skipZkLock {
            coordinators = append(coordinators,
                &zookeeper.Coordinator{
                    App: app,
                    Log: app.Logger.With(
                        zap.String("type", "coordinator"),
                        zap.String("name", "zookeeper"),
                    ),
                },
            )
        }
    }
    // ...
}
```

## 使用场景

这个修改适用于以下场景：

1. **单实例部署**：一个 Kafka 集群对应一个 Burrow 实例
2. **不需要 Zookeeper**：Kafka 4.0+ 使用 KRaft 模式，不再依赖 Zookeeper
3. **简化部署**：避免部署额外的 Zookeeper 实例

## 注意事项

1. **仅适用于单实例部署**：如果你的环境中有多个 Burrow 实例，请不要使用此配置，否则会导致重复通知
2. **向后兼容**：此修改完全向后兼容，不影响现有的多实例部署
3. **配置验证**：确保你的 notifier 配置正确，特别是模板文件和 API 端点

## 测试

编译并测试修改后的 Burrow：

```bash
go build -o burrow .
./burrow -config-dir=config
```

如果配置正确，你应该看到类似以下的日志：

```
INFO skipping zookeeper lock for single instance deployment
```

而不是之前的 Zookeeper 错误。 

## 编译
```bash
GOOS=linux GOARCH=amd64 go build -o burrow-linux-amd64 .
file burrow-linux-amd64
#从file命令的输出可以看到：
#ELF 64-bit LSB executable：Linux可执行文件
#x86-64：64位x86架构
#statically linked：静态链接
#with debug_info, not stripped：包含调试信息

# 编译多个架构
GOOS=linux GOARCH=amd64 go build -o burrow-linux-amd64 .
GOOS=linux GOARCH=arm64 go build -o burrow-linux-arm64 .
GOOS=darwin GOARCH=amd64 go build -o burrow-darwin-amd64 .
GOOS=darwin GOARCH=arm64 go build -o burrow-darwin-arm64 .
```