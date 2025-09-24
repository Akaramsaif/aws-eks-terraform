# aws-eks-terraform

Create a step by step document.
What is Kubernetics and why kubernetes.
What are the pre-requisite to provision the EKS Cluster? How to setup the Kubernetes Using the Terraform? 

How to Logging and Monitoring of EKS using the cloud watch?

Fluent Bit to be configure to collect container logs and forward them to Amazon CloudWatch.  

Step by Step Setup:

Create IAM Role with below trust policy

<img width="1855" height="797" alt="image" src="https://github.com/user-attachments/assets/70f5c837-5948-4044-973f-21fcfd1fb004" />

 

Create & attach Fluent Bit Policy to the IAM Role. 

<img width="1843" height="623" alt="image" src="https://github.com/user-attachments/assets/2ff7bdc0-e351-4cf4-b58b-cb5764a2d06f" />



Create the Kubernetes namespace using below command. 

        kubectl create namespace amazon-cloudwatch
        
Create a service account manifest fluent-bit-sa.yaml


apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: amazon-cloudwatch
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::682296103893:role/fluent-bit-role


kubectl apply -f fluent-bit-sa.yaml


Create the Fluent Bit ConfigMap fluent-bit-configmap.yaml



apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: amazon-cloudwatch
  labels:
    k8s-app: fluent-bit
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush               5
        Log_Level           info
        Daemon              off
        Parsers_File        parsers.conf
        HTTP_Server         On
        HTTP_Listen         0.0.0.0
        HTTP_Port           2020
    [INPUT]
        Name                tail
        Tag                 kube.*
        Path                /var/log/containers/*.log
        Parser              docker
        DB                  /var/log/flb_kube.db
        Mem_Buf_Limit       5MB
        Skip_Long_Lines     On
        Refresh_Interval    10
        Mem_Buf_Limit       100MB
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        Keep_log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off
    [FILTER]
        Name                rewrite_tag
        Match               kube.*
        Rule                $kubernetes['namespace_name'] ^frontend$ frontend false
        Rule                $kubernetes['namespace_name'] ^cdkgapp$ cdkgapp false
        Emitter_Name        re_emitted
        Emitter_Mem_Buf_Limit 50MB
    [OUTPUT]
        Name                cloudwatch_logs
        Match               frontend
        region              us-east-1
        log_group_name      /aws/eks/USVGA10915-EKS-D001/frontend
        log_stream_prefix   from-fluent-bit-
        auto_create_group   false
        log_key             log
    [OUTPUT]
        Name                cloudwatch_logs
        Match               cdkgapp
        region              us-east-1
        log_group_name      /aws/eks/USVGA10915-EKS-D001/cdkgapp
        log_stream_prefix   from-fluent-bit-
        auto_create_group   false
        log_key             log
  parsers.conf: |
    [PARSER]
        Name                docker
        Format              json
        Time_Key            time
        Time_Format         %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep           On




kubectl apply -f fluent-bit-configmap.yaml



Deploy the Fluent Bit DeamonSet fluent-bit-ds.yaml



apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: amazon-cloudwatch
  labels:
    k8s-app: fluent-bit
spec:
  selector:
    matchLabels:
      k8s-app: fluent-bit
  template:
    metadata:
      labels:
        k8s-app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      terminationGracePeriodSeconds: 10
      containers:
        - name: fluent-bit
          image: public.ecr.aws/aws-observability/aws-for-fluent-bit:latest
          ports:
            - containerPort: 2020
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 100Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: fluent-bit-config
              mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: fluent-bit-config
          configMap:
            name: fluent-bit-config
---


kubectl apply -f fluent-bit-ds.yaml



Create ClusterRole for Service Account we created fluent-bit-clusterrole.yaml



apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-role
rules:
- apiGroups: [""]
  resources:
    - namespaces
    - pods
  verbs: ["get", "list", "watch"]


kubectl apply -f fluent-bit-clusterrole.yaml




Create ClusterRoleBinding for Service Account we created fluent-bit-clusterrolebinding.yaml



apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit-role
subjects:
- kind: ServiceAccount
  name: fluent-bit
  namespace: amazon-cloudwatch


kubectl apply -f fluent-bit-clusterrolebinding.yaml
Restart the Pods if needed.



kubectl rollout restart daemonset fluent-bit -n amazon-cloudwatch
Verify fluent bit is collecting the logs using below command.



kubectl logs -n amazon-cloudwatch -l k8s-app=fluent-bit
Check CloudWatch Logs Console → You should see separate log groups like 



/aws/eks/USVGA10915-EKS-D001/frontend
/aws/eks/USVGA10915-EKS-D001/cdkgapp
