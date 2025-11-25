# Домашнее задание к занятию «Хранение в K8s»

------

## Задание 1. Volume: обмен данными между контейнерами в поде
### Задача

Создать Deployment приложения, состоящего из двух контейнеров, обменивающихся данными.

### Шаги выполнения
1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.
2. Настроить busybox на запись данных каждые 5 секунд в некий файл в общей директории.
3. Обеспечить возможность чтения файла контейнером multitool.


### Что сдать на проверку
- Манифесты:
  - `containers-data-exchange.yaml`
- Скриншоты:
  - описание пода с контейнерами (`kubectl describe pods data-exchange`)
  - вывод команды чтения файла (`tail -f <имя общего файла>`)

------

## Задание 2. PV, PVC
### Задача
Создать Deployment приложения, использующего локальный PV, созданный вручную.

### Шаги выполнения
1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool, использующего созданный ранее PVC
2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который busybox записывает данные каждые 5 секунд. 
4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему. (Используйте команду `kubectl describe pv`).
5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать, что произошло с файлом после удаления PV. Пояснить, почему.


### Что сдать на проверку
- Манифесты:
  - `pv-pvc.yaml`
- Скриншоты:
  - каждый шаг выполнения задания, начиная с шага 2.
- Описания:
  - объяснение наблюдаемого поведения ресурсов в двух последних шагах.

------

## Задание 3. StorageClass
### Задача
Создать Deployment приложения, использующего PVC, созданный на основе StorageClass.

### Шаги выполнения

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool, использующего созданный ранее PVC.
2. Создать SC и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который busybox записывает данные каждые 5 секунд.

### Что сдать на проверку
- Манифесты:
  - `sc.yaml`
- Скриншоты:
  - каждый шаг выполнения задания, начиная с шага 2
---

## Решение 1. Volume: обмен данными между контейнерами в поде
### Задача

Создать Deployment приложения, состоящего из двух контейнеров, обменивающихся данными.

### Шаги выполнения
1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.
2. Настроить busybox на запись данных каждые 5 секунд в некий файл в общей директории.
3. Обеспечить возможность чтения файла контейнером multitool.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-exchange
  template:
    metadata:
      labels:
        app: data-exchange
    spec:
      containers:
      - name: busybox-writer
        image: busybox:1.36.1
        command: ["/bin/sh", "-c"]
        args:
          - |
            # каждую 5 секунд дописываем текущее время в общий файл
            mkdir -p /shared
            while true; do date >> /shared/data.txt; sleep 5; done
        volumeMounts:
        - name: shared-data
          mountPath: /shared
      - name: multitool
        image: praqma/network-multitool:latest
        command: ["/bin/sh", "-c"]
        args:
          - |
            # убедимся, что файл существует, затем читаем его в режиме follow
            mkdir -p /shared
            touch /shared/data.txt
            tail -f /shared/data.txt
        volumeMounts:
        - name: shared-data
          mountPath: /shared
      volumes:
      - name: shared-data
        emptyDir: {}

