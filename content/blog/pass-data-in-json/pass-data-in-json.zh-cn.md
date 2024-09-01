---
title: "如何在配置映射中以 JSON 格式传递数据到微服务"
date: "2023-05-25"
tags: ["kubernetes"]
---
当我们需要从 Helm Chart 向服务传递少量数据时，可以很容易地在 `deployment.yaml` 中使用 `container.env` 部分。这样我们只需在代码中使用 `os.getEnv()`就能获取我们想要的值。

但是如果我们有大量的值需要传递，将它们全部添加到 Helm Chart 中会很混乱。这时，我们可以使用配置映射。个人更喜欢使用 JSON 格式，因为它易于解组为结构体，并且可以轻松访问所有值。

以下是通过配置映射传递数据的具体步骤：

1. 使用 JSON 格式的数据设置一个配置映射。

    ```helm
    apiVersion: "v1"
    kind: "ConfigMap"
    metadata:
      name: json-configmap
    data:
      config.json: |
      {
          "key1": "value1",
          "key2": "value2"
      }
    ```

2. 在 Pod 中创建一个卷并将其挂载到我们想要的容器中。

    ```helm
    apiVersion: "apps/v1"
    kind: "Deployment"
    metadata:
      name: "test-service"
    spec:
      - template:
          metadata:
            volumes:
              - name: "json-config-volume"
                configMap:
                  // 确保名称与 configmap metadata.name 中的名称相同
                  name: "json-configmap"
          containers:
          - name: "container-first"
            volumeMounts:
              - name: "config-volume"
                mountPath: "/etc/config"
            // 不是必须的，你可以在代码中硬编码，这里放置只是为了确保
            // 如果有一天我们更改了挂载路径，确保也更改了环境变量。
            env:
              - name: "CONFIG_FILE"
                value: "/etc/config/config.json"
    ```

3. 在代码中，我们可以通过 `os.getEnv()`轻松访问 JSON 文件并对其进行解组。

    ```go
    // 基于 JSON 格式
    type JsonConfig struct{
        key1 string
        key2 string
    }

    func getJsonConfig() error{
        filePath, ok := os.LookupEnv("CONFIG_FILE")
        if !ok {
            return errors.New("failed to get file path")
        }

        config, err := os.Open(filePath)
        if err != nil {
            return errors.Wrap(err, "failed to open file")
        }
        defer config.Close()

        byteValue, err := io.ReadAll(config)
        if err != nil {
            return errors.Wrap(err, "failed to read file")
        }
        jsonConfig :=  &JsonConfig{}
        if err = json.Unmarshal(byteValue, jsonConfig); err != nil {
            return errors.Wrap(err, "failed to unmarshal file")
        }
        return nil
    }
    ```
