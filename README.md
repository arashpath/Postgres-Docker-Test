# Implementing High Available Clusters for PostgresSQL with Docker Swarm

## Proposed Solutions
### [PostDock](./PostDock)
    - Original GitRepo : https://github.com/paunin/PostDock 
    - DockerHub : https://hub.docker.com/u/postdock 
### [CrunchyData](./CrunchyData)
    - Official GitRepo : https://github.com/CrunchyData/crunchy-containers
    - DockerHub : https://hub.docker.com/u/crunchydata

----
## Portainer
```bash
docker run -d -p 9000:9000 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data portainer/portainer
```

