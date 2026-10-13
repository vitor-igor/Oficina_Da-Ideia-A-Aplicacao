# Da Ideia à Aplicação Web: Desenvolvendo Soluções para Diferentes Contextos

## Guia geral de desenvolvimento

Este guia acompanha a construção de uma aplicação web pequena, desde a preparação do ambiente até a implementação e os testes de uma API. Cada participante trabalhará com um dos contextos descritos na ficha técnica: valorização e divulgação de mulheres na ciência, catálogo de projetos acadêmicos, sistema de eventos ou divulgação de laboratórios e grupos de pesquisa.

Embora os sistemas tratem de problemas diferentes, todos seguem um caminho de desenvolvimento semelhante. A proposta é perceber como decisões de Engenharia de Software, Banco de Dados e Desenvolvimento Web podem ser reaproveitadas entre aplicações.

O guia foi pensado para ser acompanhado durante a oficina. Leia cada etapa, realize as atividades na ordem indicada e utilize as verificações para identificar se já pode avançar. Ele orienta o processo, mas não entrega a implementação completa: você deverá escrever o código usando os requisitos, as regras de negócio e o diagrama da sua ficha técnica.

## Como usar este guia

Mantenha a **ficha técnica do seu contexto** por perto. Ela é a referência para o problema que será resolvido, os requisitos funcionais, as regras de negócio, as entidades, os atributos e os relacionamentos. Este guia explica como transformar essas informações em uma aplicação.

A oficina está dividida em três etapas:

| Etapa | O que será feito | Resultado esperado |
|---|---|---|
| **1. Configuração do ambiente e banco de dados** | Criar o projeto Flask, configurar o PostgreSQL, implementar os models e aplicar as migrações. | Aplicação preparada e tabelas persistidas no banco. |
| **2. API** | Implementar as operações HTTP, validar dados, aplicar regras de negócio e testar as rotas. | API funcional e documentada. |
| **3. Interface web** | Criar as telas e conectá-las à API. | Fluxo principal utilizável pelo navegador. |

Priorize os requisitos obrigatórios da ficha. Funcionalidades extras não devem impedir a conclusão do fluxo principal.

---

## Antes de começar: ferramentas necessárias

Tenha disponíveis:

- **Python 3.11 ou superior** e **uv**, para criar e gerenciar o projeto Python;
- **Docker com Docker Compose**, para executar o PostgreSQL;
- **Node.js 20.9 ou superior** e **npm**, para criar e executar o frontend Next.js;
- um editor de código e um navegador;
- **Git**, apenas se precisar recuperar um checkpoint.

Confira as ferramentas no terminal:

```bash
python --version
uv --version
docker compose version
node --version
npm --version
git --version
```

Se alguma ferramenta não estiver disponível, avise a equipe antes de avançar. Corrigir o ambiente no início evita perder tempo quando as etapas já estiverem em andamento.

## Sobre os checkpoints de recuperação

As branches de checkpoint são **cópias de referência de uma implementação funcional**, disponibilizadas para recuperação. Elas não são etapas que todo participante precisa seguir nem são necessárias se seu próprio código estiver funcionando.

Existem checkpoints somente para as duas primeiras etapas:

- **`<contexto>-bd`**: estado funcional ao final da configuração do ambiente e do banco de dados;
- **`<contexto>-api`**: estado funcional ao final da implementação da API.

O prefixo da branch depende do contexto:

| Prefixo | Contexto |
|---|---|
| `pesquisadoras` | Plataforma de valorização e divulgação de mulheres na ciência |
| `projetos` | Catálogo de projetos acadêmicos |
| `eventos` | Sistema de eventos |
| `laboratorios` | Plataforma de divulgação de laboratórios e grupos de pesquisa |

Por exemplo, no contexto de eventos, os checkpoints são `eventos-bd` e `eventos-api`; no contexto de mulheres na ciência, procure as branches com o prefixo `pesquisadoras`. Use sempre o nome exato da branch publicada no repositório.

