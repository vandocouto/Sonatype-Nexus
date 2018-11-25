# Gerenciando imagens Docker através do Sonatype - Nexus

Com o Sonatype Nexus 3 é possível criar vários repositórios privados para gerenciamento de imagens, artefatos e pacotes de sistema operacionais.

**Algumas das funcionalidades:**

-   Armazene e distribua o Maven / Java, o npm, o NuGet, o RubyGems, o Docker, o P2, o OBR, o APT e o YUM, entre outros.
-   Gerencie componentes do dev através da entrega: binários, containers, pacotes de sistemas operacionais e muito mais. 
-   Suporte incrível para o ecossistema Java Virtual Machine (JVM), incluindo Gradle, Ant, Maven e Ivy.
-   Compatível com ferramentas populares como Eclipse, IntelliJ, Hudson, Jenkins, Puppet, Chef, Docker e muito mais.

**Neste tutorial irei configurar o Nexus em um Cluster Kubernetes.**

## Pré-requisitos:

* Cluster Kubernetes
* Domain Name Server
* Letsencrypt
* Nginx Ingress

## Criando os objetos:

Passo 1 - Criando o objeto _Namespace_
```bash
vim namespaces.yaml
```
```bash
apiVersion: v1
kind: Namespace
metadata:
  name: nexus
```
```bash
kubectl apply -f namespaces.yaml
namespace/nexus created
```
Passo 3 - Criando o diretório /storage/nexus no node que vai executar o POD do Nexus.

OBS:  No deployment irei informar através qual será o nodeSelector do worker que vai executar o POD do Nexus.

