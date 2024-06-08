# Criação de uma Instância EC2 com Docker, Docker Compose, Prometheus, Grafana e Promsim

Neste artigo, vamos descrever o passo a passo para criar uma instância EC2 na AWS, configurar Docker e Docker Compose, e implementar Prometheus e Grafana para monitoramento. Utilizaremos o Promsim para gerar métricas de simulação. Este processo será automatizado utilizando um template CloudFormation.

## Passo a Passo

### 1. Definição do Template CloudFormation

O primeiro passo é criar um template CloudFormation que define todos os recursos necessários. Abaixo está o template completo:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Template para criar uma instância EC2 com Docker, Docker Compose, Prometheus, Grafana e Promsim

Resources:
  MySecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties: 
      GroupDescription: 'Security group for monitoring tools'
      VpcId: 'vpc-00c4068c76195eef9' # Substitua pelo ID do seu VPC
      SecurityGroupIngress: 
        - IpProtocol: 'tcp'
          FromPort: 8080
          ToPort: 8080
          CidrIp: '0.0.0.0/0'
        - IpProtocol: 'tcp'
          FromPort: 9090
          ToPort: 9090
          CidrIp: '0.0.0.0/0'
        - IpProtocol: 'tcp'
          FromPort: 22
          ToPort: 22
          CidrIp: '0.0.0.0/0'
        - IpProtocol: 'tcp'
          FromPort: 3000
          ToPort: 3000
          CidrIp: '0.0.0.0/0'

  MyEC2Instance:
    Type: 'AWS::EC2::Instance'
    Properties: 
      InstanceType: 't2.large'
      ImageId: 'ami-00beae93a2d981137' # ID da AMI do Amazon Linux 2 fornecido
      SubnetId: 'subnet-0d2c38ca49ba41812' # Substitua pelo ID da sua subnet
      SecurityGroupIds: 
        - !Ref MySecurityGroup
      KeyName: 'vockey' # Atualize para o nome do seu par de chaves
      UserData: 
        Fn::Base64: !Sub |
          #!/bin/bash
          # Definir o arquivo de log
          LOGFILE="/tmp/install_docker_compose.log"

          # Definir o diretório home do usuário ec2-user
          USER_HOME="/home/ec2-user"

          # Instalar Docker
          {
          echo "Instalando Docker..."
          sudo yum install -y docker
          sudo usermod -a -G docker ec2-user
          echo "Docker instalado."

          # Habilitar e iniciar o serviço Docker
          echo "Habilitando e iniciando o serviço Docker..."
          sudo systemctl enable docker
          sudo systemctl start docker
          echo "Serviço Docker iniciado."

          # Instalar Docker Compose
          echo "Instalando Docker Compose..."
          sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
          sudo chmod +x /usr/local/bin/docker-compose
          echo "Docker Compose instalado."

          # Criar diretórios e ajustar permissões para Prometheus
          echo "Criando diretórios e ajustando permissões para Prometheus..."
          mkdir -p /path/to/prometheus/data
          sudo chown -R 65534:65534 /path/to/prometheus/data
          echo "Diretórios para Prometheus criados e permissões ajustadas."

          # Criar diretórios e ajustando permissões para Grafana
          echo "Criando diretórios e ajustando permissões para Grafana..."
          mkdir -p /path/to/grafana/data
          sudo chown -R 472:472 /path/to/grafana/data
          echo "Diretórios para Grafana criados e permissões ajustadas."

          # Obter o IP público da instância
          echo "Obtendo o IP público da instância..."
          TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
          PUBLIC_IP=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" "http://169.254.169.254/latest/meta-data/public-ipv4")
          echo "IP público da instância: $PUBLIC_IP"

          # Criar o arquivo docker-compose.yml
          echo "Criando arquivo docker-compose.yml..."
          cat <<EOF > $USER_HOME/docker-compose.yml
          version: "3"

          services:
            prometheus:
              image: prom/prometheus:latest
              container_name: prometheus
              ports:
                - "9090:9090"
              volumes:
                - ./prometheus.yml:/etc/prometheus/prometheus.yml
                - /path/to/prometheus/data:/prometheus
              depends_on:
                - promsim

            grafana:
              image: grafana/grafana:latest
              container_name: grafana
              ports:
                - "3000:3000"
              volumes:
                - /path/to/grafana/data:/var/lib/grafana

            promsim:
              image: docker.io/dmitsh/promsim:0.3
              container_name: promsim
              ports:
                - "8080:8080"
              networks:
                - monitoring

          networks:
            monitoring:
              driver: bridge
          EOF
          echo "Arquivo docker-compose.yml criado."

          # Criar o arquivo prometheus.yml com IP público
          echo "Criando arquivo prometheus.yml com IP público..."
          cat <<EOF > $USER_HOME/prometheus.yml
          global:
            scrape_interval: 15s
            evaluation_interval: 15s

          scrape_configs:
            - job_name: "promsim"
              static_configs:
                - targets: ["$PUBLIC_IP:8080"]
          EOF
          echo "Arquivo prometheus.yml criado."

          # Alterar permissões dos arquivos criados
          sudo chown ec2-user:ec2-user $USER_HOME/docker-compose.yml
          sudo chown ec2-user:ec2-user $USER_HOME/prometheus.yml

          # Iniciar os serviços com Docker Compose
          echo "Iniciando os serviços com Docker Compose..."
          cd $USER_HOME
          sudo -u ec2-user docker-compose up -d
          echo "Serviços iniciados com Docker Compose."
          } &> $LOGFILE

          echo "Instalação e configuração concluídas. Veja o log em $LOGFILE"
```

### 2. Alterar o ID do VPC e Subnet

No template acima, substitua `vpc-00c4068c76195eef9` pelo ID do seu VPC e `subnet-0d2c38ca49ba41812` pelo ID da sua subnet.

### 3. Atualizar o Nome do Par de Chaves

Substitua `vockey` pelo nome do seu par de chaves que está configurado na AWS para acessar a instância EC2.

### 4. Executar o Template

1. Faça login no console da AWS.
2. Navegue até a seção de CloudFormation.
3. Crie uma nova stack utilizando o template acima.
4. Preencha os detalhes necessários e inicie a criação da stack.

### 5. Verificação da Instância e Serviços

Após a criação da stack, a instância EC2 será iniciada e o script de User Data será executado automaticamente. Verifique os seguintes pontos:

- A instância EC2 deve estar em execução.
- Docker e Docker Compose devem estar instalados.
- Os serviços Prometheus, Grafana e Promsim devem estar em execução e acessíveis através das portas 9090, 3000 e 8080, respectivamente.
- Verifique o log de instalação em `/tmp/install_docker_compose.log` para quaisquer erros.

### Conclusão

Seguindo este guia, você terá uma instância EC2 configurada com Docker, Docker Compose, Prometheus, Grafana e Promsim. Este setup permite monitorar seus serviços e infraestruturas de maneira eficiente e visualizá-los através do Grafana. Promsim será utilizado para gerar métricas de simulação, facilitando o teste e a validação do ambiente de monitoramento.
