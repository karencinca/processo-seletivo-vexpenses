# Desafio VExpenses - DevOps 

Candidata: **Karen Aires da Silva Cinca** 

✨<a href="#mag_right-análise-e-descrição-técnica-do-código-terraform">Análise e Descrição Técnica do Código Terraform</a>

✨<a href="#wrench-modificação-e-melhoria-do-código-terraform">Modificação e Melhoria do Código Terraform</a>

✨<a href="/main.tf">Arquivo modificado</a>

✨<a href="#bulb-observações">Observações</a>

✨<a href="#key-instruções">Instruções</a>

Este repositório contém o desafio proposto pela **VExpenses Despesas Corporativas**, que é uma das etapas do processo seletivo para a vaga de Estágio em DevOps. O objetivo é demonstrar conhecimentos em IaC (Infraestrutura como Código) utilizando Terraform como meio de automatizar a configuração de servidores. Também tem o propósito de expressar conhecimento em gerenciamento de recursos da AWS, priorizando a segurança e a manutenção dos recursos. 


## :mag_right: Análise e Descrição Técnica do Código Terraform

No primeiro trecho de código é definida a AWS como *cloud provider*, selecionando a região `us-east-1` (Norte da Virgínia, região Leste dos EUA), onde será provisionada a infraestrutura:

```tf
provider "aws" {
  region = "us-east-1"
}
```
Depois são criadas variáveis que armazenam nome do projeto e nome do candidato para fazer a concatenação para a criação dos nomes ao longo do projeto:

```tf
variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}


variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}
```

É criada, então, uma chave privada usando o algoritmo RSA de 2048 bits, assim é possível baixá-la e usá-la para se conectar ao servidor/instância EC2 via SSH de forma segura e sem precisar de senha:

```tf
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}
```

Também, é criada a *key pair* gerando a chave pública a partir da chave privada previamente declarada para permitir acesso à EC2, com nome baseado nas variáveis `${var.projeto}` e `${var.candidato}`:

```tf
resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
```

Em seguida, é declarado um recurso do tipo VPC com o nome `main_vpc`, criando uma rede isolada dentro da AWS servindo de alicerce para os recursos a serem criados. Depois é definido o bloco de endereços IP disponíveis para a VPC, o que significa que ela pode ter até 65.536 IPs privados (10.0.0.0 até 10.0.255.255). Além disso, a resolução de DNS é ativada, permitindo que a VPC use nomes de domínio internos em vez de apenas endereços IP. Também, permite que as instâncias dentro da VPC recebam nomes DNS públicos, essencial para acesso externo. Por último, é adicionada uma tag unindo o nome do projeto e o nome do candidato para facilitar a identificação no console:

```tf
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
```

A seguir, um recurso do tipo *subnet* é definido com o nome `main_subnet`. Como a *subnet* precisa estar dentro de uma VPC, é usado o ID da VPC já criada (`main_vpc`) para estabelecer que a *subnet* estará dentro dela. Com o `cidr_block` é definido o intervalo de endereços IP dentro da *subnet*, o que significa que essa *subnet* pode ter até 256 IPs (de 10.0.1.1 até 10.0.1.255). Esse bloco CIDR precisa estar dentro do bloco CIDR da VPC, que foi antes indicado como 10.0.0.0/16, sendo assim, um pedaço desse espaço foi reservado para essa *subnet*. Em `availability_zone` é selecionada em qual zona de disponibilidade a *subnet* será criada, que nesse caso ficará no Leste dos EUA. Por último, é adicionada uma tag unindo o nome do projeto e o nome do candidato para facilitar a identificação no console:

```tf
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
```

Depois, um recurso do tipo Internet Gateway chamado `main_igw` é definido. Ele associa o Internet Gateway à VPC criada anteriormente através do ID da VPC. Dessa forma, a VPC é conectada à internet (apesar de precisar da *route table* para funcionar). Depois, é adicionada uma tag unindo o nome do projeto e o nome do candidato para facilitar a identificação no console:

```tf
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
```

Em seguida, é criada uma *route table* chamada `main_route_table` associando à VPC anteriormente criada. Ao utilizar "0.0.0.0/0" no `cidr_block`, significa que essa rota é para todo o tráfego externo. Depois esse tráfego é enviado para o *Internet Gateway* já definido. Por último, é adicionada uma tag para facilitar a visualização no console:

```tf
resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}
```
A *route table* é associada com a *subnet* já definida, permitindo os recursos acessarem a internet. Por último, é adicionada uma tag que seria para facilitar a visualização no console, mas esse argumento não é válido para o recurso `aws_route_table_association`:

