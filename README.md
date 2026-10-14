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


### O que cada elemento do SQL-Alchemy faz?

- **`db.Model`** é a classe-base dos models. Cada classe representa uma entidade persistida.
- **`__tablename__`** define o nome da tabela no banco. Use um nome consistente e referencie esse nome nas chaves estrangeiras.
- **`db.Column`** declara uma coluna. O tipo (`db.String`, `db.Text`, `db.Integer`, `db.Date` etc.) deve combinar com a informação armazenada.
- **`primary_key=True`** define a chave primária que identifica cada registro.
- **`nullable=False`** impede que a coluna receba `NULL`. Isso representa a obrigatoriedade no banco, mas não substitui a validação da API.
- **`unique=True`** impede valores duplicados naquela coluna. Só use quando houver uma regra de unicidade; não é uma configuração automática para todo atributo.
- **`db.ForeignKey`** cria a referência entre tabelas no banco.
- **`db.relationship`** permite navegar entre objetos relacionados pelo Python. Ele não cria, sozinho, uma coluna ou chave estrangeira: a estrutura física do vínculo é definida pela chave estrangeira ou por uma tabela associativa.

### Exemplo ilustrativo

O exemplo abaixo demonstra um relacionamento um-para-um opcional entre `Cooperado` e `Usuario`. Cada usuário pode estar associado a, no máximo, um cooperado, e cada cooperado pode estar associado a, no máximo, um usuário. Essa estrutura é apenas um exemplo: consulte o diagrama do seu contexto para identificar a cardinalidade correta.

```python
from backend.extensions import db


class Cooperado(db.Model):
    __tablename__ = "cooperado"

    id = db.Column(
        db.String(36),
        primary_key=True
    )

    nome = db.Column(
        db.String(150),
        nullable=False
    )

    usuario = db.relationship(
        "Usuario",
        back_populates="cooperado",
        uselist=False
    )


class Usuario(db.Model):
    __tablename__ = "usuario"

    id = db.Column(db.Integer, primary_key=True)
    nome = db.Column(db.String(150), nullable=True)
    email = db.Column(
        db.String(150),
        nullable=False,
        unique=True
    )
    senha = db.Column(db.String(256), nullable=False)
    status = db.Column(
        db.Boolean,
        default=True,
        nullable=False
    )

    cooperado_id = db.Column(
        db.String(36),
        db.ForeignKey("cooperado.id"),
        nullable=True,
        unique=True
    )

    cooperado = db.relationship(
        "Cooperado",
        back_populates="usuario",
        uselist=False
    )
```

A coluna `cooperado_id` é a chave estrangeira armazenada na tabela `usuario`, estabelecendo o vínculo com a tabela `cooperado`. Como possui `unique=True`, um mesmo cooperado não pode ser associado a vários usuários. Já `nullable=True` permite que um usuário exista sem estar associado a um cooperado.

Os dois atributos `db.relationship()` permitem navegar entre os objetos relacionados no Python. Assim, `usuario.cooperado` permite acessar o cooperado associado a um usuário, enquanto `cooperado.usuario` permite acessar o usuário associado a um cooperado. O parâmetro `back_populates` conecta esses atributos, e `uselist=False` indica que o relacionamento retorna um único objeto, em vez de uma coleção.

Observe a diferença entre o relacionamento e a chave estrangeira: `cooperado_id` é uma coluna real no banco de dados; `cooperado` e `usuario` são atributos do ORM que facilitam o acesso aos objetos relacionados. A cardinalidade, a obrigatoriedade e a localização da chave estrangeira devem seguir o diagrama de dados do contexto escolhido.

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

# Etapa 3 — Interface web e integração com a API

Nesta etapa, você transformará os requisitos em telas e conectará a interface aos dados persistidos no backend. O frontend **não acessa o PostgreSQL diretamente**: ele envia requisições HTTP à API Flask, recebe respostas e apresenta os dados à pessoa usuária.

O projeto utilizará Next.js, React, TypeScript e Tailwind CSS. Para acelerar a construção da interface, a oficina recomenda componentes do shadcn/ui, que são adicionados ao próprio projeto e podem ser adaptados conforme necessário.

## 3.1. Confirmar os pré-requisitos

Antes de criar a interface, confirme que a API está funcionando e que você consegue consultar seus endpoints pelo Swagger. Se você recuperou o checkpoint `<contexto>-api`, prepare e valide a cópia recuperada conforme a seção de checkpoints no início deste guia.

Confira Node.js e npm:

```bash
node --version
npm --version
```

O Next.js usado nesta oficina exige Node.js 20.9 ou superior. Se a API estiver configurada em outra porta, anote o endereço correto; ele será utilizado na configuração de integração.

## 3.2. Criar o frontend no mesmo projeto local

Volte à raiz do projeto, isto é, à pasta que contém `pyproject.toml`, `compose.yaml` e `src/backend/`. Crie nela a pasta `frontend/` usando o gerador do Next.js:

```bash
npx create-next-app@latest frontend --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm --disable-git
```

A opção `--disable-git` evita que o gerador inicialize um repositório Git dentro da pasta `frontend/`. O projeto inteiro continua sendo uma pasta local criada do zero; Git só é usado se for necessário recuperar um checkpoint.

Entre no frontend e inicie o servidor de desenvolvimento:

```bash
cd frontend
npm run dev
```

Abra o endereço mostrado pelo terminal, normalmente `http://localhost:3000`. 

A estrutura inicial mais relevante será semelhante a esta:

```text
frontend/
├── package.json
├── next.config.ts
├── .env.local
└── src/
    ├── app/ 
    ├── components/ 
    └── lib/ 
```

Os arquivos podem variar conforme a versão do gerador. Não substitua configurações que já existam sem entender para que servem.

## 3.3. Planejar telas a partir dos requisitos

Antes de programar, transforme os requisitos da ficha técnica em ações que alguém poderá realizar na interface. Em geral, pense em uma listagem da entidade principal, uma visualização de detalhe com suas relações e formulários para criação ou edição quando a API oferecer essas operações.

| Necessidade | Possível elemento da interface | O que verificar na API |
|---|---|---|
| Consultar uma coleção | Lista ou tabela | Endpoint `GET` de listagem. |
| Consultar um registro e suas relações | Página de detalhe ou modal | Endpoint `GET` individual e formato das relações. |
| Cadastrar um registro | Formulário | Endpoint `POST` e schema de entrada. |
| Atualizar um registro | Formulário de edição | Endpoint `PUT`, se disponível. |
| Remover um registro | Ação com confirmação | Endpoint `DELETE`, se disponível. |
| Associar registros | Select ou formulário específico | Como a API recebe os identificadores. |

Esses são padrões, não uma exigência de reproduzir um único layout. Nem toda entidade oferece CRUD completo; consulte a ficha técnica e o Swagger antes de criar botões ou formulários.

Planeje também os estados da tela: carregamento, lista vazia, sucesso e erro. Uma lista vazia não é a mesma coisa que uma falha ao consultar a API.

**Como conferir:** cada tela ou ação planejada deve corresponder a um requisito e ter suporte no contrato da API.

## 3.4. Reaproveitar componentes prontos

