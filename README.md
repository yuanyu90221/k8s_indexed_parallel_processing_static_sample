# k8s_indexed_parallel_processing_static_sample

## 前言

參考文兼 https://kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/

今天要來實作 [Indexed Job for Parallel Processing with Static Work Assignment](https://kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/) 這個任務

這個任務的主要內容是透過 Index Job 來同步處理一個文字檔案內容

把內容讀出來在使用 rev 功能把內容字串反轉出來印到 console 上

## prequest

k8s api version 需要再 1.21 以上


## 佈署目標

1 建制一個 Index Job Deployment 來處理反轉字串的功能

2 使用 Downward API 來設定 Index

## 任務分析

### 取得完成任務的 Index

要達成這樣的設定有以下幾種作法

1 透過環境變數 JOB_COMPLETION_INDEX： Job Controller 會自動把這個變數帶入

2 讀取一個檔案包含這個完成的 Index

3 假設不能修改程式, 可以使用一個 script 從上面的方法中拿到 Index 然後帶入執行參數


## 建制一個 Index Job Deployment 來處理反轉字串的功能

建立 indexed-job.yaml 如下

```yaml=
apiVersion: batch/v1
kind: Job
metadata:
  name: 'indexed-job'
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      initContainers:
      - name: 'input'
        image: 'docker.io/libray/bash'
        command:
        - "bash"
        - "-c"
        - |
          items=(foo bar baz qux xyz)
          echo ${items[$JOB_COMPLETION_INDEX]} > /input/data.txt
        volumeMounts:
        - mountPath: /input
          name: input
      containers:
      - name: "workder"
        image: "docker.io/library/busybox"
        command:
        - "rev"
        - "/input/data.txt"
        volumeMounts:
        - mountPath: /input
          name: input
      volumes:
      - name: input
        emptyDir: {}
```

建立一個 Job 

名稱設定為 indexed-job

completions 設定為 5 代表要產生 5 個 Job 才能完成

parallelism 設定為 3 代表一次最多同時開啟 3 個 Job

initContainer 設定 image 使用 "docker.io/library/bash"

initContainer 設定執行指令如下

```bash=
bash -c items=(foo bar baz qux xyz) echo ${items[$JOB_COMPLETION_INDEX]} > /input/data.txt
```
initContainer 設定掛載路徑為 /input, 名稱為 input

container 設定 image 為 "docker.io/library/busybox"

cotainer 設定執行指令如下
```shell=
rev /input/data.txt
```

container 設定掛載路徑為 /input, 名稱為 input

注意的是 initContainer 用來做環境初始化

這邊透過 JOB_COMPLETION_INDEX 這個環境變數讓 initConatiner

把對應的值塞入分配到的 container 內

建制指令如下

```shell=
kubectl apply -f indexed-job.yaml
```

查看佈署結果使用以下指令

```shell=
kubectl describe jobs/indexed-job
```
![](https://i.imgur.com/0chXqOG.png)


查看執行內容

選取其中一個 Job 查看 log

```shell=
kubectl logs -f $job_name
```

![](https://i.imgur.com/nKqQnfe.png)


## 使用 Downward API 來設定 Index

另外一個作法是透過 Downward API 直接把 job-completion-index 當作 filecontent

設定 indexed-job-vol.yaml 如下

```yaml=
apiVersion: batch/v1
kind: Job
metadata:
  name: 'indexed-job
  annotations:
  
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: 'worker'
        image: 'docker.io/library/busybox'
        command:
        - "rev"
        - "/input/data.txt"
        volumeMounts:
        - mountPath: /input
          name: input
      volumes:
      - name: input
        downwardAPI:
          items:
          - path: "data.txt"
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
```

建立一個 Job 

名稱設定為 indexed-job

completions 設定為 5 代表要產生 5 個 Job 才能完成

parallelism 設定為 3 代表一次最多同時開啟 3 個 Job

container 設定 image 為 "docker.io/library/busybox"

cotainer 設定執行指令如下
```shell=
rev /input/data.txt
```

container 設定掛載路徑為 /input, 名稱為 input

不同的是這次的 data.txt 內容直接使用 Dowdward API 的方式

把 job-completion-index 當作檔案內容

建制指令如下

```shell=
kubectl apply -f indexed-job-vol.yaml
```

查詢執行 logs 結果如下：

![](https://i.imgur.com/lSWUqU6.png)


## 後記

這次使用 minikube 執行時, 一開始沒開啟 IndexJob API

導致 JOB_COMPLETION_INDEX 一直無法讀取到

直到筆者在 stackoverflow 查詢 minikube index job 這篇 [kubernetes-indexed-job-does-not-works](https://stackoverflow.com/questions/68958322/kubernetes-indexed-job-does-not-works)

才發現需要使用以下指令把 IndexJob 功能開啟

```shell=
minikube start --feature-gates=IndexedJob=true
```

寫到這邊有些感想

在實踐功能的時候, 檢查執行環境真的很重要

比如使用某版 go-ethereum client 要去連線 JSONRPC 就要 go1.15 以上

偏偏 debug log 在最後一行才出現版本限制訊息