```tf
resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}
```
Posteriormente, é definido um grupo de segurança criado para controlar quais conexões podem entrar e sair para os recursos da VPC (*subnets* e *route tables* associadas). No `ingress` são declaradas as regras de entrada, que nesse caso está permitindo conexões SSH (porta 22) de qualquer IP, visto que o `cidr_blocks` está como "0.0.0.0/0".

No `egress` estão as configurações de saída, que nesse caso está da forma padrão, ou seja, é permitida a conexão de qualquer porta e qualquer IP, assim a instância EC2 pode acessar qualquer serviço na internet (como baixar atualizações, acessar APIs, entre outros exemplos). Por último, é adicionada a tag para facilitar a visualização no console:

```tf
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de qualquer lugar e todo o trafego de saida"
  vpc_id      = aws_vpc.main_vpc.id

  # Regras de entrada
  ingress {
    description      = "Allow SSH from anywhere"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  # Regras de saída
  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}
```

Em seguida, é feita a busca pela versão mais recente do Debian 12. O *owner ID* reforça a busca pela AMI (*Amazon Machine Image*) esperada, sendo assim, é selecionado o Debian 12 como o sistema operacional da instância:

```tf
data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}
```

Após isso, é declarado um recurso `aws_instance` a fim de subir uma instância EC2 do tipo `t2.micro` (1 vCPU, 1GB RAM) e que será criada dentro da *subnet* definida anteriormente. O `key_name` permite acessar a EC2 usando a chave já criada com `aws_key_pair`. Depois, é associado o grupo de segurança já criado anteriormente, permitindo acesso via SSH e saída para a internet. 

Para permitir a conexão via SSH de qualquer lugar, a EC2 recebe um IP público para acessar a internet.
Por último, é definido que a EC2 terá um disco de 20GB no tipo gp2 (SSD padrão) e que o disco será excluído quando a EC2 for deletada:

```tf
resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }
```

Depois, é declarado o argumento `user_data`, que executa um script automaticamente quando a instância é iniciada. Nesse caso, ele atualiza a lista de pacotes e atualiza as dependências do sistema operacional. Para facilitar a identificação da EC2, é adicionada uma tag usando o nome do projeto e o nome do canditado:

```tf
  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
```
Então, é gerado o *output* da chave privada usada para conectar via SSH. A chave é ocultada no terminal por segurança:

```tf
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}
```

Por fim, é feito o *output* do IP público da instância EC2 que foi criada. Esse IP é gerado automaticamente pela AWS:

```tf
output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```
Apesar do código apresentado possuir os comandos essenciais para a criação de uma instância na AWS, alguns trechos não condizem com o que a documentação do Terraform orienta. Tais peculiaridades são exploradas na seção de Modificação e Melhoria do Código Terraform.


## :wrench: Modificação e Melhoria do Código Terraform

### Chave privada para conexão via SSH
Como já foi mencionado, a instância EC2 possui uma chave privada para que possa acontecer a conexão via SSH. Foi inserido o código abaixo para fazer com que essa chave seja salva automaticamente no arquivo do projeto assim que ele é executado e, assim, facilitar o processo de conexão:
```tf
resource "local_file" "TF-key" {
  content  = tls_private_key.ec2_key.private_key_pem
  filename = "tfkey"
}
```

### Route Table Connection sem tags
No código apresentado, é criada uma tag `Name` no recurso referente à conexão das tabelas de rotas, entretando, esse comando não é válido para essa situação. Sendo assim, esse trecho de código foi retirado. 

### Regras de entrada e saída no grupo de segurança
As regras de entrada e saída no grupo de segurança foram escritas usando o `ingress` e o `egress` como argumentos do recurso `aws_security_group` no código original. Porém, a fim de evitar conflitos de regras envolvendo blocos CIDR, o código foi alterado, passando a utilizar um recurso para cada regra de entrada e saída:

```tf
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH e HTTP de qualquer lugar e todo o trafego de saida"
  vpc_id      = aws_vpc.main_vpc.id
}

resource "aws_vpc_security_group_ingress_rule" "allow_ssh_key" {
  security_group_id = aws_security_group.main_sg.id

  cidr_ipv4   = "0.0.0.0/0"
  from_port   = 22
  ip_protocol = "tcp"
  to_port     = 22
}

resource "aws_vpc_security_group_ingress_rule" "allow_http" {
  security_group_id = aws_security_group.main_sg.id

  cidr_ipv4   = "0.0.0.0/0"
  from_port   = 80
  ip_protocol = "tcp"
  to_port     = 80
}

resource "aws_vpc_security_group_egress_rule" "allow_all" {
  security_group_id = aws_security_group.main_sg.id

  cidr_ipv4   = "0.0.0.0/0"
  ip_protocol = "-1"
}
```