Para reduzir o tempo gasto com estilização, utilize os componentes de [shadcn/ui](https://ui.shadcn.com/). Após criar o projeto Next.js, inicialize a biblioteca seguindo a [documentação oficial de instalação para Next.js](https://ui.shadcn.com/docs/installation/next):

```bash
npx shadcn@latest init
```

Depois, adicione somente os componentes necessários. Para começar, uma seleção pequena costuma ser suficiente:

```bash
npx shadcn@latest add button input table dialog
```

Se o seu contexto precisar de campos longos, seletores, abas ou mensagens, adicione esses componentes quando chegar a essa necessidade, por exemplo:

```bash
npx shadcn@latest add textarea select alert-dialog tabs sonner skeleton
```

Não é obrigatório instalar todos os componentes: escolha os que realmente serão usados. O comando adiciona arquivos ao projeto e pode solicitar dependências e configurações; leia as mensagens e preserve os auxiliares necessários.

| Componente ou padrão | Para que serve | Documentação |
|---|---|---|
| Button | Ações e envio de formulários | [Button](https://ui.shadcn.com/docs/components/button) |
| Input | Campo de uma linha | [Input](https://ui.shadcn.com/docs/components/input) |
| Textarea | Biografia, descrição ou resumo | [Textarea](https://ui.shadcn.com/docs/components/textarea) |
| Select | Escolher uma opção ou registro relacionado | [Select](https://ui.shadcn.com/docs/components/select) |
| Table | Listar registros em linhas e colunas | [Table](https://ui.shadcn.com/docs/components/table) |
| Dialog | Formulário ou detalhe em modal | [Dialog](https://ui.shadcn.com/docs/components/dialog) |
| Alert Dialog | Confirmar exclusão ou outra ação destrutiva | [Alert Dialog](https://ui.shadcn.com/docs/components/alert-dialog) |
| Tabs | Separar seções da aplicação | [Tabs](https://ui.shadcn.com/docs/components/tabs) |
| Toast | Mensagens breves de sucesso ou falha | [Sonner](https://ui.shadcn.com/docs/components/sonner) |
| Skeleton | Indicar carregamento | [Skeleton](https://ui.shadcn.com/docs/components/skeleton) |

Para uma oficina curta, priorize controles de formulário, listagem, detalhes e confirmação. Estados simples podem ser comunicados com texto, sem exigir componentes extras.

Outras bibliotecas e galerias, como [Untitled UI](https://www.untitledui.com/react/components) e [21st.dev](https://21st.dev/), podem servir de inspiração. Seus exemplos não fazem automaticamente parte do projeto: confira código, dependências e licença antes de utilizá-los. Para manter uma base comum entre os quatro contextos, prefira a biblioteca indicada neste guia.

**Como conferir:** antes de importar um componente, confirme que ele existe em `src/components/ui/` ou adicione-o intencionalmente. A aplicação deve continuar compilando.

## 3.5. Organizar a interface

Mantenha cada parte responsável por uma tarefa clara:

| Local sugerido | Responsabilidade |
|---|---|
| `src/app/` | Páginas, rotas e layout. |
| `src/components/` | Componentes específicos da aplicação que podem ser reutilizados. |
| `src/components/ui/` | Componentes visuais adicionados pelo shadcn/ui. |
| `src/lib/` | Funções auxiliares, tipos e chamadas HTTP. |

Não crie muitas pastas antes de precisar delas. Comece pelo fluxo principal e extraia componentes quando isso deixar o código mais claro.

No App Router do Next.js, componentes são renderizados no servidor por padrão. Partes que usam estado React, efeitos, eventos de clique ou APIs do navegador precisam ser componentes de cliente, normalmente indicados por `"use client"` no início do arquivo.

O fluxo de uma ação será:

```text
ação da pessoa
    ↓
componente da interface
    ↓
requisição HTTP
    ↓
API Flask
    ↓
resposta HTTP/JSON
    ↓
atualização da interface
```

## 3.6. Conectar o frontend à API

Esta é a parte que faz as telas deixarem de ser apenas visuais e passarem a trabalhar com os dados da aplicação. A oficina utiliza um **proxy do Next.js**: o navegador chama o próprio frontend, e o Next.js encaminha a requisição ao Flask. Assim, as chamadas da interface não precisam usar diretamente a origem da API no navegador.

### Passo 1 — Iniciar a API

Abra um terminal na raiz do projeto, onde ficam `pyproject.toml` e `src/backend/`. Com o PostgreSQL ativo e as migrações aplicadas, inicie o Flask:

```bash
uv run flask --app backend run --debug --host 127.0.0.1 --port 5000
```

Mantenha o terminal aberto. A documentação deve estar em `http://127.0.0.1:5000/docs`. Abra-a, encontre uma rota `GET` de listagem do seu contexto e anote o caminho exato, incluindo eventuais prefixos.

### Passo 2 — Configurar o proxy do Next.js

Em outro terminal, abra `frontend/next.config.ts`. Preserve as configurações existentes e acrescente uma função `rewrites()` equivalente a esta:

```typescript
import type { NextConfig } from "next";

const backendUrl = process.env.BACKEND_URL ?? "http://127.0.0.1:5000";

const nextConfig: NextConfig = {
  async rewrites() {
    return [
      {
        source: "/backend/:path*",
        destination: `${backendUrl}/:path*`,
      },
    ];
  },
};

export default nextConfig;
```

Se `next.config.ts` já possuir `nextConfig`, não crie uma segunda variável nem apague configurações necessárias: incorpore somente a função `rewrites()` ao objeto existente. Essa regra encaminha `/backend/ROTA` para `ROTA` no Flask; o prefixo `/backend` é usado pelo proxy e não faz parte da rota original da API.

### Passo 3 — Informar o endereço do backend

Na raiz de `frontend/`, ao lado de `package.json`, crie `.env.local`:

```dotenv
BACKEND_URL=http://127.0.0.1:5000
```

Ajuste a URL caso o Flask esteja em outro endereço ou porta. Essa variável é lida pelo Next.js no servidor; não use o prefixo `NEXT_PUBLIC_`.

Se alterar `next.config.ts` ou `.env.local`, pare o servidor Next.js com `Ctrl+C` e inicie-o novamente com `npm run dev` para carregar a configuração.

### Passo 4 — Testar o proxy antes de escrever a tela

Suponha apenas como exemplo que o Swagger mostre `GET /<entidade>`. Com o Next.js executando na porta 3000, abra no navegador:

`http://localhost:3000/backend/<entidade>`

Para o seu projeto, substitua `<entidade>` pela rota real de listagem. O caminho chamado pelo navegador começa com `/backend/`; o destino no Flask não tem esse prefixo. A resposta via proxy deve corresponder à resposta da rota acessada diretamente no Swagger.

Se a chamada falhar, confira nesta ordem:

1. O Flask está em execução no terminal do backend?
2. A rota funciona diretamente pelo Swagger?
3. `BACKEND_URL` aponta para o endereço e a porta corretos?
4. O arquivo `next.config.ts` contém a regra de proxy e o servidor Next foi reiniciado?
5. O caminho usado depois de `/backend/` é exatamente o caminho documentado na API?

### Passo 5 — Fazer uma chamada HTTP pelo frontend

Depois de testar o proxy, implemente no frontend uma chamada `fetch` para a rota real. O exemplo abaixo mostra o padrão de leitura; substitua `ROTA_REAL` e adapte a resposta ao JSON que sua API realmente devolve:

```typescript
const response = await fetch("/backend/ROTA_REAL", {
  cache: "no-store",
});

if (!response.ok) {
  throw new Error(`A API respondeu com HTTP ${response.status}`);
}

const dados = await response.json();
```

Receber `dados` não os coloca automaticamente na tela. Em React, normalmente você guarda o resultado em estado e o utiliza para renderizar a lista, tabela ou detalhe. Os nomes e tipos dos campos precisam seguir a resposta observada no Swagger, não este exemplo.

Para ver o fluxo completo de leitura em React, um componente de cliente pode seguir este padrão mínimo. `id` e `nome` são campos ilustrativos: substitua-os pelo formato real do JSON devolvido pela sua API.

```tsx
"use client";

import { useEffect, useState } from "react";

type Registro = { id: number; nome: string };

export default function ListaExemplo() {
  const [registros, setRegistros] = useState<Registro[]>([]);
  const [carregando, setCarregando] = useState(true);
  const [erro, setErro] = useState<string | null>(null);

  useEffect(() => {
    async function carregar() {
      try {
        const response = await fetch("/backend/ROTA_REAL", { cache: "no-store" });
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const dados = (await response.json()) as Registro[];
        setRegistros(dados);
      } catch {
        setErro("Não foi possível carregar os registros.");
      } finally {
        setCarregando(false);
      }
    }
    void carregar();
  }, []);

  if (carregando) return <p>Carregando...</p>;
  if (erro) return <p role="alert">{erro}</p>;
  if (registros.length === 0) return <p>Nenhum registro encontrado.</p>;

  return (
    <ul>
      {registros.map((registro) => (
        <li key={registro.id}>{registro.nome}</li>
      ))}
    </ul>
  );
}
```

Esse exemplo demonstra os pontos essenciais: fazer a chamada HTTP, verificar o status, converter a resposta JSON, guardar os dados em estado React e renderizar a lista. Se a API devolver um objeto que contém uma lista dentro de uma propriedade, ajuste a leitura; não presuma que toda resposta é um array.

Para enviar dados, o padrão é semelhante, mas o método e o corpo devem corresponder ao contrato da API:

```typescript
const response = await fetch("/backend/ROTA_REAL", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(dadosDoFormulario),
});

if (!response.ok) {
  throw new Error(`A API respondeu com HTTP ${response.status}`);
}
```

`dadosDoFormulario` precisa conter os campos aceitos pelo schema de entrada. Após a operação, apresente a confirmação e atualize ou recarregue os dados da interface. Em operações `DELETE` que retornem `204 No Content`, não tente ler a resposta com `response.json()`, pois ela não contém corpo.

Para chamadas repetidas, concentre o acesso HTTP em funções dentro de `src/lib/` em vez de espalhar a mesma lógica por vários componentes.

## 3.7. Tipos, formulários e relacionamentos

Consulte a ficha técnica e o Swagger/OpenAPI antes de criar tipos TypeScript ou formulários. Respeite os nomes dos campos, tipos, datas, campos obrigatórios/opcionais e valores nulos definidos pela API.

Os dados enviados e recebidos podem ter estruturas diferentes. Um identificador gerado pelo servidor, por exemplo, normalmente aparece na resposta, mas não é enviado na criação. Se a API devolver uma lista dentro de uma propriedade de um objeto, não a trate como se a resposta inteira fosse um array.

Em formulários relacionados, carregue as opções necessárias e envie os identificadores no formato esperado. Depois de uma criação ou edição bem-sucedida, atualize a interface para refletir o dado persistido. Em caso de falha, mostre uma mensagem compreensível e preserve os dados válidos que já estavam na tela.

Não presuma que todos os recursos aceitam edição ou exclusão, que todas as respostas possuem a mesma estrutura ou que os campos de um contexto servem para os outros. A ficha técnica explica a necessidade; o contrato da API mostra o que está disponível.

## 3.8. Testar o fluxo completo

Com backend e frontend executando em terminais separados, percorra a sequência principal do seu contexto:

1. Abra a interface e confirme que a página carrega.
2. Confira se a listagem obtém dados da API.
3. Crie um registro, se essa operação estiver prevista.
4. Abra o detalhe e confira os dados e relacionamentos exigidos.
5. Atualize ou remova um registro somente se a API oferecer essas operações; confirme exclusões antes de enviá-las.
6. Observe os estados de carregamento, lista vazia, sucesso e erro.
7. Se algo falhar, examine a requisição e a resposta no navegador e compare com o Swagger.
8. Recarregue a página e confirme que os dados persistidos continuam disponíveis.

Uma operação não está integrada só porque o botão responde a um clique. A ação deve produzir uma requisição válida, a API deve responder de acordo com o contrato e a interface deve apresentar o resultado coerentemente.

## 3.9. Conferência final da etapa web

Esta é uma lista de verificação da conclusão da etapa, **não um checkpoint Git**. Não existe branch de recuperação para o frontend.

- [ ] O frontend funciona dentro da pasta `frontend/` do projeto local.
- [ ] As telas e ações correspondem aos requisitos obrigatórios da ficha.
- [ ] Os tipos, formulários e chamadas seguem o contrato real da API.
- [ ] A interface consulta a API por HTTP e não acessa diretamente o banco de dados.
- [ ] O proxy do Next.js encaminha as requisições ao Flask.
- [ ] As operações disponíveis funcionam e os estados de carregamento, erro, sucesso e ausência de dados são compreensíveis.
- [ ] Os relacionamentos são exibidos de forma coerente com os requisitos.
- [ ] O fluxo principal foi percorrido no navegador e os dados continuam persistidos após recarregar a página.

---

## 3.10. Configuração completa do frontend

Este percurso detalha a construção do frontend com Untitled UI e Radix Dialog. As três partes abaixo — base do projeto, interface e integração — pertencem à etapa 3 da oficina. Ao adotá-lo, utilize de forma consistente seus componentes e a configuração de proxy `/api` com `API_URL`, como descrito nesta seção.

**Oficina BV App**

Criar uma base educacional de frontend utilizando Next.js, React, TypeScript, Tailwind CSS, Untitled UI e Radix Dialog. A interface usa componentes reutilizáveis, enquanto tipos, campos e comunicação com a API ficam separados da apresentação.

Este é o tutorial comum aos quatro contextos da oficina. `Registro` é um nome técnico para explicar o modelo de aplicação. Cada grupo usa sua ficha para definir entidades, campos, associações, operações e regras.

O guia acompanha a construção do frontend nas três etapas. Crie os arquivos na ordem apresentada e adapte o contrato antes de conectar a interface ao backend.

## 3.11. Sumário

- [1. Preparar o ambiente e criar o projeto](#etapa-1)
- [2. Instalar dependências e configurar os aliases](#etapa-2)
- [3. Adicionar componentes, tema e providers](#etapa-3)
- [4. Organizar os arquivos e o padrão visual](#etapa-4)
- [5. Definir os tipos](#etapa-5)
- [6. Configurar os campos](#etapa-6)
- [7. Criar o formulário reutilizável](#etapa-7)
- [8. Preparar os modais e a exclusão](#etapa-8)
- [9. Criar a tabela reutilizável](#etapa-9)
- [10. Montar a tela com abas e ações](#etapa-10)
- [11. Conferir o contrato da API](#etapa-11)
- [12. Configurar o endereço da API](#etapa-12)
- [13. Criar o cliente HTTP e os serviços](#etapa-13)
- [14. Conectar a página e adaptar as associações](#etapa-14)
- [15. Executar, conferir e limpar componentes extras](#etapa-15)
- [16. Entender o fluxo e adicionar uma entidade](#etapa-16)
- [Comandos necessários](#comandos-necessarios)
- [Resumo das responsabilidades](#responsabilidades)
- [Problemas comuns](#problemas-comuns)
- [Referências](#referencias)

## 3.12. Etapas e entregas

| Etapa | Resultado esperado |
| --- | --- |
| Etapa 1 — Base do projeto, passos 1 a 4 | Projeto, dependências, componentes, tema, providers e organização dos arquivos. |
| Etapa 2 — Interface, passos 5 a 10 | Tipos, campos, formulários, modais, tabelas e tela com abas e ações. |
| Etapa 3 — Integração, passos 11 a 16 | Contrato conferido, cliente HTTP, serviços, operações ligadas à API e aplicação verificada. |

## 3.13. Tecnologias

| Tecnologia | Finalidade |
| --- | --- |
| Node.js e npm | Executar o projeto e gerenciar dependências. |
| Next.js | Organizar páginas com App Router e encaminhar chamadas à API. |
| React | Construir componentes e controlar estados da interface. |
| TypeScript | Descrever dados, propriedades e operações. |
| Tailwind CSS e PostCSS | Processar estilos e aplicar o padrão visual. |
| Untitled UI | Fornecer os componentes padrão da oficina. |
| React Aria Components | Fornecer comportamento acessível aos componentes Untitled UI. |
| Radix Dialog | Controlar foco, teclado e abertura dos diálogos. |
| next-themes | Gerenciar o tema. |
| API Flask | Validar os dados, aplicar regras e persistir registros. |

## 3.14. Componentes padrão

| Componente | Referência | Uso |
| --- | --- | --- |
| Input | [Inputs](https://www.untitledui.com/react/components/inputs) | Textos curtos, e-mail, URL e datas. |
| Button | [Buttons](https://www.untitledui.com/react/components/buttons) | Ações e envio. |
| TextArea | [Textareas](https://www.untitledui.com/react/components/textareas) | Descrições e textos longos. |
| Select | [Select](https://www.untitledui.com/react/components/select) | Opções e registros associados. |
| Tabs | [Tabs](https://www.untitledui.com/react/components/tabs) | Navegação entre seções. |
| Table | [Tables](https://www.untitledui.com/react/components/tables) | Listagens e ações por registro. |
| Dialog Box | [Dialog Box](https://21st.dev/@itsankitverma/components/dialog-box) | Formulários e consultas em modal. |
| Delete Dialog | [Delete Dialog](https://21st.dev/@felipemenezes098/components/the-dialog/delete-account) | Confirmação de exclusão. |

Os diálogos adaptam as referências enviadas ao padrão da oficina, usando Radix Dialog e os botões Untitled UI.

## 3.15. Estrutura do Projeto

```text
frontend/
├── .env.local
├── .env.example
├── components.json
├── next.config.ts
├── package.json
├── package-lock.json
├── postcss.config.mjs
├── scripts/
│   └── limpar-componentes.cjs
└── src/
    ├── app/
    │   ├── layout.tsx
    │   └── page.tsx
    ├── styles/
    │   ├── globals.css
    │   └── theme.css
    ├── providers/
    │   ├── route-provider.tsx
    │   └── theme-provider.tsx
    ├── hooks/
    │   └── use-resize-observer.ts
    ├── utils/
    │   ├── cx.ts
    │   └── is-react-component.ts
    ├── lib/
    │   ├── types.ts
    │   ├── campos.ts
    │   ├── api.ts
    │   └── dialog-portal.ts
    └── components/
        ├── base/
        ├── application/
        ├── foundations/
        ├── ui/
        │   ├── dialog.tsx
        │   └── dialog-box.tsx
        └── oficina/
            ├── aplicacao.tsx
            ├── formulario.tsx
            ├── modal.tsx
            ├── excluir-registro.tsx
            ├── tabela.tsx
            └── catalogo.tsx
```

Preserve os auxiliares importados pelos componentes instalados. Acrescente os arquivos de cada entidade conforme a ficha do grupo.

## 3.16. Base do projeto

<a id="etapa-1"></a>

### 3.16.1. Preparar o ambiente e criar o projeto

Utilize Node.js 22.12 ou superior, npm e um editor. Confira:

```powershell
node --version
npm --version
```

Se o projeto ainda não existir:

```powershell
npx create-next-app@latest frontend --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm
cd frontend
```

Se já existir, entre na pasta e execute `npm install`. Mantenha `package-lock.json` no versionamento. Os comandos deste guia usam PowerShell.

<a id="etapa-2"></a>

### 3.16.2. Instalar dependências e configurar os aliases

Na raiz do frontend:

```powershell
npm install react-aria-components @untitledui/icons tailwind-merge tailwindcss-animate tailwindcss-react-aria-components next-themes lucide-react @radix-ui/react-dialog
npm install -D tailwindcss @tailwindcss/postcss postcss
```

Crie `components.json`:

```json
{
  "version": "8",
  "aliases": {
    "components": "@/components/",
    "utils": "@/utils/",
    "hooks": "@/hooks/",
    "styles": "@/styles/"
  }
}
```

O alias `@/*` aponta para `src/*`. Use a mesma configuração nos imports e na CLI.

<a id="etapa-3"></a>

### 3.16.3. Adicionar componentes, tema e providers

Adicione apenas as famílias utilizadas:

```powershell
npx untitledui@latest add input button textarea select --path src/components
npx untitledui@latest add tabs table --path src/components
```

Não utilize `--all`, `--include-all-components` ou `--yes`. Preserve os auxiliares das famílias selecionadas. A [documentação da CLI](https://www.untitledui.com/react/docs/cli) descreve a instalação por componente.

##### Estilos e PostCSS

Mantenha uma única configuração do PostCSS. Os estilos abaixo preservam os tokens usados pelos componentes padrão e seus auxiliares. Fonte, tamanhos e alturas de linha ficam em `theme.css`; as telas usam as classes do Tailwind, sem uma folha separada de tipografia.

Se acrescentar um componente que use novos tokens, inclua seus estilos ao adaptar o projeto.

**`postcss.config.mjs`**

```js
export default {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};
```

**`src/styles/theme.css`**

```css
@theme {
  --font-body: var(--font-inter, "Inter"), -apple-system, "Segoe UI", Roboto, Arial, sans-serif;
  --text-xs: calc(var(--spacing) * 3);
  --text-xs--line-height: calc(var(--spacing) * 4.5);
  --text-sm: calc(var(--spacing) * 3.5);
  --text-sm--line-height: calc(var(--spacing) * 5);
  --text-md: calc(var(--spacing) * 4);
  --text-md--line-height: calc(var(--spacing) * 6);
  --text-lg: calc(var(--spacing) * 4.5);
  --text-lg--line-height: calc(var(--spacing) * 7);
  --text-xl: calc(var(--spacing) * 5);
  --text-xl--line-height: calc(var(--spacing) * 7.5);
  --text-display-xs: calc(var(--spacing) * 6);
  --text-display-xs--line-height: calc(var(--spacing) * 8);
  --text-display-md: calc(var(--spacing) * 9);
  --text-display-md--line-height: calc(var(--spacing) * 11);
  --text-display-md--letter-spacing: -0.72px;
  --text-display-lg: calc(var(--spacing) * 12);
  --text-display-lg--line-height: calc(var(--spacing) * 15);
  --text-display-lg--letter-spacing: -0.96px;
  --text-display-xl: calc(var(--spacing) * 15);
  --text-display-xl--line-height: calc(var(--spacing) * 18);
  --text-display-xl--letter-spacing: -1.2px;
  --max-width-container: 1280px;
  --shadow-xs: 0px 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-sm: 0px 1px 3px rgba(0, 0, 0, 0.1), 0px 1px 2px -1px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0px 12px 16px -4px rgba(0, 0, 0, 0.08), 0px 4px 6px -2px rgba(0, 0, 0, 0.03), 0px 2px 2px -1px rgba(0, 0, 0, 0.04);
  --shadow-2xl: 0px 24px 48px -12px rgba(0, 0, 0, 0.18), 0px 4px 4px -2px rgba(0, 0, 0, 0.04);
  --shadow-skeuomorphic: 0px 0px 0px 1px rgba(0, 0, 0, 0.18) inset, 0px -2px 0px 0px rgba(0, 0, 0, 0.05) inset;
  --shadow-xs-skeuomorphic: var(--shadow-skeuomorphic), var(--shadow-xs);
  --color-brand-50: rgb(249 245 255);
  --color-brand-100: rgb(244 235 255);
  --color-brand-200: rgb(233 215 254);
  --color-brand-300: rgb(214 187 251);
  --color-brand-400: rgb(182 146 246);
  --color-brand-500: rgb(158 119 237);
  --color-brand-600: rgb(127 86 217);
  --color-brand-700: rgb(105 65 198);
  --color-brand-800: rgb(83 56 158);
  --color-brand-900: rgb(66 48 125);
  --color-brand-950: rgb(44 28 95);
  --color-utility-blue-50: var(--color-blue-50);
  --color-utility-blue-100: var(--color-blue-100);
  --color-utility-blue-200: var(--color-blue-200);
  --color-utility-blue-400: var(--color-blue-400);
  --color-utility-blue-500: var(--color-blue-500);
  --color-utility-blue-700: var(--color-blue-700);
  --color-utility-brand-50: var(--color-brand-50);
  --color-utility-brand-100: var(--color-brand-100);
  --color-utility-brand-200: var(--color-brand-200);
  --color-utility-brand-400: var(--color-brand-400);
  --color-utility-brand-500: var(--color-brand-500);
  --color-utility-brand-700: var(--color-brand-700);
  --color-utility-neutral-50: var(--color-neutral-50);
  --color-utility-neutral-100: var(--color-neutral-100);
  --color-utility-neutral-200: var(--color-neutral-200);
  --color-utility-neutral-300: var(--color-neutral-300);
  --color-utility-neutral-400: var(--color-neutral-400);
  --color-utility-neutral-500: var(--color-neutral-500);
  --color-utility-neutral-700: var(--color-neutral-700);
  --color-utility-red-50: var(--color-red-50);
  --color-utility-red-100: var(--color-red-100);
  --color-utility-red-200: var(--color-red-200);
  --color-utility-red-400: var(--color-red-400);
  --color-utility-red-500: var(--color-red-500);
  --color-utility-red-700: var(--color-red-700);
  --color-utility-yellow-50: var(--color-yellow-50);
  --color-utility-yellow-100: var(--color-yellow-100);
  --color-utility-yellow-200: var(--color-yellow-200);
  --color-utility-yellow-400: var(--color-yellow-400);
  --color-utility-yellow-500: var(--color-yellow-500);
  --color-utility-yellow-700: var(--color-yellow-700);
  --color-utility-green-50: var(--color-green-50);
  --color-utility-green-100: var(--color-green-100);
  --color-utility-green-200: var(--color-green-200);
  --color-utility-green-400: var(--color-green-400);
  --color-utility-green-500: var(--color-green-500);
  --color-utility-green-700: var(--color-green-700);
  --color-utility-orange-50: var(--color-orange-50);
  --color-utility-orange-100: var(--color-orange-100);
  --color-utility-orange-200: var(--color-orange-200);
  --color-utility-orange-400: var(--color-orange-400);
  --color-utility-orange-500: var(--color-orange-500);
  --color-utility-orange-700: var(--color-orange-700);
  --color-utility-indigo-50: var(--color-indigo-50);
  --color-utility-indigo-100: var(--color-indigo-100);
  --color-utility-indigo-200: var(--color-indigo-200);
  --color-utility-indigo-400: var(--color-indigo-400);
  --color-utility-indigo-500: var(--color-indigo-500);
  --color-utility-indigo-700: var(--color-indigo-700);
  --color-utility-pink-50: var(--color-pink-50);
  --color-utility-pink-100: var(--color-pink-100);
  --color-utility-pink-200: var(--color-pink-200);
  --color-utility-pink-400: var(--color-pink-400);
  --color-utility-pink-500: var(--color-pink-500);
  --color-utility-pink-700: var(--color-pink-700);
  --color-utility-purple-50: var(--color-purple-50);
  --color-utility-purple-100: var(--color-purple-100);
  --color-utility-purple-200: var(--color-purple-200);
  --color-utility-purple-400: var(--color-purple-400);
  --color-utility-purple-500: var(--color-purple-500);
  --color-utility-purple-700: var(--color-purple-700);
  --color-utility-sky-50: var(--color-sky-50);
  --color-utility-sky-100: var(--color-sky-100);
  --color-utility-sky-200: var(--color-sky-200);
  --color-utility-sky-400: var(--color-sky-400);
  --color-utility-sky-500: var(--color-sky-500);
  --color-utility-sky-700: var(--color-sky-700);
  --color-utility-slate-50: var(--color-slate-50);
  --color-utility-slate-100: var(--color-slate-100);
  --color-utility-slate-200: var(--color-slate-200);
  --color-utility-slate-400: var(--color-slate-400);
  --color-utility-slate-500: var(--color-slate-500);
  --color-utility-slate-700: var(--color-slate-700);
  --color-text-primary: var(--color-neutral-900);
  --color-text-tertiary: var(--color-neutral-600);
  --color-text-error-primary: var(--color-red-600);
  --color-text-secondary: var(--color-neutral-700);
  --color-text-secondary_hover: var(--color-neutral-800);
  --color-text-tertiary_hover: var(--color-neutral-700);
  --color-text-brand-secondary: var(--color-brand-700);
  --color-text-placeholder: var(--color-neutral-500);
  --color-text-brand-tertiary: var(--color-brand-600);
  --color-text-quaternary: var(--color-neutral-500);
  --color-text-error-primary_hover: var(--color-red-700);
  --color-text-brand-secondary_hover: var(--color-brand-800);
  --color-border-primary: var(--color-neutral-300);
  --color-border-secondary: var(--color-neutral-200);
  --color-border-secondary_alt: rgb(0 0 0 / 0.1);
  --color-border-error: var(--color-red-500);
  --color-border-error_subtle: var(--color-red-300);
  --color-border-brand: var(--color-brand-500);
  --color-fg-white: var(--color-white);
  --color-fg-success-secondary: var(--color-green-500);
  --color-fg-secondary_hover: var(--color-neutral-800);
  --color-fg-brand-secondary: var(--color-brand-500);
  --color-fg-brand-primary: var(--color-brand-600);
  --color-fg-quaternary: var(--color-neutral-400);
  --color-fg-quaternary_hover: var(--color-neutral-500);
  --color-fg-error-primary: var(--color-red-600);
  --color-fg-error-secondary: var(--color-red-500);
  --color-fg-brand-primary_alt: var(--color-fg-brand-primary);
  --color-fg-brand-secondary_alt: var(--color-fg-brand-secondary);
  --color-fg-brand-secondary_hover: var(--color-brand-600);
  --color-bg-primary: var(--color-white);
  --color-bg-tertiary: var(--color-neutral-100);
  --color-bg-brand-solid: var(--color-brand-600);
  --color-bg-error-solid: var(--color-red-600);
  --color-bg-primary_hover: var(--color-neutral-50);
  --color-bg-brand-solid_hover: var(--color-brand-700);
  --color-bg-error-primary: var(--color-red-50);
  --color-bg-secondary: var(--color-neutral-50);
  --color-bg-primary_alt: var(--color-white);
  --color-bg-brand-primary_alt: var(--color-brand-50);
  --color-bg-secondary_alt: var(--color-neutral-50);
  --color-bg-primary-solid: var(--color-neutral-950);
  --color-bg-error-solid_hover: var(--color-red-700);
  --color-focus-ring: var(--color-brand-500);
  --color-tooltip-supporting-text: var(--color-neutral-300);
  --background-color-primary: var(--color-bg-primary);
  --background-color-tertiary: var(--color-bg-tertiary);
  --background-color-brand-solid: var(--color-bg-brand-solid);
  --background-color-error-solid: var(--color-bg-error-solid);
  --background-color-primary_hover: var(--color-bg-primary_hover);
  --background-color-brand-solid_hover: var(--color-bg-brand-solid_hover);
  --background-color-error-primary: var(--color-bg-error-primary);
  --background-color-secondary: var(--color-bg-secondary);
  --background-color-primary_alt: var(--color-bg-primary_alt);
  --background-color-brand-primary_alt: var(--color-bg-brand-primary_alt);
  --background-color-secondary_alt: var(--color-bg-secondary_alt);
  --background-color-primary-solid: var(--color-bg-primary-solid);
  --background-color-error-solid_hover: var(--color-bg-error-solid_hover);
  --text-color-primary: var(--color-text-primary);
  --text-color-secondary: var(--color-text-secondary);
  --text-color-secondary_hover: var(--color-text-secondary_hover);
  --text-color-tertiary: var(--color-text-tertiary);
  --text-color-tertiary_hover: var(--color-text-tertiary_hover);
  --text-color-error-primary: var(--color-text-error-primary);
  --text-color-brand-secondary: var(--color-text-brand-secondary);
  --text-color-placeholder: var(--color-text-placeholder);
  --text-color-brand-tertiary: var(--color-text-brand-tertiary);
  --text-color-quaternary: var(--color-text-quaternary);
  --text-color-error-primary_hover: var(--color-text-error-primary_hover);
  --text-color-brand-secondary_hover: var(--color-text-brand-secondary_hover);
  --text-color-tooltip-supporting-text: var(--color-tooltip-supporting-text);
  --border-color-secondary: var(--color-border-secondary);
  --ring-color-primary: var(--color-border-primary);
  --ring-color-secondary: var(--color-border-secondary);
  --ring-color-secondary_alt: var(--color-border-secondary_alt);
  --ring-color-error: var(--color-border-error);
  --ring-color-error_subtle: var(--color-border-error_subtle);
  --ring-color-brand: var(--color-border-brand);
  --ring-color-brand-solid: var(--color-bg-brand-solid);
  --outline-color-error: var(--color-border-error);
  --outline-color-brand: var(--color-border-brand);
}

@layer base {
  .dark-mode {
    --color-utility-blue-50: var(--color-blue-950);
    --color-utility-blue-100: var(--color-blue-900);
    --color-utility-blue-200: var(--color-blue-800);
    --color-utility-blue-400: var(--color-blue-600);
    --color-utility-blue-700: var(--color-blue-300);
    --color-utility-brand-50: var(--color-brand-950);
    --color-utility-brand-100: var(--color-brand-900);
    --color-utility-brand-200: var(--color-brand-800);
    --color-utility-brand-400: var(--color-brand-600);
    --color-utility-brand-700: var(--color-brand-300);
    --color-utility-neutral-50: var(--color-neutral-900);
    --color-utility-neutral-100: var(--color-neutral-800);
    --color-utility-neutral-200: var(--color-neutral-700);
    --color-utility-neutral-300: var(--color-neutral-700);
    --color-utility-neutral-400: var(--color-neutral-600);
    --color-utility-neutral-700: var(--color-neutral-300);
    --color-utility-red-50: var(--color-red-950);
    --color-utility-red-100: var(--color-red-900);
    --color-utility-red-200: var(--color-red-800);
    --color-utility-red-400: var(--color-red-600);
    --color-utility-red-700: var(--color-red-300);
    --color-utility-yellow-50: var(--color-yellow-950);
    --color-utility-yellow-100: var(--color-yellow-900);
    --color-utility-yellow-200: var(--color-yellow-800);
    --color-utility-yellow-400: var(--color-yellow-600);
    --color-utility-yellow-700: var(--color-yellow-300);
    --color-utility-green-50: var(--color-green-950);
    --color-utility-green-100: var(--color-green-900);
    --color-utility-green-200: var(--color-green-800);
    --color-utility-green-400: var(--color-green-600);
    --color-utility-green-700: var(--color-green-300);
    --color-utility-orange-50: var(--color-orange-950);
    --color-utility-orange-100: var(--color-orange-900);
    --color-utility-orange-200: var(--color-orange-800);
    --color-utility-orange-400: var(--color-orange-600);
    --color-utility-orange-700: var(--color-orange-300);
    --color-utility-indigo-50: var(--color-indigo-950);
    --color-utility-indigo-100: var(--color-indigo-900);
    --color-utility-indigo-200: var(--color-indigo-800);
    --color-utility-indigo-400: var(--color-indigo-600);
    --color-utility-indigo-700: var(--color-indigo-300);
    --color-utility-pink-50: var(--color-pink-950);
    --color-utility-pink-100: var(--color-pink-900);
    --color-utility-pink-200: var(--color-pink-800);
    --color-utility-pink-400: var(--color-pink-600);
    --color-utility-pink-700: var(--color-pink-300);
    --color-utility-purple-50: var(--color-purple-950);
    --color-utility-purple-100: var(--color-purple-900);
    --color-utility-purple-200: var(--color-purple-800);
    --color-utility-purple-400: var(--color-purple-600);
    --color-utility-purple-700: var(--color-purple-300);
    --color-utility-sky-50: var(--color-sky-950);
    --color-utility-sky-100: var(--color-sky-900);
    --color-utility-sky-200: var(--color-sky-800);
    --color-utility-sky-400: var(--color-sky-600);
    --color-utility-sky-700: var(--color-sky-300);
    --color-utility-slate-50: var(--color-slate-950);
    --color-utility-slate-100: var(--color-slate-900);
    --color-utility-slate-200: var(--color-slate-800);
    --color-utility-slate-400: var(--color-slate-600);
    --color-utility-slate-700: var(--color-slate-300);
    --color-text-primary: var(--color-neutral-50);
    --color-text-tertiary: var(--color-neutral-400);
    --color-text-error-primary: var(--color-red-400);
    --color-text-secondary: var(--color-neutral-300);
    --color-text-secondary_hover: var(--color-neutral-200);
    --color-text-tertiary_hover: var(--color-neutral-300);
    --color-text-brand-secondary: var(--color-neutral-300);
    --color-text-brand-tertiary: var(--color-neutral-400);
    --color-text-quaternary: var(--color-neutral-400);
    --color-text-error-primary_hover: var(--color-red-300);
    --color-text-brand-secondary_hover: var(--color-neutral-200);
    --color-border-secondary: var(--color-neutral-800);
    --color-border-error_subtle: var(--color-red-500);
    --color-border-primary: var(--color-neutral-700);
    --color-border-brand: var(--color-brand-400);
    --color-border-error: var(--color-red-400);
    --color-border-secondary_alt: var(--color-neutral-800);
    --color-fg-success-secondary: var(--color-green-400);
    --color-fg-secondary_hover: var(--color-neutral-200);
    --color-fg-brand-primary: var(--color-brand-500);
    --color-fg-quaternary: var(--color-neutral-600);
    --color-fg-error-primary: var(--color-red-500);
    --color-fg-error-secondary: var(--color-red-400);
    --color-fg-brand-primary_alt: var(--color-neutral-300);
    --color-fg-brand-secondary_alt: var(--color-neutral-600);
    --color-fg-brand-secondary_hover: var(--color-neutral-500);
    --color-bg-primary: var(--color-neutral-950);
    --color-bg-tertiary: var(--color-neutral-800);
    --color-bg-primary_hover: var(--color-neutral-900);
    --color-bg-brand-solid_hover: var(--color-brand-500);
    --color-bg-error-primary: var(--color-red-950);
    --color-bg-secondary: var(--color-neutral-900);
    --color-bg-primary_alt: var(--color-bg-secondary);
    --color-bg-brand-primary_alt: var(--color-bg-secondary);
    --color-bg-secondary_alt: var(--color-bg-primary);
    --color-bg-primary-solid: var(--color-bg-secondary);
    --color-bg-error-solid_hover: var(--color-red-500);
  }
}
```

**`src/styles/globals.css`**

```css
@import "tailwindcss";
@import "./theme.css";

@plugin "tailwindcss-react-aria-components";
@plugin "tailwindcss-animate";

@custom-variant dark (&:where(.dark-mode, .dark-mode *));
@custom-variant label (& [data-label]);

@utility transition-inherit-all {
  transition-property: inherit;
  transition-duration: inherit;
  transition-timing-function: inherit;
}

@layer base {
  html,
  body {
    min-height: 100%;
    font-family: var(--font-body);
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
  }
  body {
    margin: 0;
    background: #070f1b;
    color: var(--color-text-primary);
  }
  input[type="date"],
  input[type="datetime-local"] {
    color-scheme: dark;
  }
  textarea {
    resize: vertical;
  }
}

.dark-mode {
  color-scheme: dark;
  --color-bg-primary: #0d1a2b;
  --color-bg-primary_alt: #0d1a2b;
  --color-bg-primary_hover: #14243a;
  --color-bg-secondary: #101e31;
  --color-bg-secondary_alt: #14243a;
  --color-bg-tertiary: #182a41;
  --color-border-primary: #38506e;
  --color-border-secondary: #263b54;
  --color-text-primary: #f4f5fb;
  --color-text-secondary: #b8c7de;
  --color-text-tertiary: #94a8c5;
}
```

##### Utilitários e hook

Crie os auxiliares importados pelos componentes:

**`src/utils/cx.ts`**

```ts
import { extendTailwindMerge } from "tailwind-merge";

const twMerge = extendTailwindMerge({
  extend: {
    theme: {
      text: [
        "display-xs",
        "display-sm",
        "display-md",
        "display-lg",
        "display-xl",
        "display-2xl",
      ],
    },
  },
});

/**
 * This function is a wrapper around the twMerge function.
 * It is used to merge the classes inside style objects.
 */
export const cx = twMerge;

/**
 * This function does nothing besides helping us to be able to
 * sort the classes inside style objects which is not supported
 * by the Tailwind IntelliSense by default.
 */
export function sortCx<
  T extends Record<
    string,
    string | number | Record<string, string | number | Record<string, string | number>>
  >,
>(classes: T): T {
  return classes;
}
```

**`src/utils/is-react-component.ts`**

```ts
import type React from "react";

// Below the import on purpose: the shadcn CLI drops comments that come before a file's first
// statement, which would drop this directive and fail lint in projects installed that way.
// `any` is required here: with `unknown`, the type guards below stop narrowing to component types.
/* eslint-disable @typescript-eslint/no-explicit-any */

type ReactComponent = React.FC<any> | React.ComponentClass<any, any>;

/**
 * Checks if a given value is a function component.
 */
export const isFunctionComponent = (component: any): component is React.FC<any> => {
  return typeof component === "function";
};

/**
 * Checks if a given value is a class component.
 */
export const isClassComponent = (
  component: any,
): component is React.ComponentClass<any, any> => {
  return (
    typeof component === "function" &&
    component.prototype &&
    (!!component.prototype.isReactComponent || !!component.prototype.render)
  );
};

/**
 * Checks if a given value is a forward ref component.
 */
export const isForwardRefComponent = (
  component: any,
): component is React.ForwardRefExoticComponent<any> => {
  return (
    typeof component === "object" &&
    component !== null &&
    component.$typeof?.toString() === "Symbol(react.forward_ref)"
  );
};

/**
 * Checks if a given value is a valid React component.
 */
export const isReactComponent = (component: any): component is ReactComponent => {
  return (
    isFunctionComponent(component) ||
    isForwardRefComponent(component) ||
    isClassComponent(component)
  );
};
```

**`src/hooks/use-resize-observer.ts`**

```ts
"use client";
import { useEffect, type RefObject } from "react";
type Options<T extends Element> = {
  ref: RefObject<T | null>;
  box?: ResizeObserverBoxOptions;
  onResize: ResizeObserverCallback;
};
export function useResizeObserver<T extends Element>({
  ref,
  box = "content-box",
  onResize,
}: Options<T>) {
  useEffect(() => {
    const element = ref.current;
    if (!element || typeof ResizeObserver === "undefined") return;
    const observer = new ResizeObserver(onResize);
    observer.observe(element, { box });
    return () => observer.disconnect();
  }, [ref, box, onResize]);
}
```

##### Providers e layout

O layout importa o CSS global e envolve a página com os providers:

**`src/providers/route-provider.tsx`**

```tsx
"use client";
import type { ReactNode } from "react";
import { useRouter } from "next/navigation";
import { RouterProvider } from "react-aria-components";
export function RouteProvider({ children }: { children: ReactNode }) {
  const router = useRouter();
  return (
    <RouterProvider navigate={(href) => router.push(href)}>{children}</RouterProvider>
  );
}
```

**`src/providers/theme-provider.tsx`**

```tsx
"use client";
import type { ReactNode } from "react";
import { ThemeProvider as NextThemeProvider } from "next-themes";
export function ThemeProvider({ children }: { children: ReactNode }) {
  return (
    <NextThemeProvider
      attribute="class"
      forcedTheme="dark"
      value={{
        light: "light-mode",
        dark: "dark-mode",
      }}
      disableTransitionOnChange
    >
      {children}
    </NextThemeProvider>
  );
}
```

**`src/app/layout.tsx`**

```tsx
import type { Metadata } from "next";
import type { ReactNode } from "react";
import { Inter } from "next/font/google";
import { RouteProvider } from "@/providers/route-provider";
import { ThemeProvider } from "@/providers/theme-provider";
import "@/styles/globals.css";

const inter = Inter({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-inter",
});

export const metadata: Metadata = {
  title: "Oficina BV App",
  description: "Aplicação desenvolvida durante a Oficina BV App.",
};

export default function RootLayout({
  children,
}: Readonly<{ children: ReactNode }>) {
  return (
    <html lang="pt-BR" suppressHydrationWarning>
      <body
        className={`${inter.variable} min-h-screen bg-primary text-primary antialiased`}
      >
        <RouteProvider>
          <ThemeProvider>{children}</ThemeProvider>
        </RouteProvider>
      </body>
    </html>
  );
}
```

<a id="etapa-4"></a>

### 3.16.4. Organizar os arquivos e o padrão visual

Os componentes da biblioteca ficam nas pastas geradas pela CLI. Os componentes da oficina ficam em `src/components/oficina`.

| Elemento | Padrão visual |
| --- | --- |
| Cabeçalho | Identificação da oficina, título e descrição do contexto. |
| Navegação | Abas sublinhadas, uma seção por necessidade da aplicação. |
| Listagem | Título, ação de cadastro e tabela no mesmo card. |
| Formulário | Uma coluna, campos com largura completa e rodapé com borda. |
| Modal | Largura limitada, rolagem e título da operação. |
| Exclusão | Nome do registro, consequência prevista pela API e confirmação. |
| Feedback | Carregamento, erro e sucesso próximos à operação. |

Defina as seções a partir da ficha. Uma entidade pode ter somente cadastro e listagem; outra pode ter consulta individual, edição e exclusão. Exiba as ações previstas pelas operações do backend.

Use Select para associações únicas. Seleção múltipla depende de a API aceitar várias referências.

## 3.17. Interface

<a id="etapa-5"></a>

### 3.17.1. Definir os tipos

Crie `src/lib/types.ts`. A entidade técnica demonstra o padrão de cadastro, listagem, consulta, edição e exclusão. Substitua seus campos pelo contrato do grupo.

```typescript
export type Registro = {
  id: number;
  nome: string;
  descricao: string | null;
};

export type RegistroPayload = {
  nome: string;
  descricao: string | null;
};

export type ServicoRegistro = {
  listar: () => Promise<Registro[]>;
  buscar: (id: number) => Promise<Registro>;
  criar: (dados: RegistroPayload) => Promise<Registro>;
  atualizar: (id: number, dados: RegistroPayload) => Promise<Registro>;
  excluir: (id: number) => Promise<void>;
};
```

A resposta inclui o ID gerado pelo backend. O payload descreve os campos enviados. Datas, nulabilidade, nomes JSON e tipos de ID acompanham a API. Se houver paginação, descreva o envelope da resposta em vez de tratar a listagem como array.

<a id="etapa-6"></a>

### 3.17.2. Configurar os campos

Crie `src/lib/campos.ts`:

```typescript
import type { Campo } from "@/components/oficina/formulario";

export const camposRegistro: Campo[] = [
  { name: "nome", label: "Nome", required: true },
  { name: "descricao", label: "Descrição", type: "textarea" },
];
```

Acrescente `maxLength` conforme o schema. O exemplo não define limites de negócio para os quatro contextos.

| Dado | Componente/configuração |
| --- | --- |
| Texto curto | Input, `type: "text"`. |
| E-mail | Input, `type: "email"`. |
| URL | Input, `type: "url"`. |
| Data | Input, `type: "date"`. |
| Data e hora local | Input, `type: "datetime-local"`. |
| Texto longo | TextArea, `type: "textarea"`. |
| Referência a uma entidade | Select, `options` carregadas da API. |

Os Selects usam strings no formulário; converta para o tipo exigido pelo backend ao montar o payload.

<a id="etapa-7"></a>

### 3.17.3. Criar o formulário reutilizável

O formulário controla valores, envio, obrigatórios e erros com os componentes padrão. O componente pai decide como salvar.

**`src/components/oficina/formulario.tsx`**

```tsx
"use client";

import { useState, type FormEvent } from "react";
import { Input } from "@/components/base/input/input";
import { TextArea } from "@/components/base/textarea/textarea";
import { Select } from "@/components/base/select/select";
import { Button } from "@/components/base/buttons/button";

export type Campo = {
  name: string;
  label: string;
  required?: boolean;
  maxLength?: number;
  type?: "text" | "email" | "url" | "date" | "datetime-local" | "textarea";
  options?: {
    id: string;
    label: string;
  }[];
};

type Props = {
  fields: Campo[];
  initial?: Record<string, string>;
  label?: string;
  cancelLabel?: string;
  onSave: (data: Record<string, string>) => Promise<void>;
  onCancel: () => void;
  onBusy: (value: boolean) => void;
};

export function Formulario({
  fields,
  initial = {},
  label = "Cadastrar",
  cancelLabel = "Cancelar",
  onSave,
  onCancel,
  onBusy,
}: Props) {
  const [values, setValues] =
    useState<Record<string, string>>(initial);

  const [busy, setBusy] = useState(false);
  const [error, setError] = useState("");
  const [invalid, setInvalid] = useState<string[]>([]);

  function change(name: string, value: string) {
    setValues((old) => ({
      ...old,
      [name]: value,
    }));

    setInvalid((old) =>
      old.filter((item) => item !== name),
    );
  }

  async function submit(
    event: FormEvent<HTMLFormElement>,
  ) {
    event.preventDefault();
    if (busy) return;

    const data = Object.fromEntries(
      fields.map((field) => [
        field.name,
        (values[field.name] || "").trim(),
      ]),
    );

    const missing = fields
      .filter(
        (field) =>
          field.required && !data[field.name],
      )
      .map((field) => field.name);

    setInvalid(missing);

    if (missing.length) {
      setError("Preencha os campos obrigatórios indicados.");
      return;
    }

    setBusy(true);
    onBusy(true);
    setError("");

    try {
      await onSave(data);
    } catch (err) {
      setError(
        err instanceof Error
          ? err.message
          : "Não foi possível salvar.",
      );
    } finally {
      setBusy(false);
      onBusy(false);
    }
  }

  return (
    <form onSubmit={submit} className="space-y-4">
      {fields.map((field) => {
        const common = {
          label: field.label,
          name: field.name,
          isRequired: field.required,
          isDisabled: busy,
          isInvalid: invalid.includes(field.name),
        };

        if (field.options) {
          return (
            <Select
              key={field.name}
              {...common}
              placeholder="Selecione"
              items={field.options}
              selectedKey={values[field.name] || null}
              onSelectionChange={(key) =>
                change(
                  field.name,
                  key == null ? "" : String(key),
                )
              }
            >
              {(item) => (
                <Select.Item id={item.id}>
                  {item.label}
                </Select.Item>
              )}
            </Select>
          );
        }

        if (field.type === "textarea") {
          return (
            <TextArea
              key={field.name}
              {...common}
              rows={3}
              value={values[field.name] || ""}
              maxLength={field.maxLength}
              onChange={(value) =>
                change(field.name, value)
              }
            />
          );
        }

        return (
          <Input
            key={field.name}
            {...common}
            type={field.type || "text"}
            maxLength={field.maxLength}
            value={values[field.name] || ""}
            onChange={(value) =>
              change(field.name, value)
            }
          />
        );
      })}

      {error && (
        <p
          role="alert"
          className="break-words text-sm text-red-300"
        >
          {error}
        </p>
      )}

      <div className="flex flex-wrap justify-end gap-3 border-t border-secondary pt-5">
        <Button
          type="button"
          color="secondary"
          isDisabled={busy}
          onPress={onCancel}
        >
          {cancelLabel}
        </Button>

        <Button type="submit" isLoading={busy}>
          {label}
        </Button>
      </div>
    </form>
  );
}
```

Os valores permanecem quando a API recusa a operação. Obrigatórios são conferidos depois de `trim`. E-mail, URL, datas e limites também usam a validação nativa do input.

Use `initial` para edição, `label="Salvar alterações"` no botão e `onCancel` para fechar ou limpar. Implemente regras específicas antes do envio; o backend mantém a validação definitiva.

<a id="etapa-8"></a>

### 3.17.4. Preparar os modais e a exclusão

A base combina Radix Dialog, Dialog Box e o diálogo de exclusão. Clique externo não fecha o modal. Escape e botões podem fechá-lo quando nenhuma operação estiver em andamento.

##### Compartilhar o portal do Dialog

O popover do Select deve permanecer dentro do limite de foco e interação do diálogo:

**`src/lib/dialog-portal.ts`**

```ts
"use client";

import { createContext } from "react";

// Keep Select popovers inside the Radix dialog focus and pointer boundary.
export const DialogPortalContext = createContext<HTMLElement | null>(null);
```

**`src/components/ui/dialog.tsx`**

```tsx
"use client";

import * as Primitive from "@radix-ui/react-dialog";
import {
  forwardRef,
  useState,
  useCallback,
  type ComponentPropsWithoutRef,
  type ElementRef,
  type HTMLAttributes,
} from "react";

import { DialogPortalContext } from "@/lib/dialog-portal";

export const Dialog = Primitive.Root;
export const DialogTrigger = Primitive.Trigger;
export const DialogClose = Primitive.Close;
export const DialogTitle = Primitive.Title;
export const DialogDescription = Primitive.Description;

export const DialogContent = forwardRef<
  ElementRef<typeof Primitive.Content>,
  ComponentPropsWithoutRef<typeof Primitive.Content>
>(({ className = "", children, ...props }, ref) => {
  const [container, setContainer] = useState<HTMLDivElement | null>(null);
  const attachRef = useCallback((node: HTMLDivElement | null) => {
    setContainer(node);
    if (typeof ref === "function") ref(node);
    else if (ref) ref.current = node;
  }, [ref]);
  return (
  <Primitive.Portal>
    <Primitive.Overlay className="fixed inset-0 z-40 bg-black/40 backdrop-blur-sm" />

    <Primitive.Content
      ref={attachRef}
      {...props}
      className={`fixed left-1/2 top-1/2 z-50 grid w-[calc(100%-2rem)] max-w-md -translate-x-1/2 -translate-y-1/2 gap-4 rounded-2xl border border-secondary bg-primary p-8 text-primary shadow-2xl outline-none ${className}`}
    >
      <DialogPortalContext.Provider value={container}>{children}</DialogPortalContext.Provider>
    </Primitive.Content>
  </Primitive.Portal>
  );
});

DialogContent.displayName = "DialogContent";

export function DialogHeader({
  className = "",
  ...props
}: HTMLAttributes<HTMLDivElement>) {
  return (
    <div
      className={`flex flex-col gap-2 ${className}`}
      {...props}
    />
  );
}

export function DialogFooter({
  className = "",
  ...props
}: HTMLAttributes<HTMLDivElement>) {
  return (
    <div
      className={`flex flex-wrap justify-end gap-3 ${className}`}
      {...props}
    />
  );
}
```

**`src/components/ui/dialog-box.tsx`**

```tsx
"use client";

import type { ReactNode } from "react";
import { Button } from "@/components/base/buttons/button";
import {
  Dialog,
  DialogContent,
  DialogTitle,
  DialogDescription,
} from "./dialog";

type Props = {
  open: boolean;
  title: string;
  description?: string;
  busy?: boolean;
  onClose: () => void;
  children: ReactNode;
};

export function Component({
  open,
  title,
  description = "Oficina BV App",
  busy = false,
  onClose,
  children,
}: Props) {
  return (
    <Dialog
      open={open}
      onOpenChange={(nextOpen) => {
        if (!nextOpen && !busy) {
          onClose();
        }
      }}
    >
      <DialogContent
        className="max-h-[90dvh] overflow-y-auto sm:max-w-xl"
        onInteractOutside={(event) =>
          event.preventDefault()
        }
        onEscapeKeyDown={(event) => {
          if (busy) event.preventDefault();
        }}
      >
        <div className="absolute right-4 top-4">
          <Button
            color="tertiary"
            size="sm"
            isDisabled={busy}
            onPress={onClose}
            aria-label="Fechar modal"
          >
            ×
          </Button>
        </div>

        <div className="pr-10">
          <DialogTitle className="mb-1 text-2xl font-bold tracking-tight">
            {title}
          </DialogTitle>

          <DialogDescription className="mb-3 text-sm text-secondary">
            {description}
          </DialogDescription>
        </div>

        {children}
      </DialogContent>
    </Dialog>
  );
}
```

**`src/components/base/select/popover.tsx`**

```tsx
"use client";

import type { RefAttributes } from "react";
import { useContext } from "react";
import { DialogPortalContext } from "@/lib/dialog-portal";
import type { PopoverProps as AriaPopoverProps } from "react-aria-components";
import { Popover as AriaPopover } from "react-aria-components";
import { cx } from "@/utils/cx";

interface PopoverProps extends AriaPopoverProps, RefAttributes<HTMLElement> {
    /** Caps how tall the popover can grow before its list starts scrolling: 14rem for `sm`, 16rem for `md`, and 20rem for `lg`. */
    size: "sm" | "md" | "lg";
}

export const Popover = (props: PopoverProps) => {
    const dialogContainer = useContext(DialogPortalContext);
    return (
        <AriaPopover
            placement="bottom"
            containerPadding={0}
            offset={4}
            {...props}
            UNSTABLE_portalContainer={props.UNSTABLE_portalContainer ?? dialogContainer ?? undefined}
            className={(state) =>
                cx(
                    "z-[60] w-(--trigger-width) origin-(--trigger-anchor-point) overflow-x-hidden overflow-y-auto rounded-lg bg-primary py-1 shadow-lg ring-1 ring-secondary_alt outline-hidden will-change-transform",

                    state.isEntering &&
                        "duration-150 ease-out animate-in fade-in placement-right:slide-in-from-left-0.5 placement-top:slide-in-from-bottom-0.5 placement-bottom:slide-in-from-top-0.5",
                    state.isExiting &&
                        "duration-100 ease-in animate-out fade-out placement-right:slide-out-to-left-0.5 placement-top:slide-out-to-bottom-0.5 placement-bottom:slide-out-to-top-0.5",

                    props.size === "sm" && "max-h-56!",
                    props.size === "md" && "max-h-64!",
                    props.size === "lg" && "max-h-80!",

                    typeof props.className === "function" ? props.className(state) : props.className,
                )
            }
        />
    );
};
```

A adaptação usa `UNSTABLE_portalContainer` da versão de referência. Confira os tipos instalados antes de alterar esse trecho.

##### Wrapper do modal

**`src/components/oficina/modal.tsx`**

```tsx
"use client";

import type { ReactNode } from "react";
import { Component as DialogBox } from "@/components/ui/dialog-box";

type Props = {
  title: string;
  description?: string;
  children: ReactNode;
  busy?: boolean;
  onClose: () => void;
};

export function Modal({
  title,
  description = "Oficina BV App",
  children,
  busy = false,
  onClose,
}: Props) {
  return (
    <DialogBox
      open
      title={title}
      description={description}
      busy={busy}
      onClose={onClose}
    >
      {children}
    </DialogBox>
  );
}
```

##### Confirmação de exclusão

A consequência é recebida por propriedade. Preencha conforme a API: bloqueio, remoção de vínculos ou cascata.

**`src/components/oficina/excluir-registro.tsx`**

```tsx
"use client";

import { Trash2Icon } from "lucide-react";
import { Button } from "@/components/base/buttons/button";
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";

type Props = {
  nome: string;
  consequencia: string;
  busy: boolean;
  error: string;
  onClose: () => void;
  onConfirm: () => void;
};

export function ExcluirRegistro({
  nome,
  consequencia,
  busy,
  error,
  onClose,
  onConfirm,
}: Props) {
  return (
    <Dialog
      open
      onOpenChange={(nextOpen) => {
        if (!nextOpen && !busy) onClose();
      }}
    >
      <DialogContent
        className="sm:max-w-md"
        onInteractOutside={(event) =>
          event.preventDefault()
        }
        onEscapeKeyDown={(event) => {
          if (busy) event.preventDefault();
        }}
      >
        <DialogHeader>
          <div className="flex flex-col gap-3">
            <div className="flex size-10 items-center justify-center rounded-full bg-red-500/10 text-red-400">
              <Trash2Icon className="size-5" />
            </div>

            <div className="flex flex-col gap-2">
              <DialogTitle className="text-xl font-semibold">
                Excluir registro?
              </DialogTitle>

              <DialogDescription className="text-sm text-secondary">
                Deseja excluir <strong>{nome}</strong>?
                {consequencia}{" "}
                <span className="font-medium text-primary">
                  Esta ação não pode ser desfeita.
                </span>
              </DialogDescription>
            </div>
          </div>
        </DialogHeader>

        {error && (
          <p role="alert" className="text-sm text-red-300">
            {error}
          </p>
        )}

        <DialogFooter>
          <Button
            color="secondary"
            isDisabled={busy}
            onPress={onClose}
          >
            Cancelar
          </Button>

          <Button
            color="primary-destructive"
            isLoading={busy}
            onPress={onConfirm}
          >
            Excluir registro
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

<a id="etapa-9"></a>

### 3.17.5. Criar a tabela reutilizável

A tabela recebe registros e colunas. As ações são definidas pelo componente pai, conforme as operações disponíveis.

**`src/components/oficina/tabela.tsx`**

```tsx
"use client";
import type { ReactNode } from "react";
import { Table } from "@/components/application/table/table";
export type Coluna<T> = {
  id: string;
  label: string;
  render: (item: T) => ReactNode;
};
type Props<T extends { id: number }> = {
  label: string;
  items: T[];
  columns: Coluna<T>[];
};
export function Tabela<T extends { id: number }>({ label, items, columns }: Props<T>) {
  if (!items.length) {
    return <p className="p-6 text-sm text-secondary">Nenhum registro cadastrado.</p>;
  }
  return (
    <div className="overflow-x-auto">
      <Table aria-label={label} selectionMode="none">
        <Table.Header>
          {columns.map((column, index) => (
            <Table.Head key={column.id} id={column.id} isRowHeader={index === 0}>
              {column.label}
            </Table.Head>
          ))}
        </Table.Header>
        <Table.Body items={items}>
          {(item) => (
            <Table.Row id={item.id}>
              {columns.map((column) => (
                <Table.Cell key={column.id}>{column.render(item)}</Table.Cell>
              ))}
            </Table.Row>
          )}
        </Table.Body>
      </Table>
    </div>
  );
}
```

O exemplo usa ID numérico. Para UUID ou outro identificador, adapte os tipos do registro e da tabela em conjunto.

<a id="etapa-10"></a>

### 3.17.6. Montar a tela com abas e ações

Crie `src/components/oficina/catalogo.tsx`. O catálogo recebe um serviço por propriedade e controla apresentação, consultas e modais. O cliente HTTP será criado na Etapa 3.

O exemplo utiliza uma aba para demonstrar o padrão. Acrescente as seções necessárias à ficha, reutilizando Tabs, formulários e tabelas.

```tsx
"use client";

import { useCallback, useEffect, useState } from "react";
import { Button } from "@/components/base/buttons/button";
import { Tabs } from "@/components/application/tabs/tabs";
import { camposRegistro } from "@/lib/campos";
import type { Registro, RegistroPayload, ServicoRegistro } from "@/lib/types";
import { Formulario } from "./formulario";
import { Modal } from "./modal";
import { ExcluirRegistro } from "./excluir-registro";
import { Tabela, type Coluna } from "./tabela";

type ModalState =
  | { tipo: "criar" }
  | { tipo: "editar"; item: Registro }
  | { tipo: "ver"; item: Registro }
  | { tipo: "excluir"; item: Registro }
  | null;

function mensagem(error: unknown) {
  return error instanceof Error ? error.message : "Não foi possível concluir.";
}

function payload(data: Record<string, string>): RegistroPayload {
  return {
    nome: data.nome.trim(),
    descricao: data.descricao?.trim() || null,
  };
}

export function Catalogo({ servico }: { servico: ServicoRegistro }) {
  const [items, setItems] = useState<Registro[]>([]);
  const [modal, setModal] = useState<ModalState>(null);
  const [loading, setLoading] = useState(true);
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState("");
  const [modalError, setModalError] = useState("");
  const [notice, setNotice] = useState("");

  const carregar = useCallback(async () => {
    setLoading(true);
    setError("");
    try {
      setItems(await servico.listar());
    } catch (err) {
      setError(mensagem(err));
    } finally {
      setLoading(false);
    }
  }, [servico]);

  useEffect(() => { void carregar(); }, [carregar]);

  function fechar() {
    if (!busy) {
      setModal(null);
      setModalError("");
    }
  }

  async function abrir(item: Registro, tipo: "ver" | "editar") {
    if (busy) return;
    setBusy(true);
    setError("");
    setNotice("");
    try {
      setModal({ tipo, item: await servico.buscar(item.id) });
    } catch (err) {
      setError(mensagem(err));
    } finally {
      setBusy(false);
    }
  }

  async function salvar(data: Record<string, string>) {
    if (modal?.tipo === "editar") {
      await servico.atualizar(modal.item.id, payload(data));
      setNotice("Registro atualizado.");
    } else {
      await servico.criar(payload(data));
      setNotice("Registro cadastrado.");
    }
    setModal(null);
    await carregar();
  }

  async function excluir() {
    if (busy || modal?.tipo !== "excluir") return;
    setBusy(true);
    setModalError("");
    try {
      await servico.excluir(modal.item.id);
      setModal(null);
      setNotice("Registro excluído.");
      await carregar();
    } catch (err) {
      setModalError(mensagem(err));
    } finally {
      setBusy(false);
    }
  }

  const columns: Coluna<Registro>[] = [
    { id: "nome", label: "Nome", render: (item) => item.nome },
    { id: "descricao", label: "Descrição", render: (item) => item.descricao || "—" },
    {
      id: "acoes",
      label: "Ações",
      render: (item) => (
        <div className="flex flex-wrap gap-2">
          <Button color="secondary" isDisabled={busy} onPress={() => void abrir(item, "ver")}>
            Visualizar
          </Button>
          <Button color="secondary" isDisabled={busy} onPress={() => void abrir(item, "editar")}>
            Editar
          </Button>
          <Button color="primary-destructive" isDisabled={busy} onPress={() => {
            setModalError("");
            setNotice("");
            setModal({ tipo: "excluir", item });
          }}>Excluir</Button>
        </div>
      ),
    },
  ];

  return (
    <main className="mx-auto max-w-7xl space-y-6 px-4 py-8">
      <header className="border-b border-secondary pb-6">
        <p className="text-sm text-secondary">Oficina BV App</p>
        <h1 className="mt-2 text-3xl font-semibold">Catálogo</h1>
        <p className="mt-2 text-secondary">Cadastre e consulte os registros da aplicação.</p>
      </header>
      {notice && <p role="status" className="text-green-300">{notice}</p>}
      {error && (
        <div role="alert" className="space-y-3">
          <p className="text-red-300">{error}</p>
          <Button color="secondary" isDisabled={busy || loading} onPress={() => void carregar()}>
            Atualizar listagem
          </Button>
        </div>
      )}
      {loading && <p role="status">Carregando dados...</p>}

      <Tabs defaultSelectedKey="registros">
        <Tabs.List type="underline" aria-label="Seções principais">
          <Tabs.Item id="registros" isDisabled={busy}>Registros</Tabs.Item>
        </Tabs.List>
        <Tabs.Panel id="registros" className="mt-6 rounded-xl border border-secondary bg-primary">
          <div className="flex flex-wrap items-center justify-between gap-4 p-5">
            <h2 className="text-xl font-semibold">Registros</h2>
            <Button isDisabled={busy || loading} onPress={() => {
              setNotice("");
              setModal({ tipo: "criar" });
            }}>Cadastrar</Button>
          </div>
          {!loading && <Tabela label="Registros" items={items} columns={columns} />}
        </Tabs.Panel>
      </Tabs>

      {(modal?.tipo === "criar" || modal?.tipo === "editar") && (
        <Modal title={modal.tipo === "editar" ? "Editar registro" : "Cadastrar registro"}
          busy={busy} onClose={fechar}>
          <Formulario
            key={modal.tipo === "editar" ? modal.item.id : "novo"}
            fields={camposRegistro}
            initial={modal.tipo === "editar" ? {
              nome: modal.item.nome, descricao: modal.item.descricao || "",
            } : {}}
            label={modal.tipo === "editar" ? "Salvar alterações" : "Cadastrar"}
            onSave={salvar} onCancel={fechar} onBusy={setBusy}
          />
        </Modal>
      )}
      {modal?.tipo === "ver" && (
        <Modal title={modal.item.nome} onClose={fechar}>
          <dl className="space-y-4">
            <div><dt className="text-secondary">Nome</dt><dd>{modal.item.nome}</dd></div>
            <div><dt className="text-secondary">Descrição</dt>
              <dd className="whitespace-pre-wrap">{modal.item.descricao || "—"}</dd></div>
          </dl>
        </Modal>
      )}
      {modal?.tipo === "excluir" && (
        <ExcluirRegistro nome={modal.item.nome}
          consequencia="Confira os efeitos sobre registros vinculados conforme a regra desta API."
          busy={busy} error={modalError} onClose={fechar} onConfirm={() => void excluir()} />
      )}
    </main>
  );
}
```

Após salvar, o modal fecha e a lista é consultada novamente. Se a gravação funcionar e a nova consulta falhar, o sucesso permanece acompanhado do erro de atualização; repetir a consulta não repete o cadastro.

Remova ações sem endpoint. Troque títulos, descrições e consequências de exclusão pelos textos do contexto. Consultas de vínculos podem aparecer nos detalhes ou em uma seção própria.

## 3.18. Integração com a API

<a id="etapa-11"></a>

### 3.18.1. Conferir o contrato da API

Abra o Swagger do backend do grupo e registre o contrato antes de escrever as chamadas.

| Informação | O que definir |
| --- | --- |
| Rotas | Coleção, consulta por ID e consultas de associações. |
| Métodos | Operações GET, POST, PUT/PATCH e DELETE disponíveis. |
| Entrada | Nomes JSON, obrigatórios, limites e formatos. |
| Resposta | Array, paginação, objeto, relações aninhadas e nulabilidade. |
| IDs | Inteiro, UUID ou outro identificador. |
| Datas | Data simples ou data/hora; fuso horário quando previsto. |
| Exclusão | Resposta sem corpo, bloqueio, remoção de vínculos ou cascata. |
| Erros | Códigos, mensagens e erros por campo. |

O exemplo adota `/registro` e operações por ID. O caminho é ilustrativo: substitua pelas rotas reais. Nenhum contexto precisa criar uma entidade chamada Registro.

| Operação | Método e rota ilustrativos |
| --- | --- |
| Listar | GET `/registro` |
| Consultar | GET `/registro/{id}` |
| Cadastrar | POST `/registro` |
| Atualizar | PUT `/registro/{id}` |
| Excluir | DELETE `/registro/{id}` |

Preencha uma tabela equivalente com o contrato do grupo. Relacione cada requisito a uma tela, ação ou campo e à operação que o atende.

<a id="etapa-12"></a>

### 3.18.2. Configurar o endereço da API

O navegador chama `/api` no Next.js. Os rewrites encaminham a chamada ao Flask, conforme a [documentação do Next.js](https://nextjs.org/docs/app/api-reference/config/next-config-js/rewrites).

**`.env.local`** e **`.env.example`**:

```dotenv
API_URL=http://127.0.0.1:5000
```

Use o endereço real do backend. A variável é lida na configuração do Next.js e não precisa de prefixo `NEXT_PUBLIC_`.

**`next.config.ts`**:

```typescript
import type { NextConfig } from "next";

const apiUrl = (process.env.API_URL || "http://127.0.0.1:5000").replace(/\/$/, "");

const nextConfig: NextConfig = {
  async rewrites() {
    return [{ source: "/api/:path*", destination: apiUrl + "/:path*" }];
  },
};

export default nextConfig;
```

Assim, `/api/registro` no navegador chega a `/registro` no Flask. Se todas as rotas do backend usarem um prefixo, como `/api`, você pode incluí-lo em `API_URL` e definir os caminhos dos serviços a partir dele.

| Configuração | Chamada no navegador | Destino |
| --- | --- | --- |
| API_URL termina em `:5000`; serviço usa `/registro` | `/api/registro` | `:5000/registro` |
| API_URL termina em `:5000/api`; serviço usa `/registros` | `/api/registros` | `:5000/api/registros` |

Reinicie o Next.js após alterar a variável. Versione `.env.example` e mantenha `.env.local` fora do Git.

<a id="etapa-13"></a>

### 3.18.3. Criar o cliente HTTP e os serviços

Crie `src/lib/api.ts`. O cliente concentra conexão, códigos HTTP e respostas. O serviço reúne operações de uma entidade.

```typescript
import type { Registro, RegistroPayload, ServicoRegistro } from "./types";

async function request<T>(path: string, method = "GET", dados?: unknown): Promise<T> {
  let response: Response;
  try {
    response = await fetch("/api" + path, {
      method,
      headers: dados === undefined ? undefined : { "Content-Type": "application/json" },
      body: dados === undefined ? undefined : JSON.stringify(dados),
      cache: "no-store",
    });
  } catch {
    throw new Error("Não foi possível conectar à API. Confira o backend e tente novamente.");
  }

  const text = await response.text();
  let body: unknown = null;
  if (text) {
    try { body = JSON.parse(text); }
    catch {
      if (response.ok) throw new Error("A API retornou uma resposta fora do formato JSON esperado.");
    }
  }

  if (!response.ok) {
    const details = body && typeof body === "object"
      ? body as Record<string, unknown> : {};
    const message = typeof details.message === "string" ? details.message
      : typeof details.error === "string" ? details.error
      : "A operação foi recusada pela API (HTTP " + response.status + ").";
    const fields = details.errors ? " " + JSON.stringify(details.errors) : "";
    throw new Error(message + fields);
  }
  return body as T;
}

export function criarServicoRegistro(rota: string): ServicoRegistro {
  const path = rota.replace(/\/$/, "");
  return {
    listar: () => request<Registro[]>(path),
    buscar: (id) => request<Registro>(path + "/" + id),
    criar: (dados: RegistroPayload) => request<Registro>(path, "POST", dados),
    atualizar: (id, dados: RegistroPayload) => request<Registro>(path + "/" + id, "PUT", dados),
    excluir: async (id) => { await request<unknown>(path + "/" + id, "DELETE"); },
  };
}

// Substitua pela rota da coleção informada no Swagger do grupo.
export const registroService = criarServicoRegistro("/registro");
```

O DELETE pode retornar `204` sem corpo. O cliente não tenta analisar JSON quando a resposta está vazia. Adapte o verbo de atualização e o formato de resposta ao contrato.

Os tipos TypeScript não validam o JSON em tempo de execução. Confira o formato real ao integrar. Para várias entidades, separe tipos e serviços mantendo o cliente HTTP comum.

<a id="etapa-14"></a>

### 3.18.4. Conectar a página e adaptar as associações

Crie `src/app/page.tsx`:

```tsx
import { Aplicacao } from "@/components/oficina/aplicacao";

export default function Page() {
  return <Aplicacao />;
}
```

Crie `src/components/oficina/aplicacao.tsx`:

```tsx
"use client";

import { registroService } from "@/lib/api";
import { Catalogo } from "./catalogo";

export function Aplicacao() {
  return <Catalogo servico={registroService} />;
}
```

O catálogo e o serviço ficam juntos na entrada cliente. A página apenas renderiza essa entrada, evitando passar funções pela fronteira entre Server Components e Client Components.

##### Aplicar aos quatro contextos

Cada grupo percorre a mesma sequência:

1. Identificar entidades e operações na ficha.
2. Conferir schemas e endpoints no Swagger.
3. Ajustar tipos de resposta e payload.
4. Configurar campos, limites e opções.
5. Criar o serviço de cada entidade.
6. Definir abas, colunas e modais.
7. Conectar consultas, cadastros e demais ações.
8. Conferir requisitos e regras com os dados do contexto.

##### Associações

Carregue as entidades referenciadas antes de montar as opções. Use ID como valor e texto legível como rótulo.

```typescript
import type { Campo } from "@/components/oficina/formulario";

type OpcaoRelacionada = { id: number; nome: string };

export function campoRelacionamento(
  name: string,
  label: string,
  items: OpcaoRelacionada[],
): Campo {
  return {
    name, label, required: true,
    options: items.map((item) => ({
      id: String(item.id), label: item.nome,
    })),
  };
}
```

Acrescente o helper em `src/lib/campos.ts`, reutilizando o import de `Campo` existente. Passe o nome JSON da FK, o rótulo do contexto e os registros consultados. Ajuste `required` nas associações opcionais.

Para IDs inteiros positivos:

```typescript
export function idPositivo(value: string, label: string) {
  const id = Number(value);
  if (!Number.isInteger(id) || id <= 0) {
    throw new Error("Selecione uma opção válida para " + label + ".");
  }
  return id;
}
```

Inclua o helper no arquivo que monta o payload. IDs UUID permanecem strings. Campos opcionais podem ser enviados como `null` quando previsto pelo schema. Envie IDs quando a API espera IDs, em vez de objetos de associação.

Quando não houver opções, informe qual cadastro deve ser feito antes. Atualize as opções após novos cadastros. Consultas de vínculos e exclusões seguem as regras do backend.

<a id="etapa-15"></a>

### 3.18.5. Executar, conferir e limpar componentes extras

Inicie a API conforme o guia de backend do grupo. Na raiz do frontend:

```powershell
npm run dev
```

Abra `http://localhost:3000`. Se outra aplicação usar a porta, escolha outra com `npm run dev -- --port 3001`.

| Operação | Resultado a conferir |
| --- | --- |
| Cadastro | Dados enviados; resposta e nova listagem conferidas. |
| Consulta | Listagem e detalhes correspondem aos dados persistidos. |
| Edição | Dados atuais preenchidos; alteração aparece após salvar. |
| Exclusão | Confirmação e efeitos sobre vínculos seguem a API. |
| Validação | Obrigatórios, formatos, limites e regras apresentados. |
| Erros | Rede, conflito e rejeição mantêm mensagem e permitem corrigir. |
| Modais | Foco, Escape, botões e Selects funcionam. |
| Estados | Carregamento, lista vazia, sucesso e falha diferenciados. |

Use as verificações correspondentes às operações implementadas no contexto.

##### Identificar componentes extras

Depois de ligar a aplicação à página, crie o script abaixo. Ele percorre os imports alcançados pelas páginas para identificar arquivos extras de `src/components`.

**`scripts/limpar-componentes.cjs`**

```js
const fs = require("node:fs");
const path = require("node:path");
const ts = require("typescript");

const project = path.resolve(__dirname, "..");
const root = path.join(project, "src");
const components = path.join(root, "components");
const catalog = path.join(components, "oficina", "catalogo.tsx");
const apply = process.argv.includes("--aplicar");

if (
  !fs.existsSync(path.join(root, "app", "page.tsx")) ||
  !fs.existsSync(catalog)
) {
  throw new Error(
    "Execute na raiz do frontend, depois de criar a aplicação da oficina.",
  );
}

const configPath = ts.findConfigFile(
  project,
  ts.sys.fileExists,
  "tsconfig.json",
);

if (!configPath) {
  throw new Error("tsconfig.json não encontrado.");
}

const config = ts.readConfigFile(configPath, ts.sys.readFile);

if (config.error) {
  throw new Error("Não foi possível ler tsconfig.json.");
}

const parsedConfig = ts.parseJsonConfigFileContent(
  config.config,
  ts.sys,
  path.dirname(configPath),
);

if (parsedConfig.errors.length) {
  throw new Error("Corrija os erros do tsconfig.json antes da limpeza.");
}

function walk(dir) {
  return fs.readdirSync(dir, { withFileTypes: true }).flatMap((entry) => {
    if (entry.isSymbolicLink()) {
      throw new Error(`Link simbólico não permitido: ${entry.name}`);
    }

    const file = path.join(dir, entry.name);
    return entry.isDirectory() ? walk(file) : [file];
  });
}

const source = walk(root).filter((file) =>
  /\.[cm]?[jt]sx?$/.test(file),
);

const sourceSet = new Set(source);
const kept = new Set();
const unresolved = [];

const entries = source.filter((file) => {
  const relative = path.relative(root, file);
  const inApp = relative.split(path.sep)[0] === "app";

  return (
    /\.d\.[cm]?ts$/.test(file) ||
    (inApp &&
      /^(page|layout|not-found|error|global-error|loading|template|default|route|sitemap|robots|manifest|icon|apple-icon|opengraph-image|twitter-image)\.[jt]sx?$/.test(
        path.basename(file),
      ))
  );
});

function visit(file) {
  if (kept.has(file)) return;
  kept.add(file);

  const ast = ts.createSourceFile(
    file,
    fs.readFileSync(file, "utf8"),
    ts.ScriptTarget.Latest,
    true,
  );

  if (ast.parseDiagnostics.length) {
    throw new Error(
      `Limpeza cancelada: erro de sintaxe em ${path.relative(root, file)}.`,
    );
  }

  function add(specifier) {
    const result = ts.resolveModuleName(
      specifier,
      file,
      parsedConfig.options,
      ts.sys,
    ).resolvedModule;

    if (result) {
      const resolved = path.resolve(result.resolvedFileName);

      if (sourceSet.has(resolved)) visit(resolved);
      return;
    }

    const local =
      specifier.startsWith(".") ||
      specifier.startsWith("@/");

    if (!local) return;

    const base = specifier.startsWith("@/")
      ? path.join(root, specifier.slice(2))
      : path.resolve(path.dirname(file), specifier);

    // Estilos e assets são preservados; não são candidatos à remoção.
    if (fs.existsSync(base) && fs.statSync(base).isFile()) {
      return;
    }

    unresolved.push(
      `${path.relative(root, file)}: ${specifier}`,
    );
  }

  function scan(node) {
    if (
      (ts.isImportDeclaration(node) || ts.isExportDeclaration(node)) &&
      node.moduleSpecifier &&
      ts.isStringLiteral(node.moduleSpecifier)
    ) {
      add(node.moduleSpecifier.text);
    }

    if (
      ts.isImportEqualsDeclaration(node) &&
      ts.isExternalModuleReference(node.moduleReference) &&
      node.moduleReference.expression &&
      ts.isStringLiteral(node.moduleReference.expression)
    ) {
      add(node.moduleReference.expression.text);
    }

    if (
      ts.isImportTypeNode(node) &&
      ts.isLiteralTypeNode(node.argument) &&
      ts.isStringLiteral(node.argument.literal)
    ) {
      add(node.argument.literal.text);
    }
    if (
      ts.isCallExpression(node) &&
      (node.expression.kind === ts.SyntaxKind.ImportKeyword ||
        (ts.isIdentifier(node.expression) &&
          node.expression.text === "require"))
    ) {
      const argument = node.arguments[0];

      if (argument && ts.isStringLiteralLike(argument)) {
        add(argument.text);
      } else {
        throw new Error(
          `Limpeza cancelada: import dinâmico sem caminho literal em ${path.relative(root, file)}.`,
        );
      }
    }

    ts.forEachChild(node, scan);
  }

  scan(ast);
}

entries.forEach(visit);

if (unresolved.length) {
  throw new Error(
    `Limpeza cancelada: imports não resolvidos.\n${unresolved.join("\n")}`,
  );
}

if (!kept.has(catalog)) {
  throw new Error(
    "Limpeza cancelada: o catálogo da oficina ainda não está ligado às páginas da aplicação.",
  );
}

// A remoção fica limitada a src/components.
// Declarações de tipos globais são preservadas.
const unused = source.filter(
  (file) =>
    file.startsWith(components + path.sep) &&
    !/\.d\.[cm]?ts$/.test(file) &&
    !kept.has(file),
);

console.log(`Componentes sem uso: ${unused.length}.`);

for (const file of unused) {
  console.log(
    `${apply ? "Remover" : "Sem uso"}: ${path.relative(root, file)}`,
  );
}

if (apply) {
  for (const file of unused) {
    fs.unlinkSync(file);
  }

  console.log(
    "Limpeza concluída. Execute npx tsc --noEmit e npm run build.",
  );
} else {
  console.log(
    "Nenhum arquivo alterado. Confira a lista antes de executar com --aplicar.",
  );
}
```

Confira os tipos e simule:

```powershell
npx tsc --noEmit
node scripts/limpar-componentes.cjs
```

Revise a lista. Se os arquivos indicados estiverem sem uso:

```powershell
node scripts/limpar-componentes.cjs --aplicar
npx tsc --noEmit
npm run build
```

O script preserva dependências indiretas, estilos e arquivos fora de `src/components`. Imports locais não resolvidos cancelam a limpeza. Ele verifica `catalogo.tsx`; adapte essa verificação se renomear a entrada.

Pacotes npm não são removidos automaticamente. Confira imports, plugins CSS e configurações antes de desinstalar dependências. Preserve os providers, hooks e utilitários utilizados.

Se o Windows apresentar falha de subprocessos com Turbopack, a alternativa utilizada nas referências é configurar `next dev --webpack` e `next build --webpack` nos scripts.

<a id="etapa-16"></a>

### 3.18.6. Entender o fluxo e adicionar uma entidade

```text
Pessoa usuária
      ↓
Aba / tabela / formulário / modal
      ↓
Estado React e validação da interface
      ↓
Serviço da entidade
      ↓
Cliente HTTP
      ↓
Rewrite do Next.js
      ↓
API Flask
      ↓
Schemas / routes / services / models
      ↓
Banco de dados
```

Resposta:

```text
JSON ou resposta sem corpo
      ↓
Cliente HTTP
      ↓
Atualização dos estados e nova consulta
      ↓
Tabela / detalhes / mensagem
```

Para adicionar uma entidade:

1. Defina tipos de entrada e resposta.
2. Configure campos e conversão do payload.
3. Crie o serviço usando o cliente HTTP.
4. Configure colunas e ações permitidas.
5. Adicione a seção e os modais necessários.
6. Carregue opções de relacionamento.
7. Atualize listagens após as operações.
8. Relacione a implementação aos requisitos e confira as regras.

Os componentes comuns permanecem reutilizáveis. Contrato, campos e regras mudam conforme cada um dos quatro contextos.

<a id="comandos-necessarios"></a>

## 3.19. Comandos necessários

#### Projeto criado do zero

```powershell
npx create-next-app@latest frontend --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm
cd frontend
npm install react-aria-components @untitledui/icons tailwind-merge tailwindcss-animate tailwindcss-react-aria-components next-themes lucide-react @radix-ui/react-dialog
npm install -D tailwindcss @tailwindcss/postcss postcss
```

Crie `components.json` antes de adicionar os componentes:

```powershell
npx untitledui@latest add input button textarea select --path src/components
npx untitledui@latest add tabs table --path src/components
```

Preencha os estilos, providers e arquivos da aplicação conforme as etapas. Os comandos não substituem a criação desses arquivos.

#### Uso cotidiano, com o projeto configurado

Na raiz do frontend, com a API ativa:

```powershell
npm run dev
```

#### Conferir a aplicação

```powershell
npx tsc --noEmit
npm run build
```

<a id="responsabilidades"></a>

## 3.20. Resumo das responsabilidades

| Arquivo/Pasta | Responsabilidade |
| --- | --- |
| `components.json` | Aliases da CLI. |
| `postcss.config.mjs` | Processamento do CSS. |
| `.env.local` | Endereço local da API. |
| `next.config.ts` | Encaminhamento por rewrites. |
| `src/app/layout.tsx` | Layout, providers e estilos globais. |
| `src/app/page.tsx` | Entrada da página. |
| `src/styles/` | Tokens de cor, fonte, tamanhos e padrão visual. |
| `src/providers/` | Tema e navegação. |
| `src/hooks/` e `src/utils/` | Auxiliares dos componentes. |
| `src/lib/types.ts` | Tipos dos dados e operações. |
| `src/lib/campos.ts` | Campos, limites e opções. |
| `src/lib/api.ts` | Cliente HTTP e serviços. |
| `src/lib/dialog-portal.ts` | Contexto dos popovers. |
| `src/components/base/`, `application/` e `foundations/` | Componentes Untitled UI e auxiliares. |
| `src/components/ui/` | Base Radix e Dialog Box. |
| `src/components/oficina/formulario.tsx` | Valores, campos, envio e erros. |
| `src/components/oficina/modal.tsx` | Modal comum. |
| `src/components/oficina/excluir-registro.tsx` | Confirmação de exclusão. |
| `src/components/oficina/tabela.tsx` | Listagem configurável. |
| `src/components/oficina/catalogo.tsx` | Abas, consultas, ações e modais. |
| `src/components/oficina/aplicacao.tsx` | Ligação do catálogo ao serviço no cliente. |
| `scripts/limpar-componentes.cjs` | Identificação e limpeza opcional de extras. |

<a id="problemas-comuns"></a>

---

## Visão final: o que deve estar funcionando

Ao final da oficina, a pasta do projeto deverá reunir as três partes da aplicação:

```text
oficina-eventos/
├── pyproject.toml
├── uv.lock
├── compose.yaml
├── .env
├── .env.example
├── migrations/
├── src/
│   └── backend/        # Flask, models e API
└── frontend/           # Next.js, React e interface web
```

O objetivo não é produzir um sistema de produção completo, mas chegar a uma aplicação pequena e integrada: os dados são persistidos no PostgreSQL, as regras e operações são expostas pela API Flask e a interface web permite realizar o fluxo principal do contexto escolhido.

A mensagem central da oficina é que o domínio muda, mas muitas decisões de engenharia — organização dos dados, separação de responsabilidades, definição de contratos, tratamento de erros e integração entre camadas — podem ser reaproveitadas entre aplicações.

## Referências técnicas

- [uv — documentação](https://docs.astral.sh/uv/)
- [Flask — documentação](https://flask.palletsprojects.com/)
- [Flask-SQLAlchemy — documentação](https://flask-sqlalchemy.palletsprojects.com/)
- [Flask-Migrate — documentação](https://flask-migrate.readthedocs.io/)
- [Flask-Smorest — documentação](https://flask-smorest.readthedocs.io/)
- [Docker Compose — documentação](https://docs.docker.com/compose/)
- [Next.js — instalação](https://nextjs.org/docs/app/getting-started/installation)
- [Next.js — rewrites](https://nextjs.org/docs/app/api-reference/config/next-config-js/rewrites)
- [shadcn/ui — instalação para Next.js](https://ui.shadcn.com/docs/installation/next)
- [shadcn/ui — componentes](https://ui.shadcn.com/docs/components)