# Projeto GitOps na prática - Programa de bolsas
Esse projeto consiste na realização da execução de uma aplicação demo em microserviços, rodando em cluster Kubernetes local, e seguindo os principios de GitOps com o uso do ArgoCD.

## 👨‍💻 Tecnologias utilizadas
- Kubernetes
- Minikube
- Kubectl
- ArgoCD
- Git
- Docker

<br>

## 🌐 Configurando o ambiente
Os comandos a seguir são para Linux (amd64). Para usuários de Windows ou macOS, por favor, sigam as instruções na [documentação oficial do Kubectl](https://kubernetes.io/docs/tasks/tools/) e [documentação oficial do Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download).

### Instalação do Kubectl
Kubectl é a principal ferramenta de linha de comando para administrar cluster Kubernetes, antes de tudo precisamos instalá-la. Execute os seguintes comandos para linux:

    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    kubectl version --client

### Instalação do Minikube
Minikube é a ferramenta que permite executar um cluster Kubernetes de um único nó na nossa máquina, antes de instalar o proprio Minikube é preciso o driver que será utilizado para provisionar o ambiente que o Minikube criará o nó, e também o software de container runtime que ele vai utilizar, para isso usaremos o Docker.

Para instalar o Docker siga a documentação oficial: [Guia oficial de instalação Docker.](https://docs.docker.com/engine/install/)

Após finalizado a instalação, podemos prosseguir instalando o Minikube, para isso execute os seguintes comandos:

    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    chmod +x ./minikube
    sudo mv ./minikube /usr/local/bin/minikube
    minikube version

Agora para iniciar e configurar o Minikube para utilizar o Docker, use esses comandos:

    docker context use default
    minikube config set driver docker
    minikube start

Tendo feito isso o ambiente está pronto e configurado para seguirmos para as próximas etapas.

<br><br>

## 🐙🐈‍⬛ Configurando o GitHub
Para prosseguir é preciso ter uma conta no Github, caso não tenha se cadastre clicando em [Github.](https://github.com/signup?source=form-home-signup&user_email=)

### Conectando chave SSH no Github
Para gerar a chave que usaremos para autentificação no Github execute o seguinte comando no terminal:

    ssh-keygen -t ed25519 -C "seu_email@exemplo.com"

Após vai ser perguntado o local do arquivo que deseja salvar a chave, colocar algo como: **/home/SEU_USUARIO/.ssh/github**

Também vai pedir por uma palavra passe que fica a sua escolha escolher alguma ou não. Então a chave vai ser gerada porém ela ainda precisa ser adicionada ao ssh-agent do sistema, para isso execute os comandos:

    eval "$(ssh-agent -s)"
    ssh-add <LOCAL_DA_SUA_CHAVE>

Agora ainda precisamos adicionar a chave pública no Github, use o comando *cat* pelo terminal ou então navegue até a pasta com a sua chave, então copie o seu conteúdo. (**IMPORTANTE**: a chave que deve ser copiada é a que termina com a extensão ".pub" essa é a chave pública)

    cat <LOCAL_DA_SUA_CHAVE>.pub

Com a chave copiada, acesse a [página de configurações SSH no Github](https://github.com/settings/keys). Clique no botão verde escrito **New SSH key**. Escolha um titulo para a chave, em *Key type* selecione *Authentication Key* (recomendo criar outro registro de chave com o *Key type* de *Signing Key* após esse), por último em *Key* cole a chave pública copiada e clique em adicionar chave.

Use o comando para testar se a conexão SSH foi estabelecida:

    ssh -T git@github.com


<br><br>

## Etapa 1 – Fork e repositório GitHub
**Nota**: Embora a etapa original do programa sugira um Fork do repositório da aplicação demo, para este exercício de GitOps criarei um repositório privado do zero. Farei isso pois a fonte da verdade (o repositório Git) precisa conter apenas o manifesto que aplicaremos no cluster.

Nessa etapa o objetivo é criar o repositório privado no Github, contendo o arquivo .yaml manifesto do Kubernetes com nossa aplicação *Online Boutique*, primeiramente precisamos baixar o manifesto.

Link para o manifesto Kubernetes da aplicação: https://github.com/GoogleCloudPlatform/microservices-demo/blob/main/release/kubernetes-manifests.yaml

Então com o arquivo manifesto em mãos vamos criar o repositório, na pagina principal do Github (https://github.com), clique no botão verde escrito **New** no canto esquerdo da página. Isso vai abrir a tela de criação de repositório. No campo **Repository name** escreva **gitops-microservices** e na opção **Choose visibility** marque **Private**. Ao final clique em **Create repository**.

<img width="1812" height="1100" alt="repositorio" src="https://github.com/user-attachments/assets/6311e60a-9458-4094-bb96-3895bc6ca184" />

Agora precisamos fazer o upload do arquivo manifesto Kubernetes dentro do repositório recém criado, isso pode ser feito via interface gráfica do próprio Github ou via Git que tem CLI e GUI. Nesse documento vou demostrar como fazer o commit do arquivo via Git CLI (se preferir fazer via Github pode pular para a proxima etapa).

Link do download do Git: https://git-scm.com/install/

Crie uma pasta no seu computador no local onde deseja que seja o repositório local e mova o arquivo .yaml que fizemos o download para dentro dessa pasta. Dentro dessa pasta criada, crie outra pasta chamada *k8s* para o arquivo do Kubernetes, mova o manifesto para dentro dessa pasta.

Na página do Github do seu repositório, copie o URL SSH dele. (exemplo: git@github.com:devbrunofernandes/gitops-microservices.git)

Após executes os seguintes comandos Git só substituindo as informações necessarias:

    git init
    git add k8s/kubernetes-manifests.yaml
    git commit -m "first commit"
    git branch -M main
    git remote add origin <URL_DO_SEU_REPOSITORIO>
    git push -u origin main

<!-- Ao final da execução dos comandos, o Git pedirá pelas suas credenciais do Github, primeiro escreva o *username* normalmente e depois disso ele vai pedir por uma senha, ela precisa vir de um token disponibilizado pelo próprio Github. -->

Pronto, após isso o commit com nosso arquivo foi feito no repositório remoto do Github, lembrando que para esse método a chave SSH precisa já estar conectada no computador com o Github.

<br><br>

## Etapa 2 – Instalar ArgoCD no cluster local
Nessa etapa iremos instalar o ArgoCD como operador no cluster Kubernetes, e também a ferramenta de linha de comando (CLI) do ArgoCD.

### Instalação no cluster

De início o primeiro passo é criar um namespace no Kubernetes para o ArgoCD operar:

    kubectl create namespace argocd

Em seguida instale o ArgoCD para que ele crie todos os recursos necessarios no cluster:

    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Para verificar se ele foi instalado corretamente:

    kubectl get pods -n argocd

Se a saída desse comando for diversos pods aparecendo na tela do terminal e eles estiverem no estado de *running* ou estiverem iniciando, quer dizer que o ArgoCD se instalou com sucesso no cluster.

### Ferramenta de linha de comando ArgoCD

Para instalar execute os comandos:

    curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
    sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
    rm argocd-linux-amd64

É somente isso, para testar se instalação foi um sucesso rode:

    argocd version

<br><br>

## Etapa 3 – Acessar ArgoCD localmente
O objetivo dessa etapa é acessar a interface gráfica (GUI) web do ArgoCD na rede local da nossa máquina, a partir disso podemos operar o ArgoCD por lá.

### Autentificação no ArgoCD
Precisamos expor a porta do serviço de interface web do ArgoCD rodando no cluster. Para isso execute o comando: (Para parar o port-forward, traga-o de volta para o primeiro plano com o comando fg e pressione Ctrl+C, ou use kill %1.)

    kubectl port-forward svc/argocd-server -n argocd 8080:443 &

Com isso o ArgoCD já está acessivel, porém para autentificar nele precisamos de uma senha, ele pode ser obtida com o comando abaixo, copie a saída dele:

    kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

Após obtida a senha e a aplicação exposta, podemos acessar o ArgoCD via terminal ou via navegador com interface gráfica, para isso digite **localhost:8080** na barra de endereço do seu navegador. Como já foi comentado, ele vai pedir usuario e senha, o usuario por padrão é sempre *admin* e a senha já obtemos no passo anterior. Agora é só acessar e logar no ArgoCD.

<img width="1827" height="947" alt="argocd-login" src="https://github.com/user-attachments/assets/9c17ee7a-47d8-4fab-96e1-333fc7f734a9" />

Se autentique via terminal de linha de comando também, usa o mesmo usuario e senha, para isso digite o comando:

    argocd login localhost:8080

<br><br>

## Etapa 4 – Criar o App no ArgoCD
Antes de conseguir criar o app, precisamos garantir que o ArgoCD consiga o acesso ao nosso repositório privado no Github.

### Criar chave SSH
Crie a chave SSH que será usada para acesso do ArgoCD, execute o seguinte comando no terminal:

    ssh-keygen -t rsa -b 4096 -f ./argocd-key -N "" -C "argocd-deploy-key"

Isso cria as chaves SSH pública e privada.

### Conectando chave pública
Copie o conteúdo da chave pública gerada. (a que termina com a extensão .pub)

Vá até a página do seu repositório no Github, acesse *Settings*, na barra lateral clique em *Deploy keys*, depois no botão verde escrito **Add deploy key**, no titulo da chave digite algo como *argocd*, e no outro campo cole a chave copiada.

### Conectando chave privada
Para isso, vá no Github e copie novamente a URL SSH do seu repo, também se certifique de saber o caminho no sistema que está a chave SSH que criamos nos passos anteriores (no caminho inclua a propria chave), o comando criava a chave no diretorio atual.

Com isso pronto, pode executar o comando que adiciona o repositório com a chave, só substituindo para os valores adequados:

    argocd repo add <URL_SSH_DO_SEU_REPO> --ssh-private-key-path <LOCAL_DA_CHAVE_PRIVADA>

### Criando a aplicação
Entrando no ArgoCD, na página inicial vai ter um botão escrito **CREATE APPLICATION** no centro da página, clique nele.

Ao clicar, vai abrir um formulário na tela para preencher com as informações referentes a aplicação. Preencha os seguintes campos:

- *Application name:* `(ex: gitops-microservices)`
- *Project name:* `default`
- *Sync policy:* `Automatic`
- *Enable auto-sync:* `On`
- *Repository URL:* `(ex: git@github.com:devbrunofernandes/gitops-microservices.git)`
- *Revision:* `HEAD`
- *Path:* `k8s/`
- *Cluster URL:* `https://kubernetes.default.svc`
- *Namespace:* `default`

Ao terminar de selecionar todas essas opções, clique no botão de criar no topo da página. Pronto, a aplicação vai ser criada no ArgoCD, e automaticamente vai sincronizar com o estado do nosso repositório remoto no Github, e criar todos os elementos que estão declarados no manifesto. (o app vai ficar no estado de *progressing*, pois o manifesto exemplo usa um load balancer, e nós estamos testando em ambiente local, sem load balancer)

<img width="1826" height="952" alt="argocd-app" src="https://github.com/user-attachments/assets/73c3e498-19eb-4965-9d9e-d36f9a8c8f54" />

<br><br>

## Etapa 5 – Acessar o front-end
O serviço Frontend na nossa aplicação é um ClusterIP, isso significa que ele só tem acesso a rede internamente dentro do cluster. Portanto para acessa-lo no nosso navegador, é necessario fazer um port-fowarding no serviço. Para isso execute o comando a seguir:

    kubectl port-forward service/frontend 8081:80

Simples assim, agora podemos acessar via navegador digitando: *localhost:8081* na barra de endereço.

<img width="1812" height="952" alt="online-boutique" src="https://github.com/user-attachments/assets/d5ccdfbe-8212-4c68-a01d-6752a7fa86a6" />

<br><br>

## Conclusão
Este projeto concluiu com sucesso a implementação de um fluxo GitOps de ponta a ponta. Ao final deste guia, fomos capazes de:

- Configurar um ambiente Kubernetes local com Minikube e Kubectl.

- Versionar o estado desejado de uma aplicação de microserviços (Online Boutique) em um repositório Git privado.

- Instalar e configurar o ArgoCD para monitorar o repositório.

- Conectar o ArgoCD ao cluster e ao repositório de forma segura usando chaves SSH.

- Validar o deploy acessando o frontend da aplicação.

O resultado é um sistema onde o Git é a única fonte da verdade, e o ArgoCD garante que o cluster reflita essa verdade, demonstrando na prática o poder do gerenciamento declarativo de aplicações.
