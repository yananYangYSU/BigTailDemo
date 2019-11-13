# HTTP interface of MLperf inference benchmark 

Based on the MLperf, we packaged the inference model into web services in addtion to the existing script operation mode in MLperf. And we also designed a load generator to send http requests for measurement. The web service is implemented using python flask+guicorn framework which can work in concurrent request scenarios. The load generator is implemented using java httpclient+threadpool that works in open-loop, and it has a web GUI for the realtime measurement of metrics (e.g., 99th tail-latency, QPS, RPS, etc.). Meanwhile, the load generator provides RMI interface for external call and can produce dynamically changeable request loads.

## Web inference
* currently support `mobilenet-coco300-tf` and `resnet-coco1200-tf` model inference, in continuous update... (easy to scale)
* support mutil-process server end in each GPU card
* provide docker image for fast build and source code for download 
## Load generator
* support thousands level concurrent request per second
* support web GUI for real-time view data
* support dynamiclly changeable QPS in open-loop
* support RMI interface for external call

# Building and installation
Next we need build the web service and load generator, respectively
## Part 1: Build inference web service
Here we use the mobilenet model as an example, which use coco300 dataset and tensorflow framework

### Step 1. Prepare the web service runtime environment
We provide the configured docker image for download (Recommended)
```Bash
$ docker pull ynyang/mobilenet-coco-flask:v1
```
In another way, you can use the Dockerfile to build image, the `Dockerfile` is showed as follows:

```
# Content of DockerFile
# install the runtime env and library
FROM tensorflow/tensorflow:1.13.1-gpu-py3-jupyter
MAINTAINER YananYang
RUN pip install pillow
RUN pip install opencv-python
RUN apt-get update && apt-get install -y libsm6  libxrender1 libxext-dev
RUN pip install Cython
RUN pip install pycocotools
RUN pip install gunicorn
RUN pip install flask gevent
rm -rf /var/lib/apt/lists/*
```
Command for building docker image using `Dockerfile` (Optional)
```bash
$ docker build -t ynyang/mobilenet-coco-flask:v1 .
```
### Step 2. Prepare the image dataset of coco

| dataset | picture download link | annotation download link |
| ---- | ---- | ----|
| coco (validation) | http://images.cocodataset.org/zips/val2017.zip | http://images.cocodataset.org/annotations/stuff_annotations_trainval2017.zip

