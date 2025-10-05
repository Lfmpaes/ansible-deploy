# Automação Ansible para Deploy na AWS

Este repositório contém um conjunto de playbooks e roles Ansible que automatizam o provisionamento e a implantação de uma aplicação web estática com banco de dados MySQL (Amazon RDS) na AWS. O fluxo cobre ainda atualização contínua e escalabilidade horizontal através de Auto Scaling Group.

## Pré-requisitos
- Ansible >= 2.15 com Python 3.9+
- Coleções `amazon.aws` e `community.aws`
- AWS CLI configurado e arquivo `aws_credentials` válido (mantenha no diretório raiz)
- Chave SSH existente na AWS (`ec2_key_name`)

Instale as coleções necessárias:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Organização do projeto
```
ansible.cfg
collections/requirements.yml
inventories/aws_ec2.yml
playbooks/
  provision_infra.yml
  configure_web.yml
  deploy_app.yml
  update_app.yml
  scale_app.yml
group_vars/all.yml
roles/
  provision_infra/
  web/
  app/
  db_config/
  scale/
artifacts/
```

- `ansible.cfg`: habilita o inventário dinâmico `aws_ec2` e garante paths consistentes.
- `group_vars/all.yml`: centraliza parâmetros (região, tipos de instância, versões, etc.). Ajuste conforme seu ambiente.
- `inventories/aws_ec2.yml`: usa o plugin `amazon.aws.aws_ec2` para descobrir hosts via tags (`Project=ansible-startup-lab` e `Role=<web|database>`).
- `artifacts/`: recebe saídas locais (ex.: JSON com endpoints provisionados).

## Fluxo recomendado
1. **Provisionamento** (`playbooks/provision_infra.yml`)
   - Executa a partir do `localhost` usando módulos AWS para criar security groups, instância EC2, RDS e salvar artefatos com endpoints.
   - Tenta detectar a VPC/subnets padrão caso não estejam definidos em `group_vars/all.yml`.
   - Tags aplicadas: `Project=ansible-startup-lab`, `Environment=production`, `Role=web|database`.

2. **Configuração inicial** (`playbooks/configure_web.yml`)
   - Usa as tags do inventário dinâmico para atingir os web servers.
   - Role `web`: instala Apache, cria diretório do site e disponibiliza página padrão.
   - Role `db_config`: obtém o endpoint do RDS via API e gera `/etc/app/db.env` com credenciais.

3. **Deploy da aplicação estática** (`playbooks/deploy_app.yml`)
   - Clona a versão desejada (`app_version`/`app_repository_url`) em `/opt/app/releases/<versão>` e publica em `{{ app_document_root }}`.
   - Cria arquivo `healthz.html` para verificações simples.

4. **Atualizações contínuas** (`playbooks/update_app.yml`)
   - Realiza rolling update (`serial: 1`). Informe `new_release=<tag>` ao executar para trocar a versão.

5. **Escalonamento** (`playbooks/scale_app.yml`)
   - Cria/atualiza Launch Template e Auto Scaling Group utilizando as mesmas tags.
   - Ajuste `web_min_size`, `web_max_size` e `web_desired_capacity` em `group_vars/all.yml` antes de rodar.
   - Após migrar para ASG, considere encerrar manualmente a instância criada em `provision_infra.yml` para evitar duplicidade.

## Execução sugerida
```bash
# Provisionar infraestrutura
ansible-playbook playbooks/provision_infra.yml

# Configurar servidor web e conectar ao RDS
ansible-playbook playbooks/configure_web.yml

# Fazer o primeiro deploy
ansible-playbook playbooks/deploy_app.yml -e app_version=v1.0.0

# Atualizar para uma nova release
ansible-playbook playbooks/update_app.yml -e new_release=v1.1.0

# Ajustar escala (ex. aumentar para 2 instâncias)
ansible-playbook playbooks/scale_app.yml -e web_desired_capacity=2
```

> **Dica:** Exporte `AWS_SHARED_CREDENTIALS_FILE=./aws_credentials` ou ajuste `aws_profile` para o profile correto no arquivo de credenciais.

## Próximos passos
- Criar pipelines (GitHub Actions, GitLab CI, etc.) que invoquem esses playbooks automaticamente.
- Adicionar testes com `ansible-lint` e `molecule`.
- Integrar health-check ao ALB/Target Group para validar o rolling update.