**Quando usar um checkpoint?** Se, ao final de uma etapa, você não conseguiu terminar ou seu código não funciona e não consegue corrigir o problema a tempo, clone a branch de checkpoint do **seu contexto** e continue a oficina a partir dessa implementação. Assim, um problema em uma etapa não impede você de acompanhar as seguintes.

Você começa a oficina em uma pasta local, sem precisar clonar a `main` nem inicializar um repositório Git. Se precisar recuperar um checkpoint, abra um terminal **fora da pasta do projeto atual** e execute o comando correspondente. Exemplo para o checkpoint do banco de dados de eventos:

```bash
git clone --branch eventos-bd --single-branch https://github.com/vitor-igor/Oficina_Da-Ideia-A-Aplicacao.git oficina-eventos-bd
cd oficina-eventos-bd
```

Para recuperar a API, use a branch `eventos-api` e outro nome de pasta, por exemplo `oficina-eventos-api`. Nos outros contextos, substitua o nome da branch pelo checkpoint correspondente publicado no repositório. Não faça o clone dentro da pasta do projeto que você começou a desenvolver.

Depois de clonar um checkpoint, **não reinicialize o projeto nem repita etapas que já estão prontas**. O Git recupera os arquivos de código, mas não transfere o arquivo `.env`, o ambiente virtual nem os dados que estavam no banco de outra máquina. Prepare a cópia recuperada:

```bash
uv sync
cp .env.example .env
# Confira se DATABASE_URL corresponde ao compose.yaml

docker compose up -d db
uv run flask --app backend db upgrade
```

Se o arquivo `.env` já existir, confira sua configuração em vez de sobrescrevê-lo. Use `db upgrade` para aplicar as migrações já versionadas ao banco local. Não execute `db init` novamente e não gere uma nova migração apenas por ter clonado a branch. Depois, valide o estágio recuperado e continue a partir dele.

A branch `*-api` já contém uma implementação funcional da API; a branch `*-bd` contém a preparação do ambiente e o banco, mas não a API completa. **Não há checkpoint para a interface web**: a terceira etapa deve ser desenvolvida a partir do projeto local em que você estiver trabalhando.

---

# Etapa 1 — Configuração do ambiente e banco de dados

## 1.1. Criar o projeto local

Escolha uma pasta de trabalho vazia. Os comandos a seguir usam `oficina` como nome da pasta do projeto:

```bash
mkdir oficina
cd oficina
uv init backend
```

O comando `uv init` cria a configuração inicial do projeto Python, incluindo o `pyproject.toml`. A pasta em que você está será a raiz da aplicação: nela ficarão o backend, o Docker Compose e, mais tarde, a pasta `frontend/`.

No `pyproject.toml`, confira se a seção `[project]` possui `requires-python = ">=3.11"`. Como a aplicação usará o pacote `src/backend/`, configure também o empacotamento. Preserve as informações que o `uv` já colocou no arquivo e acrescente estas seções se ainda não existirem:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/backend"]
```

Crie pelo editor a estrutura inicial abaixo **antes de sincronizar o ambiente**. Isso garante que o pacote indicado na configuração de empacotamento já exista quando o `uv` preparar o projeto:

```text
oficina/
├── pyproject.toml
├── compose.yaml
├── .env.example
├── .env
├── .gitignore
└── src/
    └── backend/
        ├── __init__.py
        ├── config.py
        ├── extensions.py
        └── models.py