Scale the coco dataset to 300pxx300px size. The scale tool is [here](../../../tools/upscale_coco).
You can run it for ssd-mobilenet like:
```bash
python upscale_coco.py --inputs /data/coco/ --outputs /data/coco-300 --size 300 300 --format png
```
The ready dataset and nanotation should be as follows:
```bash
$ ls /home/tank/yanan/data/coco-300/
annotations  val2017
```
The storage directory path can be optionally specify in your computer
### Step 3. Prepare the trained inference model
| model | framework | accuracy | dataset | model link | model source | precision | notes |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| ssd-mobilenet 300x300 quantized finetuned | tensorflow | mAP 0.23594 | coco resized to 300x300 | [from zenodo](https://zenodo.org/record/3252084/files/mobilenet_v1_ssd_8bit_finetuned.tar.gz) | Habana | int8 | ??? |

The ready trained inference model should be as follows:
```bash
$ ls /home/tank/yanan/model
ssd_mobilenet_v1_coco_2018_01_28.pb
```
### Step 4. Prepare the start script for web service
`start-service-mobilenet-coco300-tf.sh` is used to start the web service process of  inference model when container is created 

The content of `start-service-mobilenet-coco300-tf.sh`:
```bash
#!/bin/bash
cd /home/inference/v0.5/classification_and_detection/python/webservice
gunicorn -w 8 -b 0.0.0.0:5000 web_mobilenet_coco300_tf:app
```
-w: num of process in one GPU card to provide web service

The ready start script of service should be as follows:
```bash
$ ls /home/tank/yanan/script
start-service-mobilenet-coco300-tf.sh
```
### Step 5. Create docker container

We use the kubernetes to create the pod and service of inference, the container mounts host's files and directories to it's volume when it is created

The files and directories in host machine are showed as follows:
```python
/home/tank/yanan
          |----/data
          |    +----/coco-300
          |         |----/annotations/instances_val2017.json
          |         +----/val2017/00000xxxxx.jpg
          |----/model/ssd_mobilenet_v1_coco_2018_01_28.pb
          |----/script/start-service-mobilenet-coco300-tf.sh
          +----/inference/v0.5/classification_and_detection/*
```
The files and directories in container are showed as follows:
```python
/----/home/script/start-service-mobilenet-coco300-tf.sh
|    +----/inference/v0.5/classification_and_detection/*   
|
|----/data
|    +----/coco-300
|         |----/annotations/instances_val2017.json
|         +----/val2017/00000xxxxx.jpg
+----/model/ssd_mobilenet_v1_coco_2018_01_28.pb
```
Then use `create-pod.yaml` to create pod
```bash
$ kubectl create -f create-pod.yaml
pod/mobilenet-coco300-pod created
$ kubectl get pods -n yyn
NAME                     READY   STATUS    RESTARTS   AGE
mobilenet-coco300-pod0   1/1     Running   0          19s
```

The content of `create-pod.yaml` is showed as follows:
```python
apiVersion: v1                          # api version
kind: Pod                               # component type
metadata:
  name: mobilenet-coco300-pod0
  namespace: yyn
  labels:                               # label
    app: mobilenet-coco300-app-label
spec:
  nodeName: tank-node3                  # node that you want to deploy pod in
  containers:
  - name: mobilenet-coco300-flask-con      # container name
    image: ynyang/mobilenet-coco-flask:v1   # image
    imagePullPolicy: Never
    ports:
    - containerPort: 5000              # service port in container
    env:
    - name: CUDA_VISIBLE_DEVICES
      value: "0"                       # use GPU card 0 (e.g. 1,2)
    command: ["sh", "-c", "/home/script/start-service-mobilenet-coco300-tf.sh"]
    volumeMounts:                      # paths in container
    - name: model-path
      mountPath: /model
    - name: data-path
      mountPath: /data
    - name: code-path
      mountPath: /home/inference
    - name: script-path
      mountPath: /home/script
  volumes:  
  - name: model-path
    hostPath:                          # paths in host machine
      path: /home/tank/yanan/model
  - name: data-path
    hostPath:
      path: /home/tank/yanan/data
  - name: code-path
    hostPath:
      path: /home/tank/yanan/inference
  - name: script-path
    hostPath:
      path: /home/tank/yanan/script
```

After the pod is created, use `create-service.yaml` to create service
```bash
$ kubectl create -f create-service.yaml
pod/mobilenet-coco300-service created
$ kubectl get service -n yyn
NAME                        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
mobilenet-coco300-service   NodePort   10.105.250.36   <none>        5000:31500/TCP   5d6h
```

Test the web service is successfully started, `192.168.3.130` is the IP of node (tank-node3) that deployed the pod, which provides `31500` port map the pod 
```bash
$ curl 192.168.3.130:31500/gpu
ok
```
The print of "ok" means the webservice is available now, and we use `nvidia-smi -l 1` command can see the web inference processes has been created in node `192.168.3.130`

```bash
$ nvidia-smi -l 1
Wed Nov 13 22:16:48 2019
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 430.40       Driver Version: 430.40       CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce RTX 208...  Off  | 00000000:18:00.0 Off |                  N/A |
| 27%   26C    P8    15W / 250W |   5268MiB / 11019MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  GeForce RTX 208...  Off  | 00000000:3B:00.0 Off |                  N/A |
| 27%   24C    P8    19W / 250W |      0MiB / 11019MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   2  GeForce RTX 208...  Off  | 00000000:86:00.0 Off |                  N/A |
| 27%   24C    P8    22W / 250W |      0MiB / 11019MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0     38812      C   /usr/bin/python3                             657MiB |
|    0     38825      C   /usr/bin/python3                             657MiB |
|    0     38827      C   /usr/bin/python3                             657MiB |
|    0     38830      C   /usr/bin/python3                             657MiB |
|    0     38894      C   /usr/bin/python3                             657MiB |
|    0     38977      C   /usr/bin/python3                             657MiB |
|    0     39025      C   /usr/bin/python3                             657MiB |
|    0     39089      C   /usr/bin/python3                             657MiB |
+-----------------------------------------------------------------------------+
```

## Part 2: Build load generator
The load generator is writen in Java, it can be deployed in container or host machine, and we need install Java JDK, apache tomcat before using it 


### Step 1: Install Java JDK
Download java jdk and install to /usr/local/java/
```bash
$ wget https://download.oracle.com/otn/java/jdk/8u231-b11/5b13a193868b4bf28bcb45c792fce896/jdk-8u231-linux-x64.tar.gz
$ tar -zxvf jdk-8u231-linux-x64.tar.gz /usr/local/java/
```
Modify the `/etc/profile` file
```bash
$ vi /erc/profile
```
Config Java environment variables, append the following content into the file
```bash
export JAVA_HOME=/usr/local/java/jdk1.8.0_231
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=$PATH:${JAVA_HOME}/bin 
```
Enable the configuration
```
$ source /etc/profile
$ java -version
java version "1.8.0_231"
Java(TM) SE Runtime Environment (build 1.8.0_231-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.231-b12, mixed mode)
```

### Step 2: Install apache tomcat
Download java jdk and install to /usr/local/java/
```bash
$ http://mirrors.tuna.tsinghua.edu.cn/apache/tomcat/tomcat-8/v8.5.47/bin/apache-tomcat-8.5.47.tar.gz
$ tar -zxvf apache-tomcat-8.5.47.tar.gz /usr/local/tomcat/
```
### Step 3: Deploy the load generator to tomcat
Modify the configuration file in `loadGen`
```bash
$ vi LoadGen/src/main/resources/conf/sys.properties
```
The content of `sys.properties` is showed as follows:
```python
# URL of web inference service that can be accessed by http 
imageClassifyBaseURL=http://192.168.3.130:31500/gpu  
# node IP that deployed load generator
serverIp=192.168.3.110     
# the port of RMI service provided by load generator 
rmiPort=2222
# window size of latency recorder, which can be seen from GUI page
windowSize=60
# record interval of latency, default: 1000ms
recordInterval=1000
```
Build the source code into war package (e.g., uses `jar` or `eclipse IDE`)
```bash
$ java cvf sdcloud.war loadGen/*
```

Deploy the web package into tomcat
```bash
$ mv sdcloud.war /usr/local/tomcat/apache-tomcat-8.5.47/webapp
$ /usr/local/tomcat/apache-tomcat-8.5.47/bin/startup.sh
```

### Step 4: Test the load generator
Open the exlporer and visit url `http://192.168.3.110:8080/sdcloud`
