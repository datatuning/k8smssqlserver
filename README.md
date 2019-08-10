# Artefatos de para criação de um cluster SQL Server 2019 AlwaysON Availability Groups utilizando Kubernetes.

Especificacao da configuracao utilizada para este Lab:

- Sistema Operacional: Ubuntu 19.04;
- Hypervisor: Oracle VirtualBox;
- 4 maquinas virtuais para o cluster Kubernetes. Segue configuracoes:
    - Master: 1 VM com 2 vCPU's, 4GB de RAM, 15GB de disco e 2 placas de rede (1 placa configurada para rede interna); 
    - Workers: 3 VM com 2 vCPU's, 2GB de RAM, 15GB de disco e 2 placas de rede (1 placa configurada para rede interna).
- Configurar os hosts em rede interna onde os mesmos estejam se comunicando. 

## Instalando o Cluster Kubernetes

As primeiras etapas se aplicam aos **4 nos do cluster**, entao, basta seguir com o passo a passo de execucao dos comandos abaixo

Desabilitar a swap do servidor:

`` #swapoff -a ``

Instalacao do Docker:

`` #curl -fsSL https://get.docker.com | bash ``

Verificar se o Docker foi instalado:

`` #docker --version ``

Instalacao do Kubernetes:

`` #curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - ``
`` #echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list ``
`` #apt update ``
`` #apt install kubelet kubectl kubeadm -y ``

Alterar o CGROUP driver de cgroupfs para systemd pois o Kubernetes recomenda systemd como driver:

Criar o JSON do driver CGROUP para SYSTEMD:

`` #vim /etc/docker/daemon.json ``

``{``
``	"exec-opts": ["native.cgroupdriver=systemd"],``
``	"log-driver": "json-file",``
``	"log-opts": {``
``	  "max-size": "100m"``
``	},``
``	"storage-driver": "overlay2"``
``}``

Criar a pasta do servico do docker dentro do diretorio systemd:

`` #mkdir -p /etc/systemd/system/docker.service.d ``

Reiniciar o deamon e docker:

`` #systemctl daemon-reload ``
`` #systemctl restart docker ``


** A partir deste momento executaremos apenas no Master **

Baixar as imagens necessarias para subir o no Master:

`` #kubeadm config images pull ``

Iniciar o Kubernetes:

`` #kubeadm init --apiserver-advertise-address $(hostname -i) // No master ``

Executar os comandos solicitados ao final do provisionamento do k8s:

`` #mkdir -p $HOME/.kube``
`` #cp -i /etc/kubernetes/admin.conf $HOME/.kube/config``
`` #chown $(id -u):$(id -g) $HOME/.kube/config``

Habilitar a rede para que os Workers "conversem" com o Master:

`` #kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"``

**Agora vamos executar o comando de JOIN nos Workers para que eles se conectem ao Master:**

Recuperar o comando de JOIN via kubeadm (Executar no Master):

`` #kubeadm token create --print-join-command ``

Copiar, colar e executar o comando em todos os Workers (Este comando abaixo eh meramente um exemplo):

`` #kubeadm join k8smaster:6443 --token q0rhkh.mtlm2riel1gbxxut --discovery-token-ca-cert-hash sha256:3f5d522a8e6f034e5737ec2201abd0269001f30586d3573c554d37a595accd20 ``


## Provisionando o Cluster com SQL Server AlwaysON AG no K8S

Clonar este repositorio para o Master.

`` #git clone https://github.com/datatuning/k8smssqlserver.git``

Criar a pasta a seguir em todos os Workers, necessario para o armazenamento dos bancos de dados das instancias SQL Server:

`` #mkdir /var/opt/mssql ``

A partir daqui executaremos os comandos **apenas no Master**.

Criar o namespace para encapsular os recursos relacionados a infraestrutura do SQL Server:

`` #kubectl create namespace ag1 ``

Criar uma secret necessaria para distribuicao da senha do usuario SA por todos os pods onde estarao as instancias:

`` #kubectl create secret generic sql-secrets --from-literal=sapassword="P@ss0rd2019" --from-literal=masterkeypassword="P@ss0rd2019" --namespace ag1 ``

Criar o Storage Class:

`` #kubectl apply -f storageClass.yaml --namespace ag1 ``
`` #kubectl get storageclass ``

Criar os Persistent Volumes para armazenamento dos bancos de dados:

`` #kubectl apply -f storage.yaml --namespace ag1 ``
`` #kubectl get pv ``

Criar o Operator para gerenciamento do StafulSet:

`` #kubectl apply -f operator.yaml --namespace ag1 ``

Finalmente iremos provisionar os Pods com as instancias SQL Server:

`` #kubectl apply -f sqlserver.yaml --namespace ag1 ``

Verificar se os pods relacionado as instancias SQL Server estao com status Executing:

`` #kubectl get pods --namespace ag1 ``

Criar os Services para expor o no primario e os nos secundarios do AlwaysON:

`` #kubectl apply -f ag-services.yaml --namespace ag1 ``
`` #kubectl get service --namespace ag1 ``

Caso tudo tenha dado certo, agora basta verificar a porta do ag-primary e utilizar o ip externo do Master juntamente com a porta para conectar no SQL Server.

Conecte no ag-primary utilizando o IP Externo do Master e a porta listada no get services.

Apos conectado, abre uma nova query e execute os comandos a seguir para criar o banco de dados, coloca-lo no AG e crie uma tabela neste banco de dados inserindo dados para verificar se tudo esta em pleno funcionamento:


`` CREATE DATABASE [TestAG1]`` 
`` GO`` 
`` BACKUP DATABASE [TestAG1] TO DISK = 'nul'`` 
`` BACKUP LOG [TestAG1] TO DISK = 'nul'`` 
`` GO`` 
`` ALTER AVAILABILITY GROUP [ag1] ADD DATABASE [TestAG1]`` 
`` GO`` 

`` USE [TestAG1]`` 
`` GO`` 

`` CREATE TABLE TEST(`` 
`` 	A INT IDENTITY`` 
`` 	,B VARCHAR(10)`` 
`` 	,C DATETIME`` 
`` )`` 

`` INSERT INTO TEST VALUES('DOCKER',GETDATE());`` 
`` INSERT INTO TEST VALUES('K8S',GETDATE());`` 