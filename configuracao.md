# 🔸 3 METODOLOGIA

## 🟠 3.1 Preparando a Máquina Virtual

Para realizar a instalação do CentOs Stream, utilizamos o hipervisor VirtualBox em sua versão 7.x, que pode ser baixado no site oficial virtualbox.org.

As configurações da máquina virtual (VM) seguem as solicitações listadas na Tabela 3.1.1. A versão instalada do sistema é a sua versão 10, atualmente a mais recente, obtida diretamente do site oficial centos.org.

Inicializando o Virtual Box e criando uma máquina nova, colocamos os parâmetros solicitados. Os primeiros campos são o nome da máquina virtual, caminho em que ela será armazenada e a imagem (ISO) que será instalada na VM. 
* **VM name:** `CentOs_Stream_CP2_Oficial`
* **VM Folder:** Qualquer diretório/pasta conhecida que o usuário preferir.

Ao escolher o nome e ponto de montagem, o próximo passo é inserir as configurações “físicas” da máquina:
* **Memória:** 4096 MB (valor absoluto para o solicitado nominalmente de 4 GB)
* **CPUs:** 2 núcleos
* **Firmware:** Marcada a caixa “UEFI”

Por fim, a escolha do tamanho do disco, sendo criado o disco principal de 60 GB. No caso deste trabalho, criamos um disco novo, mas pode ser possível utilizar um existente, caso o usuário queira.

Clicando em finalizar, a máquina fica pronta para instalação do sistema, sendo listada como uma das opções, em caso da existência de mais de uma máquina.

---

## 🟠 3.2 Instalação do CentOs Stream

Com a máquina criada e padronizada, podemos inicializar através do botão de ícone de uma seta verde no VirtualBox, com a máquina selecionada. Como se trata de uma “simulação da vida real”, o sistema identificará que o boot deverá ser realizado pelo “CD”, no caso, a ISO que foi inserida durante a criação da máquina.

Ao aguardar o boot, entramos na tela de GRUB da instalação e selecionamos a opção de instalação, no caso, a primeira disponível. Durante a instalação, após o boot inicial, o CentOs nos guiará por diversas opções:

1. **Idioma:** Utilizaremos a opção **Português (Brasil)**. Isso permitirá menos problemas na hora de interagirmos com o teclado "brasileiro".
2. **Resumo da Instalação:** O menu contém diversos ícones. Um aviso no rodapé reforça a exigência da definição de parâmetros nas opções marcadas com uma exclamação (`!`).

Das opções do menu, fizemos o particionamento por último para documentar melhor o resultado. As solicitações a serem seguidas estão relacionadas na Tabela 3.2.1.

### Seleção de Programas
Escolhemos a **“instalação mínima” (Minimal install)** inicialmente, por ser um exercício de hardening e o modo mais rápido para reproduzir os passos. Nenhum pacote adicional foi instalado. Qualquer pacote necessário será instalado a quente. Ao confirmar a escolha, voltamos para o Menu de Resumo.

### Usuários e Credenciais
Na opção **“Conta Root”**, iremos habilitar o root e permitir conexões SSH:
* **Usuário:** `root`
* **Senha:** `root root`
> **Nota:** A senha do super usuário é fraca. Durante o hardening, iremos impedir que conexões como root sejam realizadas para provar que o processo está tendo efeito. O sistema pede confirmação dupla para aceitar senhas fracas.

Na opção **“Criar Usuários”**:
* **Usuário:** `bdoe`
* **Senha:** `Bob1234`
> O usuário não terá privilégios administrativos iniciais. Pode-se considerar colocá-lo na lista de “Sudoers” posteriormente.

### Rede e Nome do Host
A placa de rede está configurada como NAT por padrão no VirtualBox. Escolhemos manter para que o sistema possa se conectar à internet e realizar atualizações e instalação de pacotes.

### Particionamento de Disco (Destino da Instalação)
As especificações gerais estão descritas na Tabela 3.2.2. Exceto pelas partições de boot, **todas as outras devem ser criptografadas**. 

Após dimensionar as partições e clicar em “Pronto”, o sistema solicitará uma senha de criptografia:
* **Senha criptografia de disco:** `cripto1234`

Ao aceitar as mudanças, o disco foi particionado, restando aproximadamente 9 GB de espaço livre. 

