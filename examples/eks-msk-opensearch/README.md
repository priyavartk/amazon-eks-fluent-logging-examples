## Multi-Tenant logging using Fluent-bit, EKS , Amazon MSK and  Amazon OpenSearch

In this example, we will showcase how to send your application logs from EKS to OpenSearch via Amazon MSK (managed KAFKA). We will use tenant notion using EKS "namespace" such that each tenant has a seperate namespace.  The solution helps to meet following use cases in a multi-tenant deployment from EKS cluster.

* Use of Kafka as broker will de-couple log forwarding directly from your EKS cluster to OpenSearch such that KAFKA can store logs if the OpenSearch is facing any issues or challenges with consuming logs.
* you can use multilple KAFKA consumers or Connectors to send logs from your Podsto different destinations of your choice such as S3 or CloudWatch Logs .So Kafka can act as a Fan-Out source of your application logs.
* Seperate KAFKA topics for each tenant will help you meet tenant's Isolation(logs in seperate topics) and KAFKA sink connector will then send logs to OpenSearch creating unique INDEX  per topic, hence giving tenant isolation at OpenSearch also.

To achieve this we will use "fluent-bit" to collect logs from your pods. Fluent-bit is one of the most popular opensource log collector/processor for kubernetes workloads which can send logs to  many supported destniations like CloudWatch Logs, S3 and  OpenSearch. Fluent bit uses OUTPUT plugins to forward logs these destinations. In our example we will use 'KAFKA' OUTPUT plugin.

Fluent Bit will run as a DaemonSet on your EKS cluster to tail /var/log/containers/*.log on the EKS cluster and use grep FILTER to process logs for defined namespaces.
We are using a Lua script filter in the configuration file which is used to set topic names for KAFKA topics such that each namespace will have a corresponding topic "logs_namespace" this gives our topics a unique name if KAFKA broker is being used/shared between more than many applications.

In the end we are using Amazon MSK connector for OpenSearch we will send these logs to Opensearch such that each namespace(tenant) will have one Index.

The terraform code in terraform directory which will create an EKS cluster, MSK cluster, Kafka custom plugin,MSK Connector for OpenSearch  and OpenSearch domain in VPC.

Reference Architecure ![Architecture](Ref-Architecture.png?raw=true "Title")


#### Pre-requisites

* . A S3 bucket for terraform backend
* . IAM permissions to create resources(VPC/S3 bucket/EKS Cluster/MSK Brokers/OpenSearch Domain).A Administrator access IAM role is recommended.
* . Install kubectl (https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) and  kafka clients binaries to verify your KAFKA brokers (optional).


* Note. Terraform code will create VPC and all required components. But your OpenSearch dashboard will not be accessible over internet, so you might consider using a AWS client VPN ( or any connectivity method to allow you access to dashboard). you can also launch use a Microsoft windows instance in same VPC and access it via RDP to access your OpenSearch dashboard 

#### Instructions

* To get started, edit 0-proivder.tf to update backend S3 bucket , region and key prefix.
* 
* [Optional] Edit 3-variables.tf to create/edit more namespaces and enable logging on them. In this example we are creating two namespaces."enable_logs_to_es" is a boolean value which is true .i.e means enable logging for this name space
```
default = [
    {
      "name" : "logging",
      "enable_logs_to_es" = false,
    },
    {
      "name" : "example",
      "enable_logs_to_es" = true,
```

* Follow terraform instructions from section below ,terraform will create EKS cluster/MSK and OpenSearch compnonents and install fluent-bit in 'logging' namespace and also create a 'example' namespace.

1. run 
```
terraform init

```


* Wait for terraform to complete 
2. Deploy a sample nginx pod and service  in 'example' namespace.
```
kubectl config set-context --current --namespace=example
kubectl apply -f example-deployment.yaml
kubectl get svc nginx-service-loadbalancer

```
* Note down the name of LoadBalancer and copy it in your browser and hit few times to generate access logs.

3. Login to machine which has KAFKA client binary are installed and list KAFKA topics to verify logs_example and logs_logging topics are created and logs are sent to them.
 
```
./bin/kafka-topics.sh --bootstrap-server=<<list of your brokers>>  --list
./bin/kafka-console-consumer.sh --bootstrap-server <<list of your brokers>. --topic logs_example    

```
4. Login to your OpenSearch Dashboard as admin and verify the indexes are created for each of namespace enabled to log to OpenSearch. 


* If you have applications requiring different parsers for your pods, Fluent-bit allows you to choose your parser. Annotate your pods with following to choose your parser.
   ```
      fluentbit.io/parser: <parser-name>
   ```
* If you want to completely opt out of logging for any of your pods. Use

```
fluentbit.io/exclude: "true"

```
* To configure and use RBAC with OpenSearch , you can follow instructions from https://aws.amazon.com/blogs/apn/storing-multi-tenant-saas-data-with-amazon-opensearch-service/

