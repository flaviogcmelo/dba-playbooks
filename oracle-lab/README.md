# Oracle Lab — Automação com Ansible

Playbooks Ansible para montar um laboratório Oracle 19c no VirtualBox com:

- **RAC 2 nós** com Grid Infrastructure + ASM (OCR, DATA, FRA)
- **Single Instance + ASM** com Data Guard (Primary + Standby)

---

## Estrutura do projeto

```
oracle-lab/
├── inventory/
│   ├── rac/hosts.yml          → IPs e configurações dos nós RAC
│   └── si/hosts.yml           → IPs do Primary e Standby (Data Guard)
├── group_vars/
│   └── all.yml                → Variáveis globais (versão, paths, senhas, etc.)
├── roles/
│   ├── oracle-prereqs/        → Pacotes, kernel, limits, usuários, udev, SSH
│   ├── grid-install/          → Instalação silenciosa do GI 19c
│   ├── asm-diskgroups/        → Criação dos diskgroups DATA e FRA
│   ├── oracle-install/        → Instalação do software Oracle DB 19c
│   └── oracle-dbca/           → Criação do banco via DBCA + config Data Guard
└── playbooks/
    ├── setup-rac.yml           → Orquestra instalação RAC completa
    ├── setup-si.yml            → Orquestra instalação SI (para Data Guard)
    └── setup-dataguard.yml     → Configura DG entre Primary e Standby
```

---

## Pré-requisitos

### VirtualBox — configuração das VMs

#### RAC (2 VMs: rac1 e rac2)

Cada VM precisa de:
- **CPU**: 2 vCPUs
- **RAM**: 4GB (mínimo — 8GB recomendado)
- **Disco OS**: 50GB (SATA)
- **Rede**:
  - Adapter 1: Host-Only (rede pública: 192.168.56.x)
  - Adapter 2: Host-Only (rede privada/interconnect: 10.10.10.x)

**Shared disks** (criados uma vez, anexados às duas VMs):

```bash
# Criar discos compartilhados (execute no host, não na VM)
# Tipo "Shareable" é necessário para RAC no VirtualBox

VBoxManage createmedium disk --filename ~/VirtualBox\ VMs/shared/ocr1.vdi \
  --size 5120 --format VDI --variant Fixed
VBoxManage modifymedium ~/VirtualBox\ VMs/shared/ocr1.vdi --type Shareable

# Repetir para ocr2.vdi, ocr3.vdi (OCR/VOTING — 3x5GB)
# Repetir para data1.vdi, data2.vdi (DATA — 2x20GB)
# Repetir para fra1.vdi (FRA — 30GB)

# Anexar a rac1 E rac2 (com --storagectl SCSI e mtype SharedReadWrite)
VBoxManage storageattach rac1 --storagectl "SCSI" \
  --port 1 --device 0 --type hdd \
  --medium ~/VirtualBox\ VMs/shared/ocr1.vdi \
  --mtype SharedReadWrite
```

> **Dica**: Use um SCSI Controller separado para os discos compartilhados (não misture com o disco de OS).

#### Single Instance / Data Guard (2 VMs: oracle-primary e oracle-standby)

Cada VM:
- **CPU**: 2 vCPUs
- **RAM**: 4GB
- **Disco OS**: 50GB
- **Discos ASM** (por VM, não compartilhados): 2x10GB (DATA) + 1x15GB (FRA)
- **Rede**: Adapter 1: Host-Only (192.168.56.x)

---

### Binários Oracle

Coloque os ZIPs do Oracle neste diretório do **host** (ou em um mount NFS acessível das VMs):

```
/mnt/oracle_software/
├── LINUX.X64_193000_grid_home.zip    (~3.0 GB)
└── LINUX.X64_193000_db_home.zip      (~3.0 GB)
```

