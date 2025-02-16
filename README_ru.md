# Плагин Time-Slicing GPUs в Kubernetes
## Введение

NVIDIA [GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html) позволяет чередовать рабочие нагрузки на **GPU с избыточной подпиской** с помощью набора расширенных опций для NVIDIA Kubernetes Device Plugin. Внутри системы используется механизм шеринга GPU — Time-Slicing, который позволяет нагрузкам, работающим на GPU с избыточной подпиской, выполняться параллельно. В этой статье рассматриваются способы активации этой функции в Managed Service for Kubernetes с использованием GPU Operator.

Этот механизм тайм-шеринга GPU в Kubernetes позволяет системному администратору определить набор реплик для GPU, которые могут быть назначены отдельным подам для выполнения рабочих нагрузок. В отличие от [MIG](https://www.google.com/url?q=https://docs.google.com/document/d/1mdgMQ8g7WmaI_XVVRrCvHPFPOMCm5LQD5JefgAh6N8g/edit&sa=D&source=editors&ust=1655578433019961&usg=AOvVaw1F-OezvM-Svwr1lLsdQmu1) (Multi-Instance GPU) между репликами отсутствует изоляция памяти и защита от сбоев, однако для некоторых рабочих нагрузок это может быть более эффективным решением, чем отсутствие возможности разделения вовсе. Шеринг GPU с помощью Time-Slicing используется для мультиплексирования задач с разных реплик одного и того же базового GPU.

Подробнее см. в [официальной документации](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/gpu-sharing.html#configuration-for-shared-access-to-gpus-with-gpu-time-slicing).

## Начало работы


[Добавьте группу узлов](https://yandex.cloud/en/docs/managed-kubernetes/operations/node-group/node-group-create) с GPU NVIDIA T4.

Передайте конфигурацию Time-Slicing для NVIDIA Kubernetes Device Plugin через ConfigMap:
```
kubectl create -f time-slicing-config.yaml
```
Установите GPU Operator:
```
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia \
   && helm repo update \
   && helm install gpu-operator nvidia/gpu-operator \
     -n gpu-operator \
     --set devicePlugin.config.name=time-slicing-config
```
Конфигурацию Time-Slicing можно применить как на уровне кластера, так и на уровне узла. По умолчанию GPU Operator не применяет конфигурацию Time-Slicing к узлам GPU в кластере. Значение по умолчанию можно использовать с параметром `devicePlugin.config.default=<config-name>` для каждой ClusterPolicy: 
```
kubectl patch clusterpolicies.nvidia.com/cluster-policy \
   -n gpu-operator --type merge \
   -p '{"spec": {"devicePlugin": {"config": {"name": "time-slicing-config", "default": "tesla-t4"}}}}'

```
Или (для отдельных узлов с меткой узла):

 
```
yc managed-kubernetes node-group add-labels <NODE-GROUP-NAME>|<NODE-GROUP-ID> --labels nvidia.com/device-plugin.config=tesla-t4
```


## Тестирование функции Time-Slicing GPU с помощью NVIDIA GPU Operator

Выполните развертывание с несколькими репликами:
```
kubectl apply -f nvidia-plugin-test.yml
```
Убедитесь, что все 5 реплик работают:
```
kubectl get pods
```
Выполните проверку через `nvidia-smi`:
```
kubectl exec <nvidia-container-toolkit-name> -n gpu-operator -- nvidia-smi
```
Ожидаемый результат:
```

Defaulted container "nvidia-container-toolkit-ctr" out of: nvidia-container-toolkit-ctr, driver-validation (init)
Thu Jan 26 09:42:51 2023       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.65.01    Driver Version: 515.65.01    CUDA Version: N/A      |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla T4            Off  | 00000000:8B:00.0 Off |                    0 |
| N/A   72C    P0    70W /  70W |   1579MiB / 15360MiB |    100%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A     43108      C   /usr/bin/dcgmproftester11         315MiB |
|    0   N/A  N/A     43211      C   /usr/bin/dcgmproftester11         315MiB |
|    0   N/A  N/A     44583      C   /usr/bin/dcgmproftester11         315MiB |
|    0   N/A  N/A     44589      C   /usr/bin/dcgmproftester11         315MiB |
|    0   N/A  N/A     44595      C   /usr/bin/dcgmproftester11         315MiB |
+-----------------------------------------------------------------------------+
```
