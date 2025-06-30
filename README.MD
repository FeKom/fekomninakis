# Blog Pessoal - fekomninakis.com.br

Bem-vindo ao repositório do meu blog pessoal, hospedado em fekomninakis.com.br. Este é um site estático construído com o framework Hugo, configurado com DNS via Registro.br e implantado no Firebase Hosting.

## Sobre o Projeto

Este blog é um espaço onde compartilho ideias, projetos e reflexões. Ele foi desenvolvido com o Hugo, um gerador de sites estáticos rápido e flexível, ideal para blogs e sites pessoais. A hospedagem no Firebase garante alta performance e escalabilidade, enquanto o domínio foi configurado pelo Registro.br.


## Tecnologias Utilizadas

- Hugo: Framework para geração de sites estáticos.
- Firebase Hosting: Plataforma de hospedagem para o deploy do site.
- Registro.br: Serviço utilizado para compra e configuração do domínio [fekomninakis.com.br](https://fekomninakis.com.br).

## Como Configurar Localmente
Para rodar o projeto localmente, siga os passos abaixo:

### Pré-requisitos

- Hugo instalado (versão recomendada: última versão estável).
- Git para clonar o repositório.
- Conta no Firebase (opcional, para deploy).

### Passos

1. Clone o repositório:
```bash
git clone https://github.com/seu-usuario/seu-repositorio.git
```


2. Navegue até o diretório do projeto:
```bash
cd seu-repositorio
```

3. Inicie o servidor de desenvolvimento do Hugo:
```bash
hugo server
```

4. Acesse o site localmente em [http://localhost:1313].


## Configuração do Domínio

O domínio [fekomninakis.com.br](https://fekomninakis.com.br) foi registrado pelo Registro.br. 

Para configurar o DNS:

1. Acesse o painel do Registro.br.
2. Configure os registros DNS para apontar para os servidores do Firebase Hosting (instruções disponíveis na documentação do Firebase).
3. Aguarde a propagação do DNS (pode levar até 48 horas).

## Deploy no Firebase
Para implantar o site no Firebase, utilizamos o script [./script-deploy-firebase.sh](/fekomninakis/script-deploy-firebase.sh). Siga estas etapas:

1. Instale o Firebase CLI:
```bash
npm install -g firebase-tools
```

2. Faça login no Firebase:
```bash
firebase login
```

3. Execute o script de deploy:
```bash
./script-deploy-firebase.sh
```


