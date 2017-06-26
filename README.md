[![](https://images.microbadger.com/badges/image/dailyhotel/gophernotes.svg)](https://microbadger.com/images/dailyhotel/gophernotes "Get your own image badge on microbadger.com")

# dockerized-gophernotes
Unofficial Docker image of Gophernotes.

- Security improvement : no `root` account
- Better Docker support from [jupyter/base-notebook](https://github.com/jupyter/docker-stacks/tree/master/base-notebook)

## On Kubernetes

- [coreos/alb-ingress-controller](https://github.com/coreos/alb-ingress-controller) is needed to use ALB. Without ALB, websocket is working properly.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gophernotes
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:myssl
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS": 443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/subnets: Subnet1,Subnet2
    alb.ingress.kubernetes.io/security-groups: OfficeOnly
spec:
  rules:
  - host: gophernotes.mysite.com
    http:
      paths:
      - path: /
        backend:
          serviceName: gophernotes
          servicePort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: gophernotes
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8888
    protocol: TCP
  selector:
    app: gophernotes
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: gophernotes
spec:
  replicas: 1
  strategy:
    type: Recreate
  minReadySeconds: 60
  template:
    metadata:
      labels:
        app: gophernotes
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 100 
      containers:
      - name: gophernotes
        image: dailyhotel/gophernotes:latest
        imagePullPolicy: Always
        command: 
        - start-notebook.sh
        - --NotebookApp.token=''
        env:
        - name: GRANT_SUDO
          value: "yes"
        ports:
        - name: app-port
          containerPort: 8888
        volumeMounts:
          - name: gophernotes-data
            mountPath: /home/jovyan/work
      volumes:
      - name: gophernotes-data
        persistentVolumeClaim:
          claimName: gophernotes-data-volumeclaim
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gophernotes-data-volumeclaim
  annotations:
    volume.beta.kubernetes.io/storage-class: "default"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```