Deployment file created
nginx-ts-deployment.yaml 


<img width="622" height="381" alt="image" src="https://github.com/user-attachments/assets/efc8147d-e833-4e8a-a6cd-2614937f8bb9" />


Step 1: Check Cluster
        Make sure Minikube is running and kubectl is connected.
        Ran following commands to check minikube and kubctl is connected
        "minikube status,
        kubectl config current-context,
        kubectl get nodes,
        kubectl get pods -A"

        output: 
          <img width="737" height="382" alt="image" src="https://github.com/user-attachments/assets/11b07edb-a0c7-4924-b799-12849e33a689" />

Step 2: Create Namespace
        Ran below commads for creating/deleting the namespace
        "kubectl create namespace tsday2,
        kubectl get namespace tsday2,
        kubectl delete namespace tsday2
        kubectl create namespace tsday2"

        output: 
                <img width="606" height="216" alt="image" src="https://github.com/user-attachments/assets/ad0154e8-117a-4fc5-9235-b6af936fba92" 

Step 3: Apply Deployment YAML
Apply the Deployment manifest:
        Ran below command
        "kubectl apply -f day2/nginx-ts-deployment.yaml"

          output:
                  <img width="767" height="352" alt="image" src="https://github.com/user-attachments/assets/c565f27f-5e8f-45d3-a2f7-dcba8fc9a3ff" />

Step 4: Check Deployment
        Ran below command
        "kubectl get deployment -n tsday2"

        output:
                <img width="582" height="87" alt="image" src="https://github.com/user-attachments/assets/d81014cf-0497-4371-877f-6d66567e1114" />

                READY 5/5: 5 Pods are ready out of 3 desired.
                UP-TO-DATE 5: 5 Pods match the current Deployment template.
                AVAILABLE 5: 5 Pods are available

Step 5: Check ReplicaSet
         Ran below command
         "kubectl get replicaset -n tsday2"
         output:
                 <img width="582" height="52" alt="image" src="https://github.com/user-attachments/assets/7411e343-a78e-4840-9005-45e7749added" />

Step 6: Check Pods
        Ran below command
        "kubectl get pods -n tsday2 -o wide"
        Output:
                <img width="1025" height="117" alt="image" src="https://github.com/user-attachments/assets/cd70dbb7-94fc-44c1-91c3-bef08a67299d" />

Step 7: Describe Deployment
        Ran below command 
        "kubectl describe deployment nginx-ts-deployment -n tsday2"

        output:
                <img width="992" height="487" alt="image" src="https://github.com/user-attachments/assets/cbbae2fb-ba26-44bf-b3cb-37dc6c3aa5b6" />

Step 8: Check Pods Using Label
        Ran below command 
        "get pods -n tsday2 -l app=tsnginx"

        Output:
                
                <img w""idth="711" height="157" alt="image" src="https://github.com/user-attachments/assets/293ff36c-6880-4272-8c55-cc7a85dae353" />

Scaling The Deployment:
        Scaling means increasing or decreasing the number of Pod replicas.
        Current YAML says replicas: 5
        Ran below commands for scale upto 7 replicas:
        "kubectl scale deployment nginx-ts-deployment --replicas=7 -n tsday2
        kubectl get deployment -n tsday2
        kubectl get pods -n tsday2 -o wide"

        Ouptut:
                <img width="991" height="245" alt="image" src="https://github.com/user-attachments/assets/6cce9589-9247-4220-9838-c5b1620c0f1f" />


       Ran below commands to Scale Down To 2 Replicas:
        "kubectl scale deployment nginx-ts-deployment --replicas=2 -n tsday2,
        kubectl get deployment -n tsday2"

        Output:
                <img width="1037" height="282" alt="image" src="https://github.com/user-attachments/assets/a5ba312a-c21b-4543-873a-d2c3cb455200" />

Rolling Update:
        Ran following commands:
        "kubectl set image deployment/nginx-ts-deployment nginx=nginx:1.28 -n tsday2
        kubectl rollout status deployment/nginx-ts-deployment -n tsday2,
        kubectl get recpicaset -n tsday2
        kubectl get pods -n tsday2 -o wide"

        Output:
                <img width="1052" height="206" alt="image" src="https://github.com/user-attachments/assets/792157a7-53c3-47fb-a845-2b590054f106" />

Rollback update:

        Ran below commands:
        "kubectl rollout undo deployment/nginx-ts-deployment -n tsday2
        kubectl rollout status deployment/nginx-ts-deployment -n day2
        kubectl describe deployment nginx-ts-deployment -n tsday2"

        Output:
                <img width="1210" height="855" alt="image" src="https://github.com/user-attachments/assets/3cf9c4d9-efd9-4bab-80c1-024b35aa751d" />

Self-Healing Practical
        Self-healing means Kubernetes repairs the workload automatically.
        
        First list Pods:
                command : 
                "kubectl get pods -n day2,
                kubectl delete pod <pod-name> -n day2,
                "
                Output:
                "The deleted Pod disappears.
                ReplicaSet creates a replacement Pod.
                Deployment returns to the desired replica count."
                <img width="1447" height="816" alt="image" src="https://github.com/user-attachments/assets/ef2dc5a8-cb6b-4ef1-811e-51463bfe0a73" />

Check Container Version:
                Ran below commands:
                "kubectl get pods -n tsday2 and  kubectl exec nginx-ts-deployment-5478b8b8c-jjfkc -n tsday2 -- nginx -v "

                Output:
                        <img width="757" height="172" alt="image" src="https://github.com/user-attachments/assets/0e3fba36-78a5-4faf-b550-d8c95f31994a" />

                        
                


                

                



                   





