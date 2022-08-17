Run the following to build the metricbeat image for the metricbeat user-workloads deployment:

[Link](https://www.elastic.co/guide/en/beats/metricbeat/current/setup-repositories.html)


```bash
sudo -i

podman build . -t metricbeat-user-workloads:latest

# In case you are using remote registry:
podman login <REGISTRY>

podman tag metricbeat-user-workloads:latest <REGISTRY>/<REPO>/metricbeat-user-workloads:latest

podman push <REGISTRY>/<REPO>/metricbeat-user-workloads:latest
```
