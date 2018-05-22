# Airflow
## Use docker-compose deploy airflow cluster

### 1、进入airflow_master文件夹，启动master
   'docker-compose -f docker-compose-CeleryExecutor-master.yml up -d'
 
 步骤1会启动airflow的master集群服务，其中启动的有：Redis、postgresql、webserver、flower和scheduler。
   
### 2、在另外一台机器上，启动worker
   > docker-compose -f docker-compose-CeleryExecutor-worker.yml up -d
 
 步骤2只会在机器上启动一个worker节点，然后根据文件中的配置链接到master上，在master节点上可通过flower查看worker的状态。