```

- Получить имя пода 

```bash
POD=$(kubectl get pods -l app=data-exchange -o jsonpath="{.items[0].metadata.name}"
```
![scr1](https://github.com/vladrabbit/K8S-4/blob/main/SCR/%D0%A1%D0%BD%D0%B8%D0%BC%D0%BE%D0%BA%20%D1%8D%D0%BA%D1%80%D0%B0%D0%BD%D0%B0%202025-11-24%20%D0%B2%2015.27.19.png)

- описание пода с контейнерами (`kubectl describe pods data-exchange`)

![scr2](https://github.com/vladrabbit/K8S-4/blob/main/SCR/%D0%A1%D0%BD%D0%B8%D0%BC%D0%BE%D0%BA%20%D1%8D%D0%BA%D1%80%D0%B0%D0%BD%D0%B0%202025-11-23%20%D0%B2%2018.51.18.png)

![scr3](https://github.com/vladrabbit/K8S-4/blob/main/SCR/%D0%A1%D0%BD%D0%B8%D0%BC%D0%BE%D0%BA%20%D1%8D%D0%BA%D1%80%D0%B0%D0%BD%D0%B0%202025-11-23%20%D0%B2%2018.51.39.png)

- вывод команды чтения файла (`tail -f <имя общего файла>`)

![scr4](https://github.com/vladrabbit/K8S-4/blob/main/SCR/%D0%A1%D0%BD%D0%B8%D0%BC%D0%BE%D0%BA%20%D1%8D%D0%BA%D1%80%D0%B0%D0%BD%D0%B0%202025-11-23%20%D0%B2%2018.52.48.png)


## Решение 2. PV, PVC


### Шаги выполнения
1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool, использующего созданный ранее PVC
2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который busybox записывает данные каждые 5 секунд. 
4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему. (Используйте команду `kubectl describe pv`).
5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать, что произошло с файлом после удаления PV. Пояснить, почему.


### Что сдать на проверку
- Манифесты:
  - `pv-pvc.yaml`

```yaml
  apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange-pvc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-exchange-pvc
  template:
    metadata:
      labels:
        app: data-exchange-pvc
    spec:
      # Под запустится на ноде с меткой storage=local1
      nodeSelector:
        storage: local1
      containers:
      - name: busybox-writer
        image: busybox:1.36.1
        command: ["/bin/sh", "-c"]
        args:
          - |
            mkdir -p /data
            # каждую 5 секунд дописываем дату во /data/data.txt
            while true; do date >> /data/data.txt; sleep 5; done
        volumeMounts:
        - name: shared-pvc
          mountPath: /data
      - name: multitool
        image: praqma/network-multitool:latest
        command: ["/bin/sh", "-c"]
        args:
          - |
            mkdir -p /data
            touch /data/data.txt
            tail -f /data/data.txt
        volumeMounts:
        - name: shared-pvc
          mountPath: /data
      volumes:
      - name: shared-pvc
        persistentVolumeClaim:
          claimName: pvc-local


  ```

- Скриншоты:
  - каждый шаг выполнения задания, начиная с шага 2.

    - Создание/описание PV/PVC
      ![scr1](https://github.com/vladrabbit/K8S-4/blob/main/SCR/2-1.1.png)

    - Описание POD

      ```bash
      POD=$(kubectl get pods -l app=data-exchange-pvc -o jsonpath="{.items[0].metadata.name}")
      ```

      ![scr2](https://github.com/vladrabbit/K8S-4/blob/main/SCR/2-2.1.png)
      ![scr3](https://github.com/vladrabbit/K8S-4/blob/main/SCR/2-2.2.png)

    - Вывод чтения файла

      ![scr4](https://github.com/vladrabbit/K8S-4/blob/main/SCR/2-3.png)

    - Скриншот kubectl describe pv pv-local после удаления PVC

      ![scr5](https://github.com/vladrabbit/K8S-4/blob/main/SCR/2-4.png)

    - Объяснение поведения
      
      - В манифесте persistentVolumeReclaimPolicy: Retain. Это значит:

        1. При удалении PVC Kubernetes не удаляет автоматически данные на хосте и не удаляет объект PV. Вместо этого PV перейдёт в состояние Released, но контент на диске останется нетронутым.

        2. Поэтому kubectl describe pv pv-local покажет, что PV Released (или Available/Bound). 

        3. Причина: Retain — политика сохранения данных, чтобы администратор мог вручную очистить или перенести данные перед удалением PV.

    - Скриншот содержимого директории на ноде

      ![scr6](https://github.com/vladrabbit/K8S-4/blob/main/SCR/2-5.png)

    - Скриншот содержимого директории на ноде после удаления PV

      ![scr7](https://github.com/vladrabbit/K8S-4/blob/main/SCR/2-5.1.png)

    - Объяснение
      - hostPath — это просто путь на хосте. 
      Удаление объекта PV в Kubernetes не удаляет файлы в этой директории. 
      PersistentVolume — это мета-объект, управляющий привязкой; реальное содержимое находится в файловой системе ноды.


      - Поскольку persistentVolumeReclaimPolicy: Retain, и это hostPath,
       Kubernetes не будет удалять данные на диске ни при удалении PVC, 
       ни при удалении PV (удаление PV — только удаление объекта в Kubernetes API). 
       Поэтому файл останется в /mnt/local-pv/data.txt до тех пор, пока не удалить его вручную.


## Решение 3. StorageClass

### Шаги выполнения

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool, использующего созданный ранее PVC.

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-local
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
# Static PV which will be bound to PVC created with storageClassName: sc-local
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-sc-local
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: sc-local
  hostPath:
    path: /mnt/sc-pv
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: storage
          operator: In
          values:
          - local1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-sc
spec:
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: sc-local
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange-sc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-exchange-sc
  template:
    metadata:
      labels:
        app: data-exchange-sc
    spec:
      nodeSelector:
        storage: local1
      containers:
      - name: busybox-writer
        image: busybox:1.36.1
        command: ["/bin/sh", "-c"]
        args:
          - |
            mkdir -p /data
            # append current date every 5 seconds into /data/data.txt
            while true; do date >> /data/data.txt; sleep 5; done
        volumeMounts:
        - name: sc-storage
          mountPath: /data
      - name: multitool
        image: praqma/network-multitool:latest
        command: ["/bin/sh", "-c"]
        args:
          - |
            mkdir -p /data
            touch /data/data.txt
            tail -f /data/data.txt
        volumeMounts:
        - name: sc-storage
          mountPath: /data
      volumes:
      - name: sc-storage
        persistentVolumeClaim:
          claimName: pvc-sc

```
2. Создать SC и PVC для подключения папки на локальной ноде, которая будет использована в поде.

  - создание директории на ноде

    ![scr1](https://github.com/vladrabbit/K8S-4/blob/main/SCR/11.png)

  - назначение label на ноде описана в манифесте

    ![scr2](https://github.com/vladrabbit/K8S-4/blob/main/SCR/12.png)

  - проверка состояния 

    ![scr3](https://github.com/vladrabbit/K8S-4/blob/main/SCR/21.png)

  - проверка того, что происходит запись в файл на ноде после применения манифеста

    ![scr4](https://github.com/vladrabbit/K8S-4/blob/main/SCR/22.png)



3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который busybox записывает данные каждые 5 секунд.
  
  - назначение переменной значения = имени пода

    ![scr5](https://github.com/vladrabbit/K8S-4/blob/main/SCR/31.png)

  - информация о поде

    ![scr6](https://github.com/vladrabbit/K8S-4/blob/main/SCR/32.png)
    ![scr7](https://github.com/vladrabbit/K8S-4/blob/main/SCR/33.png)

  - проверка чтения из файла

    ![scr8](https://github.com/vladrabbit/K8S-4/blob/main/SCR/34.png)



------