```

Os arquivos Python podem começar vazios e serão preenchidos nas próximas seções. As responsabilidades de cada arquivo são:

| Arquivo | Responsabilidade |
|---|---|
| `src/backend/__init__.py` | Criar a aplicação Flask por meio de `create_app()`. |
| `src/backend/config.py` | Carregar configurações, inclusive a URL de conexão com o banco. |
| `src/backend/extensions.py` | Declarar extensões compartilhadas, como SQLAlchemy e Flask-Migrate. |
| `src/backend/models.py` | Definir as entidades persistidas com base no diagrama. |
| `.env` | Guardar a configuração local de conexão. Não compartilhe credenciais pessoais. |
| `.env.example` | Mostrar quais variáveis de ambiente precisam ser configuradas. |

## 1.2. Instalar as dependências do backend

Adicione as bibliotecas necessárias para Flask, mapeamento objeto-relacional, migrações, conexão com PostgreSQL e carregamento das variáveis de ambiente:

```bash
uv add flask flask-sqlalchemy flask-migrate "psycopg[binary]" python-dotenv marshmallow flask-smorest
uv remove flasgger ## opcional
uv sync
```

O `uv add` registra as dependências no `pyproject.toml` e atualiza o `uv.lock`; `uv sync` sincroniza o ambiente local com o projeto. Como `uv add` já sincroniza as dependências, o comando `uv sync` funciona como uma conferência explícita de que o ambiente está alinhado ao projeto.

Crie um `.gitignore` com, pelo menos, estas entradas:

```gitignore
.env
.venv/
__pycache__/
*.pyc
```

Mesmo que você não esteja usando Git no início da oficina, o `.gitignore` ajuda a evitar que arquivos locais sejam incluídos por engano caso o projeto seja versionado posteriormente.

## 1.3. Configurar o PostgreSQL com Docker Compose

O Docker evita que cada participante precise instalar e configurar o PostgreSQL manualmente. Crie `compose.yaml` na raiz do projeto com uma configuração equivalente a esta:

```yaml
services:
  db:
    image: postgres:18-alpine
    container_name: backend-postgres

    environment:
      POSTGRES_DB: backend
      POSTGRES_USER: backend
      POSTGRES_PASSWORD: backend

    ports:
      - "5432:5432"

    volumes:
      - postgres_data:/var/lib/postgresql

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U backend -d backend"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

O banco, o usuário e a senha usados neste exemplo são credenciais locais para a oficina. Mantenha os valores consistentes com a URL de conexão configurada na próxima seção. Se a porta `5432` já estiver ocupada, escolha outra porta do host e ajuste a URL de acordo.

Suba o banco de dados e confira se o container está em execução:

```bash
docker compose up -d db
docker compose ps
```

O PostgreSQL deve indicar que está aceitando conexões. Se isso não acontecer, confira se o Docker está em execução, se o serviço `db` está ativo e se a porta escolhida está disponível.

## 1.4. Configurar a conexão e a aplicação Flask

Crie `.env.example` com a variável usada para conectar a aplicação ao banco:

```dotenv
DATABASE_URL=postgresql+psycopg://backend:backend@localhost:5432/backend
```

Copie o exemplo para o arquivo local `.env`:

```bash
cp .env.example .env
```

Em `src/backend/config.py`, carregue a variável de ambiente `.env` e disponibilize-a para o Flask-SQLAlchemy:

```python
import os

from dotenv import load_dotenv


load_dotenv()


class Config:
    SQLALCHEMY_DATABASE_URI = os.getenv("DATABASE_URL")
    SQLALCHEMY_TRACK_MODIFICATIONS = False

    API_TITLE = "Oficina BV App API"
    API_VERSION = "v1"
    OPENAPI_VERSION = "3.0.3"

    OPENAPI_URL_PREFIX = "/"
    OPENAPI_JSON_PATH = "openapi.json"

    OPENAPI_SWAGGER_UI_PATH = "/docs"
    OPENAPI_SWAGGER_UI_URL = (
        "https://cdn.jsdelivr.net/npm/swagger-ui-dist/"
    )
```

Com essa configuração, a interface Swagger ficará disponível em http://localhost:5000/docs e a especificação OpenAPI em http://localhost:5000/openapi.json.


Em `src/backend/extensions.py`, crie instâncias compartilhadas de SQLAlchemy, Flask-Migrate e API do Flask-Smorest:

```python
from flask_migrate import Migrate
from flask_smorest import Api
from flask_sqlalchemy import SQLAlchemy


db = SQLAlchemy()
migrate = Migrate()
api = Api()
```

O objeto `db` fornece a integração com o SQLAlchemy; `migrate` integra Flask-Migrate/Alembic para controlar as alterações estruturais do banco; e `api` é o componente responsável por gerenciar e estruturar a API REST da aplicação.

Em `src/backend/__init__.py`, implemente a fábrica inicial da aplicação:

```python
from flask import Flask

from backend.config import Config
from backend.extensions import api, db, migrate


def create_app():
    app = Flask(__name__)

    app.config.from_object(Config)

    db.init_app(app)
    migrate.init_app(app, db)
    api.init_app(app)

    from backend import models

    from backend.routes.usuario_routes import usuario_blp

    api.register_blueprint(
        usuario_blp
    )

    @app.get("/")
    def index():
        return {
            "mensagem": "API funcionando"
        }

    return app
```

Os trechos são a base de configuração da oficina, não a implementação das entidades e das rotas. Neste momento, o objetivo é garantir que o Flask carregue a aplicação e as extensões corretamente. Ao adicionar novas entidades, importe e registre seus respectivos Blueprints (Flask-Smorest) nesse ponto por meio de `api.register_blueprint(<entidade>_blp)`.

## 1.5. Traduzir o diagrama em models

Antes de escrever as classes das entidades, leia os requisitos, as regras de negócio e o diagrama ER da sua ficha técnica. **Não invente entidades nem relacionamentos:** o diagrama do seu contexto é a referência para decidir o que deve ser persistido.

A correspondência entre o diagrama e o código é:

| Elemento do diagrama | Como aparece na implementação |
|---|---|
| Entidade | Classe que herda de `db.Model` e representa uma tabela. |
| Atributo | Coluna declarada com `db.Column` e um tipo adequado. |
| Identificador | Chave primária, geralmente `db.Integer` com `primary_key=True`. |
| Relacionamento | Chave estrangeira (`db.ForeignKey`) e, quando útil, atributo de navegação ORM (`db.relationship`). |
| Cardinalidade | Define onde ficam as chaves estrangeiras, quantas referências são permitidas e se é necessária uma tabela associativa. |
| Obrigatoriedade | `nullable=False` para impedir `NULL`; outras restrições podem ser necessárias conforme a regra. |


### O que cada elemento faz?

- **`db.Model`** é a classe-base dos models do Flask-SQLAlchemy. Cada classe representa uma entidade persistida.
- **`__tablename__`** define o nome da tabela no banco. Use um nome consistente e referencie esse nome nas chaves estrangeiras.
- **`db.Column`** declara uma coluna. O tipo (`db.String`, `db.Text`, `db.Integer`, `db.Date` etc.) deve combinar com a informação armazenada.
- **`primary_key=True`** define a chave primária que identifica cada registro.
- **`nullable=False`** impede que a coluna receba `NULL`. Isso representa a obrigatoriedade no banco, mas não substitui a validação da API.
- **`unique=True`** impede valores duplicados naquela coluna. Só use quando houver uma regra de unicidade; não é uma configuração automática para todo atributo.
- **`db.ForeignKey`** cria a referência entre tabelas no banco.
- **`db.relationship`** permite navegar entre objetos relacionados pelo Python. Ele não cria, sozinho, uma coluna ou chave estrangeira: a estrutura física do vínculo é definida pela chave estrangeira ou por uma tabela associativa.

### Exemplo ilustrativo: uma relação um-para-muitos

O exemplo abaixo usa nomes genéricos para mostrar como um registro principal pode estar relacionado a várias pessoas. **Ele demonstra apenas uma cardinalidade; não copie essa estrutura automaticamente.** Confira o diagrama do seu contexto, pois a posição da chave estrangeira e o tipo de relacionamento dependem dele.

