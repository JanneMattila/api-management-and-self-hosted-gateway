# API Management and self-hosted gateway

API Management and self-hosted gateway related notes

## Scenario: Internal API Management

You deploy your API Management using these [instructions](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-internal-vnet).
You then want to use self-hosted gateway using that internal
API Management instance.

Next you'll create Windows VM to quickly test this setup:

![architecture](https://user-images.githubusercontent.com/2357647/104581271-01fe5f00-5667-11eb-9fd6-e5bf4393e893.png)

You update you Windows VM hosts file as per [documentation](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-internal-vnet#access-on-default-host-names),
so that you can reach the API Management from your VM. 

Example `C:\Windows\System32\drivers\etc\hosts`:

```text
172.16.3.5 <your-instance>.azure-api.net
172.16.3.5 <your-instance>.management.azure-api.net
172.16.3.5 <your-instance>.portal.azure-api.net
172.16.3.5 <your-instance>.developer.azure-api.net
172.16.3.5 <your-instance>.scm.azure-api.net
```

*Note*: If you do not set these private IPs then `<your-instance>.management.azure-api.net`
will resolve using the public IP (which is not  accessible).

Now you can test the following:

```rest
GET https://<your-instance>.azure-api.net/echo/resource?param1=sample HTTP/1.1
Ocp-Apim-Subscription-Key: 4d1a6...<your subscription key>...8a483
```

Above should work now correctly. 

Next you install [Docker Desktop](https://docs.docker.com/docker-for-windows/install/),
enable Kubernetes to it and you're ready to  run self-hosted gateway in that VM.

Copy the Gateway related configuration yaml contents from Azure Portal to local file in VM.
That file is slightly modified for demo purposes here [internal-mode/internalgw.yaml](internal-mode/internalgw.yaml).

**Note**: Here comes _critical_ piece: You need to update yaml file
to include `hostAliases` so that gateway container can get the correct IPs
for those API Management services:

```yaml
      hostAliases:
      - ip: "172.16.3.5"
        hostnames:
        - "<your-instance>.management.azure-api.net"
        - "<your-instance>.azure-api.net"
        - "<your-instance>.portal.azure-api.net"
        - "<your-instance>.developer.azure-api.net"
        - "<your-instance>.scm.azure-api.net"
```

*Note*: If you only want to use `docker run` for running your gateway (instead of Kubernetes),
then you can use [--add-host](https://docs.docker.com/engine/reference/commandline/run/)
switch for providing the host to ip mapping to the container. 

Deploy your gateway:

```cmd
kubectl apply -f internalgw.yaml

# Get NodePort 
kubectl get svc internalgw

# e.g. 80:32108/TCP,443:31022/TCP
```

Use port from above output and now your local gateway should respond succesfully:

```
GET https://localhost:31022/echo/resource?param1=sample HTTP/1.1
Ocp-Apim-Subscription-Key: 4d1a6...<your subscription key>...8a483
```
