# Choreo Connect Deployment with APIM

This README file provides instructions on how to set up and deploy Choreo Connect with API Manager and the Redis server for distributed caching, which is used for backend throttling and burst control, using the configurations provided in this repository.

Furthermore, the following section provides instructions on how to perform the performance test on this deployment using the JMeter client deployed in the same Kubernetes cluster

- [Conduct Performance Tests on Choreo Connect to Verify Backend Throttling and Burst Control Features](#Conduct Performance Tests on Choreo Connect to Verify Backend Throttling and Burst Control Features)

## Deployment Architecture


![Diagram](/home/sajith-madhusanka/Downloads/setup.png)

## Checkout the Repository

First, log into the cluster. Create a directory structure as needed and clone the repository that contains all the configurations for the deployment:

```sh
git clone https://github.com/Sumudu-Sahan/CC_Deployment_APIM.git
```

Navigate to the `CC_Deployment_APIM` directory:

``` bash
cd CC_Deployment_APIM
```

It's recommended to fork this repository and clone your fork to the cluster to avoid difficulties when making changes and pulling updates.

## Deploy the Nginx Ingress Load Balancer

Deploy the ingress load balancer by executing the following command. This will automatically create the `ingress-nginx` namespace and deploy all services, pods, and other configurations under that namespace:

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

## Create Namespace

Next, create a namespaces for the API Manager, Choreo Connect, and the Redis server:

```sh
kubectl create namespace wso2
```

## Deploy the Choreo Connect With API Manager

Add the secrets for UMT authentication to get the latest images from the WSO2 docker registry. Replace `<YOUR_WSO2_MAIL>` and `<YOUR_WSO2_PASSWORD>` with your actual WSO2 email and password:

```sh
kubectl create secret docker-registry customcred --docker-server=docker.wso2.com --docker-username=<YOUR_WSO2_MAIL> --docker-password=<YOUR_WSO2_PASSWORD> --docker-email=<YOUR_WSO2_MAIL> -n wso2
```

Deploy the API Manager and Choreo Connect by executing the following commands in the `CC_Deployment_APIM` directory:

```sh
kubectl apply -f apim -n wso2
kubectl apply -f choreo-connect -n wso2
```

This will deploy the API Manager with Choreo Connect and ingress routing mappings. Under Choreo Connect, there will be two container replicas for the enforcer and router pods.

## Deploy the Redis Cache server

Deploy the Redis cache server by executing the following commands in the `CC_Deployment_APIM` directory:

```sh
kubectl apply -f redis -n wso2
```
## Deploy the Backend server cluster

Deploy the Backend server cluster by executing the following commands in the `CC_Deployment_APIM` directory:

```sh
kubectl apply -f backend-cluster -n wso2
```

## Verify Deployments

To verify if all the pods have been deployed properly, execute:

```sh
kubectl get po -n wso2
```

To verify the ingress routings, execute:

```sh
kubectl get ingress -n wso2
```

To verify the services, execute:

```sh
kubectl get svc -n wso2
```

## Add /etc/hosts Mappings

Retrieve the external IP address of the ingress nginx load balancer to update the `/etc/hosts` file on your local machine. Execute the following command to get the public IP address:

```sh
kubectl get all -n ingress-nginx
```

Locate the `LoadBalancer` TYPE record and copy the value under the `EXTERNAL-IP` column. Add the following mapping to your `/etc/hosts` file, replacing `<EXTERNAL_IP>` with the actual IP address:

```sh
<EXTERNAL_IP> gw.wso2.com apim.wso2.com
```

## Access the Portals

After updating the `/etc/hosts` file, you can access the portals via the following URLs:

- Publisher Portal: [https://apim.wso2.com/publisher](https://apim.wso2.com/publisher)
- Developer Portal: [https://apim.wso2.com/devportal](https://apim.wso2.com/devportal)
- Admin Portal: [https://apim.wso2.com/admin](https://apim.wso2.com/admin)
- Carbon Management: [https://apim.wso2.com/carbon](https://apim.wso2.com/carbon)

To access the Choreo Connect router, use the following URL:

- [https://gw.wso2.com](https://gw.wso2.com)

## Access the backend servers

 - Backend Server 1 : http://mockoon:80/
 - Backend Server 2 : http://mockoon-2:80/
 
 

## Conduct Performance Tests on Choreo Connect to Verify Backend Throttling and Burst Control Features

- Deploy two APIs with the configurations below in Choreo Connect via the APIM publisher portal.
  - API 1
    - Name: TEST_API_1
    - context: test_1
    - version: 1.0.0
    - resources:
        - GET /username
    - endpoint: http://mockoon:80/
  - API 2
     - Name: TEST_API_2
     - context: test_2
     - version: 1.0.0
     - resources:
        - GET /username
     - endpoint: http://mockoon-2:80/
 
- Configure either backend throttling or burst control based on your requirements.
- Subscribe to the APIs and generate the access token via the APIM DevPortal.
- To perform the load test on single API
    - Update the **jmeter-config-map.yaml** file residing under the CC_Deployment_APIM/jmeter/single directory with the generated access token.
    - Deploy the JMeter config map by executing the following commands in the `CC_Deployment_APIM` directory:
        ```sh
        kubectl apply -f jmeter/single/jmeter-config-map.yaml -n wso2
        ```
- To perform the load test on both APIs
    - Update the **jmeter-config-map.yaml** file residing under the CC_Deployment_APIM/jmeter/two directory with the generated access token.
    - Deploy the JMeter config map by executing the following commands in the `CC_Deployment_APIM` directory:
         ```sh
            kubectl apply -f jmeter/two/jmeter-config-map.yaml -n wso2
         ```
- If you want to deploy a custom JMeter config map, you can create a JMeter script via the JMeter UI and execute the command below to deploy it as a config map. Please make sure to define the JTL save path as similar to /opt/apache-jmeter-5.5/test_case.jtl when creating the JMeter script.
    ```sh
     kubectl create configmap jmeter-test-script --from-file=<PATH_TO_JMETER_SCRIPT>/performance_test_v1.jmx -n wso2
    ```
- Deploy the JMeter pod by executing the following commands in the `CC_Deployment_APIM` directory:

    ```sh
    kubectl apply -f jmeter/jmeter-deployment.yaml -n wso2
    ```
- If the JMeter pod is already deployed, redeploy the pod by executing the command below to apply the JMeter config map changes.
    ```sh
    kubectl delete pod  $(kubectl get pods -n wso2 -l app=jmeter -o jsonpath="{.items[0].metadata.name}")  -n wso2
    ```
 - If you have performed a load test already and need to clear the logs of the backend server before starting the next test, please execute the commands below to redeploy the backend servers.
    - Backend Server 1
      ```sh
       kubectl delete pod  $(kubectl get pods -n wso2 -l app=mockoon -o jsonpath="{.items[0].metadata.name}") -n wso2 
      ```
   - Backend Server 2
     ```sh
     kubectl delete pod  $(kubectl get pods -n wso2 -l app=mockoon-2 -o jsonpath="{.items[0].metadata.name}")  -n wso2
     ```
- Execute the below command to save the backend server logs in your local machine.
    - Backend Server 1
       ```sh
        kubectl logs -f  $(kubectl get pods -n wso2 -l app=mockoon -o jsonpath="{.items[0].metadata.name}")  -n wso2 >>  <PATH_TO_SAVE_TEST_RESULT_DIRECTORY_OF_YOUR_LOCAL_MACHINE>/backend1.log 2>&1
       ```
   - Backend Server 2
     ```sh
     kubectl logs -f $(kubectl get pods -n wso2 -l app=mockoon-2 -o jsonpath="{.items[0].metadata.name}") -n wso2 >> <PATH_TO_SAVE_TEST_RESULT_DIRECTORY_OF_YOUR_LOCAL_MACHINE>/backend2.log 2>&1
     ```
 - To check the performance test execution summary, you can tail the logs of the JMeter pod by executing the command below.
      ```sh
       kubectl logs -f  $(kubectl get pods -n wso2 -l app=jmeter -o jsonpath="{.items[0].metadata.name}")  -n wso2
      ```
 - Once the performance test is completed, you can execute the following command to copy the JMeter summary reports (JTL) to the local machine.
    - API Invocation 1
        ```sh
        kubectl cp -n wso2 $(kubectl get pods -n wso2 -l app=jmeter -o jsonpath="{.items[0].metadata.name}"):/opt/apache-jmeter-5.5/test_case1.jtl <PATH_TO_SAVE_TEST_RESULT_DIRECTORY_OF_YOUR_LOCAL_MACHINE>/test_case1.jtl
        ```
    -  API Invocation 2
      ```sh
      kubectl cp -n wso2 $(kubectl get pods -n wso2 -l app=jmeter -o jsonpath="{.items[0].metadata.name}"):/opt/apache-jmeter-5.5/test_case2.jtl <PATH_TO_SAVE_TEST_RESULT_DIRECTORY_OF_YOUR_LOCAL_MACHINE>/test_case2.jtl
      ```
 - Then, you can execute the following command inside the `<JMETER_APP>/bin` directory on your local machine to generate the JMeter summary report.
    - API Invocation 1
         ```sh
         ./jmeter.sh -g <PATH_TO_SAVE_TEST_RESULT_DIRECTORY_OF_YOUR_LOCAL_MACHINE>/test_case1.jtl -o <PATH_TO_SAVE_TEST_RESULT_DIRECTORY_OF_YOUR_LOCAL_MACHINE>/OUTPUT1
         ```
     -  API Invocation 2
      ```sh
       ./jmeter.sh -g <PATH_TO_SAVE_TEST_RESULT_DIRECTORY_OF_YOUR_LOCAL_MACHINE>/test_case2.jtl -o <PATH_TO_SAVE_TEST_RESULT_DIRECTORY_OF_YOUR_LOCAL_MACHINE>/OUTPUT2
      ```
 - Then you can view the JMeter summary report by opening the index.html file residing under the `<PATH_TO_SAVE_TEST_RESULT_DIRECTORY_OF_YOUR_LOCAL_MACHINE>/OUTPUT1` directory.
 - If you want to check the TPS of the backend servers, you can execute the following command to obtain that statistic.
    - Backend Server 1
        ```sh
         grep -oP '"timestamp":"\K[^"]+' backend1.log | cut -d'.' -f1 | sort | uniq -c > output1.txt
        ```
    - Backend Server 2
      ```sh
      grep -oP '"timestamp":"\K[^"]+' backend2.log | cut -d'.' -f1 | sort | uniq -c > output2.txt
      ```