```python
from backend.extensions import db


class EntidadePrincipal(db.Model):
    __tablename__ = "entidade_principal"

    id = db.Column(db.Integer, primary_key=True)
    nome = db.Column(db.String(100), nullable=False)

    pessoas = db.relationship(
        "PessoaRelacionada",
        back_populates="principal",
    )


class PessoaRelacionada(db.Model):
    __tablename__ = "pessoa_relacionada"

    id = db.Column(db.Integer, primary_key=True)
    nome = db.Column(db.String(100), nullable=False)
    principal_id = db.Column(
        db.Integer,
        db.ForeignKey("entidade_principal.id"),
        nullable=False,
    )

    principal = db.relationship(
        "EntidadePrincipal",
        back_populates="pessoas",
    )
```

Neste exemplo, a chave estrangeira `principal_id` fica na tabela `pessoa_relacionada`, pois cada pessoa aponta para uma entidade principal. A mesma entidade principal pode ser referenciada por várias pessoas, formando uma relação um-para-muitos.

Os dois `db.relationship` permitem navegar nos dois sentidos: `principal.pessoas` fornece a coleção de pessoas relacionadas; `pessoa.principal` fornece a entidade principal daquele registro. Os valores de `back_populates` devem corresponder aos nomes dos atributos definidos na outra classe.

### Como representar outras cardinalidades?

- **Um-para-muitos (1:N):** a chave estrangeira normalmente fica na tabela do lado “muitos”.
- **Um-para-um (1:1):** a chave estrangeira precisa também impedir múltiplas referências, normalmente por uma restrição de unicidade; a configuração ORM deve representar uma relação singular.
- **Muitos-para-muitos (N:N):** normalmente requer uma tabela associativa que armazene as chaves estrangeiras das duas entidades. Se o vínculo tiver atributos próprios, pode fazer sentido representá-lo como um model.

Para cada relacionamento do seu diagrama, responda antes de programar:

1. Um registro pode se relacionar com nenhum, um ou vários registros?
2. A relação é obrigatória ou opcional?
3. Qual entidade guarda a chave estrangeira? É necessária uma tabela associativa?
4. A chave estrangeira pode aceitar `NULL`? Deve ser única?
5. O que deve acontecer com os registros relacionados quando um registro é removido?

Não presuma que toda relação exige uma tabela associativa nem que uma exclusão deva apagar automaticamente todos os registros relacionados. As regras de exclusão precisam ser escolhidas com base nas regras de negócio e implementadas de forma coerente entre o banco e o ORM.

Implemente os models em `src/backend/models.py`. Ao terminar, confira se todas as entidades da ficha foram representadas, se tipos e nulabilidade fazem sentido, se as chaves estrangeiras apontam para as tabelas corretas e se os relacionamentos ORM refletem as cardinalidades definidas.

## 1.5. Criar e aplicar migrações

Definir models no Python não cria automaticamente as tabelas no PostgreSQL. O Flask-Migrate integra o Alembic ao projeto para registrar mudanças de estrutura em arquivos de migração e aplicá-las ao banco.

Se a pasta `migrations/` ainda não existir, inicialize o histórico uma única vez:

```bash
uv run flask --app backend db init
```

Depois de implementar os models, gere uma migração, examine o arquivo criado e aplique-a:

```bash
uv run flask --app backend db migrate -m "cria as tabelas iniciais"
uv run flask --app backend db upgrade
```

Caso realize alguma alteração aos models posteriormente, gere uma nova migração e aplique-a novamente. Não apague o histórico nem substitua migrações por `db.create_all()`. Para visualizar as migrações realizadas, examine `migrations/versions/`.

Para listar as tabelas no banco:

```bash
docker compose exec db psql -U backend -d backend -c "\\dt"
```

Se precisar examinar uma tabela específica, abra o `psql` e consulte sua definição com `\d nome_da_tabela`.

## 1.7. Conferência da etapa 1