Vale observar que no recurso `allow_ssh_key` não foi definido um `cidr_ipv4` para um IP específico. O ideal nesse caso seria atribuir para um IP fixo de uma VPN, onde os usuários seriam obrigados a estarem conectados para conseguir acessar o recurso alvo via SSH (uma EC2, por exemplo). Isso serviria como uma camada adicional de segurança, além do uso de chave assimétrica para SSH configurada anteriormente nesse desafio. Para fins de conveniência, foi permitido o acesso SSH de qualquer IP.

### Definição do grupo de segurança da instância
No código apresentado, o grupo de segurança é relacionado com a instância a partir do seguinte código no `resource "aws-instance"`:

```tf
security_groups = [aws_security_group.main_sg.name]
```
Entretanto, ao criar instâncias em uma VPC, a relação é para ser feita através do ID do grupo de segurança, trocando a linha de código anterior pela linha abaixo:

```tf
vpc_security_group_ids = [aws_security_group.main_sg.id]
```

### Automação da instalação do Nginx
Ao executar uma instância, existe a opção de passar dados que podem ser usados para realizar tarefas de configuração comuns automatizadas ou executar scripts logo após a inicialização da instância. Dessa forma, foram utilizados comandos de instalação do Nginx no `user_data`, que fica no recurso da `aws_instance`. Assim, o Ngnix é instalado e iniciado automaticamente após a criação da instância. 
Para fins de organização do código, foi criado um arquivo separado com o script de instalação e inicialização do Nginx: 

```tf
user_data = file("./install-nginx.sh")
```

Arquivo `install-nginx.sh`:

```bash
#!/bin/bash
sudo apt-get update -y
sudo apt-get install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

Para fins de assertividade, o comando `apt-get upgrade -y` foi removido, visto que ele não é necessário para o desafio proposto. Além disso, há o risco do comando causar conflitos ou erros, pois atualiza todas as dependências do sistema operacional de forma indiscriminada, podendo causar efeitos colaterais indesejados. O recomendado é sempre atualizar as dependências verificando as suas versões uma a uma.

Para que seja possível acessar o Nginx no navegador, foi adicionada uma regra de entrada no grupo de segurança, sendo permitido o acesso à instância via HTTP:

```tf
resource "aws_vpc_security_group_ingress_rule" "allow_http" {
  security_group_id = aws_security_group.main_sg.id

  cidr_ipv4   = "0.0.0.0/0"
  from_port   = 80
  ip_protocol = "tcp"
  to_port     = 80
}
```

Se o Nginx for utilizado como um proxy reverso de um *web server* servindo requisições HTTP, ainda é possível liberar a porta 443 para o protocolo TCP/IP a fim de prover HTTPS para as requisições, uma vez que o TLS for corretamente configurado no Nginx.

## :bulb: Observações
Tendo em vista a utilização do Terraform com GitHub Actions, uma sugestão é utilizar o [OpenTofu](https://opentofu.org/) (um fork do Terraform), já que ele usa licença MPL, que é mais permissiva do que a licença utilizada atualmente pelo Terraform da HashiCorp.

##  :key: Instruções
Considerando que o Terraform e a AWS CLI (AWS Command Line Interface) já estão instalados e configurados, seguir os seguintes passos para executar o projeto:

1. Clone o projeto:
```
$ git clone https://github.com/karencinca/processo-seletivo-vexpenses
```

2. Acesse a pasta do projeto:
```
$ cd processo-seletivo-vexpenses
```
3. Inicialize o Terraform:
```
$ terraform init
```
4. Veja o plano de execução do Terraform:
```
$ terraform plan
```
5. Execute as ações do Terraform:
```
$ terraform apply
```

Com esses passos, a instância é criada na plataforma da AWS.
Para fazer a conexão via SSH, dentro do diretório do repositório, execute o seguinte comando no terminal, colocando no `ec2_public_ip` o endereço IP que foi gerado:
```
$ ssh -i tfkey admin@ec2_public_ip
```

Para conectar via HTTP, basta acessar o endereço IP no navegador.

##

###### Feito com 💙 por [Karen Cinca](https://www.linkedin.com/in/karencinca/).