```bash
mkdir -p /storage/nexus && chmod 777 /storage/nexus -R
```
Passo 4 - Criando o objeto PersistentVolume e PersistentVolumeClaim
```bash
vim storage.yaml
```
```bash
kind: PersistentVolume
apiVersion: v1
metadata:
  name: volume-local
  labels:
    type: local
spec:
  capacity:
    storage: 30Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    # CHANGE ME
    path: "/storage/nexus"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nexus-pvc
  namespace: nexus
  labels:
    app: nexus
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      # CHANGE ME
      storage: 30Gi
```
Passo 5 - Criando o objeto Secret que irá armazenar as chaves .crt e .key do nexus.dominio.com.br  e docker.dominio.com.br
```bash
vim secret.yaml
```
```bash
apiVersion: v1
kind: Secret
metadata:
  name: nexus-tls
  namespace: nexus
type: Opaque
data:
  tls.crt: INSIRA AQUI O CRT CODIFICADO COM BASE64
  tls.key: INSIRA AQUI O KEY CODIFICADO COM BASE64
---
apiVersion: v1
kind: Secret
metadata:
  name: docker-tls
  namespace: nexus
type: Opaque
data:
  tls.crt: INSIRA AQUI O CRT CODIFICADO COM BASE64
  tls.key: INSIRA AQUI O KEY CODIFICADO COM BASE64
```
Para gerar o certificado utilize: [Letsencrypt](https://letsencrypt.org/)
Para codificar o .crt e .key utilize: [Base64encode](https://www.base64encode.org/)

Passo 6 - Criando o Label.
_Listando todos os labels de todos os nodes_
```bash
kubectl get nodes --show-labels
NAME           STATUS   ROLES    AGE   VERSION   LABELS
kubernetes-1   Ready    master   14d   v1.12.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=kubernetes-1,node-role.kubernetes.io/master=
kubernetes-2   Ready    node     14d   v1.12.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=kubernetes-2,node-role.kubernetes.io/node=
```
Passo 7 - Criando o Label nexusStorage=nexus para o node kubernetes-2. 
Neste node contém o diretório /storage/nexus, diretório que será mapeado para o POD do Nexus.
```bash
kubectl label nodes kubernetes-2 nexusStorage=nexus
```
_Caso precise excluir o Label_:
```bash
kubectl label node kubernetes-2 nexusStorage-
```
```bash
kubectl get nodes --show-labels
NAME           STATUS   ROLES    AGE   VERSION   LABELS
kubernetes-1   Ready    master   14d   v1.12.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=kubernetes-1,node-role.kubernetes.io/master=
kubernetes-2   Ready    node     14d   v1.12.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=kubernetes-2,nexusStorage=nexus,node-role.kubernetes.io/node=
```
Passo 8 - Criando o objeto Deployment 
```bash
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nexus
  namespace: nexus
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nexus
    spec:
      containers:
      - image: sonatype/nexus3
        imagePullPolicy: Always
        name: nexus
        ports:
        - containerPort: 8081
        - containerPort: 5000
        volumeMounts:
          - mountPath: /nexus-data
            name: nexus-data-volume
      volumes:
        - name: nexus-data-volume
          persistentVolumeClaim:
            claimName: nexus-pvc
      nodeSelector:
          nexusStorage: nexus
```
Passo 9 - Criando o objeto Service
```bash
vim service.yaml
```
```bash
apiVersion: v1
kind: Service
metadata:
  name: nexus-service
  namespace: nexus
spec:
  ports:
 - port: 80
    targetPort: 8081
    protocol: TCP
    name: http
 - port: 5000
    targetPort: 5000
    protocol: TCP
    name: docker
  selector:
    app: nexus
```
Passo 10 - Criando o objeto Ingress
Documentação do [Ingress com Nginx](https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/installation.md) 
```bash
vim ingress.yaml
```
```bash
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nexus-ingress
  namespace: nexus
  annotations:
    ingress.kubernetes.io/proxy-body-size: 100m
    kubernetes.io/tls-acme: "true"
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
 - hosts:
    - nexus.dominio.com.br
    secretName: nexus-tls
  rules:
 - host: nexus.dominio.com.br
    http:
      paths:
      - path: /
        backend:
          serviceName: nexus-service
          servicePort: 80
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: docker-ingress
  namespace: nexus
  annotations:
    ingress.kubernetes.io/proxy-body-size: "0"
    ingress.kubernetes.io/proxy-read-timeout: "600"
    ingress.kubernetes.io/proxy-send-timeout: "600"
    kubernetes.io/tls-acme: "true"
    kubernetes.io/ingress.class: "nginx"
    nginx.org/client-max-body-size: "100m"
spec:
  tls:
 - hosts:
    - docker.dominio.com.br
    secretName: docker-tls
  rules:
 - host: docker.dominio.com.br
    http:
      paths:
      - path: /
        backend:
          serviceName: nexus-service
          servicePort: 5000
```
Depois que a implantação estiver concluída e a interface do Nexus 3 estiver disponível no nexus.dominio.com.br, basta criar o repositório do docker. 
Entre no Nexus com admin / admin123, vá para “Server administration and configuration”, “Repositories”, “Create repository”, “docker (hosted)”
e publique o docker hosted http na porta 5000.

## Acessando o Nexus

![enter image description here](https://lh3.googleusercontent.com/V-sYzYwT0914Q98id2s2OMOyANQBMLjJDIUxzKWZmB1laIEvEn3LiUu7k-xR2s3By95zK-REOHryyQ=s1024 "Nexus-Kubernetes")

Create Repository

![enter image description here](https://lh3.googleusercontent.com/NW9iaSpYhE4d0Tf1DyElIZH-URA0LlS6R15CpbGYaqTlpzkJW47E-j_LRg8T3AuvJE-f-FTl0vPVpw=s1024 "Nexus-2")

docker (hosted)
![enter image description here](https://lh3.googleusercontent.com/YXTF8noB4qmUJ3_CJbCPw85j9gdlgBS4XOEEzaT4MqRvlQCpf7scz0mzrzWVMTRCTst_JbrtogZ3Pw=s1024 "Nexus-3")

docker-privado
![enter image description here](https://lh3.googleusercontent.com/HX5PcIm6sEA6WmjNJyPr8Cu9hkE9xAXSDZO3bN6nFbwdYdOeGkt_UpBSX2weAP4_ZUWizZcIQZ-nlw=s1024 "Nexus-4")


## Testando! 
Realizando o login
```bash
docker login docker.dominio.com.br
Username: admin
Password: admin123
Login Succeeded
```
Testando o push da imagem ubuntu para o repositório docker.dominio.com.br
```bash
docker pull ubuntu
docker tag ubuntu docker.dominio.com.br/ubuntu:1.0.0
docker push docker.dominio.com.br/ubuntu:1.0.0
```