Antes de começar a API, verifique se:

- [ ] A aplicação Flask é importada sem erros.
- [ ] O PostgreSQL está ativo e aceita conexões.
- [ ] `DATABASE_URL` corresponde ao banco definido no Compose.
- [ ] Os models representam as entidades e relações da ficha técnica.
- [ ] As migrações foram geradas, revisadas e aplicadas.
- [ ] As tabelas, chaves primárias e estrangeiras, nulabilidade e restrições importantes existem no banco.
- [ ] Não há divergências conhecidas entre o diagrama e a estrutura implementada.

Esta lista é uma verificação de conclusão. Se não conseguiu terminar ou corrigir um problema a tempo, utilize a branch `<contexto>-bd`, conforme explicado no início do guia.

---

# Etapa 2 — API

A API disponibiliza operações HTTP para consultar e alterar os dados persistidos. Nesta etapa, você definirá como as funcionalidades descritas nos requisitos serão acessadas por requisições e respostas. A ficha técnica continua sendo a referência para decidir quais operações são obrigatórias e quais regras precisam ser respeitadas.

## 2.1. Planejar as rotas a partir dos requisitos

Consulte os requisitos funcionais e liste quais operações devem existir. Nem toda entidade precisa de CRUD completo: as operações obrigatórias vêm dos requisitos e das regras de negócio.

| Método HTTP | Uso comum |
|---|---|
| `GET` | Listar recursos ou consultar um recurso sem alterá-lo. |
| `POST` | Criar um recurso ou executar uma operação prevista. |
| `PUT` | Atualizar um recurso identificado. |
| `DELETE` | Remover um recurso, quando permitido pelas regras. |

Como padrão ilustrativo, uma coleção pode ser representada por `/entidade-principal` e um registro individual por `/entidade-principal/{id}`. Esses caminhos são marcadores, não rotas literais da oficina. Defina os nomes e operações com base nas entidades da ficha e mantenha um padrão consistente.

Para cada operação, pergunte:

1. Quais dados a requisição deve receber?
2. Quais dados precisam aparecer na resposta?
3. Que validações são necessárias?
4. Quais registros relacionados precisam existir?
5. Que regra de negócio pode impedir a operação?
6. Qual código HTTP representa o resultado?

Antes de programar, compare a lista de operações com os requisitos. Isso reduz o risco de implementar rotas desnecessárias ou deixar de atender uma funcionalidade obrigatória.

## 2.2. Schemas: validar entradas e serializar respostas

Os schemas ficam em `src/backend/schemas/` e utilizam Marshmallow. Um schema de entrada define quais campos o cliente pode enviar e valida tipos, formatos, obrigatoriedade e limites. Um schema de saída define quais campos serão devolvidos pela API.

Os schemas de entrada e saída podem ser diferentes: um identificador gerado pelo banco, por exemplo, normalmente aparece na resposta, mas não é enviado pelo cliente na criação. Compare cada campo com a ficha técnica e com os models, incluindo datas, nulabilidade, limites de texto e identificadores.

A validação de formato não substitui a validação de negócio. Um `id` pode ser um inteiro válido, mas apontar para um registro inexistente, por isso essa situação precisa ser tratada pela lógica da operação.

#### Exemplo:
```python
from marshmallow import Schema, fields, validate


class UsuarioCreateSchema(Schema):
    nome = fields.String(
        allow_none=True,
        validate=validate.Length(max=150)
    )

    email = fields.Email(
        required=True
    )

    senha = fields.String(
        required=True,
        load_only=True,
        validate=validate.Length(min=6, max=256)
    )
```

**Explicação**: `required=True` exige que o campo seja enviado. `allow_none=True` permite que seu valor seja `null`; isso não é o mesmo que permitir que o campo seja omitido. Os limites do schema devem ser coerentes com os limites e restrições definidos nos models.

## 2.3. Services: implementar operações e regras de negócio

