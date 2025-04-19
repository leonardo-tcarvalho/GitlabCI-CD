# ⚙️ Setup de GitLab + Produção com CI/CD via SSH

## 🧱 Ambiente

- 2 VMs Linux no WSL2 (GitLab + Produção)
- Ambas com o mesmo IP local
- Redirecionamento de portas feito via `netsh` no Windows

## 📁 Pipeline CI/CD no GitLab

- A pipeline que usei está no repositório, caso queira dar uma olhada.

## 🚀 Passo a passo completo

```bash


# ================================
# 1. SERVIDOR DO GITLAB
# ================================

# Atualizar pacotes
sudo apt update && sudo apt upgrade -y

# Instalar dependências
sudo apt install -y curl openssh-server ca-certificates tzdata perl

# Adicionar repositório do GitLab CE
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash

# Instalar o GitLab
sudo EXTERNAL_URL="http://localhost" apt install gitlab-ce -y

# Alterar porta SSH para 2222
sudo nano /etc/ssh/sshd_config
# Altere:
#Port 22 → Port 2222

# Reinicie o serviço do SSH para que funcione corretamente
sudo systemctl restart ssh

# Adicionar repositório do GitLab Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash

# Instalar o GitLab Runner
sudo apt install gitlab-runner -y

# Registrar o Runner
sudo gitlab-runner register
# → Use as infos do GitLab: URL, token do projeto, executor: shell

# Gerar chave SSH
ssh-keygen -t rsa -b 4096 -C "gitlab"
# Salve como ~/.ssh/id_rsa (Enter nos prompts)

# Adicionar a chave privada no GitLab
# → Settings > CI/CD > Variables
# Nome: SSH_PRIVATE_KEY
# Valor: conteúdo de ~/.ssh/id_rsa
# Marcar como "Protected" e "Masked"



# ================================
# 2. SERVIDOR DE PRODUÇÃO
# ================================

# Atualizar pacotes
sudo apt update && sudo apt upgrade -y

# Instalar Apache
sudo apt install apache2 -y

# Alterar porta do Apache para 84
sudo nano /etc/apache2/ports.conf
# Adicione:
Listen 84

# Reinicie o Apache para que a porta altere corretamente
sudo systemctl restart apache2

# Alterar porta SSH para 2223
sudo nano /etc/ssh/sshd_config
# Altere:
#Port 22 → Port 2223

# Reinicie o serviço do SSH para que funcione corretamente
sudo systemctl restart ssh



# ============================================
# 3. CONFIGRAR AS CHAVES ENTRE OS SERVIDORES
# ============================================

# Copiar chave pública para Produção
# (Nessa parte é necessário utilizar senha para acessar o servidor)
scp -P 2223 ~/.ssh/id_rsa.pub <USUARIO_SERVIDOR_PRODUCAO>@<IP_PRODUCAO>:~/.ssh/authorized_keys

# No servidor de produção:
mkdir -p ~/.ssh
cat ~/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Acessar GitLab por SSH para salvar nos known_hosts
ssh -p 2222 <USUARIO_SERVIDOR_GITLAB>@<IP_GITLAB>

# Acessar Produção por SSH para salvar nos known_hosts
ssh -p 2223 <USUARIO_SERVIDOR_PRODUCAO>@<IP_PRODUCAO>

# Dar permissão para o runner escrever no Apache
sudo chown -R <USUARIO_SERVIDOR_PRODUCAO>:www-data /var/www/html
sudo chmod -R 775 /var/www/html



# ================================
# 4. CONFIGURAÇÃO DE REDIRECIONAMENTO DE PORTAS NO WINDOWS (PowerShell)
# ================================

# Siga a ordem abaixo, pois o servidor do GitLab e o Runner devem ser criados primeiro

# Redirecionar porta para o servidor GitLab, Executar o seguinte comando no PowerShell:
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=127.0.0.1 connectport=2222 connectaddress=<IP_DA_VM_GITLAB>

# Redirecionar porta para o servidor de Produção, Executar o seguinte comando no PowerShell:
netsh interface portproxy add v4tov4 listenport=2223 listenaddress=127.0.0.1 connectport=2223 connectaddress=<IP_DA_VM_PRODUCAO>
```
