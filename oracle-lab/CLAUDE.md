# Contexto do Projeto — oracle-lab

> Este arquivo é lido automaticamente pelo Claude Code ao iniciar neste diretório.
> Última atualização: 2026-06-25

## O que é este projeto

Automação Ansible para montar laboratório Oracle 19c no VirtualBox, cobrindo dois ambientes:

1. **Oracle RAC 2 nós** — Grid Infrastructure + ASM (OCR/DATA/FRA) + Oracle DB 19c
2. **Single Instance + ASM + Data Guard** — Primary + Standby físico via RMAN DUPLICATE

## Estado atual

Projeto **criado e funcional**. Todos os arquivos estão em `D:\work\dba-playbooks\oracle-lab\`.
Documentação detalhada salva no Obsidian Vault em `D:\Users\OneDrive\Documentos\ObsidianVault\00-Inbox\` (8 notas com prefixo "Oracle Lab 19c").

## Estrutura do projeto

```
oracle-lab/
├── ansible.cfg
├── CLAUDE.md                          ← este arquivo
├── inventory/
│   ├── rac/hosts.yml                  → rac1 (192.168.56.101) + rac2 (.102) + SCAN (.120-122)
│   └── si/hosts.yml                   → oracle-primary (.201) + oracle-standby (.202)
├── group_vars/
│   └── all.yml                        → TODAS as variáveis — editar aqui antes de executar
├── roles/
│   ├── oracle-prereqs/                → pacotes OL8, kernel, limits, udev, SSH equivalency
│   ├── grid-install/                  → gridSetup.sh -silent (CRS_CONFIG para RAC, HA_CONFIG para SI)
│   ├── asm-diskgroups/                → CREATE DISKGROUP DATA + FRA via SQL*Plus as sysasm
│   ├── oracle-install/                → runInstaller -silent INSTALL_DB_SWONLY
│   └── oracle-dbca/                   → dbca -silent + archivelog + DG Broker + tnsnames/listener
└── playbooks/
    ├── setup-rac.yml                  → instala RAC completo (7 fases)
    ├── setup-si.yml                   → instala SI + ASM (cria banco só no primary)
    └── setup-dataguard.yml            → configura DG via RMAN DUPLICATE FROM ACTIVE DATABASE
```

## Decisões técnicas tomadas

- **udev rules** em vez de ASMLib — ASMLib descontinuado no OL8 kernel 5.x+ (MOS 2745017.1)
- **SELinux permissive** em vez de disabled — evita reboot obrigatório no lab
- **serial: 1** no root.sh do GI — obrigatório para RAC (race condition no OCR se paralelo)
- **INSTALL_DB_SWONLY** — software separado da criação do banco (permite múltiplos bancos)
- **RMAN DUPLICATE FROM ACTIVE DATABASE** — sem necessidade de backup prévio para criar standby
- **DG_BROKER_START=TRUE** — habilita DGMGRL para gerenciar o Data Guard

## GIDs — Alinhados com ambiente de produção INEP

```
oinstall:  54321    dba:       54322    oper:      54323
backupdba: 54324    dgdba:     54325    kmdba:     54326
asmdba:    54327    asmoper:   54328    asmadmin:  54329    racdba: 54330
```

UIDs: grid=54321, oracle=54322

> ⚠️ IMPORTANTE: asmdba=54327 (não 54327 para racdba como estava errado em drafts anteriores).
> Referência: serverConfig.yml em D:\Users\OneDrive - INEP\Documentos\git\ansible-wsl\ansible_x86\oracle-preinstall\

## Variáveis obrigatórias a ajustar antes de executar

Arquivo: `group_vars/all.yml`

```yaml
oracle_installer_source: /mnt/oracle_software   # Onde estão os ZIPs do Oracle
oracle_sys_password: "Oracle_1234#"             # Trocar ou usar Ansible Vault
```

Inventário `inventory/rac/hosts.yml`:
```yaml
rac1:
  ansible_host: 192.168.56.101   # IP real da VM rac1
rac2:
  ansible_host: 192.168.56.102   # IP real da VM rac2
```

## Comandos de execução

```bash
# RAC completo
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml

# RAC por fase
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags prereqs
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags grid,grid_root,grid_config
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags asm
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags oracle_sw,oracle_root
ansible-playbook -i inventory/rac/hosts.yml playbooks/setup-rac.yml --tags dbca

# SI + Data Guard
ansible-playbook -i inventory/si/hosts.yml playbooks/setup-si.yml
ansible-playbook -i inventory/si/hosts.yml playbooks/setup-dataguard.yml
```

## Playbooks existentes em outros diretórios (contexto)

- `D:\Users\OneDrive - INEP\Documentos\git\ansible-wsl\` — Playbooks do ambiente de trabalho (INEP)
  - `ansible_x86\oracle-preinstall\serverConfig.yml` — referência de GIDs de produção
  - `install-oracle\` — role de instalação Oracle (mais antiga, sem GI)
  - `power-aix-oracle-rac-asm\` — projeto RAC em AIX (IBM Power)
  
- `D:\work\dba-playbooks\` — Este repositório (GitHub pessoal)
  - `roles\install-packages-19c\` — instalação de pacotes OL7/OL8
  - `roles\install-server\` — configuração básica de servidor
  - `roles\satelliteRegister-oracle\` — registro no Red Hat Satellite/Katello
  - `backup\` — playbooks antigos (pré-roles), estilo mais simples

## O que falta / próximos passos possíveis

- [ ] Aplicar Release Updates (RUs) via OPatch após instalação base
- [ ] Adicionar role para configurar HugePages (recomendado para produção)
- [ ] Criar role de patching (OPatch + datapatch)
- [ ] Testar e validar o playbook completo numa VM real
- [ ] Adicionar suporte a Oracle Linux 9 (OL9)
- [ ] Considerar Vagrant para o ambiente SI/DG (funciona bem sem shared disks)
- [ ] Fast-Start Failover (FSFO) com observer para o Data Guard

## Referências

- [GI Installation Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/)
- [DB Installation Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/)
- [Data Guard 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/sbydb/)
- [ASM Admin Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/ostmg/)
- MOS 1937060.1 — udev para ASM no OL8
- MOS 1962946.1 — Instalação silenciosa do GI 19c
- MOS 2745017.1 — ASMLib deprecation no OL8