Os services ficam em `src/backend/services/`. Eles concentram consultas e alterações persistentes, verificam a existência dos registros relacionados e aplicam as regras que dependem do estado do sistema.

Ao implementar uma operação, siga esta linha de raciocínio:

1. Os dados recebidos passaram pela validação do schema?
2. Os registros referenciados existem?
3. A regra de negócio permite a operação?
4. Como os dados devem ser consultados ou alterados?
5. A transação só será confirmada quando a operação for válida?
6. Se ocorrer um erro, como impedir que uma alteração parcial permaneça gravada?

Uma restrição de tipo ou formato costuma pertencer ao schema; uma regra que depende da relação entre registros costuma exigir uma consulta no service; e as restrições de integridade devem ser protegidas também pelo banco quando apropriado. Implemente as condições da sua ficha, não regras de outros contextos.

Um service pode, por exemplo, receber os dados já validados pelo schema, construir o model correspondente, adicioná-lo à sessão do SQLAlchemy, confirmar a transação e devolver o objeto criado. Os nomes e os campos utilizados dependem do model real; não copie uma função genérica sem adaptá-la.

#### Exemplo:
```python
from werkzeug.security import generate_password_hash

from backend.extensions import db
from backend.models import Usuario

def criar_usuario(dados):
    usuario = Usuario(
        nome=dados.get("nome"),
        email=dados["email"],
        senha=generate_password_hash(
            dados["senha"]
        )
    )

    db.session.add(usuario)
    db.session.commit()

    return usuario
```

## 2.4. Routes: conectar a API às operações

As rotas ficam em `src/backend/routes/` e são registradas pela fábrica da aplicação, em `src/backend/__init__.py`. Uma rota recebe a requisição, valida os dados com o schema apropriado, chama o service responsável e devolve uma resposta HTTP.

Flask-Smorest utiliza **Blueprints** para agrupar rotas e gerar a documentação OpenAPI/Swagger a partir dos decorators, schemas e respostas declarados na implementação. Portanto, a documentação é construída junto com as rotas; não é necessário manter uma tabela manual de cada parâmetro dentro deste guia.

Um Blueprint ilustrativo tem esta forma:

```python
from flask.views import MethodView
from flask_smorest import Blueprint, abort


usuario_blp = Blueprint(
    "usuarios",
    "usuarios",
    url_prefix="/api/usuarios",
    description="Operações relacionadas aos usuários"
)
```

Os decorators mais comuns são:

- `@<entidade>_blp.route(...)`: associa uma URL a uma classe ou função de recurso;
- `@<entidade>_blp.arguments(...)`: valida os dados recebidos usando um schema e disponibiliza os valores processados à função;
- `@<entidade>_blp.response(...)`: define o schema e o código HTTP da resposta, serializa o resultado e fornece essas informações à documentação automática;
- `@<entidade>_blp.alt_response(...)`: documenta respostas alternativas, como um recurso inexistente ou uma regra não atendida.

Uma rota de criação pode seguir este esqueleto conceitual — adapte nomes, schemas e service às operações definidas na sua ficha:

```python
@usuario_blp.route("")
class UsuarioListResource(MethodView):

    @usuario_blp.response(200, UsuarioResponseSchema(many=True))
    def get(self):
        return usuario_service.listar_usuarios()

    @usuario_blp.arguments(UsuarioCreateSchema)
    @usuario_blp.response(201, UsuarioResponseSchema)
    def post(self, dados):
        usuario_existente = (
            usuario_service.buscar_usuario_por_email(dados["email"])
        )

        if usuario_existente:
            abort(
                409,
                message="E-mail já cadastrado."
            )

        return usuario_service.criar_usuario(dados)
```

Depois de criar os Blueprints, atualize `create_app()` em `src/backend/__init__.py` para inicializar o objeto `api` e registrar cada Blueprint. Faça isso depois de criar os arquivos das rotas; não deixe a fábrica importar módulos que ainda não existem.

