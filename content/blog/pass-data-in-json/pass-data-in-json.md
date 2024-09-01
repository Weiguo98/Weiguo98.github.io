---
title: "How to pass data in JSON format in config map to the microservice"
date: "2023-05-25"
# draft: true
tags: ["kubernetes"]
---
<!-- markdownlint-disable MD010 -->
 When we have little data need to pass from helm chart to service, it quit easy to use the `container.env` part in the `deployment.yaml`. So that we could just `os.getEnv()` in the code, we got the value we want.

 But what if we have a lot of values to pass, it is a mess to add them all in the helm chart. Then, we could use config map here. Personally, I prefer to use the JSON format, because it is easy to un-marshal to struct, and we can get easy access to all the values.

 Here are the exact steps to pass data through config map:

  1. Set up a config map with json format data.
  
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

  2. Create a volume in the pod and mount it to the container we want.

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
	                    // make sure the name equals to the name in configmap metadata.name
	                    name: "json-configmap"
	          containers:
	          - name: "container-first"
	            volumeMounts:
	                - name: "config-volume"
	                mountPath: "/etc/config"
	            // not necessary, you can hardcoded in the code base, put it here just make sure
	            // if anyday we changed the mountpath, make sure change the env as well.
	            env:
	                - name: "CONFIG_FILE"
	                value: "/etc/config/config.json"

	  ```

  3. In the code base, we can through `os.getEnv()` get easy access to the JSON file and do the un-marshal to it.

		```go
		// Based on the json format
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