Download: [Oracle Software Delivery Cloud](https://edelivery.oracle.com) ou [My Oracle Support](https://support.oracle.com)

---

## Configuração

Antes de executar, ajuste as variáveis conforme seu ambiente:

### 1. Editar `group_vars/all.yml`

```yaml
# Caminho onde estão os ZIPs do Oracle (no host ou mount na VM)
oracle_installer_source: /mnt/oracle_software

# Senhas (use ansible-vault em produção)
oracle_sys_password: "Sua_Senha_Aqui"
```

### 2. Editar `inventory/rac/hosts.yml`

```yaml
rac1:
  ansible_host: 192.168.56.101    # IP da sua VM rac1
rac2:
  ansible_host: 192.168.56.102    # IP da sua VM rac2
```

### 3. Editar `inventory/si/hosts.yml`

```yaml
oracle-primary:
  ansible_host: 192.168.56.201
oracle-standby:
  ansible_host: 192.168.56.202
```

---

## Execução

### Ambiente RAC

```bash
# Instalação completa
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml

# Por fase (tags disponíveis):
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags prereqs
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags grid
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags grid_root
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags asm
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags oracle_sw
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags dbca
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags validate
```

### Ambiente Data Guard (Single Instance)

```bash
# 1. Instalar GI + DB nos dois nós e criar banco PRIMARY
ansible-playbook -i inventory/si/hosts.yml playbooks/setup-si.yml

# 2. Configurar Data Guard (standby criado via RMAN DUPLICATE)
ansible-playbook -i inventory/si/hosts.yml playbooks/setup-dataguard.yml
```

---

## Ordem de execução correta para RAC

O Oracle exige uma sequência rígida durante a instalação:

```
1. Pré-requisitos (todos os nós em paralelo)
2. gridSetup.sh -silent       → apenas nó 1
3. orainstRoot.sh             → todos os nós (sequencial)
4. root.sh do GI              → todos os nós (sequencial, um por vez)
5. gridSetup.sh -executeConfigTools → apenas nó 1
6. Criar diskgroups DATA e FRA → apenas nó 1 (via SQL*Plus como grid)
7. runInstaller -silent       → todos os nós (pode ser paralelo)
8. root.sh do Oracle DB       → todos os nós (sequencial)
9. DBCA                       → apenas nó 1
```

> **Por que o root.sh deve ser sequencial?** O `root.sh` do GI registra o nó no cluster e inicia o `crsd`. Se dois nós executarem simultaneamente, pode ocorrer race condition no OCR. A Oracle documenta isso no [Grid Infrastructure Installation Guide, seção 11](https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/).

---

## Senhas seguras com Ansible Vault

Para não expor senhas em texto claro:

```bash
# Criar arquivo de vault
ansible-vault create group_vars/vault.yml

# Conteúdo do vault.yml:
oracle_sys_password: "Minha_Senha_Segura_123#"
oracle_asm_monitor_password: "Minha_Senha_Segura_123#"

# Referenciar no all.yml:
oracle_sys_password: "{{ vault_oracle_sys_password }}"

# Executar com vault:
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml \
  --ask-vault-pass
```

---

## Troubleshooting comum

| Problema | Causa | Solução |
|----------|-------|---------|
| `gridSetup.sh` retorna código 6 | Warning (não erro) — ocorre sem display | Normal — verifique o log em `/tmp/oracle_install_logs/grid_install.log` |
| Discos ASM não encontrados | udev rules não aplicadas | Execute `udevadm trigger` e verifique `ls -la /dev/sd*` |
| root.sh falha com "CRS-4406" | Nó anterior não finalizou | Aguarde clusterware estabilizar e re-execute |
| DBCA falha com "ORA-01034" | ASM não está ativo | `srvctl start asm` como grid |
| SSH user equivalency falha no GI | Chaves não distribuídas | Re-execute a tag `ssh` do prereqs |

---

## Referências

- [Oracle Grid Infrastructure 19c Installation Guide for Linux](https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/)
- [Oracle Database 19c Installation Guide for Linux](https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/)
- [Oracle Data Guard Concepts and Administration 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/sbydb/)
- [ASM Administrator's Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/ostmg/)
- MOS Note 1937060.1 — Using udev for ASM on Oracle Linux
- MOS Note 1962946.1 — Silent Installation of Grid Infrastructure 19c