Os códigos HTTP mais relevantes para a oficina são:

| Código | Uso comum |
|---|---|
| `200 OK` | Consulta ou atualização concluída. |
| `201 Created` | Recurso criado. |
| `204 No Content` | Exclusão concluída sem corpo de resposta. |
| `404 Not Found` | Recurso ou referência necessária não existe. |
| `409 Conflict` | Conflito com uma restrição de integridade, quando aplicável. |
| `422 Unprocessable Entity` | Entrada inválida ou regra de negócio não atendida. |

O código escolhido deve descrever o resultado da operação. Não é necessário implementar um sistema complexo de erros: trate com clareza as situações relevantes para os requisitos e para as regras de negócio.

## 2.5. Executar e testar a API

Quando todos os Blueprints tiverem sido registrados, confira se o Flask reconhece as rotas:

```bash
uv run flask --app backend routes
```

Inicie o servidor de desenvolvimento:

```bash
uv run flask --app backend run --debug --host 127.0.0.1 --port 5000
```

Mantenha o terminal aberto. Com a configuração apresentada neste guia, a documentação Swagger estará em `http://127.0.0.1:5000/docs` e a especificação OpenAPI em `http://127.0.0.1:5000/openapi.json`. Se você configurou outro endereço ou porta, utilize os valores reais do projeto. 

No Swagger, confirme se as operações, os métodos, os parâmetros, os schemas e as respostas aparecem corretamente. Use a interface para executar os testes seguintes com dados fictícios do seu contexto. Anote os IDs retornados e reutilize-os nas operações seguintes; não presuma que serão sempre `1`.

1. **Criar:** cadastre os registros necessários, respeitando a ordem das dependências entre entidades.
2. **Listar e consultar:** consulte a coleção e depois um registro específico. Confira se os campos esperados aparecem.
3. **Relacionar:** crie ou consulte os vínculos definidos no modelo. Verifique se a API apresenta as entidades relacionadas da maneira prevista.
4. **Atualizar:** quando a operação existir, altere um campo e consulte o registro novamente para confirmar a persistência.
5. **Validar entradas:** omita um campo obrigatório ou envie um valor incompatível com o tipo/limite esperado. Confira a resposta de erro.
6. **Testar referências:** envie um identificador inexistente quando a operação depender de outro registro. A API deve tratar o caso sem gravar dados inválidos.
7. **Excluir:** confirme o resultado de uma exclusão e verifique o que aconteceu com os registros relacionados, de acordo com as regras de negócio.

Depois de cada alteração, consulte novamente o recurso: a resposta imediata, sozinha, não comprova que o dado foi persistido. Após um teste inválido, faça uma operação válida para confirmar que a aplicação continua funcionando.

Se algo falhar, investigue em ordem: a URL e o método chamados, o JSON enviado, a resposta e os logs do Flask; compare o resultado com o schema, o service, a rota e as regras da ficha técnica. Isso ajuda a localizar o problema sem reescrever partes que já estão funcionando.

## 2.6. Conferência da etapa 2

Antes de iniciar a interface, verifique se:

- [ ] Todas as operações obrigatórias da ficha foram implementadas e registradas.
- [ ] Os schemas correspondem aos atributos e tipos dos models.
- [ ] As relações e regras de negócio foram testadas.
- [ ] Criações e alterações persistem no banco.
- [ ] Consultas apresentam os dados relacionados esperados.
- [ ] Exclusões respeitam a integridade definida.
- [ ] As respostas HTTP são coerentes com sucessos e erros relevantes.
- [ ] A documentação automática está acessível e descreve as operações registradas.
- [ ] Os requisitos obrigatórios foram verificados com testes que podem ser repetidos.

Esta lista é uma verificação de conclusão. Se não conseguiu concluir ou corrigir a API a tempo, utilize a branch `<contexto>-api`. Ela contém o estado do banco e uma implementação funcional da API para que você possa seguir para a etapa 3.

---


