# kubernetes-local-instalation

Aprenda a provisionar um cluster Kubernetes, na sua máquina, sem o Minikube ou Kind

## Quais serão os pré-requisitos?

O virtualizador Virtualbox
No mínimo 3 instâncias do Rocky Linux 9 ou AlmaLinux 9 (sendo uma para o master e duas workers)
Usuário sudo com direitos de administrador
Mínimo de 2 GB de RAM, 2 vCPUs e 20 GB de espaço em disco
Uma conexão confiável com a Internet

## Introdução

Neste artigo, mostrarei como instalar e configurar um cluster Kubernetes no Rocky Linux 9. O número de workers dependerá da quantidade de memória ram que você tem disponível em sua máquina. Considere alocar pelo menos dois workers.

Faremos uma instalação template, que servirá como template para todas as outras máquinas. A configuração no Virtualbox deve ficar assim:

![Configuração do Virtualbox](https://github.com/andersonvm/kubernetes-local-instalation/blob/main/1.png)

Observe que, a PAE/NX deve estar habilitada e é recomendável alocar dois núcleos de CPU.

![Configuração do Virtualbox](https://github.com/andersonvm/kubernetes-local-instalation/blob/main/2.png)

Visando facilitar os trabalhos, para fins de acesso SSH, coloque a configuração da placa de rede em modo bridge. Assim, a VM irá oter um IP da rede onde sua máquina está ligada.

A instalação do sistema pode seguir os procedimentos costumeiros.

Observação: SEM INTERFACE GRÁFICA

Para fins de facilitação, acesse a VM através da sua máquina hospedeira por meio de SSH.

## Vamos aos primeiros passos

Agora, precisamos criar um usuário comum que irá elevar seus privilégios com sudo.

A configuração padrão do 'sudo' no Red Hat geralmente envolve a inclusão de usuários específicos em um grupo denominado 'wheel'. Os membros deste grupo têm permissão para executar comandos com 'sudo', desde que estejam autorizados no arquivo de configuração '/etc/sudoers'.

Além disso, o arquivo '/etc/sudoers' oferece um alto nível de flexibilidade na definição de permissões, permitindo que administradores concedam acesso a comandos específicos ou diretórios, limitando assim o escopo do que um usuário pode fazer com privilégios elevados.

Criação do usuário para administração do cluster

Faça login como root no sistema:

su - root

Agora, criaremos um usuário chamado sysops para administrar o cluster, e o adicionaremos no grupo wheel:

useradd -G wheel sysops

Agora, vamos atribuir um password para o novo usuário:

echo "SUA SENHA AQUI" | passwd sysops --stdin

Lembre-se de colocar sua senha.

Caso você já tenha criado um usuário, que não seja o root, e queira utilizá-lo, faça o seguinte:

usermod -aG wheel  NOME DO USUÁRIO

Agora, vamos configurar para que o usuário criado não precise utilizar senha ao invocar os privilégios de super user com o comando sudo:

su -

echo -e "sysops\tALL=(ALL)\tNOPASSWD: ALL" > /etc/sudoers.d/sysops

Entenda que, este passo é EXTREMAMENTE DESENCORAJADO EM PRODUÇÃO. Este artigo tem apenas o intuito de ensinar a criar um cluster Kubernetes para fins de estudo e ensaios.

## Instalação do Kubernetes

Para melhor funcionamento do kubelet, devemos desabilitar o swap:

Atenção! Todas as operações agora serão realizadas com o usuário sysops

sudo swapoff -a

sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

As distribuições baseadas no Red Hat, tem como adicional de segurança o chamado SELinux.

SELinux (Security-Enhanced Linux) é um mecanismo de controle de acesso obrigatório (MAC) implementado no kernel do Linux. Desenvolvido pela National Security Agency (NSA) em parceria com a Red Hat, SELinux reforça as políticas de segurança do sistema, oferecendo um nível adicional de proteção contra ameaças, mesmo em ambientes multiusuário.

Em sistemas Red Hat, SELinux é frequentemente utilizado para garantir a integridade do sistema e a segurança das informações. Ele trabalha controlando o acesso de usuários, programas e processos aos recursos do sistema, como arquivos, diretórios, portas de rede e sockets. Ao fazer isso, SELinux impõe políticas de segurança rigorosas, especificando exatamente quais ações são permitidas e quais são negadas, mesmo para usuários privilegiados.

O usuário "sudo" é comumente usado em sistemas Red Hat para executar comandos com privilégios de superusuário ou root temporariamente. No entanto, em ambientes SELinux, mesmo quando um usuário é autorizado a usar o "sudo", as políticas de segurança do SELinux ainda se aplicam. Isso significa que as permissões concedidas pelo "sudo" podem ser restringidas ou ampliadas com base nas políticas SELinux configuradas no sistema.

Devemos fazer a seguinte alteração:

sudo setenforce 0

sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux

Agora, permitiremos o trafego em algumas portas através do firewalld:

sudo firewall-cmd --permanent --add-port={6443,2379,2380,10250,10251,10252,10257,10259,179}/tcp

sudo firewall-cmd --permanent --add-port=4789/udp

sudo firewall-cmd --reload

A explicação é a seguinte:

Porta TCP 179: Esta porta é usada para o protocolo BGP (Border Gateway Protocol). No contexto do Kubernetes, pode ser necessária para comunicação entre nós de diferentes clusters, especialmente em cenários de cluster federado ou em configurações de rede complexas onde o roteamento dinâmico é necessário.

Porta TCP 10250: Esta porta é usada pelo Kubelet, que é um dos principais componentes em cada nó do Kubernetes. O Kubelet é responsável por gerenciar os pods e os containers em um nó específico. Abrir essa porta permite que o kubelet seja acessado por outros componentes do cluster, como o API Server, para monitoramento e gerenciamento.

Portas TCP 30000-32767: Essa faixa de portas é usada para os serviços do tipo NodePort. Quando um serviço é exposto com o tipo NodePort, o Kubernetes aloca automaticamente uma porta na faixa especificada para encaminhar o tráfego para o serviço. 
Abrir essas portas no firewall permite que os clientes externos acessem os serviços implantados no cluster através de seus NodePorts.

Porta UDP 4789: Esta porta é usada pelo VXLAN (Virtual eXtensible LAN), que é um mecanismo de encapsulamento de rede comumente usado em redes overlay no Kubernetes, como o Calico ou o Flannel. VXLAN é usado para permitir a comunicação entre pods em diferentes nós do cluster, formando uma rede virtual privada. Abrir esta porta é necessário para garantir que os pacotes VXLAN possam ser transmitidos entre os nós do cluster.

Agora, devemos adicionar os módulos de kernel overlay e br_netfilter:

sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

Agora, devemos carregar os módulos:

sudo modprobe overlay

sudo modprobe br_netfilter

Agora, adicionamos os seguintes parâmetros:

sudo vim /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1

Rode o comando:

sudo sysctl --system

Uma breve explicação:

Overlay: O módulo "overlay" é fundamental para redes overlay no Kubernetes. Redes overlay são uma técnica de virtualização de rede que permite que os pods em diferentes nós do cluster se comuniquem como se estivessem na mesma rede local, independentemente da rede física subjacente. O "overlay" fornece uma camada de abstração sobre a rede física, permitindo que os pods se comuniquem por meio de uma rede virtual.

Por que é necessário: No Kubernetes, redes overlay são amplamente utilizadas por plugins de rede como o Flannel, Calico e Weave, que criam uma rede virtual para conectar os pods em todo o cluster. Para que esses plugins funcionem corretamente, o módulo "overlay" deve estar disponível no kernel do sistema.

br_netfilter: O módulo "br_netfilter" é usado para habilitar regras de filtragem de pacotes no nível da bridge no kernel Linux. Ele permite que o sistema aplique regras de filtragem de pacotes em pontes de rede, como a utilizada pelo Kubernetes para encaminhar o tráfego de rede entre os pods.

Por que é necessário: No Kubernetes, a filtragem de pacotes é fundamental para garantir a segurança e o isolamento entre os pods. O "br_netfilter" é necessário para aplicar as políticas de segurança definidas pelo administrador do cluster, como regras de firewall e políticas de segurança de rede, garantindo que apenas o tráfego autorizado seja permitido entre os pods e para fora do cluster.

Agora, devemos instalar o containerd:

sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

Observe que ele não vem por default nos repositórios do Rocky Linux, e por isso que adicionamos este repositório.

Agora, instalemos o pacote propriamente dito:

sudo yum install containerd.io -y

Agora devemos configurar o containerd para que ele utilize o systemcgroup:

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

Agora, devemos restartar o serviço e habilitá-lo para início automático com o sistema:

sudo systemctl restart containerd

sudo systemctl enable containerd

Uma breve explicação:

Instalar o containerd é fundamental para gerenciar eficientemente os containers em um ambiente Kubernetes. Aqui estão algumas razões pelas quais o containerd é essencial:

Gerenciamento de containers: O containerd é um dos componentes principais do ecossistema de containers do Kubernetes. Ele é responsável por executar e gerenciar os containers no nível do sistema operacional. Isso inclui a criação, execução, parada e remoção de containers, bem como o gerenciamento de volumes, redes e namespaces.

Compatibilidade com padrões de containers: O containerd implementa os padrões do Open Container Initiative (OCI), o que o torna compatível com uma ampla variedade de ferramentas e frameworks de containers. Isso significa que você pode usar o containerd com uma variedade de ferramentas de desenvolvimento e implantação de containers, garantindo a interoperabilidade e a portabilidade das suas cargas de trabalho.

Integração com o ecossistema Kubernetes: O containerd é amplamente integrado com o Kubernetes e é suportado como o runtime de containers padrão no Kubernetes. Ele é utilizado pelo kubelet (um dos principais componentes do Kubernetes) para executar e gerenciar os containers em cada nó do cluster. Isso significa que ao instalar o containerd, você estará usando uma solução que é bem suportada e otimizada para ambientes Kubernetes.

Segurança e confiabilidade: O containerd é desenvolvido com foco na segurança e na confiabilidade. Ele oferece recursos avançados de isolamento de containers, garantindo que cada container tenha seu próprio ambiente isolado e seguro. Além disso, o containerd é altamente testado e utilizado em ambientes de produção, proporcionando confiabilidade e estabilidade para suas cargas de trabalho.

Desempenho: O containerd é projetado para ser leve e eficiente, minimizando o consumo de recursos do sistema. Ele oferece um desempenho excepcional, garantindo que suas cargas de trabalho de containers sejam executadas de forma rápida e eficiente, sem comprometer a performance do sistema.

Agora, instalaremos os componentes do Kubernetes (kubeadm, kubctl e o kublet) na versão 1.28.

Como não faz parte dos repositórios padrões, devemos adicionar o mesmo:

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

Agora, instalar os pacotes:

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

Após a instalação, vamos habilitar o kublet para início:

sudo systemctl enable --now kubelet

Pronto, o básico foi concluído

Observe que estes passos são comuns tanto para o nó master quanto para todos os workers. Por isso, no início deste artigo, criamos uma máquina template, pois as configurações seriam as mesmas e não faz sentido repetir para cada uma das VMs.

Agora vem o "pulo do gato"

Iremos clonar a máquina template, herdando assim todas as configurações já realizadas, poupando trabalho adicional desnecessário.

Para isso, desligue a VM e faça os seguintes passos:

Selecione a VM criada e clique com o botão direito do mouse:

![Template](https://github.com/andersonvm/kubernetes-local-instalation/blob/main/4.png)

Clique na opção Clonar e configure desta maneira para o master:

![Clone da Máquina Virtual](https://github.com/andersonvm/kubernetes-local-instalation/blob/main/5.png)

Não se esqueça de alterar a opção Política de Endereço MAC para gerar um novo endereço para a VM clonada.

Clique em proximo.

Agora, selecione a opção Clone Linkado.

Aqui estão as vantagens de criar um clone linkado:

![Clone da Máquina Virtual](https://github.com/andersonvm/kubernetes-local-instalation/blob/main/6.png)

Economia de espaço em disco: Um dos principais benefícios de um clone linkado é a economia de espaço em disco. Como os clones linkados compartilham o mesmo arquivo de disco rígido virtual com a máquina virtual original, eles ocupam muito menos espaço em disco do que uma cópia completa. Isso pode ser especialmente útil quando você precisa criar múltiplas instâncias de uma máquina virtual, mas deseja minimizar o uso de espaço em disco.

Rápida criação de clones: Clonar uma máquina virtual linkada é geralmente mais rápido do que criar uma cópia completa, pois não é necessário copiar todos os arquivos do disco rígido virtual. Isso é útil quando você precisa criar clones rapidamente para testes, desenvolvimento ou implantação de múltiplas instâncias de um ambiente.

Facilidade de gerenciamento: Como os clones linkados compartilham os mesmos arquivos de disco rígido virtual, qualquer alteração feita no disco pela máquina virtual original é refletida nos clones. Isso simplifica o gerenciamento, pois você só precisa atualizar um conjunto de arquivos para aplicar alterações a todos os clones.

Isolamento de ambientes: Embora os clones linkados compartilhem os mesmos arquivos de disco rígido virtual, eles ainda mantêm configurações de máquina virtual individuais, como configurações de rede, recursos de hardware virtual, etc. Isso permite que você crie ambientes isolados para testes ou desenvolvimento, mesmo compartilhando os mesmos dados de disco.

Como não iremos realizar alterações de hardware e ou alterações de dispositivos, está é a melhor opção neste cenário.

Faça este procedimento para o k8s-master1, k8s-worker1, k8s-worker2...

Agora, vamos realizar algumas configurações em cada uma das VMs

Primeiro, alterar o hostname:

Em cada um das VMs criadas, entre com o seguinte comando:

sudo hostnamectl set-hostname “k8s-master1” && exec bash

sudo hostnamectl set-hostname “k8s-worker1” && exec bash

sudo hostnamectl set-hostname “k8s-worker2” && exec bash

Nota: Considere que cada comando é realizado em sua respectiva VM

Para saber qual o IP foi atribuído a cada VM, utilize o comando:

ip a | grep enp0s3

Faça em cada uma e anote os repectivos IPs.

Agora, devemos adicionar as seguintes entradas em cada uma das VMs, no arquivo /etc/hosts:

192.168.X.XXX   k8s-master1
192.168.X.XXX   k8s-worker1
192.168.X.XXX   k8s-worker2

Agora, com todos estes procedimentos já realizados, estamos prontos para iniciar e configurar o cluster Kubernetes.

No nó master, rode o seguinte comando:

sudo kubeadm init --control-plane-endpoint=k8s-master1

Agora, realizaremos algumas configurações para perfeito funcionamento.

No nó master, rode os seguintes comandos:

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

Agora, devemos adicionar os workers ao cluster.

Observer que ao executar o comando

sudo kubeadm init --control-plane-endpoint=k8s-master1

foi informado, ao final do mesmo, um outro comando para adcionar os nós workers ao cluster. Algo parecido com este:

kubeadm join k8s-master1:6443 --token 69s57o.3mu6sa54f645564dfq69 \
  --discovery-token-ca-cert-hash sha256:8000dff8e803e2bfasdf654687f896saf4asd6f5sd654sadf8752a3c9fa314de6449fe

Copie este comando para adcionar cada um dos nós workers.

No terminal do nó 1, execute o comando copiado e aguarde finalizar.

Faça o mesmo para o nó 2.

Agora, no nó master, execute o seguinte comando:

kubectl get nodes

Você irá perceber que serão exibidos os nós do control-plane, e os dois nós workers adicionados. Porém, o status será NotReady.

Para que eles estejam com status Ready, é necessário instalar um componente para controle e comunicação entre os pods e fazer o serviço de DNS funcionar.

Iremos utilizar o Calico.

Para instalar, execute o seguinte comando no nó master:

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

Agora, verifique se está tudo ok:

kubectl get pods -n kube-system

Aguarde os pods do Calico estarem com status Running.

Agora, excute novamente o comando:

kubectl get nodes

Você irá perceber que agora os nós estão se comunicando entre si e o seu cluster está pronto para uso.