### Kdump
De volta ao menu inicial, a configuração Kdump (serviço de captura de falhas do kernel) passa a ser obrigatória devido à criptografia. Inicialmente, iremos **desabilitar** essa opção.

Após essas configurações, clicamos em **“Iniciar instalação”**. Ao terminar, clicamos em “Reiniciar o sistema”. 

Durante o boot, a senha de criptografia do disco é solicitada. Após inseri-la, a tela de login é exibida.
Logando como `bdoe` e utilizando o comando `lsblk`, confirmamos os discos (partições criptografadas apresentam o Type `crypt`).

---

## 🟠 3.3 Configurações Pós Instalação

Logando como usuário root, realizamos o update do sistema utilizando o `dnf`. Notou-se que o serviço SSH e o editor `vim` não estavam disponíveis inicialmente. Após reiniciar, o serviço ssh ficou ativo.

Foi adicionada uma placa de rede Host-Only à VM. Um acesso remoto (com `bdoe` e depois `root`) foi realizado com sucesso.

### 🔸 Hardening do Serviço SSH

Iniciamos as modificações em `/etc/ssh/sshd_config`. Para evitar perder o acesso ("se trancar do lado de fora"), mantivemos uma sessão SSH paralela aberta.

Lendo o arquivo, há um aviso de que o SELinux pode impedir acessos SSH em portas diferentes (linha 17). Alteramos a porta padrão para `1919`. Ao testar, a conexão foi recusada na porta 22 e sofreu time out na 1919.

Para aplicar a permissão no SELinux, instalamos a ferramenta `semanage` (pacote `policycoreutils-python`):

