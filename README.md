# Servidor de Arquivos com Samba no Ubuntu Server

## 1. Atualização do Sistema

Primeiro, atualize os pacotes do sistema:

```bash
sudo apt update
sudo apt upgrade -y
```

## 2. Instalação do Samba

Instale o Samba e suas ferramentas:

```bash
sudo apt install samba samba-common-bin -y
```

## 3. Backup da Configuração Original

Faça um backup do arquivo de configuração original:

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
```

## 4. Criação de Diretórios Compartilhados

Crie os diretórios que serão compartilhados:

```bash
# Diretório público (leitura/escrita para todos)
sudo mkdir -p /srv/samba/publico

# Diretório privado (apenas usuários autorizados)
sudo mkdir -p /srv/samba/privado

# Diretório apenas leitura
sudo mkdir -p /srv/samba/somente-leitura
```

## 5. Configuração de Permissões

Configure as permissões dos diretórios:

```bash
# Diretório público
sudo chmod -R 775 /srv/samba/publico
sudo chown -R nobody:nogroup /srv/samba/publico

# Diretório privado
sudo chmod -R 755 /srv/samba/privado
sudo chown -R root:sambashare /srv/samba/privado

# Diretório somente leitura
sudo chmod -R 755 /srv/samba/somente-leitura
sudo chown -R root:root /srv/samba/somente-leitura
```

## 6. Criação de Usuários Samba

Crie um grupo para usuários do Samba:

```bash
sudo groupadd sambashare
```

Adicione usuários ao sistema e ao Samba:

```bash
# Criar usuário do sistema
sudo useradd -M -d /srv/samba/privado -s /usr/sbin/nologin -G sambashare usuario1

# Adicionar usuário ao Samba (será solicitada uma senha)
sudo smbpasswd -a usuario1

# Habilitar o usuário
sudo smbpasswd -e usuario1
```

## 7. Configuração do Samba

Edite o arquivo de configuração:

```bash
sudo nano /etc/samba/smb.conf
```

Substitua o conteúdo pelo seguinte:

```ini
[global]
# Configurações globais
workgroup = WORKGROUP
server string = Servidor de Arquivos Ubuntu
netbios name = UBUNTU-SERVER
security = user
map to guest = bad user
dns proxy = no
log file = /var/log/samba/log.%m
max log size = 1000
logging = file
panic action = /usr/share/samba/panic-action %d

# Configurações de rede
interfaces = lo eth0
bind interfaces only = yes
server role = standalone server
passdb backend = tdbsam
obey pam restrictions = yes
unix password sync = yes
passwd program = /usr/bin/passwd %u
passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
pam password change = yes

# Configurações de desempenho
socket options = TCP_NODELAY IPTOS_LOWDELAY SO_RCVBUF=524288 SO_SNDBUF=524288
deadtime = 30
getwd cache = yes

#======================= Compartilhamentos =======================

# Diretório Home dos usuários
[homes]
comment = Diretórios Home
browseable = no
read only = no
create mask = 0700
directory mask = 0700
valid users = %S

# Compartilhamento público
[Publico]
comment = Diretório Público
path = /srv/samba/publico
browseable = yes
writable = yes
guest ok = yes
read only = no
create mask = 0775
directory mask = 0775
force user = nobody
force group = nogroup

# Compartilhamento privado
[Privado]
comment = Diretório Privado
path = /srv/samba/privado
browseable = yes
valid users = @sambashare
writable = yes
guest ok = no
read only = no
create mask = 0664
directory mask = 0775
force group = sambashare

# Compartilhamento somente leitura
[Somente-Leitura]
comment = Arquivos Somente Leitura
path = /srv/samba/somente-leitura
browseable = yes
read only = yes
guest ok = yes
```

## 8. Verificação da Configuração

Teste a configuração do Samba:

```bash
sudo testparm
```

## 9. Configuração do Firewall

Configure o firewall para permitir o tráfego Samba:

```bash
# Verificar status do UFW
sudo ufw status

