# Automação Ansible para Deploy na AWS


Este repositório contém playbooks e roles Ansible que automatizam o provisionamento de infraestrutura na AWS (VPC, sub-redes, security groups, EC2 e RDS) e a implantação de um servidor Nginx que expõe um site estático com integração a um banco MySQL. O fluxo também cobre atualização contínua e escalabilidade horizontal via Auto Scaling Group.

## Alunos:
- Luiz Fernando Menezes Paes
- Maria Eduarda Pessanha Gonçalves

## Pré-requisitos
- Ansible >= 2.15 com Python 3.9+
- Coleções `amazon.aws` e `community.aws`
- AWS CLI configurado **e arquivo `aws_credentials` obrigatório** (mantido na raiz do repositório)
- Chave SSH existente na AWS (`ec2_key_name`) ou caminho local para a chave pública (`ec2_public_key_path`)

Crie/atualize o arquivo `aws_credentials` antes de executar qualquer playbook, seguindo o formato padrão da AWS:

```ini
[default]
aws_access_key_id=SEU_ACCESS_KEY
aws_secret_access_key=SEU_SECRET_KEY
aws_session_token=SEU_SESSION_TOKEN  # remova esta linha se usar credenciais permanentes
```

> Os playbooks usam `AWS_SHARED_CREDENTIALS_FILE=./aws_credentials`; credenciais ausentes ou inválidas resultam em erros `UnauthorizedOperation` imediatamente no provisionamento.

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
docs/
```

- `ansible.cfg`: habilita o inventário dinâmico `aws_ec2` e garante paths consistentes.
- `group_vars/all.yml`: centraliza parâmetros (região, rede, tamanhos de instância, mensagens da aplicação, etc.). Ajuste antes da execução.
- `inventories/aws_ec2.yml`: usa o plugin `amazon.aws.aws_ec2` para descobrir hosts via tags (`Project=ansible-startup-lab`, `Role=<web|database>`).
- `artifacts/`: recebe saídas locais (ex.: JSON com detalhes do provisionamento).

## Fluxo recomendado
1. **Provisionamento** (`playbooks/provision_infra.yml`)
   - Cria (por padrão) uma VPC dedicada, sub-redes públicas/privadas, Internet Gateway, security groups, uma instância EC2 e um RDS MySQL.
   - Caso deseje reutilizar uma VPC existente, defina `create_vpc: false` e informe `vpc_id`, `public_subnet_ids` e `private_subnet_ids` em `group_vars/all.yml`.
   - Gera `artifacts/provisioning_output.json` com metadados (endpoint do RDS, dados da EC2).

2. **Configuração inicial** (`playbooks/configure_web.yml`)
   - Usa o inventário dinâmico para alcançar hosts com `Role=web`.
   - Role `web`: instala e configura Nginx, publica o virtual host na porta 80 e garante o serviço ativo.
   - Role `db_config`: obtém o endpoint do RDS via API e gera `/etc/app/db.env` para consumo futuro.

3. **Deploy da aplicação estática** (`playbooks/deploy_app.yml`)
   - Role `app`: publica `index.html`, `RELEASE` e a rota de health-check (`{{ app_healthcheck_path }}`) diretamente em `{{ app_document_root }}`.
   - Utilize `-e app_release_label=v1.0.0` para definir o identificador da release aplicada.

4. **Atualizações contínuas** (`playbooks/update_app.yml`)
   - Realiza rolling update (`serial: 1`) recarregando Nginx após publicar a nova release.
   - Informe `new_release=<identificador>` ao executar para propagar a alteração em sequência controlada.

5. **Escalonamento** (`playbooks/scale_app.yml`)
   - Cria/atualiza Launch Template e Auto Scaling Group com as tags e security groups do projeto.
   - Ajuste `web_min_size`, `web_max_size` e `web_desired_capacity` em `group_vars/all.yml` (ou via `-e`) antes de rodar.
   - Após migrar para ASG, considere encerrar manualmente a instância "pet" criada no provisionamento inicial.

## Execução sugerida
```bash
# Provisionar infraestrutura (cria VPC, sub-redes e instâncias)
ansible-playbook playbooks/provision_infra.yml -i inventories/aws_ec2.yml

# Configurar Nginx e gerar arquivo de conexão com o RDS
ansible-playbook playbooks/configure_web.yml

# Fazer o primeiro deploy da página estática (usa app_release_label atual)
ansible-playbook playbooks/deploy_app.yml -e app_release_label=v1.0.0

# Atualizar conteúdo exibido pela aplicação
ansible-playbook playbooks/update_app.yml -e new_release=v1.1.0

# Ajustar escala (ex.: aumentar para 2 instâncias)
ansible-playbook playbooks/scale_app.yml -e web_desired_capacity=2
```

> **Importante:** sempre execute os playbooks com `AWS_SHARED_CREDENTIALS_FILE=./aws_credentials` (já configurado nos playbooks) e mantenha o perfil `default` do arquivo sincronizado com as credenciais atuais.

## Próximos passos
- Conectar os playbooks a pipelines (GitHub Actions, GitLab CI, etc.) para provisionamento/deploy contínuos.
- Adicionar testes automáticos com `ansible-lint` e `molecule`.
- Integrar health-check com ALB/Target Group para validar o rolling update do Auto Scaling Group.