```bash
# Verificando portas permitidas
[root@localhost ~]# semanage port -l | grep ssh

# Adicionando a porta 1919
[root@localhost ~]# semanage port -a -t ssh_port_t -p tcp 1919
Configuração do Firewall para permitir a porta 1919:

Bash
[root@localhost ~]# firewall-cmd --permanent --add-port=1919/tcp
success
[root@localhost ~]# firewall-cmd --reload
success
[root@localhost ~]# firewall-cmd --zone=public --query-port=1919/tcp
yes
[root@localhost ~]# systemctl restart sshd
Com a nova porta funcionando, tentamos remover a porta 22 do SELinux:

Bash
[root@localhost ~]# semanage port -d -t ssh_port_t -p tcp 22
ValueError: Port tcp/22 is defined in policy, cannot be deleted
A exclusão não é permitida pelo SELinux, mas a porta 22 já é recusada pelo Firewall e pelo sshd_config.

🔸 Bloqueio de Login Root e Escalada de Privilégios
Promovemos o usuário bdoe a sudoer para continuar as configurações remotamente:

Bash
[root@localhost ~]# usermod -aG wheel bdoe
[root@localhost ~]# id bdoe
uid=1000(bdoe) gid=1000(bdoe) grupos=1000(bdoe),10(wheel)
Para impedir o login remoto do root, alteramos PermitRootLogin no em /etc/ssh/sshd_config. Porém, o root continuava conectando.
Verificamos que havia um arquivo residual sobrescrevendo a configuração:

Bash
[bdoe@localhost ~]$ sudo sshd -T | grep permitrootlogin
permitrootlogin yes
Acessamos e modificamos o arquivo /etc/ssh/sshd_config.d/01-permitrootlogin.conf para no e reiniciamos o sshd. O root perdeu a permissão de acesso remoto.

Outras opções editadas no sshd_config:

ClientAliveInterval = 300 (tempo de vida)

LoginGraceTime = 30 (tempo mínimo)

MaxAuthTries = 3 (tentativas máximas)

Adicionamos um banner informativo através do arquivo /etc/issues.net.

🔸 Autenticação por Chave Pública (Passwordless)
Desabilitamos a solicitação de senha alterando PasswordAuthentication no no sshd_config.

No cliente, geramos a chave:

PowerShell
PS D:\> ssh-keygen -t ed25519
No servidor, preparamos o diretório para receber a chave pública:

Bash
[bdoe@localhost ~]$ mkdir .ssh
[bdoe@localhost ~]$ chmod 700 .ssh
[bdoe@localhost ~]$ cd .ssh/
[bdoe@localhost .ssh]$ touch authorized_keys
[bdoe@localhost .ssh]$ chmod 600 authorized_keys
Copiamos o conteúdo da chave pública (.pub) gerada no cliente (via PowerShell) e colamos dentro de ~/.ssh/authorized_keys no servidor utilizando um editor de texto. Após reiniciar o serviço SSH, o login sem senha funcionou perfeitamente.

🔸 Proteção de Partições (/etc/fstab)
Criou-se um snapshot como checkpoint. Para evitar a execução de binários maliciosos em diretórios sensíveis (/tmp, /home, /boot), manipulamos as opções de montagem.

Realizamos um backup do arquivo:

Bash
[bdoe@localhost etc]$ cp /etc/fstab /etc/fstab.bak
O quarto campo do fstab (fs_mntops) permite adicionar proteções separadas por vírgula. As opções sugeridas estão relacionadas na Tabela 3.3.1.

As modificações no /etc/fstab estão documentadas na seção “4 - Script: Baseline CIS”, confirmando a eficácia através de um script antes e depois das alterações.



🔸 4 SCRIPT: BASELINE CIS


🟠 4.1 Preparação do Ambiente

Antes de qualquer configuração de segurança, foi criado um diretório próprio dentro do sistema para guardar os scripts que seriam usados durante todo o processo. Esse diretório serve como um local organizado e centralizado para armazenar as ferramentas de auditoria e verificação que seriam desenvolvidas.

bash
[root@localhost ~]# mkdir -p /root/scripts

Em seguida foi criado o script chamado cis_baseline_check.sh, o programa responsável por verificar automaticamente se o servidor está seguindo todas as boas práticas de segurança esperadas. Esse script foi escrito para checar diversos pontos do sistema, como permissões de arquivos, configurações de montagem, programas com privilégios especiais e regras de auditoria.

Nota: o script não executa nenhuma modificação no sistema, apenas realiza leitura e verificação (auditoria).

bash
[root@localhost scripts]# vim /root/scripts/cis_baseline_check.sh
🟠 4.2 Hardening de Montagens (/etc/fstab)

Depois de ter o script pronto, o próximo passo foi abrir o arquivo /etc/fstab, responsável por dizer ao sistema como cada partição do disco deve ser montada.

bash
[root@localhost ~]# vim /etc/fstab

Nesse arquivo foram adicionadas três opções de segurança em várias partições, chamadas nodev, nosuid e noexec, conforme a Tabela 3.3.1:

Partição	Opções adicionadas
/boot	nodev,nosuid
/home	nodev,nosuid
/tmp	nodev,nosuid,noexec
/var	nodev
/var/log	nodev,nosuid,noexec
/var/tmp	nodev,nosuid,noexec

Depois de editar o arquivo, foi necessário avisar o sistema que a configuração mudou:

bash
[root@localhost ~]# systemctl daemon-reload

Em seguida, cada partição foi remontada individualmente, aplicando as novas regras de segurança sem precisar reiniciar a máquina:

bash
[root@localhost ~]# mount -o remount /boot
[root@localhost ~]# mount -o remount /home
[root@localhost ~]# mount -o remount /tmp
[root@localhost ~]# mount -o remount /var
[root@localhost ~]# mount -o remount /var/log
[root@localhost ~]# mount -o remount /var/tmp
[root@localhost ~]# mount -a

Durante esse processo apareceu um pequeno erro na partição /boot (fsconfig system call failed), resolvido apenas repetindo o comando.

Para confirmar que as opções realmente foram aplicadas, consultamos o estado real das montagens direto do kernel:

bash
[root@localhost ~]# findmnt -o TARGET,OPTIONS | grep -E '/tmp|/var|/home|/boot'
🟠 4.3 Execução Inicial do Script

O script criado anteriormente recebeu permissão de execução:

bash
[root@localhost scripts]# chmod +x /root/scripts/cis_baseline_check.sh

Após a atribuição da permissão, o script foi executado, realizando o benchmark do framework CIS contra o servidor:

bash
[root@localhost scripts]# ./cis_baseline_check.sh

Considerando a pontuação do script, foram realizadas as configurações necessárias para atender às sugestões do benchmark, preparando o ambiente para um segundo teste.

🟠 4.4 Redução de Superfície SUID/SGID

Foi executado um comando de busca que varreu todo o sistema de arquivos procurando por programas com o bit especial de permissão SUID ou SGID:

bash
[root@localhost scripts]# find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null

Esses são programas que, quando executados, ganham temporariamente os privilégios do dono do arquivo, geralmente o usuário root. O comando listou todos os programas com esse tipo de permissão ativa naquele momento.

Depois de ver a lista completa, foi feita a remoção desse privilégio especial em vários programas que não precisavam dele para funcionar corretamente no ambiente do servidor:

bash
[root@localhost scripts]# chmod u-s,g-s /usr/bin/chfn
[root@localhost scripts]# chmod u-s,g-s /usr/bin/chsh
[root@localhost scripts]# chmod u-s,g-s /usr/bin/write
[root@localhost scripts]# chmod u-s,g-s /usr/bin/newgrp
[root@localhost scripts]# chmod u-s,g-s /usr/bin/gpasswd
[root@localhost scripts]# chmod u-s,g-s /usr/bin/pkexec
[root@localhost scripts]# chmod u-s,g-s /usr/bin/chage
[root@localhost scripts]# chmod u-s,g-s /usr/bin/crontab
[root@localhost scripts]# chmod u-s,g-s /usr/sbin/pam_timestamp_check
[root@localhost scripts]# chmod u-s,g-s /usr/sbin/grub2-set-bootflag
[root@localhost scripts]# chmod u-s,g-s /usr/libexec/utempter/utempter
[root@localhost scripts]# chmod u-s,g-s /usr/lib/polkit-1/polkit-agent-helper-1
[root@localhost scripts]# chmod u-s,g-s /usr/sbin/unix_chkpwd
[root@localhost scripts]# chmod u-s,g-s /usr/bin/mount
[root@localhost scripts]# chmod u-s,g-s /usr/bin/umount

Apenas os programas essenciais para o funcionamento básico do sistema — passwd, su e sudo — mantiveram esse privilégio, já que sem ele deixariam de funcionar.

Para confirmar o resultado da limpeza, o comando de busca foi executado novamente:

bash
[root@localhost scripts]# find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null
🟠 4.5 Regras de Auditoria (auditd)

Foi criado um arquivo de regras para o serviço auditd, responsável por registrar e monitorar eventos importantes que acontecem no sistema. Dentro desse arquivo foram escritas três regras, vigiando qualquer alteração feita nos arquivos passwd, shadow e sudoers:

bash
[root@localhost scripts]# cat << 'EOF' > /etc/audit/rules.d/identity.rules
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k identity
EOF

Escrever o arquivo de regras não é suficiente para que elas comecem a funcionar de verdade. Por isso foi executado o comando augenrules com a opção load, que lê todas as regras escritas e carrega elas diretamente na memória do sistema:

bash
[root@localhost scripts]# augenrules --load

Para confirmar que as regras estavam realmente ativas em memória:

bash
[root@localhost scripts]# auditctl -l
🟠 4.6 Execução Final e Validação

O script de auditoria foi executado novamente para conferir o estado geral do sistema depois de todas as correções feitas:

bash
[root@localhost scripts]# ./cis_baseline_check.sh

Na primeira vez que ele rodou, antes das regras do auditd serem criadas, o resultado mostrou falhas relacionadas exatamente à ausência dessas regras (PASS=34 FAIL=3 WARN=1). Depois de criar e carregar as regras corretamente, o script foi executado mais uma vez e dessa vez todas as verificações passaram sem nenhuma falha (PASS=37 FAIL=0 WARN=1), mostrando que o sistema estava devidamente configurado e protegido conforme o esperado pelo framework.

🟠 4.7 Exportação do Relatório

Devido ao hardening aplicado, que bloqueou o acesso remoto direto do superusuário (root) via SSH, o relatório HTML foi copiado para o diretório home de um usuário comum antes de ser transferido:

bash
[root@localhost scripts]# cp /var/log/cis-baseline-check-report.html /home/bdoe/
[root@localhost scripts]# chown bdoe:bdoe /home/bdoe/cis-baseline-check-report.html

A transferência para a máquina física foi feita via SCP, utilizando a porta SSH customizada:

bash
PS C:\Users\maick\Desktop> scp -P 1919 bdoe@192.168.56.101:~/cis-baseline-check-report.html ~/Downloads/

