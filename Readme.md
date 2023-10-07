# Run Pytorch Workload on Kubernetes
This repository contains a simple PyTorch model training in [main.py](src%2Fmain.py) based on the [PyTorch quickstart guide](https://pytorch.org/tutorials/beginner/basics/quickstart_tutorial.html). 
The repository also covers the deployment on Docker and Kubernetes. 
Here, the Docker image [bitnami/pytorch](https://hub.docker.com/r/bitnami/pytorch) (version 2.0.1) is used. 
The goal is to provide a proof of concept for running real-world Kubernetes workloads on certain Kubernetes clusters.


The first execution downloads the dataset to the directory `data` and executes the training. 
After that, following executions skip the downloading and uses the existing dataset.
The training runs for certain epochs (and therefore a certain time. 

## Setup
1. Clone this repository 
2. Run the training either using Docker or Kubernetes
3. When the training is finished, the model `model.pth` is stored in the directory `out`

### Run using Docker
To run the training using Docker, run the following from the root of this project:
```bash
docker run -it --rm --name pytorch --user $UID -v $PWD:/app bitnami/pytorch:2.0.1 python src/main.py
```

### Run using Kubernetes
To run the training using Kubernetes, follow following instructions.

For testing purpose, you can create a [Kind](https://github.com/kubernetes-sigs/kind) Kubernetes cluster.
Use the [kind-config.yaml](kind-config.yaml) 
that creates a mapping for the current directory into the `/app` directory inside the cluster container.
```bash
kind create cluster --wait 60s --config ./kind-config.yaml
```

Then, run the following from the root of this project.

> In case of running in a Kubernetes cluster without Kind, 
> you need to adjust the volume `project-vol` mentioned below.

```bash
kubectl create namespace pytorch

kubectl create -n pytorch -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: pytorch-example
spec:
  template:
    spec:
      securityContext:
        runAsUser: $UID
      containers:
      - name: pytorch
        image: bitnami/pytorch:2.0.1
        command: ['python', '/app/src/main.py']
        volumeMounts:
          - name: project-vol
            mountPath: /app
      restartPolicy: OnFailure
      volumes:
        - name: project-vol
          hostPath:
            path: /app
            type: Directory
EOF

# wait for training to finish
kubectl wait --for=condition=complete --timeout=10h job/pytorch-example -n pytorch

kubectl logs -n pytorch job/pytorch-example
```
