# Note
## Install & use Docker + Postgres + TablePlus to create DB schema
* easy read :https://hackmd.io/@byETi8KzTUyys88QkasF2w/Sk_1uaXqo
1. install postgres in docker, 15-alpine is the postgres version
:::info
docker pull < image>:<tag>
:::
```bash=
docker pull postgres:15-alpine
```
![](https://i.imgur.com/4aVLMFv.png)
    
2. start a container
:::info
docker run --name <container_name> 
-e <environment_variable>
-p <host_ports:container_ports>
        -d
< image>:<tag>
:::

* a container is 1 instance of the application contained in the image, which is started by the docker run command. We can start multiple containers from 1 single images
    
```
docker run --name postgres15 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres:15-alpine
```

![](https://i.imgur.com/td42dle.png)
    
3. Run command in container
:::info
docker exec -it <container_name_or_id >
    <command> [args]
:::

```
docker exec -it postgres15 psql -U root
```
![](https://i.imgur.com/jyDEitJ.png)

4. View container logs
:::info
docker logs
<container_name_or_id>
:::
see the information logs in the container you created
```
docker logs postgres15
```

5. TablePlus
![](https://i.imgur.com/gCqga9x.png)
press "Create a new connection..."
    
![](https://i.imgur.com/Pba3rpc.png)
* password = secret, test passed
![](https://i.imgur.com/gD9QoGP.png)
    
* open simple_bank.sql we created at dbdiagram in tablePlus and press [command + enter]
![](https://i.imgur.com/O8BC87B.png)
* it creates the tables
![](https://i.imgur.com/kYI1aD3.png)