# Permitir Samba
sudo ufw allow samba

# Ou permitir portas específicas
sudo ufw allow 137/udp
sudo ufw allow 138/udp
sudo ufw allow 139/tcp
sudo ufw allow 445/tcp
```

## 10. Inicialização dos Serviços

Inicie e habilite os serviços do Samba:

```bash
# Iniciar serviços
sudo systemctl start smbd
sudo systemctl start nmbd

# Habilitar inicialização automática
sudo systemctl enable smbd
sudo systemctl enable nmbd

# Verificar status
sudo systemctl status smbd
sudo systemctl status nmbd
```

## 11. Teste de Conectividade Local

Teste a conectividade local:

```bash
# Listar compartilhamentos
smbclient -L localhost

# Conectar ao compartilhamento público
smbclient //localhost/Publico

# Conectar ao compartilhamento privado (com usuário)
smbclient //localhost/Privado -U usuario1
```

## 12. Comandos de Gerenciamento

### Gerenciar usuários Samba:

```bash
# Listar usuários
sudo pdbedit -L

# Adicionar usuário
sudo smbpasswd -a nome_usuario

# Remover usuário
sudo smbpasswd -x nome_usuario

# Desabilitar usuário
sudo smbpasswd -d nome_usuario

# Habilitar usuário
sudo smbpasswd -e nome_usuario
```

### Monitoramento:

```bash
# Ver conexões ativas
sudo smbstatus

# Ver logs
sudo tail -f /var/log/samba/log.smbd
sudo tail -f /var/log/samba/log.nmbd
```

### Reiniciar serviços:

```bash
# Reiniciar após mudanças na configuração
sudo systemctl restart smbd
sudo systemctl restart nmbd
```

## 13. Conectar de Clientes

### Windows:
- Abra o Explorer
- Digite na barra de endereços: `\\IP_DO_SERVIDOR`
- Ou: `\\UBUNTU-SERVER`

### Linux:
```bash
# Instalar cliente
sudo apt install smbclient cifs-utils

# Conectar
smbclient //IP_DO_SERVIDOR/Compartilhamento -U usuario

# Montar permanentemente
sudo mkdir /mnt/samba
sudo mount -t cifs //IP_DO_SERVIDOR/Privado /mnt/samba -o username=usuario1
```

### Para montar automaticamente no boot, adicione ao `/etc/fstab`:
```
//IP_DO_SERVIDOR/Privado /mnt/samba cifs username=usuario1,password=senha,uid=1000,gid=1000,iocharset=utf8 0 0
```

## 14. Segurança Adicional

### Configurar acesso por IP:
Adicione ao `smb.conf` na seção `[global]`:

```ini
# Permitir apenas redes específicas
hosts allow = 192.168.1. 127.0.0.1
hosts deny = 0.0.0.0/0
```

### Configurar log detalhado:
```ini
# Na seção [global]
log level = 2
max log size = 50000
```

## 15. Troubleshooting

### Problemas comuns:

1. **Serviço não inicia:**
   ```bash
   sudo journalctl -u smbd
   sudo journalctl -u nmbd
   ```

2. **Problemas de permissão:**
   ```bash
   sudo chown -R nobody:nogroup /srv/samba/publico
   sudo chmod -R 775 /srv/samba/publico
   ```

3. **Não consegue conectar:**
   - Verificar firewall
   - Verificar se o usuário existe no Samba
   - Testar com `testparm`

4. **Recriar configuração:**
   ```bash
   sudo cp /etc/samba/smb.conf.backup /etc/samba/smb.conf
   ```

## Conclusão

Seu servidor Samba está configurado com:
- **Compartilhamento público**: Acesso livre para todos
- **Compartilhamento privado**: Apenas usuários autenticados
- **Compartilhamento somente leitura**: Acesso de leitura para todos
- **Diretórios home**: Cada usuário tem sua pasta privada

Lembre-se de fazer backups regulares da configuração e manter o sistema atualizado!
