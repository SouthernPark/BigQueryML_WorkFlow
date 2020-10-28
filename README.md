这个过程描述了 如何将BigQuery ML生成的模型 部署到docker内部， 并且把docker部署到k8s上，实现autoScale  
这个过程是 https://cloud.google.com/bigquery-ml/docs/export-model-tutorial 内容 和 https://www.tensorflow.org/tfx/serving/serving_kubernetes 内容的结合    
1. 先利用BigQuery创造一个 模型  
    (1) 先在BigQuery上创建一个名字为bqml_tutorial的数据库    
    (2) 利用BigQuery的 SQL语言训练模型  
    bq query --use_legacy_sql=false \
  'CREATE MODEL `bqml_tutorial.iris_model`
  OPTIONS (model_type="logistic_reg",
      max_iterations=10, input_label_cols=["species"])
  AS SELECT
    *
  FROM
    `bigquery-public-data.ml_datasets.iris`;'  
  
2. 在google云上 创建一个bucke 用于储存训练好的模型  
 
3. 将训练好的模型放在 bucket当中  
  bq extract -m bqml_tutorial.iris_model gs://<your_bucket_name>/iris_model  

4. 下载bucket中的模型到local host当中  
  (1) 先创建一个临时文件夹  
    mkdir tmp_dir  
  (2) 把bucket中的模型复制到临时文件夹当中  
    gsutil cp -r gs://<your_bucket_name>/iris_model tmp_dir  
    gsutil cp -r gs://bigqueryml-293607/iris_model tmp_dir 
    
5. 对模型进行版本编号  
  (1) 创建一个子文件夹名为1的文件夹(1代表版本编号为1)  
    mkdir -p serving_dir/iris_model/1   
  (2) 把临时文件夹中的模型复制到1文件夹中  
    cp -r tmp_dir/iris_model/* serving_dir/iris_model/1   
  (3) 删除临时文件夹  
    rm -r tmp_dir  
    
6. 将训练好的模型部署到tensorflow/serving的docker容器当中  
  (1) 从docker hub上pull 下来tensorflow/serving镜像  
    docker pull tensorflow/serving   
  (2) 启动tensorflow/serving 成为container, 重新命名container为serving_base     
    docker run -d --name serving_base tensorflow/serving   
  (3) 把训练好的model放入刚才启动的docker container当中    
    docker cp serving_dir/iris_model serving_base:/models/iris_model  
  (4) 改变container的环境变量Model_NAME属性，并且生成image iris_serving   
    docker commit --change "ENV MODEL_NAME iris_model" serving_base iris_serving  
  (5) 启动docker image 测试能否prediction(tensorflow serving 通过8501端口进行RESTful API响应)  
    docker run -p 8501:8501 -t iris_serving &  
    测试:  
      curl -d '{"instances": [{"sepal_length":5.0, "sepal_width":2.0, "petal_length":3.5, "petal_width":1.0}]}' -X POST http://localhost:8501/v1/models/iris_model:predict  
  (6) kill刚才启动的container, 防止与k8s端口重合   
    docker kill <container_id>  
    docker rm <container_id>  
    
 7. 将刚才生成的docker image 部署到k8s当中  
  (1) 将刚才生成的docker image push到谷歌的container registry 上面  
      a. 上传之前需要对image进行一定规则的tag  
        docker tag <you_image> gcr.io/<your_project_id>/<your_image>:v0.1.0  
      b.  上传你刚才tag的docker image  
        docker push <you-pre-taged-image>   
  (2) 部署k8s中  
      a. 需要先gcloud login  
        gcloud auth login --project <your_project_id>  
        按着console的提示完成验证操作  
      b. 设置地区(k8s根据地区创建节点)   
        gcloud config set compute/zone us-central1-b  
      c. Enable k8s API  
        https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster   
        点击上方网址, 点击文档中的Enabled Google Kubernetes Engine API 按钮  
      d. 创建一个节点数为3的k8s cluster  
        gcloud container clusters create <name_of_your_luster> --num-nodes 3  
      e. 设置默认cluster, 获得credential  
        gcloud config set container/cluster <your_cluster_name>  
        (如果你忘了cluster的名字可以通过: gcloud container clusters list 查看已经创建的cluster)  
        gcloud container clusters get-credentials <your_cluster_name>  
      f. 创建k8s deployment 利用 .yaml文件(该文件在github上，名为iris_k8s.yaml. 其他module适当改变名称即可以用)  
        I. 改变.yaml文件中的image的值, 为bucket中image的值.  
        II. 利用.yaml产生k8sdeployment  
            kubectl create -f iris_k8s.yaml   
  (3) 测试prediction   
      a. 找到网址(一开始url处显示pending, 多运行几次就好了)   
        kubectl get services    
      b. 测试  
        curl -d '{"instances": [{"sepal_length":5.0, "sepal_width":2.0, "petal_length":3.5, "petal_width":1.0}]}' -X POST http://<k8s url>:8501/v1/models/iris_model:predict  
      (如果失败很大一部分原因是: yaml file中的container image没有设置对)  
  
  8. 其他可能用到的指令  
    kubectl delete -f <yaml-file> 根据yaml-file关闭deployment  
    cp <file> <dir> 把一个文件copy到另一个文件夹  
    rm <file> 删除文件  
    rm -r <dir> 删除文件件  
    rm -rf <git_dir> 删除含有git的文件夹  
    gcloud container clusters list  查看已经创建的cluster  
    gcloud container clusters delete <cluster_name>  关闭cluster  
  9. 补充  
    由于已经上传了module到github, 所以从第6步开始就行  
    有时候在gcp上git不能push, 使用git push origin main  
     
  
    
  
  
  
  

